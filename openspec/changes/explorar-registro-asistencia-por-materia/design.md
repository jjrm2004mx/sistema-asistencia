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

**Razonamiento**: el patrón de uso real (ráfagas cortas de tráfico repartidas durante todo el día escolar, con el profesor necesitando que el QR se proyecte de inmediato al abrir sesión) exige que el backend esté siempre activo. OCI Always Free es la única opción evaluada que es simultáneamente (a) gratis sin fecha de corte y (b) sin cold-start, a cambio de que el desarrollador (solo, cómodo con Spring Boot/Docker) asuma el poco de ops que implica administrar la VM y Postgres.

### Mecanismo de tiempo real: Server-Sent Events (SSE), no WebSocket

**Decisión**: el contador en vivo, el roster del profesor y la cuenta regresiva de cierre se implementan con SSE (`SseEmitter` en Spring Boot / `EventSource` en el navegador), no WebSocket.

**Alternativas consideradas**:
- **WebSocket** (típicamente STOMP + SockJS en Spring) — descartado: da comunicación bidireccional, pero todo el flujo de este sistema es unidireccional servidor→cliente (el profesor dispara acciones vía POST normal; lo único que viaja en tiempo real es la notificación del cambio de estado resultante). WebSocket agregaría protocolo y piezas (STOMP, negociación de conexión) sin un caso de uso que las requiera.

**Razonamiento**: ningún flujo del sistema necesita que el cliente empuje datos en tiempo real de vuelta por el mismo canal — confirmación de asistencia, marcado manual, cierre/apertura y justificación son todas acciones puntuales vía REST. SSE cubre el 100% de la necesidad (contador, roster, countdown) con menos código, reconexión automática nativa del navegador, y sin upgrade de protocolo (menor fricción con la red del plantel).

**Alcance de la difusión SSE**: el mismo stream de eventos de una sesión alimenta tanto la vista pública (proyector) como el tablero privado del profesor — la diferencia entre ambas vistas es de **presentación** (qué campos se renderizan), no de canal. Ver `attendance-session` y `teacher-dashboard` specs para el detalle de qué expone cada vista (ej. la vista pública nunca expone nombres de alumnos, ver spec).

### Cierre del registro: cuenta regresiva cancelable

**Decisión**: "cerrar registro" no es una acción instantánea — dispara una cuenta regresiva visible (duración configurable en el archivo de propiedades, mismo mecanismo que tolerancia/rotación de QR/intentos de PIN) durante la cual el registro sigue abierto. El profesor puede cancelarla en cualquier momento; si llega a cero, el registro se cierra automáticamente.

**Razonamiento**: evita dos fricciones opuestas — cerrar de golpe sin darle a un alumno la oportunidad de escanear a último momento, y que el profesor tenga que calcular manualmente cuánto esperar antes de cerrar. Al ser cancelable, mantiene la misma filosofía de reversibilidad que ya tiene el cierre/apertura del registro.

## Risks / Trade-offs

- **[Riesgo] Provisionar la VM Ampere A1 gratis en OCI puede fallar por "out of capacity" en la región** al momento de crearla (fricción reportada por la comunidad) → **Mitigación**: reintentar la creación de la instancia y/o probar otra availability domain/región si la primera falla; no es un bloqueo definitivo, solo requiere paciencia en el setup inicial.
- **[Riesgo] Postgres self-hosted implica que el desarrollador es responsable de backups, updates de versión y parches de seguridad del SO** (sin un proveedor administrando esto) → **Mitigación**: automatizar backups (cron + `pg_dump` a un storage externo) y mantener el Docker Compose simple y documentado como parte del setup del proyecto.
- **[Riesgo] Una sola VM es un punto único de falla** (sin redundancia/alta disponibilidad) → **Mitigación aceptada por ahora**: consistente con el alcance de "un solo plantel" en esta fase; revisitar si el proyecto madura a SaaS multi-plantel con más de un cliente real dependiendo de uptime.
