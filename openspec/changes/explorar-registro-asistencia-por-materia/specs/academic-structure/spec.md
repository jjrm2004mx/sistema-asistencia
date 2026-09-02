## Purpose

Modela la estructura académica multi-plantel del sistema — planteles, profesores, materias, grupos, alumnos, inscripciones, asignaciones y horario — sobre la cual corre toda la operación de toma de asistencia.

## ADDED Requirements

### Requirement: Modelo de plantel multi-tenant
El sistema SHALL soportar múltiples planteles, cada uno con sus propios profesores, materias, grupos y alumnos, aislados entre sí.

#### Scenario: Aislamiento de datos entre planteles
- **WHEN** existen dos planteles distintos en el sistema
- **THEN** los datos de profesores, materias, grupos y alumnos de un plantel no son visibles ni accesibles desde el otro plantel

### Requirement: Alta de catálogo base por dirección
Dirección/Admin de plantel SHALL poder dar de alta materias, profesores, grupos y alumnos de su plantel, de forma individual o mediante importación por CSV.

#### Scenario: Alta individual de un profesor
- **WHEN** dirección crea un nuevo profesor con nombre y datos básicos
- **THEN** el profesor queda registrado en el plantel y disponible para ser asignado a materias y grupos

#### Scenario: Alta masiva por CSV
- **WHEN** dirección sube un archivo CSV con múltiples alumnos
- **THEN** el sistema crea todos los registros válidos del archivo

### Requirement: Inscripción de alumnos a grupos
Un alumno SHALL estar inscrito a uno o más grupos, y esa inscripción determina las materias en las que puede confirmar asistencia.

#### Scenario: Alumno inscrito ve sus materias
- **WHEN** un alumno está inscrito en un grupo
- **THEN** puede confirmar asistencia únicamente en las sesiones de las materias asignadas a ese grupo

### Requirement: Asignación de profesor a materia y grupo
Dirección SHALL poder crear una asignación que vincule un profesor, una materia y un grupo. Un profesor puede tener múltiples asignaciones y una materia puede tener múltiples profesores.

#### Scenario: Profesor con varias materias
- **WHEN** un profesor tiene asignaciones a dos materias distintas
- **THEN** puede iniciar sesiones de asistencia para ambas

### Requirement: Horario informativo, no restrictivo
Cada asignación SHALL poder tener un horario (día de la semana y hora) que sirve como referencia, sin restringir cuándo el profesor puede abrir una sesión.

#### Scenario: Profesor abre sesión fuera de horario
- **WHEN** un profesor con una asignación abre una sesión de asistencia en un horario distinto al registrado
- **THEN** el sistema permite la acción sin bloquearla

### Requirement: Materias siempre ligadas a un grupo fijo
Toda materia asignada SHALL estar ligada a un grupo fijo de alumnos inscritos; el sistema no SHALL soportar materias con lista de inscritos variable independiente de un grupo.

#### Scenario: Asignación sin grupo asociado
- **WHEN** se intenta crear una asignación sin especificar un grupo
- **THEN** el sistema rechaza la operación
