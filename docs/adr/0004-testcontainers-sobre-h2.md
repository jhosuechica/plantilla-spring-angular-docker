# ADR-0004 · Testcontainers en lugar de H2 para pruebas de integración

- **Estado:** Aceptada
- **Fecha:** 2026-09-01
- **Aplica a:** `backend/src/test/`, `backend/pom.xml`

## Contexto

Las pruebas de repositorio necesitan una base de datos. La opción tradicional es
H2 en memoria: arranca en milisegundos y no requiere nada instalado. El problema
es que H2 no es PostgreSQL.

Diferencias que aparecen en la práctica: tipos como `JSONB` o `UUID`, funciones
de fecha, sintaxis de `LIMIT`/`OFFSET`, `ON CONFLICT`, el comportamiento de las
secuencias y el de las transacciones. Una consulta nativa puede pasar en H2 y
fallar en producción, que es el peor resultado posible: una prueba verde que no
protege de nada.

## Decisión

Usar **Testcontainers** para levantar un PostgreSQL real (la misma imagen que
usa el `docker-compose.yml`) durante las pruebas de integración.

Separación en dos niveles:

| Tipo | Sufijo | Plugin | Docker | Velocidad |
|---|---|---|---|---|
| Unitarias | `*Test.java` | Surefire (`mvn test`) | No | Milisegundos |
| Integración | `*IT.java` | Failsafe (`mvn verify`) | Sí | Segundos |

La separación importa: durante el desarrollo se ejecuta `mvn test` decenas de
veces al día y debe ser instantáneo. El pipeline ejecuta `mvn verify`, que
incluye todo.

Se usa `@ServiceConnection` (Spring Boot 3.1+), que inyecta automáticamente URL,
usuario y contraseña del contenedor sin necesidad de `@DynamicPropertySource`.

## Alternativas consideradas

| Alternativa | Por qué se descartó |
|---|---|
| H2 en modo compatibilidad PostgreSQL | Reduce las diferencias pero no las elimina; sigue sin ser el mismo motor |
| PostgreSQL instalado y compartido para tests | Estado compartido entre ejecuciones; pruebas que dependen del orden y fallan de forma intermitente |
| Servicio `postgres` de GitHub Actions | Solo funciona en el pipeline; en local seguiría sin haber nada |
| Solo pruebas unitarias con mocks | Los mocks no validan SQL. Una `@Query` mal escrita pasaría todas las pruebas |

## Consecuencias

**Positivas**

- Las pruebas se ejecutan contra el mismo motor y la misma versión que
  producción.
- Cada ejecución arranca con un contenedor limpio: sin estado compartido ni
  pruebas dependientes del orden.
- Funciona igual en local y en el pipeline, sin configuración distinta.
- Permite probar funcionalidades específicas de PostgreSQL sin renunciar a la
  cobertura.

**Negativas**

- Requiere Docker corriendo para ejecutar `mvn verify`. Quien no lo tenga
  levantado verá el fallo. Está documentado en el README.
- Las pruebas de integración tardan segundos, no milisegundos. Por eso están
  separadas de las unitarias.
- Primera ejecución más lenta: hay que descargar la imagen de PostgreSQL.

**A vigilar**

- Si el número de pruebas de integración crece, conviene reutilizar un único
  contenedor entre clases (contenedor `static` compartido o el modo *reusable*)
  en lugar de uno por clase.
