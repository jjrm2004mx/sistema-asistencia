# Contribuir a sistema-asistencia

Convenciones de trabajo para este repo. Sigue el mismo git flow que el resto del ecosistema (aunque `sistema-asistencia` es un proyecto independiente a nivel de infraestructura, la forma de trabajar con git se mantiene consistente).

## Ramas base

- **`main`** — rama estable, siempre desplegable. Nunca se commitea directo aquí.
- **`develop`** — rama de integración. Todo el trabajo en curso se integra aquí antes de llegar a `main`. Nunca se commitea directo aquí tampoco.

Todo cambio se hace en una rama propia creada **a partir de `develop`**, y se integra de vuelta vía Pull Request.

## Convención de nombres de rama

```
<tipo>/<descripcion-corta-en-guiones>
```

Tipos permitidos:

| Prefijo    | Uso                                          |
|------------|-----------------------------------------------|
| `feature/` | Nueva funcionalidad                           |
| `bugfix/`  | Corrección de un bug                          |
| `chore/`   | Mantenimiento, dependencias, configuración    |
| `docs/`    | Cambios solo de documentación                 |

Reglas:
- Palabras separadas con **guiones** (`-`), no camelCase ni underscores.
- **Sin fechas ni nombres de personas** en el nombre de la rama.
- Descriptiva pero corta — el objetivo, no la implementación. Ej. `feature/qr-rotativo-sesion`, no `feature/agrego-endpoint-nuevo-para-qr`.

Ejemplos:
```
feature/registro-asistencia-qr
bugfix/tolerancia-tardanza-no-aplica
chore/actualiza-dependencias
docs/design-stack-tecnico
```

## Flujo de trabajo

1. Actualiza y parte de `develop`:
   ```bash
   git checkout develop
   git pull origin develop
   git checkout -b <tipo>/<descripcion>
   ```
2. Commitea tus cambios en la rama.
3. Push de la rama y abre un **Pull Request hacia `develop`** (nunca hacia `main`).
4. `main` solo recibe merges desde `develop`, en puntos de release.

## Reglas

- **Nunca push directo a `develop` ni a `main`.**
- `git fetch --all --prune` antes de cualquier `checkout`/`pull`, para evitar trabajar sobre referencias obsoletas.
- Un PR por cambio lógico — evita mezclar features no relacionadas en la misma rama.

## Mensajes de commit

- En español, imperativo, sin punto final (ej. `agrega validacion de tolerancia por profesor`).
- Referencia la capability/spec de OpenSpec cuando aplique (ej. `attendance-session: agrega rotacion de QR`).
