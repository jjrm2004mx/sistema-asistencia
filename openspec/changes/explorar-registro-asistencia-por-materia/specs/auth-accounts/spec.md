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
Cada cuenta de login tradicional SHALL tener un rol explícito — **profesor** o **dirección** — asignado al momento de su alta y no modificable por el propio usuario. El sistema SHALL exponer un único formulario de login para ambos roles; tras autenticar, SHALL redirigir a la cuenta al tablero correspondiente a su rol (`teacher-dashboard` o `admin-dashboard`) sin que el usuario deba seleccionarlo manualmente.

#### Scenario: Login de profesor redirige a su tablero
- **WHEN** una cuenta con rol profesor se autentica correctamente
- **THEN** el sistema la redirige a su tablero de profesor

#### Scenario: Login de dirección redirige a su tablero
- **WHEN** una cuenta con rol dirección se autentica correctamente
- **THEN** el sistema la redirige a su tablero de dirección

### Requirement: Cuenta ligada a exactamente un plantel
Cada cuenta de login tradicional (profesor o dirección) SHALL pertenecer a exactamente un plantel, asignado al momento de su alta y no modificable por el usuario. Si la misma persona da clases en más de un plantel, SHALL tener una cuenta separada (correo distinto) por cada uno. Tras autenticarse, el sistema SHALL resolver el plantel de la cuenta y limitar tanto el acceso a datos (aislamiento entre planteles, ver `academic-structure`) como las funcionalidades visibles en el tablero a las capabilities habilitadas para ese plantel (ver `academic-structure`, "Funcionalidades habilitadas por plantel").

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
Al dar de alta a un profesor, dirección SHALL generar una contraseña temporal que se entrega por fuera del sistema. El sistema no SHALL ofrecer un flujo de auto-registro ni de recuperación de contraseña por el propio profesor.

#### Scenario: Alta de un nuevo profesor
- **WHEN** dirección da de alta a un profesor
- **THEN** el sistema genera una contraseña temporal que dirección debe entregarle por un medio externo al sistema

### Requirement: Bootstrap manual de la primera cuenta de dirección
La primera cuenta de dirección de un plantel nuevo SHALL crearse mediante un procedimiento operativo manual fuera de la interfaz del sistema, dado que no existe todavía un rol de super-administrador cross-plantel.

#### Scenario: Alta de un plantel nuevo
- **WHEN** se incorpora un plantel nuevo al sistema
- **THEN** la primera cuenta de dirección de ese plantel se crea directamente en la base de datos, no a través de un flujo de autoservicio

### Requirement: Configuración centralizada en archivo de propiedades
Los valores de tolerancia default, intervalo de rotación del QR e intentos fallidos de PIN permitidos SHALL vivir en un único archivo de propiedades del sistema, no hardcodeados ni dispersos en el código. Las credenciales de usuarios individuales (profesor, dirección) no SHALL vivir en este archivo — viven en base de datos, ligadas al alta dinámica de cada cuenta.

#### Scenario: Cambio de un valor de configuración
- **WHEN** se actualiza el valor de tolerancia default en el archivo de propiedades
- **THEN** el nuevo valor aplica a las sesiones de clase futuras sin requerir cambios en el código
