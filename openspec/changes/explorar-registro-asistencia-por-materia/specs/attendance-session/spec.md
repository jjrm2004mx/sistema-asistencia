## Purpose

Define el ciclo de vida de una sesión de clase — el QR rotativo, su apertura y cierre, el contador en vivo, y la regla de tolerancia/tardanza — que es el mecanismo central de la toma de asistencia.

## ADDED Requirements

### Requirement: Inicio de sesión de clase
El profesor SHALL poder iniciar una sesión de clase para una de sus asignaciones (materia+grupo), lo cual genera y proyecta un QR rotativo.

#### Scenario: Profesor inicia sesión
- **WHEN** el profesor selecciona una asignación y pulsa iniciar sesión
- **THEN** el sistema crea la sesión y comienza a rotar el QR proyectado

### Requirement: Rotación del QR
El sistema SHALL rotar el token codificado en el QR proyectado cada N segundos, donde N es configurable en el archivo de propiedades del sistema (valor de referencia: 30-45 segundos).

#### Scenario: Token vencido
- **WHEN** un token de QR ha superado su intervalo de vigencia
- **THEN** el sistema rechaza cualquier confirmación de asistencia que use ese token

### Requirement: Contador en vivo
El sistema SHALL mostrar, tanto en la vista pública proyectada como en el tablero del profesor, el conteo de alumnos confirmados sobre el total de alumnos inscritos en la sesión, actualizado en tiempo real.

#### Scenario: Actualización del contador
- **WHEN** un alumno confirma su asistencia
- **THEN** el contador visible en ambas vistas se incrementa sin necesidad de recargar la página

### Requirement: Tolerancia configurable por profesor
Cada profesor SHALL tener un valor de tolerancia en minutos (default del sistema configurable en archivo de propiedades, 5 minutos), aplicado a sus sesiones para determinar si una confirmación cuenta como Presente o Tardanza.

#### Scenario: Confirmación dentro de tolerancia
- **WHEN** un alumno confirma antes de que transcurran los minutos de tolerancia del profesor desde el inicio de la sesión
- **THEN** su registro queda como Presente

#### Scenario: Confirmación fuera de tolerancia
- **WHEN** un alumno confirma después de la tolerancia pero antes del cierre de la sesión
- **THEN** su registro queda como Tardanza

### Requirement: Tardanza vigente durante toda la sesión
El sistema no SHALL transicionar automáticamente un registro de Tardanza a Falta por el solo paso del tiempo dentro de la sesión; la entrada tardía queda a discreción del profesor mediante el cierre del registro.

#### Scenario: Escaneo tardío antes del cierre
- **WHEN** un alumno escanea el QR después de la tolerancia pero la sesión sigue con el registro abierto
- **THEN** el sistema acepta la confirmación y la marca como Tardanza

### Requirement: Cierre y reapertura del registro
El profesor SHALL poder cerrar el registro de una sesión activa, deteniendo nuevas confirmaciones por QR, y reabrirlo cuando lo decida.

#### Scenario: Registro cerrado bloquea confirmaciones
- **WHEN** el registro de una sesión está cerrado
- **THEN** un alumno que intenta escanear el QR no puede confirmar su asistencia hasta que el profesor lo reabra

### Requirement: Alumno sin confirmación queda en falta
Un alumno inscrito en la sesión que no confirma asistencia por QR ni es marcado manualmente por el profesor SHALL quedar registrado como Falta.

#### Scenario: Sesión sin confirmación del alumno
- **WHEN** una sesión concluye y un alumno inscrito no tiene ningún registro de asistencia
- **THEN** el sistema lo considera Falta
