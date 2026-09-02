# Notas de exploración — Registro de asistencia por materia

> Estado: exploración con un bloque sólido de decisiones cerradas (ver abajo) — mecanismo de asistencia, tolerancia, anti-fraude, justificantes, roles/acceso y resiliencias — sin preguntas abiertas bloqueantes por ahora. Suficiente para arrancar un `proposal.md` cuando se decida formalizar. Este documento no es un `proposal.md` ni un `design.md` formal.

## Contexto confirmado

- Sistema de asistencia escolar, en un plantel de **secundaria**.
- Los alumnos cuentan con **tablet propia** (1:1). La maestra también tiene tablet y hay **proyector** disponible en el salón.
- Se asume que hay tecnología suficiente en el plantel para mecanismos digitales (no se ha confirmado el estado de la red WiFi por salón — es una pregunta abierta, ver más abajo).
- **Decisión explícita del usuario**: este sistema será un **ecosistema nuevo e independiente**. No reutiliza infraestructura de `notification-service` / `ticket-management` ni otros repos del ecosistema de tickets, aunque esos repos pueden servir como inspiración de patrones (notificación desacoplada vía eventos, clasificación automática, etc.) sin acoplamiento.

## Decisiones cerradas

### Mecanismo de asistencia: QR proyectado (único)
Se evaluaron dos direcciones (QR proyectado por la maestra vs. QR personal del alumno escaneado por la maestra). Se descarta la segunda. El sistema usa **un solo mecanismo**: la maestra proyecta un QR dinámico que rota cada N segundos, y cada alumno lo escanea con su propia tablet para confirmar asistencia.

### Alcance: SaaS multi-plantel desde el modelo de datos
Razonamiento del usuario: el costo de modelarlo N:N desde ahora es bajo comparado con modelarlo para un solo profesor, y deja la puerta abierta a vender el sistema como SaaS si madura. El dominio es:

```
                    [ Plantis SaaS ]
                          |
        +-----------------+-----------------+
        |                                   |
    Plantel A                           Plantel B  ... N
        |
        +-- N Profesores  ----N:N----  N Materias
        |        (un profesor puede dar varias materias;
        |         una materia puede tener varios profesores
        |         -- ej. dos grupos del mismo grado)
        |
        +-- N Grupos ----N:N---- N Alumnos (inscripción)
        |
        +-- Sesión de clase = Materia + Grupo + Fecha/Hora
                 |
                 +-- QR rotativo (config. por archivo de propiedades)
                 +-- Contador en vivo (confirmados / total inscritos)
                 +-- N Registros de asistencia (1 por alumno inscrito,
                 |    o marcado manual por el profesor)
                 +-- Tolerancia aplicada = la del profesor de esa
                      sesión (ver regla de tolerancia abajo)
```

### Roles
- **Profesor** — toma lista, justifica faltas de sus materias, ve su tablero.
- **Dirección/Admin de plantel** — alta de profesores/materias/grupos dentro de su plantel, aprueba motivos nuevos del catálogo de justificantes.
- **Alumno** — cuenta propia (ver mecanismo de acceso abajo), escanea el QR de forma autenticada (el registro de asistencia queda ligado a `alumno_id`, no solo al dispositivo), y puede ver su **historial de asistencia** (día actual y días anteriores).
- **Padre/Tutor** — mismo mecanismo de acceso que el alumno (ver abajo), consulta el historial de asistencia de su hijo/a. No tiene rol de justificación (eso quedó descartado — solo el profesor justifica).
- **Super-admin SaaS** (cross-plantel) — **pospuesto** hasta que exista un segundo plantel real operando. No se diseña todavía.
- **Tutor de grupo / coordinador** — **descartado**, no existe esa figura en el plantel de referencia.

