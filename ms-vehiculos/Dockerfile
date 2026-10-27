
# ETAPA 1: COMPILACION
FROM maven:3.9-eclipse-temurin-21 AS build

WORKDIR /app

# Copiar configuracion de Maven
COPY pom.xml .

# Descargar dependencias
RUN mvn -B dependency:go-offline

# Copiar codigo fuente y pruebas
COPY src ./src

# Ejecutar pruebas y construir JAR
RUN mvn -B clean package

# ETAPA 2: EJECUCION
FROM eclipse-temurin:21-jre

WORKDIR /app

# Crear usuario sin privilegios
RUN groupadd --system appgroup \
    && useradd --system --gid appgroup appuser

# Copiar  el JAR compilado
COPY --from=build /app/target/cl-matiivilla-vehiculos-0.0.1-SNAPSHOT.jar app.jar

USER appuser

EXPOSE 9004

ENTRYPOINT ["java", "-jar", "app.jar"]
