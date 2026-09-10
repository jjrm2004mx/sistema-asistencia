## Purpose

Define el tablero del super-administrador — la única cuenta con alcance cross-plantel del sistema. Reemplaza lo que antes eran procedimientos manuales en base de datos (alta de plantel, funcionalidades habilitadas, cupo de cuentas, restablecimiento de acceso de dirección) por acciones reales de UI, salvo la deshabilitación completa de un plantel, que se mantiene manual.

## ADDED Requirements

### Requirement: Alta de plantel nuevo
El super-administrador SHALL poder dar de alta un plantel nuevo desde su tablero, creando el registro del plantel y su primera cuenta de dirección en un solo paso — con aprovisionamiento por contraseña temporal, igual que el resto de altas de cuentas (ver `auth-accounts`) — sin necesidad de intervención manual en base de datos.

#### Scenario: Alta de plantel nuevo desde la UI
- **WHEN** el super-administrador da de alta un plantel nuevo junto con los datos de su primera cuenta de dirección
- **THEN** el sistema crea el plantel y esa cuenta de dirección, con una contraseña temporal lista para entregarse por fuera del sistema

### Requirement: Gestión de funcionalidades habilitadas por plantel
El super-administrador SHALL poder ver y cambiar, desde su tablero, qué capabilities tiene habilitadas cada plantel (ver `academic-structure`, "Funcionalidades habilitadas por plantel").

#### Scenario: Super-administrador habilita una funcionalidad
- **WHEN** el super-administrador habilita una capability para un plantel
- **THEN** esa funcionalidad queda disponible de inmediato para las cuentas de ese plantel

#### Scenario: Solo dos capabilities son toggleables
- **WHEN** el super-administrador abre la gestión de funcionalidades de un plantel
- **THEN** solo ve `attendance-justification` y `student-parent-access` como opciones — las demás 6 capabilities de plantel están siempre activas y no aparecen ahí; el mecanismo queda listo para que futuras capabilities de valor agregado se sumen a esta lista sin rediseñar el esquema

### Requirement: Gestión de cupo de cuentas por plantel
El super-administrador SHALL poder ver y cambiar, desde su tablero, el cupo máximo de cuentas de profesor y de dirección de cada plantel (ver `academic-structure`, "Cupo de cuentas por plantel").

#### Scenario: Super-administrador amplía un cupo
- **WHEN** el super-administrador aumenta el cupo de cuentas de profesor de un plantel
- **THEN** dirección de ese plantel puede dar de alta profesores hasta el nuevo máximo

### Requirement: Restablecer acceso de dirección bloqueada
El super-administrador SHALL poder restablecer el acceso de un plantel cuando ninguna de sus cuentas de dirección puede autenticarse, generando una nueva contraseña temporal para al menos una de sus cuentas de dirección.

#### Scenario: Restablecimiento desde la UI
- **WHEN** el super-administrador restablece el acceso de un plantel sin ninguna cuenta de dirección funcional
- **THEN** el sistema genera una contraseña temporal para una cuenta de dirección de ese plantel

### Requirement: Deshabilitación completa de un plantel (manual, sin UI)
El super-administrador SHALL poder deshabilitar por completo un plantel, revocando el acceso de todas sus cuentas (profesor y dirección) — a diferencia del resto de las funcionalidades de este tablero, este paso SHALL realizarse directamente en base de datos por ahora, sin una acción dedicada en la interfaz, dado que es un caso poco frecuente (cancelación de un plantel-cliente) frente al resto de la operación diaria del tablero.

#### Scenario: Plantel deshabilitado bloquea todo acceso
- **WHEN** un plantel es deshabilitado
- **THEN** ninguna de sus cuentas (profesor o dirección) puede autenticarse, sin importar que sus credenciales sean correctas

#### Scenario: Deshabilitación es manual
- **WHEN** se requiere deshabilitar un plantel
- **THEN** el cambio se realiza directamente en base de datos por quien administra el sistema, no desde el tablero de super-administrador

### Requirement: Alta de otra cuenta de super-administrador
El super-administrador SHALL poder dar de alta a otra cuenta de super-administrador, con el mismo mecanismo de aprovisionamiento por contraseña temporal usado para profesor y dirección. No hay límite en el número de cuentas de super-administrador.

#### Scenario: Alta de otro super-administrador
- **WHEN** un super-administrador da de alta a otra cuenta con ese mismo rol
- **THEN** el sistema genera una contraseña temporal que se entrega por fuera del sistema
