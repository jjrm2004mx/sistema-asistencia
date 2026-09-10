## Context

Ver `proposal.md` para la motivación completa. Este documento cubre el **cómo** técnico: stack, infraestructura y decisiones de arquitectura. Se construyó **de forma incremental** junto con la conversación de diseño técnico. Todas las decisiones fundacionales (hosting, tiempo real, backend, frontend, esquema de base de datos, incluyendo qué capabilities son toggleables por plantel) ya están cerradas; queda abierto solo lo que surja al implementar.

## Goals / Non-Goals

**Goals:**
- Documentar las decisiones técnicas de infraestructura y stack a medida que se cierran, con su razonamiento.

**Non-Goals:**
- Este documento no re-explica el dominio ni las reglas de negocio — eso vive en `proposal.md` y en `specs/`.

## Decisions

### Hosting: OCI (Oracle Cloud Infrastructure) Always Free

**Decisión**: backend y base de datos se despliegan en una VM Ampere A1 del tier Always Free de OCI (hasta 4 OCPU / 24GB RAM repartibles), con Postgres self-hosted vía Docker Compose en la misma VM.

**Alternativas consideradas**:
- **AWS Free Tier** (EC2 + RDS) — descartado: el free tier clásico es un trial de 12 meses, no permanente; después empieza a cobrar. Requiere ops equivalente a OCI sin la ventaja de ser gratis para siempre.
- **Vercel** — descartado para el backend: modelo serverless sin soporte para procesos de larga duración ni WebSockets persistentes, incompatible con Spring Boot y con el contador en vivo/roster en tiempo real que requiere el sistema. Vercel Postgres (free tier chico, pensado para pairearse con cómputo en Vercel) tampoco aplica si el backend vive en otro lado.
- **Railway / Render (PaaS)** — descartado como opción principal: cero ops (deploy vía git push, TLS y Postgres administrados), pero el free tier de Render duerme el servicio tras inactividad (cold start ~30-60s en la siguiente request), lo cual choca con el requisito explícito de "cero fricción para el profesor" del `proposal.md` — un salón esperando a que despierte el backend no es aceptable. Railway no duerme, pero su free tier es por créditos limitados, no gratis permanente.

**Razonamiento**: el patrón de uso real (ráfagas cortas de tráfico repartidas durante todo el día escolar, con el profesor necesitando que el QR se proyecte de inmediato al abrir sesión de clase) exige que el backend esté siempre activo. OCI Always Free es la única opción evaluada que es simultáneamente (a) gratis sin fecha de corte y (b) sin cold-start, a cambio de que el desarrollador (solo, cómodo con Spring Boot/Docker) asuma el poco de ops que implica administrar la VM y Postgres.

### Mecanismo de tiempo real: Server-Sent Events (SSE), no WebSocket

**Decisión**: el contador en vivo, el roster del profesor y la cuenta regresiva de cierre se implementan con SSE (`SseEmitter` en Spring Boot / `EventSource` en el navegador), no WebSocket.

**Alternativas consideradas**:
- **WebSocket** (típicamente STOMP + SockJS en Spring) — descartado: da comunicación bidireccional, pero todo el flujo de este sistema es unidireccional servidor→cliente (el profesor dispara acciones vía POST normal; lo único que viaja en tiempo real es la notificación del cambio de estado resultante). WebSocket agregaría protocolo y piezas (STOMP, negociación de conexión) sin un caso de uso que las requiera.

**Razonamiento**: ningún flujo del sistema necesita que el cliente empuje datos en tiempo real de vuelta por el mismo canal — confirmación de asistencia, marcado manual, cierre/apertura y justificación son todas acciones puntuales vía REST. SSE cubre el 100% de la necesidad (contador, roster, countdown) con menos código, reconexión automática nativa del navegador, y sin upgrade de protocolo (menor fricción con la red del plantel).

**Alcance de la difusión SSE — corregido**: la formulación original de esta decisión decía que la vista pública y el tablero privado se alimentan del "mismo stream" y que la diferencia es solo de presentación en el cliente. Eso es impreciso y, tal cual estaba escrito, inseguro: si el payload de un solo canal incluyera nombres de alumnos y el cliente público simplemente los ocultara al renderizar, cualquiera con las herramientas de desarrollador del navegador vería los nombres en el stream crudo — violaría directamente el requirement de `attendance-session` de que la vista pública "nunca expone nombres de alumnos". La decisión correcta es **dos endpoints SSE distintos**, alimentados por el mismo evento interno de dominio pero serializados distinto en el servidor:
- `GET /api/sesiones-clase/{id}/eventos/publico` — sin autenticación, el payload nunca incluye nombres de alumnos, solo el conteo agregado y el estado de la cuenta regresiva de cierre.
- `GET /api/sesiones-clase/{id}/eventos/profesor` — requiere sesión autenticada (ver "Mecanismo de sesión" abajo) del profesor de esa asignación o de dirección de ese plantel; el payload incluye el roster completo con nombres.

