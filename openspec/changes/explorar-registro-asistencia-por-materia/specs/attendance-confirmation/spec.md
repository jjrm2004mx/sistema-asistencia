## Purpose

Define cómo un alumno confirma su propia asistencia al escanear el QR proyectado, incluyendo su identificación, el reconocimiento de su dispositivo y el manejo de reintentos y errores.

## ADDED Requirements

### Requirement: Confirmación vía escaneo de QR
El alumno SHALL confirmar su asistencia escaneando el QR proyectado con la cámara de su dispositivo, lo cual abre una vista web asociada al contexto de esa sesión.

#### Scenario: Escaneo abre la confirmación
- **WHEN** un alumno escanea el QR proyectado
- **THEN** el sistema abre una página web asociada a esa sesión y al token vigente en ese momento

### Requirement: Identificación por matrícula y PIN
El sistema SHALL identificar al alumno mediante su matrícula y un PIN (su fecha de nacimiento en formato ddmmaaaa), sin requerir un flujo de usuario/contraseña tradicional.

#### Scenario: Identificación correcta
- **WHEN** el alumno ingresa su matrícula y PIN correctos
- **THEN** el sistema lo identifica y procede a registrar su confirmación

### Requirement: Bloqueo por intentos fallidos de PIN
El sistema SHALL bloquear nuevos intentos de identificación de una matrícula tras un número configurable de intentos fallidos consecutivos de PIN (default: 15, en archivo de propiedades).

#### Scenario: Bloqueo tras agotar intentos
- **WHEN** un alumno falla su PIN el número de veces configurado
- **THEN** el sistema bloquea nuevos intentos para esa matrícula hasta que el profesor o dirección lo desbloqueen

### Requirement: Reconocimiento ligero de dispositivo
El sistema SHALL requerir la identificación completa (matrícula + PIN) solo la primera vez que un dispositivo confirma asistencia. En confirmaciones subsecuentes desde el mismo dispositivo, SHALL reconocer al alumno sin solicitar el PIN nuevamente.

#### Scenario: Segunda confirmación desde el mismo dispositivo
- **WHEN** un alumno ya identificado en su tablet escanea un nuevo QR de una materia distinta el mismo día
- **THEN** el sistema lo reconoce sin pedirle matrícula y PIN de nuevo

#### Scenario: Confirmación desde un dispositivo distinto
- **WHEN** el alumno intenta confirmar asistencia desde un dispositivo no reconocido previamente
- **THEN** el sistema solicita matrícula y PIN completos

### Requirement: Manejo de QR expirado
Si el token leído por el alumno ya no está vigente, el sistema SHALL rechazar la confirmación e indicarle que debe volver a escanear el QR actual, sin ofrecer margen de gracia sobre el token vencido.

#### Scenario: Escaneo de token vencido
- **WHEN** el alumno envía un token que ya rotó
- **THEN** el sistema rechaza la solicitud y le indica que escanee el código vigente

### Requirement: Re-captura y doble escaneo
Si el alumno ya tiene un registro de asistencia para la sesión, el sistema SHALL informarle su estado actual y, si decide volver a capturar, SHALL advertirle que esto sustituye el registro anterior antes de aplicar el cambio — salvo que el nuevo estado calculado sea idéntico al existente, en cuyo caso SHALL mostrar un mensaje neutro sin advertencia.

#### Scenario: Reconfirmación sin cambio de estado
- **WHEN** un alumno ya registrado como Presente vuelve a escanear el QR dentro de la misma sesión y el nuevo cálculo también resulta en Presente
- **THEN** el sistema le informa que ya está registrado, sin mostrar una advertencia de sustitución

#### Scenario: Reconfirmación con cambio de estado
- **WHEN** un alumno ya registrado como Presente escanea de nuevo y el nuevo cálculo resulta en Tardanza
- **THEN** el sistema le advierte que esto sustituirá su registro anterior antes de aplicar el cambio

### Requirement: Escaneo fuera de la materia-grupo inscrita
El sistema SHALL validar en el backend que el alumno esté inscrito en la materia-grupo de la sesión escaneada como salvaguarda de integridad de datos, aunque no requiere una experiencia de usuario dedicada para este caso (los alumnos están físicamente supervisados por el profesor en su salón).

#### Scenario: Validación de inscripción en backend
- **WHEN** una solicitud de confirmación llega para un alumno no inscrito en la materia-grupo de la sesión
- **THEN** el sistema rechaza el registro