### Acceso de alumno y padre — sin login tradicional
Decisión explícita: no se expone a los alumnos (ni a los padres) a un flujo de login con usuario/contraseña. En su lugar:
- **Identificador**: matrícula/ID de estudiante.
- **PIN**: fecha de nacimiento del alumno, formato `ddmmaaaa` (ej. `19072007`).
- **Bloqueo por intentos fallidos**: 15 intentos, **configurable en el mismo archivo de propiedades** que la tolerancia, la rotación del QR y (ahora) este límite.
- Razonamiento: es un dato de bajo riesgo (consulta de asistencia, no notas ni datos financieros); el PIN no necesita ser único a nivel plantel, solo desambigua dentro de la consulta de una matrícula específica.
- **Revisado con la lupa de simplicidad (ver sección "Simplificaciones" abajo): matrícula+PIN completo aplica siempre para CONSULTAR historial desde cualquier dispositivo. Para CONFIRMAR asistencia desde la tablet propia del alumno en clase, el PIN completo solo se pide la primera vez en ese dispositivo — de ahí en adelante el navegador ya "sabe" quién es en ESE dispositivo (token/cookie local, no un login formal).** Esto reemplaza la decisión anterior de "sin sesión persistente en ningún caso" — ver detalle en "Simplificaciones".

### Detalle fino de permisos alumno/padre
- **Padre con más de un hijo/a**: **repite el flujo completo** (matrícula + PIN) por cada hijo por separado. No existe un vínculo formal padre-múltiples-hijos que evite volver a identificarse por cada uno.
- **Campos visibles en el historial** (transparencia total, sin ocultar detalle): estado (Presente/Tardanza/Falta/Justificada), **hora exacta del registro**, **el texto del motivo** cuando está justificada, y **si fue auto-registro por QR o marcado manual del profesor**.
- **Alcance temporal**: el historial visible **se acota al mes en curso** (no es histórico completo sin límite desde el inicio del sistema).
- **Alcance de "lo propio"**: el alumno **solo ve su propio historial, nunca el de compañeros**, sin excepción.

### Flujo de confirmación de asistencia (vista del alumno)
1. El alumno escanea el QR proyectado con la cámara de su tablet → **el QR abre un navegador** (el QR codifica una URL con `sesión_id` + token de rotación vigente; consistente con que todo el sistema es aplicación web, no una app nativa).
2. El navegador pide **matrícula + fecha de nacimiento** — **solo la primera vez en ese dispositivo** (ver "Simplificaciones"); en escaneos posteriores desde la misma tablet, el dispositivo ya identifica al alumno sin volver a teclear el PIN completo.
3. Al enviar, el servidor resuelve el estado según lo que encuentre para ese alumno en esa sesión:
   - **No hay registro previo** → se crea el registro (Presente/Tardanza según hora vs. tolerancia del profesor).
   - **Ya existe un registro** (el alumno ya había confirmado, o está reintentando/doble-escaneo) → se le informa y se le ofrece **volver a capturar**, con un **warning explícito de que esto sustituye el registro anterior** antes de aplicar el cambio. Esta misma lógica cubre el caso de doble escaneo accidental — no hay una ruta de error separada para eso.
4. **QR expirado** (el token ya rotó antes de que se procesara la solicitud): no hay margen de gracia ni mensaje especial más allá de pedir que **vuelva a leer el código actual** — se resuelve re-escaneando, no reintentando el mismo token.
5. **Caso descartado explícitamente**: escanear el QR de una materia-grupo donde el alumno no está inscrito (ej. otro salón) — no se considera un caso real a manejar en UX, porque los alumnos están físicamente en el salón bajo supervisión del profesor; no pueden estar escaneando el QR de otro salón. (La validación de inscripción puede seguir existiendo en el backend como salvaguarda de integridad de datos, pero no requiere una experiencia de usuario dedicada.)
6. El contador en vivo del profesor/proyector se actualiza con cada confirmación (ya confirmado en rondas anteriores).

### Tolerancia: configurable por profesor, default en archivo de propiedades
- La tolerancia (minutos tras el inicio de la materia antes de contar "falta") es **configurable por profesor**, no por materia ni global-fija.
- Existe un **default del sistema = 5 minutos**, pero ese valor vive en un **archivo de propiedades/configuración**, no hardcodeado — mismo mecanismo de configuración que la rotación del QR (ver siguiente punto).
- **Tardanza dura toda la clase**: no hay transición automática tardanza→falta por tiempo. Si el alumno escanea después de la tolerancia pero antes de que termine la sesión, es tardanza. **La entrada física tardía al salón queda a discreción del profesor, no del sistema** — el sistema no le cierra la puerta a un escaneo tardío, es el profesor quien decide si deja entrar al alumno. Si no lo deja entrar, el alumno se queda sin escanear (falta por ausencia de confirmación, o el profesor la marca manualmente vía el fallback ya confirmado).

