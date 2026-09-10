## Purpose

Define el tablero del profesor — la vista unificada de calendario, roster en vivo, marcado manual y resúmenes operativos con la que interactúa día a día para tomar y revisar asistencia.

## ADDED Requirements

### Requirement: Navegación unificada por calendario
El tablero SHALL presentar una vista de calendario navegable (mes/semana/día) combinada con un selector de materia/hora, como único punto de entrada tanto para iniciar sesiones de clase del día como para revisar sesiones de clase pasadas. El sistema no SHALL ofrecer una sección separada de "historial".

#### Scenario: Entrada por default
- **WHEN** el profesor entra a su tablero
- **THEN** el sistema muestra el día actual con la materia+grupo+hora sugerida según el horario registrado y la hora actual, editable por el profesor

#### Scenario: Navegación a un día pasado
- **WHEN** el profesor navega el calendario a una fecha anterior y selecciona una materia/hora
- **THEN** el sistema muestra el roster y controles de esa sesión de clase pasada en la misma pantalla, sin una sección aparte

### Requirement: Roster en vivo de la sesión de clase
El tablero SHALL mostrar, para la sesión de clase seleccionada, el listado de alumnos inscritos con su estado individual (Presente, Tardanza, Falta, Falta Justificada, Pendiente) actualizado en tiempo real. Falta Justificada SHALL distinguirse visualmente de Falta — no son el mismo estado.

#### Scenario: Alumno confirma mientras el profesor observa
- **WHEN** un alumno confirma su asistencia
- **THEN** el estado de ese alumno se actualiza en el roster del profesor sin recargar la página

#### Scenario: Falta justificada se distingue de falta sin justificar
- **WHEN** el profesor justifica la falta de un alumno (ver `attendance-justification`)
- **THEN** el roster deja de mostrarlo como Falta y lo muestra como Falta Justificada

### Requirement: Marcado manual sin restricción de momento
El profesor SHALL poder marcar manualmente el estado de asistencia de cualquier alumno en cualquier sesión de clase — activa, cerrada, o de un día anterior — en cualquier momento.

#### Scenario: Marcado manual en sesión de clase pasada
- **WHEN** el profesor navega a una sesión de clase de una semana anterior y marca manualmente a un alumno
- **THEN** el sistema aplica el cambio sin bloquear la acción por tratarse de una fecha pasada

### Requirement: Advertencia al sobrescribir un auto-registro
Si el profesor marca manualmente a un alumno que ya tiene un registro por auto-confirmación de QR, el sistema SHALL advertirle antes de aplicar el cambio, salvo que el nuevo estado sea idéntico al existente.

#### Scenario: Profesor sobrescribe un registro por QR
- **WHEN** el profesor marca manualmente a un alumno que ya escaneó el QR, con un estado distinto al registrado
- **THEN** el sistema muestra una advertencia antes de confirmar el cambio

### Requirement: Desbloqueo de identificador de alumno bloqueado
El profesor SHALL poder ver, dentro de su roster, qué alumnos de sus propias asignaciones tienen su identificador bloqueado por intentos fallidos de PIN (ver `attendance-confirmation`), y desbloquearlo seleccionándolo de una lista — la misma acción que dirección tiene disponible en `admin-dashboard`, pero acotada a los alumnos que el profesor atiende.

#### Scenario: Alumno bloqueado visible en el roster
- **WHEN** un alumno de una sesión de clase del profesor tiene su identificador bloqueado
- **THEN** el roster del profesor lo muestra con un estatus "Bloqueado"

#### Scenario: Profesor desbloquea a un alumno
- **WHEN** el profesor selecciona a un alumno bloqueado y confirma el desbloqueo
- **THEN** el sistema le permite al alumno volver a intentar su identificación

### Requirement: Resumen de cierre con alumnos sin registrar
Al cerrarse el registro de una sesión de clase (manual o por cuenta regresiva vencida), el tablero del profesor SHALL mostrar un resumen con el conteo de confirmados y la lista nominal de los alumnos inscritos que no registraron asistencia. Esta lista SHALL ser visible únicamente en el tablero privado del profesor, nunca en la vista pública proyectada.

#### Scenario: Registro se cierra con alumnos pendientes
- **WHEN** el registro de la sesión de clase se cierra
- **THEN** el tablero del profesor muestra el resumen de confirmados y la lista con nombre de los alumnos sin registro

### Requirement: Resúmenes del mes seleccionado
El tablero SHALL mostrar cinco resúmenes calculados sobre el mes actualmente seleccionado en el calendario: (1) sesiones de clase del día pendientes/completadas, (2) porcentaje de asistencia por grupo, (3) ranking de faltas/tardanzas por alumno, (4) racha de faltas consecutivas por alumno, y (5) conteo de justificantes justificados vs. sin justificar.

#### Scenario: Cambio de mes recalcula resúmenes
- **WHEN** el profesor navega el calendario a un mes anterior
- **THEN** los cinco resúmenes se recalculan sobre ese mes en vez del mes actual
