# =============================================================================
# Backend Spring Boot - build multi-etapa
#
# Etapa 1 (builder): compila con Maven + JDK completo.
# Etapa 2 (runtime): solo el JAR sobre un JRE minimo.
#
# Por que multi-etapa: la imagen final no contiene Maven, el JDK, el codigo
# fuente ni el cache de dependencias. Pasa de ~700 MB a ~200 MB y reduce la
# superficie de ataque (menos binarios = menos CVEs).
# =============================================================================

ARG JAVA_VERSION=21

# ---------- Etapa 1: build ----------
FROM maven:3.9-eclipse-temurin-${JAVA_VERSION} AS builder
WORKDIR /build

# Copiamos primero el pom.xml solo para descargar dependencias.
# Si el pom no cambia, Docker reutiliza esta capa y el build baja de
# minutos a segundos. Este orden es la optimizacion mas importante del archivo.
COPY pom.xml .
RUN mvn -B -ntp dependency:go-offline

# Ahora si el codigo fuente (cambia en cada commit)
COPY src ./src
RUN mvn -B -ntp clean package -DskipTests \
    && mv target/*.jar target/app.jar

# ---------- Etapa 2: runtime ----------
FROM eclipse-temurin:${JAVA_VERSION}-jre-alpine

# Usuario sin privilegios: si alguien logra ejecutar codigo dentro del
# contenedor, no lo hace como root.
RUN addgroup -S spring && adduser -S spring -G spring

WORKDIR /app
COPY --from=builder --chown=spring:spring /build/target/app.jar app.jar

USER spring
EXPOSE 8080

# MaxRAMPercentage hace que la JVM respete el limite de memoria del contenedor
# en lugar de leer la RAM total del host.
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:+UseContainerSupport"

# Requiere spring-boot-starter-actuator (ver POM-ADICIONES.md).
# Docker Compose usa este healthcheck para no arrancar el frontend
# hasta que el backend responda de verdad.
HEALTHCHECK --interval=30s --timeout=5s --start-period=60s --retries=3 \
  CMD wget -q --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS -jar /app/app.jar"]