### Rotación del QR: configurable en archivo de propiedades
El intervalo de rotación del QR (**referencia ajustada a 30-45s**, ver "Simplificaciones" — antes 10-15s) es configurable desde el mismo archivo de propiedades que la tolerancia — un solo lugar de configuración del sistema, no valores repartidos ni hardcodeados.

### Plataforma: aplicación web
El sistema es una **aplicación web**, accesible desde cualquier dispositivo con navegador (tablet, laptop), no una app nativa atada a un dispositivo específico. Esto resuelve de raíz la resiliencia de hardware del profesor: si su tablet falla, simplemente usa una laptop u otro dispositivo disponible. No se decidió lo mismo explícitamente para el alumno (se sigue asumiendo tablet 1:1), pero al ser web, el mismo argumento aplicaría si hiciera falta.

### Resiliencia humana: sin mecanismos dedicados (por ahora)
- **Modo pánico / pausa de sesión ante interrupción (simulacro, salida imprevista)**: se evaluó y se decidió **no construir un estado dedicado**. Si la sesión se interrumpe, el profesor simplemente vuelve a capturar la asistencia con normalidad (reinicia/continúa el registro) — el sistema no necesita "saber" que hubo una interrupción.
- **Recordatorio automático si el profesor no ha abierto su sesión de clase**: **fuera de alcance por ahora**, descartado explícitamente, no queda como pendiente activo.

### Resiliencia de red: fuera de alcance por ahora
Se decidió explícitamente **no explorar** cola local / sync diferido por el momento. No es prioridad — descartado, no pendiente activo.

### Anti-fraude: riesgo de foto-reenvío aceptado por ahora
Riesgo identificado: el QR proyectado es una imagen; se puede fotografiar y reenviar (ej. WhatsApp) a un alumno ausente para que "confirme" remotamente desde su propia tablet, sin estar físicamente en el plantel.
- Se descartó atar la confirmación a la red del plantel como mitigación — **decisión explícita: la red del plantel no debe intervenir en el proceso** (hay una sola red para todo el plantel, no AP separado por salón).
- **Decisión: riesgo aceptado por ahora.** Las únicas mitigaciones en alcance son la rotación del QR y el contador en vivo (ver siguiente punto) como disuasión pasiva — no hay verificación activa (geolocalización, vínculo dispositivo-alumno tipo MDM, etc.). Si en la práctica esto duele, se revisita más adelante.

### Contador en vivo (confirmado, antes era brainstorm)
Debe mostrarse en dos lugares:
- En el **tablero del profesor** (vista privada, para que dé seguimiento).
- **Junto al QR proyectado** (vista pública en el proyector) — ej. "27/32 confirmados".

### Fallback manual del profesor (confirmado, antes era brainstorm)
Si un alumno no tiene su tablet disponible (olvidada, rota, sin batería), el **profesor puede marcar la asistencia manualmente** desde su tablero. Es el mecanismo de resiliencia de hardware.

### Flujo de asignación materia-grupo-profesor
- **Alta del catálogo base** (materias, profesores, grupos, alumnos + inscripción a grupo) la hace **dirección/admin de plantel**, manualmente. **Importación por CSV es opcional** (para altas masivas), no el único camino.
- **Asignación** = Profesor + Materia + Grupo. La crea dirección manualmente (CSV opcional también aquí).
- **Horario formal existe** (día(s) de la semana + hora de inicio, ej. "Matemáticas, Profesor X, Grupo 1A, Lun/Mié/Vie 7:00am") y el sistema lo conoce de antemano.
- **El horario es informativo/orientativo, no una restricción dura**: ayuda a saber qué toca, pero **cualquier profesor con una asignación a esa materia-grupo puede abrir sesión (proyectar el QR) en cualquier momento**, no solo en su horario programado.
- **No existen materias sin grupo fijo** (se descarta el caso de talleres/electivas con lista de inscritos variable — todas las materias del plantel de referencia cuelgan de un grupo fijo).
- **Suplencias/cobertura: fuera de alcance como mecanismo de sistema.** Si un profesor suplente cubre una clase, toma la asistencia por un medio externo (Excel u otro), y **el profesor titular actualiza el sistema después usando el fallback manual** (el mismo mecanismo ya confirmado para "alumno sin tablet disponible") — no se crea ningún rol de suplente ni flujo dedicado.

