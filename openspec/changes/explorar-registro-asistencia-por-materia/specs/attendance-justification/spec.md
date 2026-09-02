## Purpose

Define cómo el profesor justifica faltas de sus alumnos y cómo se gestiona el catálogo de motivos de justificación de cada plantel.

## ADDED Requirements

### Requirement: Justificación exclusiva del profesor
Solo el profesor de la materia/sesión SHALL poder justificar una falta de un alumno en esa sesión. El sistema no SHALL ofrecer un flujo de justificación por padres/tutores ni por el alumno.

#### Scenario: Profesor justifica una falta
- **WHEN** el profesor de una sesión marca una falta de un alumno como justificada
- **THEN** el registro cambia su estado a Falta Justificada de inmediato, sin requerir aprobación previa

### Requirement: Alcance de la justificación
Solo las faltas SHALL poder justificarse; las tardanzas no SHALL ser justificables. La justificación SHALL aplicarse por materia/sesión individual, sin propagarse automáticamente a otras sesiones del mismo día.

#### Scenario: Falta en múltiples materias el mismo día
- **WHEN** un alumno tiene falta en tres materias del mismo día por el mismo motivo
- **THEN** cada profesor debe justificar su propia materia por separado; el sistema no propaga la justificación entre materias

### Requirement: Motivo con catálogo o texto libre
El profesor SHALL poder seleccionar un motivo del catálogo del plantel o escribir un motivo en texto libre, y opcionalmente adjuntar evidencia.

#### Scenario: Justificación con texto libre nuevo
- **WHEN** el profesor escribe un motivo que no existe en el catálogo del plantel
- **THEN** la justificación se aplica de inmediato al registro y el motivo se agrega al catálogo del plantel con estado "pendiente de aprobar"

### Requirement: Aprobación del catálogo por dirección
Dirección SHALL poder aprobar o eliminar un motivo pendiente del catálogo de su plantel. El catálogo de motivos SHALL ser específico de cada plantel (no compartido entre planteles).

#### Scenario: Dirección elimina un motivo pendiente
- **WHEN** dirección elimina un motivo marcado como pendiente de aprobar
- **THEN** el motivo deja de estar disponible para selección futura en el catálogo de ese plantel

### Requirement: Sin fecha límite para justificar
El sistema SHALL permitir justificar una falta en cualquier momento, sin fecha límite.

#### Scenario: Justificación de una falta antigua
- **WHEN** el profesor justifica una falta ocurrida varias semanas atrás
- **THEN** el sistema aplica el cambio sin restricción de tiempo
