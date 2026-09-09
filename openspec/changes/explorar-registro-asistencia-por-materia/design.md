## Context

Ver `proposal.md` para la motivación completa. Este documento cubre el **cómo** técnico: stack, infraestructura y decisiones de arquitectura. Se construyó **de forma incremental** junto con la conversación de diseño técnico. Las decisiones fundacionales (hosting, tiempo real, backend, frontend, esquema de base de datos) ya están cerradas; queda abierto el punto marcado como pendiente en "Esquema de base de datos" (qué capabilities son toggleables por plantel) y lo que surja al implementar.

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

**Alcance de la difusión SSE**: el mismo stream de eventos de una sesión de clase alimenta tanto la vista pública (proyector) como el tablero privado del profesor — la diferencia entre ambas vistas es de **presentación** (qué campos se renderizan), no de canal. Ver `attendance-session` y `teacher-dashboard` specs para el detalle de qué expone cada vista (ej. la vista pública nunca expone nombres de alumnos, ver spec).

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

### Entidades

- **`plantel`**: id, nombre, `tipo_identificador_alumno` (MATRICULA | CORREO_INSTITUCIONAL, bloqueado tras el primer alumno — ver `academic-structure`), `cupo_profesores`, `cupo_direccion`, `activo` (deshabilitación completa, ver `super-admin-dashboard`).
- **`plantel_funcionalidad`**: `plantel_id` (FK), `capability` (enum de las capabilities toggleables), `habilitada` (bool) — tabla puente N:N para "Funcionalidades habilitadas por plantel". *Pendiente de decidir cuáles de las 9 capabilities son toggleables vs. siempre activas (`academic-structure` y `auth-accounts` son candidatas obvias a "siempre activas", son la base del sistema) — no está definido en los specs todavía.*
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
