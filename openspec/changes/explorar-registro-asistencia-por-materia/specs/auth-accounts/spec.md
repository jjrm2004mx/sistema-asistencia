## Purpose

Define los mecanismos de autenticación y aprovisionamiento de cuentas para cada rol del sistema: login tradicional para profesor/dirección y su aprovisionamiento, y la configuración centralizada de valores del sistema.

## ADDED Requirements

### Requirement: Login tradicional para profesor y dirección
Profesor y dirección SHALL autenticarse mediante usuario y contraseña, con credenciales almacenadas en base de datos, a diferencia del mecanismo de identificador+PIN (matrícula o correo institucional, según el plantel) usado por alumno y padre/tutor. El campo "usuario" SHALL ser el correo electrónico del profesor/dirección, capturado por dirección al momento del alta — no un username independiente. El correo SHALL ser único dentro del plantel.

#### Scenario: Login exitoso
- **WHEN** un profesor ingresa su correo y contraseña correctos
- **THEN** el sistema lo autentica y le da acceso a su tablero

#### Scenario: Correo duplicado dentro del plantel
- **WHEN** dirección intenta dar de alta a un profesor con un correo que ya pertenece a otra cuenta de ese plantel
- **THEN** el sistema rechaza el alta

### Requirement: Rol explícito por cuenta y enrutamiento post-login
Cada cuenta de login tradicional SHALL tener un rol explícito — **profesor**, **dirección** o **super-administrador** — asignado al momento de su alta y no modificable por el propio usuario. El sistema SHALL exponer un único formulario de login para los tres roles; tras autenticar, SHALL redirigir a la cuenta al tablero correspondiente a su rol (`teacher-dashboard`, `admin-dashboard` o `super-admin-dashboard`) sin que el usuario deba seleccionarlo manualmente.

#### Scenario: Login de profesor redirige a su tablero
- **WHEN** una cuenta con rol profesor se autentica correctamente
- **THEN** el sistema la redirige a su tablero de profesor

#### Scenario: Login de dirección redirige a su tablero
- **WHEN** una cuenta con rol dirección se autentica correctamente
- **THEN** el sistema la redirige a su tablero de dirección

#### Scenario: Login de super-administrador redirige a su tablero
- **WHEN** una cuenta con rol super-administrador se autentica correctamente
- **THEN** el sistema la redirige a `super-admin-dashboard`

### Requirement: Cuenta ligada a exactamente un plantel (profesor y dirección)
Cada cuenta de login tradicional con rol profesor o dirección SHALL pertenecer a exactamente un plantel, asignado al momento de su alta y no modificable por el usuario. Si la misma persona da clases en más de un plantel, SHALL tener una cuenta separada (correo distinto) por cada uno. Tras autenticarse, el sistema SHALL resolver el plantel de la cuenta y limitar tanto el acceso a datos (aislamiento entre planteles, ver `academic-structure`) como las funcionalidades visibles en el tablero a las capabilities habilitadas para ese plantel (ver `academic-structure`, "Funcionalidades habilitadas por plantel"). Una cuenta de super-administrador no SHALL estar ligada a ningún plantel — su alcance es cross-plantel por definición (ver `super-admin-dashboard`).

#### Scenario: Funcionalidad deshabilitada oculta tras login
- **WHEN** un profesor de un plantel con una capability deshabilitada (ej. `attendance-justification`) inicia sesión
- **THEN** su tablero no muestra ninguna opción relacionada con esa funcionalidad

#### Scenario: Misma persona en dos planteles
- **WHEN** una persona da clases en dos planteles distintos
- **THEN** necesita una cuenta (correo) separada por cada plantel; no existe una sola cuenta que abarque ambos

### Requirement: Cierre de sesión de usuario (logout)
Profesor y dirección SHALL poder cerrar su sesión de usuario de forma explícita desde su tablero. Al cerrarla, el sistema SHALL revocar el acceso a su tablero hasta que vuelva a autenticarse. Este cierre es independiente del cierre del registro de una sesión de clase (ver `attendance-session`) — son dos conceptos distintos que no deben confundirse.

#### Scenario: Logout explícito
- **WHEN** un profesor o dirección pulsa cerrar sesión en su tablero
- **THEN** el sistema termina su sesión de usuario y lo regresa al formulario de login

#### Scenario: Acceso tras logout
- **WHEN** un usuario que cerró sesión intenta volver a su tablero sin autenticarse de nuevo
- **THEN** el sistema le niega el acceso y lo redirige al formulario de login

### Requirement: Aprovisionamiento por contraseña temporal
Al dar de alta a un profesor o a otra cuenta de dirección de su mismo plantel, dirección SHALL generar una contraseña temporal que se entrega por fuera del sistema. El sistema no SHALL ofrecer un flujo de auto-registro ni de recuperación de contraseña por el propio usuario. Cada alta está sujeta al cupo de cuentas por rol del plantel (ver `academic-structure`, "Cupo de cuentas por plantel").

#### Scenario: Alta de un nuevo profesor
- **WHEN** dirección da de alta a un profesor
- **THEN** el sistema genera una contraseña temporal que dirección debe entregarle por un medio externo al sistema

#### Scenario: Alta de otra cuenta de dirección
- **WHEN** una cuenta de dirección da de alta a otra cuenta con rol dirección de su mismo plantel
- **THEN** el sistema genera una contraseña temporal que se entrega por fuera del sistema, igual que con un profesor

### Requirement: Bootstrap manual de la primera cuenta de super-administrador
La primera cuenta de super-administrador del sistema SHALL crearse mediante un procedimiento operativo manual fuera de la interfaz, directamente en base de datos — una sola vez en la vida del sistema, dado que no existe una cuenta superior que pueda darla de alta. Toda alta posterior (planteles nuevos, otras cuentas de super-administrador, cuentas de dirección) se realiza desde `super-admin-dashboard`, no mediante intervención manual.

#### Scenario: Arranque del sistema
- **WHEN** el sistema se pone en operación por primera vez
- **THEN** la primera cuenta de super-administrador se crea directamente en la base de datos, no a través de un flujo de autoservicio ni de una UI

### Requirement: Configuración centralizada en archivo de propiedades
Los valores de tolerancia default, intervalo de rotación del QR e intentos fallidos de PIN permitidos SHALL vivir en un único archivo de propiedades del sistema, no hardcodeados ni dispersos en el código. Las credenciales de usuarios individuales (profesor, dirección) no SHALL vivir en este archivo — viven en base de datos, ligadas al alta dinámica de cada cuenta.

#### Scenario: Cambio de un valor de configuración
- **WHEN** se actualiza el valor de tolerancia default en el archivo de propiedades
- **THEN** el nuevo valor aplica a las sesiones de clase futuras sin requerir cambios en el código
