# ADR-0001 · Builds multi-etapa en lugar de imagen única

- **Estado:** Aceptada
- **Fecha:** 2026-09-01
- **Aplica a:** `backend/Dockerfile`, `frontend/Dockerfile`

## Contexto

Ambos servicios necesitan herramientas de compilación que no hacen falta en
tiempo de ejecución:

- El backend se compila con Maven y un JDK completo, pero para ejecutarse solo
  necesita un JRE y el JAR.
- El frontend se compila con Node y el CLI de Angular, pero el resultado son
  archivos estáticos que sirve cualquier servidor web.

Una imagen de una sola etapa arrastra todo eso al servidor.

## Decisión

Usar builds multi-etapa: una etapa `builder` con el toolchain completo y una
etapa final que solo copia el artefacto compilado.

- Backend: `maven:3.9-eclipse-temurin-21` → `eclipse-temurin:21-jre-alpine`
- Frontend: `node:20-alpine` → `nginx:1.27-alpine`

Además, en la etapa de build se copian primero los archivos de dependencias
(`pom.xml`, `package*.json`) y después el código fuente. Docker cachea cada
instrucción: si las dependencias no cambian, esa capa se reutiliza y el build
se salta la descarga completa.

## Alternativas consideradas

| Alternativa | Por qué se descartó |
|---|---|
| Imagen única con Maven/Node | Imagen 3–4 veces más grande; incluye JDK, Maven, caché `.m2` y el código fuente en el servidor |
| Compilar fuera y copiar el JAR con `COPY` | Rompe la reproducibilidad: el resultado depende de la máquina que compiló, no del Dockerfile |
| `distroless` en lugar de Alpine | Aún más pequeña y segura, pero sin shell: depurar dentro del contenedor se vuelve muy incómodo. Se descarta por el momento |

## Consecuencias

**Positivas**

- Imagen final considerablemente más pequeña (ver la seccion *Resultados medibles* del README).
- Menor superficie de ataque: ni JDK, ni Maven, ni código fuente en producción.
- Builds incrementales rápidos gracias al orden de las capas.
- El proceso corre como usuario sin privilegios (`USER spring`).

**Negativas**

- Dockerfile más largo y con más conceptos que explicar.
- Alpine usa musl en vez de glibc; alguna librería nativa poco común podría no
  funcionar. Para una aplicación Spring Boot estándar no es un problema.

**A vigilar**

- Si aparecen bibliotecas nativas con problemas en musl, cambiar la etapa final
  a `eclipse-temurin:21-jre-jammy`.
- Una optimización posterior son los *layered jars* de Spring Boot, que separan
  las dependencias de la aplicación en capas distintas.
