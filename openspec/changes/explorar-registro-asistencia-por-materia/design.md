## Context

Ver `proposal.md` para la motivación completa. Este documento cubre el **cómo** técnico: stack, infraestructura y decisiones de arquitectura. Se está construyendo **de forma incremental** junto con la conversación de diseño técnico — no todas las secciones están completas todavía; se amplían a medida que se cierran decisiones (backend, frontend, DB, mecanismo de tiempo real, etc.).

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

### Funcionalidades habilitadas y cupo de cuentas por plantel: control manual, sin UI de super-admin todavía

**Decisión**: cada plantel tiene (a) un conjunto de capabilities habilitadas de forma independiente entre sí (no un plan cerrado tipo Básico/Completo), y (b) un cupo máximo de cuentas de profesor y de dirección, independientes entre sí — dado que el modelo de negocio SaaS cobra por licencia/cuenta, no tendría sentido que dirección pudiera dar de alta cuentas sin límite. Ambos se cambian vía configuración/base de datos directamente por el operador del sistema — no existe (todavía) un rol de super-administrador cross-plantel con interfaz propia para esto. Dirección puede ver qué tiene habilitado y su cupo/consumo de cuentas, pero no puede autohabilitarse funcionalidades ni ampliarse el cupo ella misma.

**Alternativas consideradas**:
- **Plan cerrado (enum Básico/Completo) en vez de toggles independientes** — descartado: agrupa capabilities en paquetes fijos, más simple de modelar y de vender, pero menos flexible para paquetes a la medida por plantel a futuro.
- **Construir ya una UI de super-administrador cross-plantel** — descartado por ahora: sin múltiples planteles-cliente reales operando en paralelo, es costo hundido especulativo sin caso de uso real que valide el diseño de esa UI (¿solo toggles de features? ¿también facturación? ¿métricas agregadas?).

**Razonamiento**: dirección administra su propio plantel, pero no puede ser quien se autohabilite funcionalidades ni se autoasigne cuentas ilimitadas si ambas están atadas a un plan/pricing — es un conflicto de interés obvio. Esa autoridad debe vivir por encima del nivel de plantel. Sin embargo, mientras haya un solo operador (el desarrollador) y pocos planteles, un cambio manual vía DB/config es suficiente y evita construir una herramienta antes de tener un caso de uso real que la valide — el mismo patrón que el bootstrap manual de la primera cuenta de dirección (ver `auth-accounts`).

**Trigger para revisitar**: cuando el cambio manual por DB se vuelva una fricción operativa frecuente (más de un plantel-cliente real pidiendo cambios de funcionalidades con regularidad), construir la UI de super-administrador cross-plantel deja de ser prematuro.

### Identificación del alumno: PIN en cada confirmación, sin reconocimiento de dispositivo

**Decisión**: cada confirmación de asistencia solicita identificador+PIN completos, sin importar si el dispositivo ya confirmó antes. No existe un mecanismo de "dispositivo reconocido" que recuerde al alumno entre escaneos.

**Alternativas consideradas**:
- **Reconocimiento ligero de dispositivo** (la versión original de este requirement) — descartado: asumía una relación 1 dispositivo : 1 alumno ("su tablet"), pero los dispositivos de los alumnos se comparten en la práctica (hermanos, tablet familiar, una prestada un día). Si el sistema recuerda un dispositivo como "ya identificado", un segundo alumno que lo use heredaría silenciosamente la identidad del primero sin que el sistema se lo pida — atribuyendo la asistencia a la persona equivocada.

**Razonamiento**: el PIN es una fricción baja (4 dígitos derivados de la fecha de nacimiento, ya memorizados) comparado con el riesgo de una mala atribución de asistencia por dispositivo compartido, que es un caso real y frecuente en el contexto de secundaria (no todos los alumnos tienen dispositivo propio). Pedirlo siempre es más simple de implementar y de razonar que diseñar un mecanismo de reconocimiento que distinga correctamente "mismo alumno, mismo dispositivo" de "dispositivo compartido, alumno distinto".

## Risks / Trade-offs

- **[Riesgo] Provisionar la VM Ampere A1 gratis en OCI puede fallar por "out of capacity" en la región** al momento de crearla (fricción reportada por la comunidad) → **Mitigación**: reintentar la creación de la instancia y/o probar otra availability domain/región si la primera falla; no es un bloqueo definitivo, solo requiere paciencia en el setup inicial.
- **[Riesgo] Postgres self-hosted implica que el desarrollador es responsable de backups, updates de versión y parches de seguridad del SO** (sin un proveedor administrando esto) → **Mitigación**: automatizar backups (cron + `pg_dump` a un storage externo) y mantener el Docker Compose simple y documentado como parte del setup del proyecto.
- **[Riesgo] Una sola VM es un punto único de falla** (sin redundancia/alta disponibilidad) → **Mitigación aceptada por ahora**: consistente con el alcance de "un solo plantel" en esta fase; revisitar si el proyecto madura a SaaS multi-plantel con más de un cliente real dependiendo de uptime.