Ver `attendance-session` y `teacher-dashboard` specs para el detalle de qué expone cada vista.

### Cierre del registro: cuenta regresiva cancelable

**Decisión**: "cerrar registro" no es una acción instantánea — dispara una cuenta regresiva visible (duración configurable en el archivo de propiedades, mismo mecanismo que tolerancia/rotación de QR/intentos de PIN) durante la cual el registro sigue abierto. El profesor puede cancelarla en cualquier momento; si llega a cero, el registro se cierra automáticamente.

**Razonamiento**: evita dos fricciones opuestas — cerrar de golpe sin darle a un alumno la oportunidad de escanear a último momento, y que el profesor tenga que calcular manualmente cuánto esperar antes de cerrar. Al ser cancelable, mantiene la misma filosofía de reversibilidad que ya tiene el cierre/apertura del registro.

### Rol de super-administrador con UI propia (`super-admin-dashboard`)

**Decisión**: se revierte la decisión original de mantener todo el control cross-plantel (funcionalidades habilitadas, cupo de cuentas, alta de plantel, restablecimiento de acceso de dirección) como procedimiento manual en base de datos. Se construye `super-admin-dashboard`, un tercer rol de login (junto a profesor y dirección) sin plantel asociado, con UI real para: alta de plantel nuevo, funcionalidades habilitadas por plantel, cupo de cuentas por plantel, y restablecimiento de acceso de dirección bloqueada. La única acción que se mantiene manual (sin UI) es la **deshabilitación completa de un plantel** — caso poco frecuente (cancelación de un cliente) frente al resto de la operación.

Consecuencia directa: el bootstrap manual en base de datos deja de repetirse por cada plantel nuevo — se reduce a un evento único en la vida del sistema, la primera cuenta de super-administrador (ver `auth-accounts`). Todo plantel posterior se da de alta desde `super-admin-dashboard`, sin tocar la base de datos.

**Alternativas consideradas**:
- **Mantener todo manual (decisión original)** — revertida: aunque válida como punto de partida, una vez que el propio proyecto identificó suficientes operaciones recurrentes de este tipo (alta de plantel, cambio de funcionalidades, cupo, reset de acceso), construir la UI dejó de ser prematuro — son acciones que se repiten por cada plantel-cliente nuevo, no un caso aislado.
- **Login/acceso separado del de profesor/dirección** — descartado: se reutiliza el mismo formulario unificado y el mismo mecanismo de rol (ver `auth-accounts`, "Rol explícito por cuenta y enrutamiento post-login"), evitando construir una superficie de autenticación aparte para un solo rol adicional.

**Razonamiento**: dirección sigue sin poder autohabilitarse funcionalidades ni ampliarse su propio cupo — ese conflicto de interés no cambia — pero ahora existe una cuenta real (no solo "el desarrollador con acceso a la BD") con una interfaz para ejercer esa autoridad, lo cual es más sostenible en cuanto haya más de un plantel-cliente.

### Identificación del alumno: PIN en cada confirmación, sin reconocimiento de dispositivo

**Decisión**: cada confirmación de asistencia solicita identificador+PIN completos, sin importar si el dispositivo ya confirmó antes. No existe un mecanismo de "dispositivo reconocido" que recuerde al alumno entre escaneos.

**Alternativas consideradas**:
- **Reconocimiento ligero de dispositivo** (la versión original de este requirement) — descartado: asumía una relación 1 dispositivo : 1 alumno ("su tablet"), pero los dispositivos de los alumnos se comparten en la práctica (hermanos, tablet familiar, una prestada un día). Si el sistema recuerda un dispositivo como "ya identificado", un segundo alumno que lo use heredaría silenciosamente la identidad del primero sin que el sistema se lo pida — atribuyendo la asistencia a la persona equivocada.

