## Why

Los planteles de secundaria hoy toman asistencia de forma manual y una sola vez al día, sin capturar variaciones por materia/periodo, sin trazabilidad de tardanzas ni justificantes, y sin visibilidad en tiempo real para profesores, dirección, alumnos o padres. Este cambio introduce un sistema de registro de asistencia digital, por materia, basado en QR proyectado — diseñado explícitamente para ser una herramienta simple y de bajo esfuerzo operativo para el profesor (esto es solo toma de asistencia, no debe generarle fricción), con un modelo de dominio que puede crecer a SaaS multi-plantel si el proyecto madura.

## What Changes

- Introduce el modelo de dominio SaaS multi-plantel: `Plantel` > N `Profesores` / N `Materias` (N:N) / N `Grupos` (N:N con `Alumnos` vía inscripción), con `Asignación` (Profesor + Materia + Grupo) y horario informativo (no restrictivo).
- Introduce el mecanismo de toma de asistencia por materia/sesión de clase vía **QR proyectado y rotativo** (intervalo configurable, referencia 30-45s), con **contador en vivo** visible tanto en la vista pública (proyector) como en el tablero del profesor.
- Introduce el flujo de confirmación de asistencia del alumno: escaneo → navegador (app web) → identificación por identificador (matrícula o correo institucional, según el plantel) + PIN (fecha de nacimiento) en **cada** confirmación, sin reconocimiento de dispositivo entre escaneos (los dispositivos pueden compartirse entre alumnos).
- Introduce tolerancia configurable por profesor (default 5 min, en archivo de propiedades) y la regla de tardanza: dura toda la clase, sin transición automática a falta; la entrada tardía queda a discreción del profesor mediante apertura/cierre reversible del registro.
- Introduce el **marcado manual del profesor** como mecanismo de resiliencia de hardware (alumno sin tablet, suplencias capturadas por fuera y transcritas después) — deliberadamente sin cola offline, sin modo pánico dedicado y con el riesgo de fraude por QR reenviado aceptado sin mitigación activa adicional (rotación + contador como única disuasión).
- Introduce la **justificación de faltas** por el profesor (solo faltas, granularidad por materia/sesión de clase, evidencia opcional), con catálogo de motivos por plantel sujeto a aprobación de dirección (motivo nuevo se usa de inmediato y queda pendiente de aprobar para reutilización futura).
- Introduce los **tableros de profesor y dirección**: navegación por calendario (mes/semana/día) unificada con la operación diaria, y resúmenes operativos (asistencia por grupo, ranking de faltas, rachas, justificantes, sesiones de clase pendientes) calculados sobre el periodo visible.
- Introduce el **acceso de solo lectura para alumno y padre** a su historial de asistencia (matrícula + PIN, acotado al mes en curso, transparencia total de campos, sin capacidad de justificar).
- Introduce **login tradicional** (usuario/contraseña, credenciales en base de datos) para profesor y dirección, con alta dinámica de profesores por dirección y aprovisionamiento por contraseña temporal.

## Capabilities

### New Capabilities
- `academic-structure`: modelo de plantel, profesores, materias, grupos, alumnos, inscripciones, asignaciones profesor-materia-grupo y horario informativo.
- `attendance-session`: sesión de clase, QR rotativo, apertura/cierre reversible del registro, contador en vivo, tolerancia y regla de tardanza.
- `attendance-confirmation`: flujo de escaneo y confirmación del alumno, identificación por identificador+PIN en cada confirmación, manejo de QR expirado y re-captura/doble-escaneo.
- `attendance-justification`: justificación de faltas por el profesor, catálogo de motivos por plantel y su aprobación por dirección.
- `teacher-dashboard`: tablero del profesor — navegación por calendario, roster en vivo, marcado manual, resúmenes.
- `admin-dashboard`: tablero de dirección — alta del catálogo base (materias/profesores/grupos/alumnos), resúmenes a nivel plantel, aprobación del catálogo de justificantes, reset de contraseñas y de PIN.
- `student-parent-access`: acceso y consulta de historial de asistencia para alumno y padre/tutor.
- `auth-accounts`: autenticación y aprovisionamiento de cuentas — login tradicional de profesor/dirección con credenciales en base de datos, acceso sin login tradicional de alumno/padre, bootstrap manual de la primera cuenta de dirección por plantel.

### Modified Capabilities
(N/A — proyecto nuevo, sin specs existentes.)

## Impact

- Proyecto **nuevo e independiente**: no reutiliza infraestructura de `notification-service` ni `ticket-management` (aunque puede inspirarse en sus patrones de eventos/notificación desacoplada sin acoplarse a ellos).
- Repositorio actualmente vacío de código — este cambio es la base fundacional; el stack técnico, esquema de base de datos y mecanismo de tiempo real quedan pendientes de definir en `design.md`.
- Alcance deliberadamente amplio en capacidades (8), pero se espera una implementación **por fases**: un núcleo mínimo de uso diario (sesión de clase, QR, confirmación, marcado manual, cierre) antes que las capacidades de valor agregado (tableros, resúmenes, catálogo con aprobación) — el secuenciamiento exacto se define en `tasks.md`.
