## Purpose

Define los mecanismos de autenticación y aprovisionamiento de cuentas para cada rol del sistema: login tradicional para profesor/dirección y su aprovisionamiento, y la configuración centralizada de valores del sistema.

## ADDED Requirements

### Requirement: Login tradicional para profesor y dirección
Profesor y dirección SHALL autenticarse mediante usuario y contraseña, con credenciales almacenadas en base de datos, a diferencia del mecanismo de identificador+PIN (matrícula o correo institucional, según el plantel) usado por alumno y padre/tutor.

#### Scenario: Login exitoso
- **WHEN** un profesor ingresa su usuario y contraseña correctos
- **THEN** el sistema lo autentica y le da acceso a su tablero

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
- **THEN** el nuevo valor aplica a las sesiones futuras sin requerir cambios en el código