### Flujo de sesión — vista del profesor
Dos vistas distintas (no una sola pantalla espejada), consistente con que el contador en vivo ya se había confirmado en ambos lugares:
- **Vista pública (proyector)**: QR grande rotando + contador simple ("27/32 confirmados").
- **Vista privada (tablero del profesor, en su tablet o cualquier dispositivo — es app web)**: roster completo de alumnos inscritos con su estado en tiempo real (Presente/Tardanza/Falta/pendiente), controles de la sesión, marcado manual, acceso a justificar faltas.

Flujo:
1. Profesor entra a su tablero, ve sus asignaciones (el horario es orientativo, puede elegir cualquiera de sus materia-grupo asignadas, no solo "la de ahora").
2. Selecciona materia+grupo, inicia la sesión → arranca la rotación del QR, se proyecta la vista pública.
3. Alumnos escanean; el roster privado se actualiza en vivo.
4. **Marcado manual**: el profesor puede marcarlo **en cualquier momento**, sin restricción de que sea "en vivo" o "al final" — no está atado al estado abierto/cerrado del registro.
5. **Cierre del registro**: el tablero tiene un botón para **cerrar el registro de asistencia** — mientras está cerrado, los alumnos ya no pueden auto-registrarse escaneando el QR. Es **reversible**: el profesor puede "volver a poner disponible el QR" para reabrir el registro cuando quiera (ej. si decide dejar entrar a un alumno tarde). Esto es, en la práctica, el mecanismo concreto que materializa la decisión ya tomada de que "la entrada tardía queda a discreción del profesor" — cerrar/abrir el QR es esa discreción en acción.
6. **Sobrescritura por marcado manual**: si el profesor marca manualmente a un alumno que **ya se había auto-registrado por QR**, el sistema muestra un **warning** de que ya tiene asistencia registrada, antes de aplicar el cambio (mismo patrón de advertencia que la re-captura del propio alumno).

### Tablero del profesor — navegación y resúmenes
- **Un solo lugar unificado** (no dos secciones separadas): el tablero **es** un calendario (vista mes/semana/día) con selector de materia/hora. Al entrar, aterriza en "hoy" con la **materia+grupo+hora pre-seleccionada automáticamente** según la hora actual + el horario registrado (el horario se reusa aquí como sugerencia inteligente por default, sin dejar de ser "informativo" para efectos de restricción — el profesor puede cambiar la selección libremente).
- **Sin concepto separado de "reabrir sesión pasada"**: navegar a un día anterior es simplemente moverse en el mismo calendario y seleccionar otra materia/hora — no hay una sección aparte de "historial".
- **En un día pasado**: mismo roster + marcado manual + justificar, todo "en cualquier momento" (ya confirmado antes). Lo único que no aplica a un día pasado es la rotación de QR en vivo — no se puede "reabrir" un QR para una fecha ya ocurrida, solo edición manual/justificación. *(Inferido por consistencia con decisiones previas, pendiente de confirmación explícita si no es correcto.)*
- **Resúmenes en el tablero** (los 5 propuestos entran al alcance):
  1. Resumen del día: cuántas de sus sesiones de hoy ya pasó lista, cuántas pendientes.
  2. % de asistencia por grupo.
  3. Ranking de faltas/tardanzas: top alumnos con más inasistencias en sus materias.
  4. Racha de faltas consecutivas por alumno (alerta simple, no motor de alertas complejo).
  5. Justificantes: cuántas faltas están justificadas vs. sin justificar.