**Razonamiento**: el PIN es una fricción baja (4 dígitos derivados de la fecha de nacimiento, ya memorizados) comparado con el riesgo de una mala atribución de asistencia por dispositivo compartido, que es un caso real y frecuente en el contexto de secundaria (no todos los alumnos tienen dispositivo propio). Pedirlo siempre es más simple de implementar y de razonar que diseñar un mecanismo de reconocimiento que distinga correctamente "mismo alumno, mismo dispositivo" de "dispositivo compartido, alumno distinto".

### Backend: Spring Boot (Java)

**Decisión**: el backend se construye en Spring Boot con Java — formaliza lo que ya se venía asumiendo implícitamente en decisiones anteriores (`SseEmitter` para SSE, "el desarrollador... cómodo con Spring Boot/Docker" en la decisión de hosting).

**Alternativas consideradas**:
- **Spring Boot (Kotlin)** — descartado: mismo ecosistema/infraestructura (SseEmitter, Spring Data JPA), pero sin necesidad de justificar el cambio de lenguaje sobre lo ya asumido.
- **Otro framework (Node/Express, Django, etc.)** — descartado: las decisiones de hosting y de tiempo real ya asumen JVM; reabrir el framework reabriría también esas decisiones sin una razón nueva que lo justifique.

**Razonamiento**: formaliza la decisión implícita en vez de dejarla flotando como una suposición no declarada — sin alternativas nuevas que cambien el cálculo ya hecho en las decisiones de hosting y SSE.

### Organización de paquetes: monolito modular por capability (vertical slices)

**Decisión**: los paquetes del backend se organizan por **capability** (uno por cada una de las 9 del `proposal.md`), con `Controller` → `Service` → `Repository`/`Entity` dentro de cada paquete — no por capa técnica horizontal (`controller/`, `service/`, `repository/` a nivel raíz agrupando todo el proyecto). Sigue siendo un solo artefacto desplegable (la modularidad es interna, no física) — coherente con la decisión de backend/hosting.

```
com.sistemaasistencia
├── common/                        # entidades usadas por 3+ capabilities
│   ├── Plantel.java, PlantelRepository.java, PlantelService.java
│   └── Cuenta.java, CuentaRepository.java, CuentaService.java   (login unificado)
│
├── academic_structure/            # CRUD-heavy, sin capa domain/
│   ├── Materia.java, Grupo.java, Alumno.java, Inscripcion.java, Asignacion.java
│   ├── AcademicStructureController.java
│   ├── AcademicStructureService.java
│   └── *Repository.java
│
├── attendance_session/            # la única con domain/ aislado de Spring/JPA
│   ├── domain/
│   │   ├── EstadoSesionClase.java     (transiciones ABIERTA→CERRANDO→CERRADA)
│   │   ├── TokenQr.java               (rotación/expiración)
│   │   └── ReglaTolerancia.java       (Presente vs Tardanza)
│   ├── SesionClase.java, RegistroAsistencia.java   (entidades JPA)
│   ├── SesionClaseController.java
│   ├── SesionClaseSseController.java  (endpoints /eventos/publico y /eventos/profesor)
│   ├── SesionClaseService.java
│   └── *Repository.java
│
├── attendance_confirmation/       # usa attendance_session.RegistroAsistenciaService
│   ├── AttendanceConfirmationController.java
│   ├── AttendanceConfirmationService.java
│   └── IdentificacionAlumnoService.java   (identificador+PIN, bloqueo por intentos)
│
├── attendance_justification/
│   ├── MotivoJustificacion.java
│   ├── AttendanceJustificationController.java
│   └── AttendanceJustificationService.java
│
├── teacher_dashboard/             # sin entidades propias, agrega datos de otras capabilities
├── admin_dashboard/               # ídem, a nivel plantel
├── student_parent_access/         # ídem, de solo lectura
├── super_admin_dashboard/         # ídem, cross-plantel
│
└── auth_accounts/
    ├── LoginController.java
    └── AuthService.java           (login/logout/sesión — Cuenta en sí vive en common/)
```

`teacher_dashboard`, `admin_dashboard`, `student_parent_access` y `super_admin_dashboard` no son dueños de entidades propias — son capas de agregación/consulta que orquestan servicios de las demás capabilities.

