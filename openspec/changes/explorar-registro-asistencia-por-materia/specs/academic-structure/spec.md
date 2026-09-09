## Purpose

Modela la estructura académica multi-plantel del sistema — planteles, profesores, materias, grupos, alumnos, inscripciones, asignaciones y horario — sobre la cual corre toda la operación de toma de asistencia.

## ADDED Requirements

### Requirement: Modelo de plantel multi-tenant
El sistema SHALL soportar múltiples planteles, cada uno con sus propios profesores, materias, grupos y alumnos, aislados entre sí.

#### Scenario: Aislamiento de datos entre planteles
- **WHEN** existen dos planteles distintos en el sistema
- **THEN** los datos de profesores, materias, grupos y alumnos de un plantel no son visibles ni accesibles desde el otro plantel

### Requirement: Funcionalidades habilitadas por plantel
Cada plantel SHALL tener un conjunto de capabilities habilitadas de forma independiente entre sí (no un plan cerrado tipo Básico/Completo), determinando qué funcionalidades del sistema están disponibles para ese plantel. Dirección SHALL poder ver qué funcionalidades tiene habilitadas su plantel, pero no SHALL tener ningún mecanismo en la interfaz para habilitarlas o deshabilitarlas ella misma — ese cambio requiere una autoridad por encima del nivel de plantel (ver `auth-accounts`, rol de super-administrador cross-plantel).

#### Scenario: Plantel con funcionalidad deshabilitada
- **WHEN** una capability está deshabilitada para un plantel
- **THEN** el sistema oculta o bloquea el acceso a esa funcionalidad para todas las cuentas de ese plantel (profesor y dirección)

#### Scenario: Dirección no puede autohabilitarse funcionalidades
- **WHEN** dirección busca en su tablero un mecanismo para habilitar una funcionalidad no disponible en su plantel
- **THEN** el sistema no ofrece ninguna acción para hacerlo; solo puede ver cuáles tiene habilitadas

#### Scenario: Cambio de funcionalidades habilitadas
- **WHEN** se requiere habilitar o deshabilitar una funcionalidad para un plantel
- **THEN** el cambio SHALL hacerse mediante un procedimiento operativo manual (configuración/base de datos) fuera de la interfaz del sistema, dado que no existe todavía un rol de super-administrador cross-plantel con una interfaz dedicada para esto

### Requirement: Tipo de identificador de alumno configurable por plantel
Dirección SHALL configurar, desde su tablero y como parte del setup inicial de su plantel, un único tipo de identificador para sus alumnos — **matrícula** o **correo institucional** — dado que no todas las escuelas manejan matrícula. Todos los alumnos de un mismo plantel SHALL usar el mismo tipo (no se mezclan ambos dentro de un mismo plantel). Este identificador SHALL ser único dentro del plantel y es el que usan alumno y padre/tutor para identificarse en los flujos de matrícula/correo+PIN (ver `attendance-confirmation` y `student-parent-access`). El sistema puede mantener además un ID interno propio del alumno, pero alumno, padre/tutor y profesor no SHALL necesitar conocerlo ni usarlo.

Una vez que el plantel tiene al menos un alumno dado de alta, el tipo de identificador configurado SHALL quedar bloqueado — el sistema no SHALL permitir cambiarlo, para evitar inconsistencias con alumnos ya identificados bajo el tipo anterior.

#### Scenario: Plantel configurado con matrícula
- **WHEN** un plantel tiene configurado el tipo de identificador "matrícula"
- **THEN** todos sus alumnos se dan de alta con una matrícula como identificador único, y los flujos de confirmación de asistencia y consulta de historial solicitan matrícula+PIN

#### Scenario: Plantel configurado con correo institucional
- **WHEN** un plantel tiene configurado el tipo de identificador "correo institucional"
- **THEN** todos sus alumnos se dan de alta con su correo institucional como identificador único, y los flujos de confirmación de asistencia y consulta de historial solicitan correo institucional+PIN en vez de matrícula

#### Scenario: Identificador duplicado dentro del plantel
- **WHEN** se intenta dar de alta un alumno cuyo identificador (matrícula o correo institucional, según lo configurado) ya existe en ese plantel
- **THEN** el sistema rechaza el alta

#### Scenario: Configuración libre antes del primer alumno
- **WHEN** el plantel todavía no tiene ningún alumno dado de alta
- **THEN** dirección puede configurar o cambiar libremente el tipo de identificador desde su tablero

#### Scenario: Tipo de identificador bloqueado tras el primer alumno
- **WHEN** el plantel ya tiene al menos un alumno dado de alta y dirección intenta cambiar el tipo de identificador
- **THEN** el sistema rechaza el cambio

### Requirement: Alta de catálogo base por dirección
Dirección/Admin de plantel SHALL poder dar de alta materias, profesores, grupos y alumnos de su plantel, de forma individual o mediante importación por CSV.

#### Scenario: Alta individual de un profesor
- **WHEN** dirección crea un nuevo profesor con nombre y datos básicos
- **THEN** el profesor queda registrado en el plantel y disponible para ser asignado a materias y grupos

#### Scenario: Alta masiva por CSV
- **WHEN** dirección sube un archivo CSV con múltiples alumnos, con la columna de identificador correspondiente al tipo configurado para su plantel (matrícula o correo institucional)
- **THEN** el sistema crea todos los registros válidos del archivo

### Requirement: Inscripción de alumnos a grupos
Un alumno SHALL estar inscrito a uno o más grupos, y esa inscripción determina las materias en las que puede confirmar asistencia.

#### Scenario: Alumno inscrito ve sus materias
- **WHEN** un alumno está inscrito en un grupo
- **THEN** puede confirmar asistencia únicamente en las sesiones de clase de las materias asignadas a ese grupo

### Requirement: Asignación de profesor a materia y grupo
Dirección SHALL poder crear una asignación que vincule un profesor, una materia y un grupo. Un profesor puede tener múltiples asignaciones y una materia puede tener múltiples profesores.

#### Scenario: Profesor con varias materias
- **WHEN** un profesor tiene asignaciones a dos materias distintas
- **THEN** puede iniciar sesiones de clase para ambas

### Requirement: Horario informativo, no restrictivo
Cada asignación SHALL poder tener un horario (día de la semana y hora) que sirve como referencia, sin restringir cuándo el profesor puede abrir una sesión de clase.

#### Scenario: Profesor abre sesión de clase fuera de horario
- **WHEN** un profesor con una asignación abre una sesión de clase en un horario distinto al registrado
- **THEN** el sistema permite la acción sin bloquearla

### Requirement: Materias siempre ligadas a un grupo fijo
Toda materia asignada SHALL estar ligada a un grupo fijo de alumnos inscritos; el sistema no SHALL soportar materias con lista de inscritos variable independiente de un grupo.

#### Scenario: Asignación sin grupo asociado
- **WHEN** se intenta crear una asignación sin especificar un grupo
- **THEN** el sistema rechaza la operación