- **Ventana de los resúmenes = el mes que el profesor tiene seleccionado en el calendario** (no una ventana fija de 30 días rodantes). Por default, al entrar, es el mes actual; si navega a un mes anterior, los resúmenes se recalculan sobre ese mes. Reusa la navegación de calendario ya definida arriba en vez de un selector de rango aparte. **Distinto del "mes en curso" fijo del historial de alumno/padre** — ahí sí es una ventana fija porque no tienen navegación de calendario, es una vista simple de consulta.

### Login de profesor y dirección
- **Profesor y dirección/admin de plantel usan login tradicional** (usuario/contraseña), a diferencia del mecanismo de matrícula+PIN de alumno/padre. Confirmado por contraste explícito con esa decisión.
- **Las credenciales viven en base de datos, ligadas al alta dinámica de profesor/dirección** (uno por uno o CSV) — **no en el archivo de propiedades**. El archivo de propiedades queda reservado para configuración del sistema (tolerancia default, rotación de QR, intentos fallidos de PIN); mezclar ahí credenciales por entidad rompería la promesa de que dirección puede dar de alta profesores sin depender de un desarrollador editando un archivo y redesplegando.
- **Aprovisionamiento**: al dar de alta a un profesor, **dirección genera una contraseña temporal** y se la entrega por fuera del sistema (no hay flujo de recuperación de contraseña por ahora — pospuesto, no bloquea el alta dinámica).

### Dirección/Admin de plantel — detalle
- **Alta de la primera cuenta de dirección (bootstrap)**: sin super-admin SaaS (pospuesto), no hay nadie en el sistema que pueda crear la primera cuenta de dirección de un plantel. Se siembra **por fuera del sistema, directo en base de datos**, documentado como paso operativo temporal — mismo criterio que las credenciales de profesor viviendo en BD en vez de en el archivo de propiedades.
- **Una sola cuenta de dirección por plantel**, por el momento (no hay director + subdirector + coordinador académico como cuentas separadas todavía).
- **Tablero de dirección**: entra al alcance. Mismo patrón que el tablero del profesor — **calendario navegable (mes/semana/día)**, resúmenes calculados sobre el mes/día seleccionado. A nivel **plantel completo** (todos los grupos, todos los profesores), con 7 resúmenes:
  1. Resumen del día: de todas las sesiones programadas hoy en el plantel, cuántas ya pasaron lista, cuántas pendientes.
  2. % de asistencia por grupo (todos los grupos del plantel).
  3. Ranking de faltas/tardanzas: top alumnos del plantel completo (más fuerte que a nivel profesor — un alumno con faltas repartidas entre varias materias distintas no se ve "crónico" para ningún profesor individual, pero sí agregado a nivel plantel).
  4. Racha de faltas consecutivas por alumno, vista plantel completo.
  5. Justificantes: justificadas vs. sin justificar, plantel completo.
  6. **Profesores que aún no han pasado lista hoy** — lista accionable con nombre (no solo el conteo del punto 1).
  7. **Motivos de justificantes pendientes de aprobar** — conteo/badge de la cola de aprobación del catálogo (ver punto anterior).
  - **Fuera de alcance por ahora, deliberadamente**: tendencia mes a mes del plantel (se siente más a reporte que a tablero operativo), y alertas automáticas de intervención por ausentismo crónico con umbrales/notificaciones (los puntos 3 y 4 ya dan la información sin necesitar un motor de alertas).
- **Reset de contraseña de profesor**: dirección puede **generar una nueva contraseña temporal** para un profesor (mismo mecanismo que el alta inicial) — es el mecanismo interino mientras no exista un flujo de recuperación de contraseña propio.
- **Aprobación del catálogo de justificantes** — mecánica concreta: cuando un profesor escribe un motivo nuevo (texto libre), **se agrega automáticamente al catálogo del plantel con una marca de "pendiente de aprobar"** (la justificación en sí ya aplicó de inmediato al registro del alumno, independientemente de este estado — ver sección "Justificantes"). Dirección puede **aprobar** (queda como motivo normal reusable) o **eliminar** (para descartar entradas basura/de prueba, ej. "motivo prueba 2") desde su tablero.

