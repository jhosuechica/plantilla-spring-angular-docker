# ADR-0002 · PostgreSQL en Compose con volumen nombrado

- **Estado:** Aceptada
- **Fecha:** 2026-09-01
- **Aplica a:** `docker-compose.yml`

## Contexto

Hasta ahora, cada persona que quería trabajar en el proyecto tenía que instalar
PostgreSQL en su máquina, crear la base de datos y el usuario a mano, y ajustar
la configuración local. Eso produce tres problemas:

1. Versiones distintas de PostgreSQL entre desarrolladores.
2. Pasos manuales no documentados que solo conoce quien los hizo.
3. Credenciales terminando en archivos versionados.

## Decisión

Declarar PostgreSQL como un servicio más del `docker-compose.yml`, con:

- **Volumen nombrado** `${PROYECTO}-pgdata` para los datos, separado del ciclo de vida
  del contenedor.
- **Healthcheck** con `pg_isready`, y `condition: service_healthy` en el
  `depends_on` del backend.
- **Puerto 5432 no publicado** al host por defecto: la base de datos solo es
  visible dentro de la red interna del proyecto.
- **Credenciales por variables de entorno** leídas de `.env`, que no se
  versiona; en el repositorio solo va `.env.example`.

## Alternativas consideradas

| Alternativa | Por qué se descartó |
|---|---|
| PostgreSQL instalado en el host | Es exactamente el problema que se quiere resolver |
| Base de datos gestionada (RDS) también en desarrollo | Costo, latencia y dependencia de internet para trabajar |
| Bind mount (`./data:/var/lib/postgresql/data`) en vez de volumen nombrado | Problemas de permisos y de rendimiento en Windows y macOS; ensucia el árbol del repositorio |
| H2 en memoria para desarrollo | Comportamiento distinto al de PostgreSQL; los errores aparecerían solo en producción |

## Consecuencias

**Positivas**

- La misma versión exacta de PostgreSQL para todos, incluido el pipeline.
- `docker compose up -d` sustituye a media página de instrucciones de
  instalación.
- Reset completo del entorno con `docker compose down -v`.
- El backend no arranca contra una base de datos que aún no acepta conexiones.

**Negativas**

- Para conectarse con DBeaver o pgAdmin hay que descomentar el bloque `ports`.
  Es fricción deliberada: el estado por defecto es el seguro.
- Los scripts de `db/init/` se ejecutan solo al crear el volumen por primera
  vez. Quien espere que corran en cada arranque se confundirá.

**A vigilar**

- `db/init/` no es un sistema de migraciones. En cuanto el esquema empiece a
  evolucionar, hay que incorporar Flyway o Liquibase y pasar `DDL_AUTO` a
  `validate` en todos los entornos.
