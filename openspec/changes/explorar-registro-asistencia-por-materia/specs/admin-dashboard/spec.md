## Purpose

Define el tablero de dirección/admin de plantel — la gestión del catálogo base, los resúmenes agregados a nivel plantel y la aprobación del catálogo de justificantes.

## ADDED Requirements

### Requirement: Tablero a nivel plantel completo
El tablero de dirección SHALL usar la misma navegación de calendario (mes/semana/día) que el tablero del profesor, pero con datos agregados de todos los grupos y profesores del plantel.

#### Scenario: Vista agregada del día
- **WHEN** dirección entra a su tablero
- **THEN** ve el resumen del día con todas las sesiones de clase programadas del plantel, no solo las de un profesor

### Requirement: Siete resúmenes a nivel plantel
El tablero SHALL mostrar: (1) resumen del día del plantel completo, (2) porcentaje de asistencia por grupo, (3) ranking de faltas/tardanzas del plantel, (4) racha de faltas consecutivas por alumno, (5) justificantes justificados vs. sin justificar, (6) lista de profesores que aún no han pasado lista en sus sesiones de clase del día, y (7) conteo de motivos de justificantes pendientes de aprobar.

#### Scenario: Profesores pendientes de pasar lista
- **WHEN** un profesor tiene una sesión de clase programada según su horario y no la ha iniciado
- **THEN** ese profesor aparece en la lista de "profesores que aún no han pasado lista hoy"

### Requirement: Aprobación del catálogo de justificantes
Dirección SHALL poder ver, aprobar o eliminar los motivos de justificantes marcados como "pendiente de aprobar" en el catálogo de su plantel.

#### Scenario: Dirección aprueba un motivo pendiente
- **WHEN** dirección aprueba un motivo marcado como pendiente
- **THEN** el motivo queda disponible como parte normal del catálogo para futuras justificaciones

### Requirement: Reset de credenciales de profesor y PIN de alumno
Dirección SHALL poder generar una nueva contraseña temporal para un profesor, y desbloquear/resetear el PIN de un alumno bloqueado.

#### Scenario: Profesor olvida su contraseña
- **WHEN** dirección genera una nueva contraseña temporal para un profesor
- **THEN** el profesor puede iniciar sesión de usuario (login) con la nueva contraseña entregada por dirección
