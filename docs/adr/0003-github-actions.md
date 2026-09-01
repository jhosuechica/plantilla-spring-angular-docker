# ADR-0003 · GitHub Actions como plataforma de CI/CD

- **Estado:** Aceptada
- **Fecha:** 2026-09-01
- **Aplica a:** `.github/workflows/ci.yml`

## Contexto

El proyecto necesita que cada cambio se compile, se pruebe y se empaquete
automáticamente, sin depender de que alguien recuerde ejecutar los comandos en
su máquina. El código ya vive en GitHub.

## Decisión

Usar GitHub Actions con cuatro jobs:

| Job | Disparador | Función |
|---|---|---|
| `backend` | siempre | `mvn verify`: unitarias, integración, cobertura y umbral |
| `frontend` | siempre | `npm ci`, lint, tests, build de producción |
| `smoke` | tras los dos anteriores | Levanta el stack con Compose y verifica por HTTP que responde |
| `publish` | solo push a `main` | Construye y publica las imágenes en Docker Hub |

Decisiones internas relevantes:

- **`backend` y `frontend` corren en paralelo.** Son independientes; encadenarlos
  solo alargaría el pipeline.
- **`smoke` existe deliberadamente.** Sin él, el pipeline solo demostraría que
  las imágenes se construyen, no que el sistema arranca. Un error de
  configuración de red o de variables de entorno no aparece en un `docker build`.
- **Etiquetado doble:** cada imagen se publica como `latest` y como
  `<sha-del-commit>`. `latest` es mutable y sirve para el flujo habitual; el
  tag con SHA permite volver a una versión exacta.
- **`concurrency` con `cancel-in-progress`.** Si llegan dos commits seguidos a
  la misma rama, se cancela el pipeline viejo. Ahorra minutos de runner.
- **Credenciales en Secrets.** El usuario de Docker Hub va como *Variable*
  (no es secreto) y el token como *Secret*. Nunca la contraseña de la cuenta.

## Alternativas consideradas

| Alternativa | Por qué se descartó |
|---|---|
| Jenkins | Requiere un servidor propio, mantenerlo, actualizarlo y asegurarlo. Desproporcionado para este proyecto |
| GitLab CI | Excelente, pero implicaría migrar el repositorio |
| Sin CI, builds locales | El problema que se quiere resolver |
| Publicar en GHCR en vez de Docker Hub | Válido y hasta más integrado, pero Docker Hub es más reconocible en un portafolio y no requiere autenticación para descargar |

## Consecuencias

**Positivas**

- Cada Pull Request se valida solo; nada llega a `main` sin pasar las pruebas.
- El umbral de cobertura se vuelve obligatorio, no una sugerencia.
- Las imágenes publicadas son reproducibles y trazables al commit exacto.
- El badge del README muestra el estado real del proyecto.

**Negativas**

- Acoplamiento a GitHub. Migrar a otra plataforma implicaría reescribir el
  workflow (aunque los Dockerfiles y el Compose son portables).
- El job `smoke` alarga el pipeline: levanta el stack completo en cada
  ejecución.
- Los minutos de runner son limitados en el plan gratuito para repositorios
  privados. En repositorios públicos son ilimitados.

**A vigilar**

- Si el pipeline empieza a tardar demasiado, `smoke` puede restringirse a los
  PR hacia `main` en vez de a todas las ramas.
- Falta el paso de despliegue continuo al servidor. Es la evolución natural del
  workflow.
