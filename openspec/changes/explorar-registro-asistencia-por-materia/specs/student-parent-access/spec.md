## Purpose

Define el acceso de solo lectura de alumnos y padres/tutores a su historial de asistencia, sin capacidades de edición ni justificación. Padre/tutor **no es una identidad técnica separada** del alumno — no tiene credencial propia ni cuenta propia; accede con el mismo identificador+PIN del alumno, a la misma pantalla. "Alumno" y "padre/tutor" en este documento describen quién está mirando la pantalla, no dos mecanismos de acceso distintos.

## ADDED Requirements

### Requirement: Acceso sin login tradicional
Alumno y padre/tutor SHALL identificarse mediante el identificador del alumno (matrícula o correo institucional, según el tipo configurado por su plantel — ver `academic-structure`) y un PIN (fecha de nacimiento del alumno), sin un flujo de usuario/contraseña tradicional. Este endpoint no tiene la supervisión física que sí tiene el escaneo de QR en el salón, así que el bloqueo por intentos fallidos de PIN (ver `attendance-confirmation`, "Bloqueo por intentos fallidos de PIN, compartido entre flujos") SHALL aplicar aquí también, compartiendo el mismo contador por identificador.

#### Scenario: Identificación para consulta
- **WHEN** un padre ingresa el identificador y PIN de su hijo/a
- **THEN** el sistema lo identifica y le muestra el historial correspondiente a ese identificador

#### Scenario: Identificador bloqueado por intentos fallidos
- **WHEN** un identificador ya está bloqueado (por intentos fallidos previos en este endpoint o en el de confirmación de asistencia)
- **THEN** el sistema rechaza el intento de consulta hasta que el profesor o dirección lo desbloqueen

### Requirement: Historial acotado al mes en curso
El sistema SHALL mostrar el historial de asistencia del alumno acotado al mes en curso, con estado, hora exacta del registro, motivo de justificación (cuando aplica) y origen del registro (auto-confirmación por QR o marcado manual del profesor).

#### Scenario: Consulta del historial
- **WHEN** un alumno o padre se identifica correctamente
- **THEN** ve el detalle completo de cada sesión de clase del mes en curso, incluyendo estado, hora, motivo (si está justificada) y origen del registro

### Requirement: Alcance de "lo propio"
Un alumno SHALL ver únicamente su propio historial. Un padre/tutor SHALL ver únicamente el historial del hijo/a cuyo identificador y PIN ingresó, repitiendo el flujo de identificación por cada hijo/a distinto.

#### Scenario: Padre con más de un hijo/a
- **WHEN** un padre quiere consultar la asistencia de un segundo hijo/a
- **THEN** debe identificarse de nuevo con el identificador y PIN de ese segundo hijo/a

### Requirement: Sin capacidad de edición
Alumno y padre/tutor SHALL tener acceso exclusivamente de solo lectura; no SHALL poder justificar faltas ni modificar ningún registro de asistencia.

#### Scenario: Intento de modificar un registro
- **WHEN** un alumno o padre accede a su historial
- **THEN** el sistema no ofrece ninguna acción de edición o justificación sobre los registros mostrados
