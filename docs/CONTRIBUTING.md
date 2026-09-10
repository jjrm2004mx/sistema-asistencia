# Guía de Contribución

¡Gracias por tu interés en contribuir a este proyecto!  
Este documento describe las reglas básicas que debes seguir para colaborar con el equipo de desarrollo.

---

## Ramas base

- **`main`** — rama estable, siempre desplegable. Nunca se commitea directo aquí.
- **`develop`** — rama de integración. Todo el trabajo en curso se integra aquí antes de llegar a `main`. Nunca se commitea directo aquí tampoco.

Todo cambio se hace en una rama propia creada **a partir de `develop`**, y se integra de vuelta vía Pull Request.

---

## 🧱 Requisitos generales 

Antes de contribuir, asegúrate de que:

- Has clonado el repositorio correctamente.
- Estás trabajando sobre la rama correcta (`develop`, no `main`).
- Conoces y aplicas nuestras reglas de nomenclatura para ramas (ver abajo).
- Tus cambios están bien documentados y justificados en los *commits* y/o *pull requests*.

---

## 🪜 Flujo de trabajo sugerido

1. Crea una rama a partir de `develop`.
2. Usa el prefijo correcto para el tipo de trabajo (ver sección siguiente).
3. Realiza tus cambios y haz commits claros y frecuentes.
4. Abre un Pull Request contra `develop`.
5. Espera revisión del equipo. Una vez aprobado, se integrará.

---

## 🌿 Convención para nombres de ramas

Para asegurar una buena organización y facilitar futuras automatizaciones (CI/CD), sigue esta convención:

| Tipo         | Prefijo     | Ejemplo                            |
|--------------|-------------|------------------------------------|
| Funcionalidad nueva | `feature/`  | `feature/login-usuarios`           |
| Corrección de errores | `bugfix/`   | `bugfix/error-carga-imagen`        |
| Corrección urgente en producción | `hotfix/`   | `hotfix/caida-servidor`            |
| Preparación de una versión | `release/`  | `release/1.3.0`                     |
| Mantenimiento / tareas técnicas | `chore/`    | `chore/actualizar-dependencias`    |
| Documentación | `docs/`     | `docs/actualizar-readme`           |
| Experimentación o prototipo | `test/`     | `test/ui-nueva-navbar`             |

---

## Reglas

- **Nunca push directo a `develop` ni a `main`.**
- `git fetch --all --prune` antes de cualquier `checkout`/`pull`, para evitar trabajar sobre referencias obsoletas.
- Un PR por cambio lógico — evita mezclar features no relacionadas en la misma rama.

---

## 🧪 Buenas prácticas en commits

- Usa mensajes de commit claros y en presente:
  - ✔ `Agrega validación para formulario de login`
  - ✖ `Agregado validación...` o `Cambios varios`
- Agrupa cambios relacionados en un mismo commit.
- Evita *commits* automáticos del IDE o sin descripción.

---

## 🤝 Código de conducta

Este proyecto busca mantener un ambiente profesional y colaborativo.  
Por favor, sé respetuoso en los comentarios y revisiones.

---

## ❓¿Dudas?

Si tienes dudas sobre cómo contribuir, contacta con el líder técnico del equipo o abre un issue en GitHub.

¡Gracias por colaborar!