### Justificantes
- **Quién justifica**: solo el **profesor** de esa materia/sesión, en este momento. La idea de brainstorm de justificante subido por el padre/tutor queda **descartada por ahora** (no pendiente, descartada) — ver nota en la lista de brainstorm.
- **Alcance**: solo se justifican **faltas** (no tardanzas). La fecha/hora del registro es la del sistema.
- **Granularidad**: por **materia/sesión individual**, sin propagación automática entre sesiones del mismo día. Ejemplo discutido: un alumno asiste a las 7am y se retira a las 10am por enfermedad — cada materia posterior se marca como falta y **cada profesor la justifica por separado**, sin que el sistema infiera o propague la justificación entre materias. Decisión deliberada: hay demasiados supuestos posibles (ej. "salió a tomarse la foto de la credencial y volvió") como para modelar propagación automática; se mantiene simple.
- **Aprobación previa del registro**: no aplica — el profesor decide y aplica la justificación directo, sin flujo de aprobación intermedio.
- **Evidencia adjunta**: opcional. El profesor puede o no asociar un justificante/comprobante.
- **Motivo**: **texto libre por default** (la opción más simple). También existe la posibilidad de un **catálogo de motivos**; cuando el profesor usa un motivo nuevo que no está en el catálogo, darlo de alta **requiere aprobación de dirección/admin de plantel** antes de sumarse.
- **Alcance del catálogo**: **por plantel** (tenant-scoped) — cada plantel tiene su propio catálogo de motivos, aunque el mismo motivo pueda repetirse entre planteles distintos (no hay catálogo global compartido).
- **Fecha límite para justificar**: ninguna, de momento.

### Simplificaciones — revisión con la lupa "herramienta simple para el profesor"
El usuario pidió revisar todo el contexto acumulado bajo un principio explícito: **esto es solo una toma de asistencia, debe ser lo más sencillo posible, sin dolores de cabeza para el profesor.** De esa revisión salieron 5 ajustes, todos adoptados:

1. **Identidad ligera en el dispositivo propio del alumno para CONFIRMAR asistencia.** Antes: matrícula+PIN completo en cada escaneo (16 dígitos x 6-7 materias/día x 30-40 alumnos = fricción real que se convierte en interrupciones de clase). Ahora: el PIN completo solo se pide **la primera vez en esa tablet**; de ahí en adelante el dispositivo ya identifica al alumno (token/cookie local, no un login formal ni sesión de usuario tradicional). **CONSULTAR historial desde cualquier otro dispositivo sigue pidiendo matrícula+PIN completo cada vez** — ahí sí hace falta identificar a alguien que no se conoce el dispositivo. Ver secciones "Acceso de alumno y padre" y "Flujo de confirmación de asistencia" arriba, ya actualizadas.
2. **Rotación del QR ajustada de 10-15s a 30-45s** (configurable). Ya se había aceptado el riesgo de fraude por QR reenviado sin mitigación fuerte — subir el intervalo no cambia ese nivel de riesgo aceptado, pero da margen real para completar el flujo sin que expire a medio camino.
3. **Desbloqueo tras agotar los 15 intentos fallidos de PIN**: **el profesor puede desbloquear/resetear el PIN de un alumno**, no solo dirección — el profesor está presente en el salón cuando ocurre el bloqueo, así que resolverlo ahí mismo evita mandar al alumno a buscar a dirección en medio de la clase (la misma interrupción que se busca evitar). Dirección conserva esa misma capacidad como respaldo general.
4. **Separar núcleo mínimo (MVP) de fase 2** al momento de bajar esto a `tasks.md`: el núcleo es lo que el profesor usa todos los días sin fricción (proyectar QR, alumnos confirman, ver quién falta, marcar manual, cerrar registro). Los 5+7 resúmenes, rankings, rachas, y el catálogo de justificantes con aprobación son valiosos pero **no bloquean una primera versión funcional** — se secuencian como fase 2, no se descartan.
5. **El warning de "sobrescribir" en re-captura/doble-escaneo debe reservarse para cuando el estado realmente cambia** (ej. pasar de Tardanza a Presente, o viceversa) — si el alumno reconfirma sin que nada cambie, un mensaje neutro ("ya estabas registrado, sin cambios") es suficiente, sin la fricción de una advertencia grande. Ajuste al flujo ya descrito en "Flujo de confirmación de asistencia" y en el marcado manual del profesor.