**Alternativas consideradas**:
- **Capas técnicas horizontales** (`controller/`, `service/`, `repository/`, `model/` a nivel raíz) — evaluada y descartada: es el patrón más familiar de tutoriales Spring Boot, pero con 9 capabilities cada carpeta técnica termina con 10-12 clases sin relación obvia entre sí — encontrar "todo lo de sesión de clase" implica mirar en 4 carpetas distintas en vez de una.
- **Hexagonal completo (puertos/adaptadores en todo el backend)** — descartado: el valor de hexagonal se paga cuando se espera cambiar infraestructura sin tocar lógica de negocio, o un equipo grande necesita fronteras estrictas — ninguno de los dos aplica aquí (infraestructura ya fijada, desarrollador único). La mayoría de las capabilities son CRUD-heavy; ponerles puertos/adaptadores es indirección sin beneficio real.

**Razonamiento**: por capability calca 1:1 la estructura de `specs/`, así que localizar código para un requirement es directo. Dentro de cada capability, Controller→Service→Repository es la capa de aplicación normal, no ceremonia extra. El aislamiento de dominio (`domain/`) se reserva para `attendance_session`, la única capability con lógica de estado real (máquina de estados de la sesión, rotación de QR, cálculo de tolerancia) que vale la pena proteger de anotaciones de Spring/JPA — el resto no lo necesita.

### Estructura de la API: REST + SSE por capability

**Decisión**: API REST convencional bajo `/api`, recursos en plural y kebab-case (independiente de que los paquetes Java usen snake_case), JSON como formato, autenticación por cookie de sesión (ver "Mecanismo de sesión" abajo, no header `Authorization`). Los dos endpoints SSE (público/privado) ya decididos arriba son la única excepción al patrón request/response normal.

