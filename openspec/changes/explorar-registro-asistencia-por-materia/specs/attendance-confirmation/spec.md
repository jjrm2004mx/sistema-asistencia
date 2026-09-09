## Purpose

Define cómo un alumno confirma su propia asistencia al escanear el QR proyectado, incluyendo su identificación y el manejo de reintentos y errores.

## ADDED Requirements

### Requirement: Confirmación vía escaneo de QR
El alumno SHALL confirmar su asistencia escaneando el QR proyectado con la cámara de su dispositivo, lo cual abre una vista web asociada al contexto de esa sesión de clase.

#### Scenario: Escaneo abre la confirmación
- **WHEN** un alumno escanea el QR proyectado
- **THEN** el sistema abre una página web asociada a esa sesión de clase y al token vigente en ese momento

### Requirement: Identificación por identificador de alumno y PIN
El sistema SHALL identificar al alumno mediante su identificador (matrícula o correo institucional, según el tipo configurado por su plantel — ver `academic-structure`) y un PIN (su fecha de nacimiento en formato ddmmaaaa), sin requerir un flujo de usuario/contraseña tradicional.

#### Scenario: Identificación correcta
- **WHEN** el alumno ingresa su identificador y PIN correctos
- **THEN** el sistema lo identifica y procede a registrar su confirmación

### Requirement: Confirmación visible del resultado del registro
Tras identificarse correctamente, el sistema SHALL registrar la asistencia del alumno y mostrarle en su propio dispositivo el estado resultante (Presente o Tardanza) antes de que pueda cerrar la vista.

#### Scenario: Alumno ve su estado tras confirmar
- **WHEN** el alumno se identifica correctamente y el sistema calcula y registra su asistencia
- **THEN** la vista le muestra su estado resultante (ej. "Registrado: Presente" o "Registrado: Tardanza")

### Requirement: Bloqueo por intentos fallidos de PIN
El sistema SHALL bloquear nuevos intentos de identificación de un identificador de alumno tras un número configurable de intentos fallidos consecutivos de PIN (default: 15, en archivo de propiedades).

#### Scenario: Bloqueo tras agotar intentos
- **WHEN** un alumno falla su PIN el número de veces configurado
- **THEN** el sistema bloquea nuevos intentos para ese identificador hasta que el profesor o dirección lo desbloqueen

### Requirement: Identificación completa en cada confirmación, sin reconocimiento de dispositivo
El sistema SHALL solicitar identificador y PIN completos en cada confirmación de asistencia, sin excepción por dispositivo ya usado previamente. No SHALL existir un mecanismo de "dispositivo reconocido" que omita el PIN, dado que un dispositivo puede ser compartido entre distintos alumnos (hermanos, tablet prestada, dispositivo familiar) — recordar un dispositivo como "ya identificado" arriesgaría atribuir la asistencia al alumno equivocado.

#### Scenario: Segunda confirmación el mismo día desde el mismo dispositivo
- **WHEN** un alumno que ya confirmó una materia escanea el QR de una materia distinta el mismo día, desde el mismo dispositivo
- **THEN** el sistema le solicita identificador y PIN completos otra vez, igual que en su primera confirmación

#### Scenario: Dispositivo compartido entre dos alumnos
- **WHEN** dos alumnos distintos usan el mismo dispositivo para confirmar asistencia en sesiones de clase distintas
- **THEN** cada uno debe identificarse con su propio identificador y PIN; el sistema nunca asume que el segundo alumno es el mismo que el primero

### Requirement: Manejo de QR expirado
Si el token leído por el alumno ya no está vigente, el sistema SHALL rechazar la confirmación e indicarle que debe volver a escanear el QR actual, sin ofrecer margen de gracia sobre el token vencido.

#### Scenario: Escaneo de token vencido
- **WHEN** el alumno envía un token que ya rotó
- **THEN** el sistema rechaza la solicitud y le indica que escanee el código vigente

### Requirement: Re-captura y doble escaneo
Si el alumno ya tiene un registro de asistencia para la sesión de clase, el sistema SHALL informarle su estado actual y, si decide volver a capturar, SHALL advertirle que esto sustituye el registro anterior antes de aplicar el cambio — salvo que el nuevo estado calculado sea idéntico al existente, en cuyo caso SHALL mostrar un mensaje neutro sin advertencia.

#### Scenario: Reconfirmación sin cambio de estado
- **WHEN** un alumno ya registrado como Presente vuelve a escanear el QR dentro de la misma sesión de clase y el nuevo cálculo también resulta en Presente
- **THEN** el sistema le informa que ya está registrado, sin mostrar una advertencia de sustitución

#### Scenario: Reconfirmación con cambio de estado
- **WHEN** un alumno ya registrado como Presente escanea de nuevo y el nuevo cálculo resulta en Tardanza
- **THEN** el sistema le advierte que esto sustituirá su registro anterior antes de aplicar el cambio

### Requirement: Escaneo fuera de la materia-grupo inscrita
El sistema SHALL validar en el backend que el alumno esté inscrito en la materia-grupo de la sesión de clase escaneada como salvaguarda de integridad de datos, aunque no requiere una experiencia de usuario dedicada para este caso (los alumnos están físicamente supervisados por el profesor en su salón).

#### Scenario: Validación de inscripción en backend
- **WHEN** una solicitud de confirmación llega para un alumno no inscrito en la materia-grupo de la sesión de clase
- **THEN** el sistema rechaza el registro