## Dimensiones de "robustez" identificadas (estado final)

1. **Anti-fraude** — resuelto como riesgo aceptado. Cerrado.
2. **Resiliencia de red** — fuera de alcance por ahora, descartado explícitamente.
3. **Resiliencia de hardware** — resuelta: fallback manual del profesor cubre al alumno sin tablet; el profesor mismo está cubierto porque el sistema es aplicación web (puede usar cualquier dispositivo con navegador).
4. **Resiliencia humana** — resuelta: no se construyen mecanismos dedicados (ni modo pánico, ni recordatorio automático) por ahora, ambos descartados explícitamente.
5. **Consistencia de datos** — cerrada implícitamente al descartar el Supuesto 3: al haber un solo mecanismo de confirmación ya no hay riesgo de que dos fuentes se contradigan.

## Ideas adicionales (brainstorm, sin decidir cuáles entran al alcance)

- QR grupal de excepción para eventos no ligados a materia (asambleas, simulacros, salidas).
- Feedback inmediato en la tablet del alumno al escanear (sonido/vibración/check visual).
- Señal cruzada con control de acceso general del plantel (torniquete/tarjeta), si existe, para detectar anomalías (alumno "presente" en clase pero nunca entró al plantel).
- ~~Vista del propio alumno de su historial/racha de asistencia~~ — **confirmado** (ver sección "Roles" arriba): el alumno ve su historial de asistencia de días actual y anteriores. La parte de "racha" (streak) sigue sin decidirse, queda como variante de la idea ya confirmada.
- ~~Justificantes subidos por el padre/tutor desde una app, con flujo de aprobación~~ — **descartado por ahora**: se decidió que solo el profesor justifica (ver sección "Justificantes" arriba). Podría revisitarse en el futuro como extensión, no como pendiente activo.
- Alertas automáticas de ausentismo crónico (umbral de faltas/tardanzas en un periodo).
- ~~Dashboard en vivo para dirección: qué salones ya pasaron lista y cuáles no~~ — **confirmado** (ver sección "Dirección/Admin de plantel — detalle" arriba), aunque el detalle fino de qué resúmenes exactos entran no está bajado todavía.
- Reportes periódicos automáticos de asistencia por grupo/materia.
- Auditoría de cambios manuales a registros de asistencia (quién, cuándo, por qué) — cobra más relevancia ahora que el marcado manual del profesor es un mecanismo confirmado, no solo brainstorm.
- Registro del propio maestro (llegó a tiempo a su clase) como subproducto gratis del evento de toma de lista.
- ~~Modelo de datos debe contemplar materias no ligadas a un grupo fijo (talleres/electivas con lista de inscritos variable)~~ — **descartado**: no existe ese caso en el plantel de referencia, todas las materias cuelgan de un grupo fijo.
- Notificación a padres en tiempo real ante tardanza/falta (patrón de evento + plantilla + proveedor de correo, sin compartir infra con el ecosistema de tickets).

## Preguntas abiertas para la próxima sesión

El terreno cubierto (mecanismo, tolerancia, anti-fraude, justificantes, roles/acceso, resiliencias) es suficiente para considerar iniciar un `proposal.md` si se decide formalizar. Posibles hilos nuevos a futuro (no urgentes, no bloqueantes):
- Revisitar resiliencia de red y el modo pánico/recordatorio automático si en la práctica resultan necesarios (quedaron descartados por ahora, no prohibidos permanentemente).
- Revisitar el riesgo de fraude por QR reenviado si en la práctica resulta un problema real (riesgo aceptado, no resuelto de raíz).
- Super-admin SaaS, cuando exista un segundo plantel real.