**Endpoints principales por capability** (no exhaustivo — cubre los casos de uso ya spec'd, el resto se completa al implementar):

- **`auth_accounts`**: `POST /api/auth/login`, `POST /api/auth/logout`, `GET /api/auth/me` (cuenta actual: rol, plantel, nombre — para restaurar sesión al recargar el frontend).
- **`academic_structure`**: `GET|POST /api/materias`, `GET|POST /api/grupos`, `GET|POST /api/alumnos`, `POST /api/alumnos/import-csv`, `POST /api/alumnos/{id}/inscripciones` (inscribir a un grupo adicional), `DELETE /api/alumnos/{id}/inscripciones/{grupoId}` (retirar de un grupo), `GET|POST /api/asignaciones`, `GET|PUT /api/plantel/configuracion` (tipo de identificador, bloqueado tras el primer alumno), `PATCH /api/alumnos/{id}/desbloquear`.
- **`attendance_session`**: `POST /api/sesiones-clase` (iniciar), `GET /api/sesiones-clase/{id}`, `POST /api/sesiones-clase/{id}/cerrar`, `POST /api/sesiones-clase/{id}/cancelar-cierre`, `POST /api/sesiones-clase/{id}/reabrir`, `PATCH /api/sesiones-clase/{id}/registros/{alumnoId}` (marcado manual), `GET /api/sesiones-clase/{id}/eventos/publico` (SSE), `GET /api/sesiones-clase/{id}/eventos/profesor` (SSE, autenticado).
- **`attendance_confirmation`**: `POST /api/confirmaciones/verificar` (identificador+PIN+token QR — valida y devuelve el estado actual del alumno en la sesión si ya tenía uno, más un `tokenVerificacion` de un solo uso; endpoint público, sin sesión, sujeto al bloqueo por intentos fallidos), `POST /api/confirmaciones/confirmar` (con el `tokenVerificacion`, registra la asistencia y devuelve el estado resultante).
- **`attendance_justification`**: `POST /api/registros-asistencia/{id}/justificar`, `GET /api/motivos-justificacion`.
- **`teacher_dashboard`**: `GET /api/tablero-profesor/sesiones?fecha=`, `GET /api/tablero-profesor/resumenes?mes=`, `GET /api/sesiones-clase/{id}/resumen-cierre` (conteo de confirmados + lista nominal de alumnos sin registro; vive en `attendance_session` pero atiende este requirement).
- **`admin_dashboard`**: `GET /api/tablero-direccion/sesiones?fecha=` (todas las sesiones del plantel, agregado), `GET /api/tablero-direccion/resumenes?mes=`, `PATCH /api/motivos-justificacion/{id}/aprobar`, `DELETE /api/motivos-justificacion/{id}`, `POST /api/cuentas/{id}/reset-password`.
- **`student_parent_access`**: `POST /api/consulta-historial` (identificador+PIN, abre la sesión corta), `GET /api/consulta-historial/mes-actual`, `GET /api/consulta-historial/sesion-actual` (dentro de la sesión corta ya abierta — informa si hay una sesión de clase activa ahora mismo para alguna asignación del alumno, sin exponer token de QR ni permitir confirmar desde aquí).
- **`super_admin_dashboard`**: `GET /api/planteles` (para elegir cuál administrar), `POST /api/planteles` (alta de plantel nuevo), `PATCH /api/planteles/{id}/funcionalidades`, `PATCH /api/planteles/{id}/cupo-cuentas`, `POST /api/planteles/{id}/restablecer-acceso-direccion`, `POST /api/cuentas` (alta de otra cuenta super-admin). Deshabilitar un plantel por completo sigue sin endpoint — se mantiene manual en base de datos.

**Alternativas consideradas**:
- **GraphQL** — descartado: da flexibilidad de consulta que ningún requirement pide (los tableros tienen resúmenes fijos, no consultas ad-hoc del cliente), y agrega una pieza más (resolvers, schema) sin necesidad — mismo criterio que descartó WebSocket y Nginx aparte en decisiones previas.

**Razonamiento**: REST + los dos SSE ya decididos cubren el 100% de los flujos spec'd sin agregar protocolo nuevo. Los nombres de recurso siguen el vocabulario de los specs (`sesiones-clase`, no `sessions`) para que la API se lea igual que los requirements que implementa.

### Mecanismo de sesión: HttpSession stateful para login tradicional, sesión corta para alumno/padre

**Decisión**:
- **Profesor/dirección/super-administrador**: sesión stateful vía `HttpSession` de Spring Security (cookie `HttpOnly` + `Secure`), en memoria en la propia instancia — dado que ya decidimos una sola VM / una sola instancia, no hace falta un store distribuido (Redis) para compartir sesión entre réplicas que no existen. El logout invalida la sesión en el servidor de inmediato, cumpliendo el requirement ya escrito en `auth-accounts` ("revoca el acceso... hasta que vuelva a autenticarse") sin necesidad de una blocklist de tokens.
- **Alumno/padre**: al identificarse (identificador+PIN) en `student-parent-access`, el sistema abre una sesión corta y de bajo privilegio para navegar su historial del mes sin repetir el PIN en cada clic. Esta sesión corta **no aplica a `attendance-confirmation`** — ahí ya decidimos pedir identificador+PIN completos en cada confirmación de asistencia, sin excepción (ver "Identificación del alumno: PIN en cada confirmación, sin reconocimiento de dispositivo" arriba); una sesión de conveniencia ahí reabriría el mismo riesgo de dispositivo compartido que esa decisión evitó.

**Alternativas consideradas**:
- **JWT sin estado** — no descartado de forma permanente, dejado como puerta abierta: si el sistema eventualmente necesita más de una instancia del backend (más allá del alcance actual de una sola VM), JWT evita depender de sesión en memoria de una sola instancia. Por ahora, con una sola instancia, "logout inmediato" con JWT normalmente requeriría igual una blocklist de tokens revocados del lado servidor — reintroduce el mismo estado que se supone que JWT evita, sin ganar nada a cambio en el escenario actual.

**Razonamiento**: la decisión de sesión sigue la misma lógica que las demás decisiones de esta fase — resolver con lo más simple que satisface los requirements ya escritos, dado el contexto ya fijado (una VM, un operador), dejando explícito el trigger para reconsiderar (pasar a más de una instancia) en vez de sobre-diseñar para una escala que no existe todavía.

### Portal de entrada del alumno: orienta hacia el QR, no lo sustituye

**Decisión**: la identificación de `student-parent-access` (identificador+PIN, sesión corta) funciona como punto de entrada único del alumno. Al identificarse, si existe una sesión de clase activa en ese momento para alguna de sus asignaciones, el sistema le muestra un aviso informativo (materia y grupo) invitándolo a escanear el QR proyectado — junto con el acceso a su historial. Si no hay ninguna sesión activa, solo se muestra el acceso al historial. El aviso es puramente informativo: no expone ningún mecanismo para registrar asistencia directamente desde ahí. No requiere cambios al esquema de base de datos — es una lectura sobre `sesion_clase`/`asignacion`/`inscripcion`, entidades ya existentes.

**Alternativas consideradas**:
- **Permitir confirmar asistencia directamente desde el portal, sin escanear el QR** — descartado explícitamente: anularía la decisión de "Mecanismo de tiempo real" que hace del QR proyectado con rotación la única defensa contra suplantación (ver también "Identificación del alumno: PIN en cada confirmación, sin reconocimiento de dispositivo"). El QR obliga a que el alumno esté físicamente frente al proyector en ese momento; un botón de confirmación en un portal accesible desde cualquier dispositivo no da esa garantía y reabriría el riesgo de asistencia falsa sin necesidad de estar en el salón.

**Razonamiento**: el alumno hoy no tiene ninguna señal de "es tu hora de confirmar" fuera de ver el proyector — este aviso mejora el descubrimiento del flujo real (que sigue siendo 100% vía QR) sin tocar el modelo de amenaza ya cerrado. Reutiliza la sesión corta que `student-parent-access` ya abre, sin introducir un mecanismo de autenticación nuevo.

### Frontend: SPA con React, servida como recurso estático del propio backend

**Decisión**: el frontend es una SPA en React, que consume la API REST y los streams SSE del backend. El build de producción se sirve como recursos estáticos del propio Spring Boot (un solo artefacto desplegable, un solo contenedor en el Docker Compose de la VM) — no un servidor Nginx ni un deploy separado para el frontend.

**Alternativas consideradas**:
- **Server-rendered (Thymeleaf)** — descartado: un solo deploy y menos piezas para el desarrollador solo, pero los tableros son la parte más compleja de UI del sistema (calendario mes/semana/día, roster en vivo actualizado por SSE, hasta 7 resúmenes recalculados, marcado manual con confirmaciones) — ese nivel de interactividad se maneja mejor con un framework de UI con estado que con vistas renderizadas en servidor.
- **Frontend servido aparte (Nginx/CDN + build propio)** — descartado: agrega una pieza más (proxy, CORS entre orígenes, un segundo pipeline de deploy) sin necesidad real dado que backend y frontend viven en la misma VM — mismo criterio que descartó WebSocket sobre SSE (menos protocolo/piezas sin un caso de uso que las requiera).

**Razonamiento**: React da mejor manejo de estado interactivo en tiempo real para los tableros, que es donde vive la complejidad real de UI del sistema. Servirlo como estático desde el propio Spring Boot mantiene la filosofía de "una sola VM, un solo operador, mínima superficie operativa" ya establecida en la decisión de hosting.

### Sistema de diseño visual: azul institucional, registro tabular, sin patrones de "IA genérica"

**Decisión**: la identidad visual del frontend usa una paleta de azul institucional (`#2A5C99` como acento primario sobre fondo blanco/gris-azulado, con un verde apagado como acento secundario para métricas positivas), tipografía `Newsreader` (display, serif editorial) + `Work Sans` (cuerpo) + `JetBrains Mono` (cifras/datos tabulares), y datos operativos presentados como **registro tabular** (filas, no tarjetas kanban) cuando el contenido es naturalmente una lista — encaja con el roster de alumnos, el calendario de sesiones y los resúmenes de los tableros de `teacher-dashboard`/`admin-dashboard`.

**Cómo se llegó a esto**: se evaluaron dos prototipos con contenido de referencia (no del dominio real, solo para comparar dirección visual) — uno con paleta bóveda/latón, otro con esta paleta azul institucional — contra dos demos públicos de un skill de generación de sistemas de diseño de terceros (`ui-ux-pro-max`, evaluado y descartado como dependencia, ver más abajo). Ambos demos de terceros salieron genéricos en la revisión (héroe centrado, badges "AI-Powered", emojis como marcador de sección, integraciones estilo Zapier calcado) — la paleta azul institucional en sí no era el problema; la ejecución sí. Se optó por esta dirección, ejecutada a mano, sin la dependencia de terceros.

**Alternativas consideradas**:
- **Skill de terceros `ui-ux-pro-max`** (github.com/nextlevelbuilder/ui-ux-pro-max-skill) para generar el sistema de diseño — descartado: MIT, procesamiento local sin llamadas de red según su documentación, pero sus propios demos públicos (`sales-crm-platform`, `pet-grooming`) se evaluaron como genéricos ("cookie-cutter", sin identidad visual propia) — no da garantía de evitar el look de "hecho con IA" que se buscaba evitar, que era el motivo original para considerarlo.

**Razonamiento**: el registro tabular no es solo una preferencia estética — encaja con el propio dominio (asistencia por sesión, listas de alumnos, resúmenes por periodo son inherentemente tabulares) mejor que un patrón de tarjetas prestado de un CRM. Falta adaptar el contenido de los dos prototipos (que usaban un dominio de CRM de ventas como referencia neutral) al dominio real del sistema al implementar las pantallas.

## Esquema de base de datos (entidades y relaciones)

Nivel arquitectura — entidades, campos clave y relaciones. Tipos de columna, constraints e índices exactos quedan para las migrations al implementar.

### Cuenta unificada para login tradicional, en vez de tablas separadas por rol

**Decisión**: una sola tabla `cuenta` para los tres roles de login tradicional (profesor, dirección, super-administrador), con un campo `rol` discriminador, en vez de tablas `Profesor`/`Direccion`/`SuperAdministrador` separadas.

**Alternativas consideradas**:
- **Tabla separada por rol** — descartado: `auth-accounts` ya unificó login, logout, rol y enrutamiento post-login como un solo mecanismo (ver "Rol explícito por cuenta y enrutamiento post-login") — separar en tablas distintas duplicaría esa lógica en la capa de datos sin necesidad, ya que los tres roles comparten correo+contraseña, alta con contraseña temporal, y pertenencia opcional a un plantel.

**Razonamiento**: `tolerancia_minutos` es el único campo que aplica exclusivamente a rol=PROFESOR (queda NULL para dirección y super-administrador); todo lo demás (correo, password_hash, rol, plantel_id) es común a los tres. Una tabla con un campo específico NULL para dos de tres roles es más simple que tres tablas con lógica de auth repetida.

### Funcionalidades toggleables por plantel: solo 2 de las 9 capabilities

**Decisión**: de las 9 capabilities, solo `attendance-justification` y `student-parent-access` son toggleables por plantel vía `plantel_funcionalidad` (ver `super-admin-dashboard`, "Gestión de funcionalidades habilitadas por plantel"). Las otras 6 — `auth-accounts`, `academic-structure`, `attendance-session`, `attendance-confirmation`, `teacher-dashboard`, `admin-dashboard` — SHALL estar siempre activas, sin representarse en `plantel_funcionalidad`. `super-admin-dashboard` no aplica al concepto: no es una capability "por plantel", es cross-plantel y no depende de ningún plantel específico.

**Criterio usado**: una capability es candidata a toggleable solo si deshabilitarla no deja a ningún rol sin pantalla de destino tras el login. `teacher-dashboard` y `admin-dashboard` son el destino de enrutamiento post-login de profesor y dirección (ver `auth-accounts`, "Rol explícito por cuenta y enrutamiento post-login") — y toda alta de plantel nuevo crea automáticamente una cuenta de dirección (ver `super-admin-dashboard`, "Alta de plantel nuevo"), así que deshabilitar `admin-dashboard` dejaría a esa cuenta sin ningún lugar a donde ir tras autenticarse; lo mismo aplica a `teacher-dashboard` en cuanto exista al menos un profesor. `attendance-session` y `attendance-confirmation` son el mecanismo central de toma de asistencia — deshabilitarlos no es "quitar una funcionalidad", es dejar de usar el sistema. `attendance-justification` y `student-parent-access` sí son seguras: son capas de valor agregado sobre el núcleo, y si se deshabilitan ningún rol pierde su pantalla de entrada — el profesor simplemente deja de ver el botón "Justificar" y el alumno/padre deja de tener el portal, sin dejar ninguna cuenta varada.

**Alternativas consideradas**:
- **Las 9 capabilities toggleables** (interpretación original, ambigua) — descartada: deshabilitar `admin-dashboard` o `teacher-dashboard` para un plantel dejaría cuentas ya creadas (dirección o profesor) sin ruta de destino tras el login, un estado inconsistente que el sistema no maneja en ningún lado.
- **Usar la deshabilitación completa de un plantel para el caso "dirección sin tablero"** — ya existe como mecanismo separado (`super-admin-dashboard`, "Deshabilitación completa de un plantel"); no hace falta duplicar ese caso de uso a nivel de capability individual.

**Razonamiento**: el mecanismo de toggle está pensado principalmente para funcionalidades **futuras** que se agreguen al sistema más adelante — con las 9 capabilities actuales, el margen real de "apagar sin romper nada" resultó ser angosto (solo 2), pero la tabla puente `plantel_funcionalidad` queda lista para crecer cuando aparezcan nuevas capabilities de valor agregado sin necesidad de rediseñar el esquema.

### Entidades

- **`plantel`**: id, nombre, `tipo_identificador_alumno` (MATRICULA | CORREO_INSTITUCIONAL, bloqueado tras el primer alumno — ver `academic-structure`), `cupo_profesores`, `cupo_direccion`, `activo` (deshabilitación completa, ver `super-admin-dashboard`).
- **`plantel_funcionalidad`**: `plantel_id` (FK), `capability` (enum: `JUSTIFICACION` | `CONSULTA_HISTORIAL` — las únicas 2 toggleables, ver decisión arriba), `habilitada` (bool) — tabla puente N:N para "Funcionalidades habilitadas por plantel". Las demás 7 capabilities no tienen fila aquí: siempre están activas y no pasan por esta tabla.
- **`cuenta`**: id, `correo` (único por `plantel_id`; sin restricción de unicidad cruzada para super-administrador, que no tiene plantel), `password_hash`, `rol` (PROFESOR | DIRECCION | SUPER_ADMINISTRADOR), `plantel_id` (FK, NULL si rol=SUPER_ADMINISTRADOR), `tolerancia_minutos` (solo aplica si rol=PROFESOR; default 5, configurable — ver `attendance-session`).
- **`materia`**: id, `plantel_id` (FK), nombre.
- **`grupo`**: id, `plantel_id` (FK), nombre.
- **`alumno`**: id, `plantel_id` (FK), `identificador` (matrícula o correo institucional según `plantel.tipo_identificador_alumno`; único por plantel), `pin` (fecha de nacimiento, ddmmaaaa), `intentos_fallidos_pin`, `bloqueado` (bool — compartido entre `attendance-confirmation` y `student-parent-access`, ver esa decisión arriba), nombre.
- **`inscripcion`**: `alumno_id` (FK), `grupo_id` (FK) — N:N alumno↔grupo.
- **`asignacion`**: id, `cuenta_id` (FK, rol=PROFESOR), `materia_id` (FK), `grupo_id` (FK), `horario_dia`, `horario_hora` (informativo, no restrictivo).
- **`sesion_clase`**: id, `asignacion_id` (FK), fecha, `hora_inicio`, `estado` (ABIERTA | CERRANDO | CERRADA), `token_qr_actual`, `token_qr_expira_en`, `cierre_countdown_fin` (nullable — cuenta regresiva cancelable).
- **`registro_asistencia`**: id, `sesion_clase_id` (FK), `alumno_id` (FK), `estado` (PRESENTE | TARDANZA | FALTA | FALTA_JUSTIFICADA), `origen` (QR | MANUAL), `hora_registro`, `motivo_justificacion_id` (FK nullable), `evidencia_url` (nullable), `registrado_por_cuenta_id` (FK nullable a `cuenta` — quién hizo el marcado manual o la justificación, para trazabilidad).
- **`motivo_justificacion`**: id, `plantel_id` (FK), texto, `estado` (PENDIENTE_APROBAR | APROBADO).

No hay entidad de "dispositivo" — se descartó el reconocimiento de dispositivo (ver decisión arriba). No hay entidad de "padre/tutor" — no es una identidad técnica separada del alumno (ver `student-parent-access`).

### Relaciones clave
- `plantel` 1:N `cuenta`, `materia`, `grupo`, `alumno`, `motivo_justificacion`.
- `alumno` N:N `grupo` vía `inscripcion`.
- `cuenta` (rol=PROFESOR) + `materia` + `grupo` → `asignacion` (relación ternaria).
- `asignacion` 1:N `sesion_clase` 1:N `registro_asistencia`.
- `alumno` 1:N `registro_asistencia`; `motivo_justificacion` 1:N `registro_asistencia`.

## Risks / Trade-offs

- **[Riesgo] Provisionar la VM Ampere A1 gratis en OCI puede fallar por "out of capacity" en la región** al momento de crearla (fricción reportada por la comunidad) → **Mitigación**: reintentar la creación de la instancia y/o probar otra availability domain/región si la primera falla; no es un bloqueo definitivo, solo requiere paciencia en el setup inicial.
- **[Riesgo] Postgres self-hosted implica que el desarrollador es responsable de backups, updates de versión y parches de seguridad del SO** (sin un proveedor administrando esto) → **Mitigación**: automatizar backups (cron + `pg_dump` a un storage externo) y mantener el Docker Compose simple y documentado como parte del setup del proyecto.
- **[Riesgo] Una sola VM es un punto único de falla** (sin redundancia/alta disponibilidad) → **Mitigación aceptada por ahora**: consistente con el alcance de "un solo plantel" en esta fase; revisitar si el proyecto madura a SaaS multi-plantel con más de un cliente real dependiendo de uptime.
