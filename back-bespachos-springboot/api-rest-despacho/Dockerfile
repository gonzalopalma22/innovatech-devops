# Etapa 1: Build
FROM maven:3.9.6-eclipse-temurin-17 AS build

WORKDIR /app

COPY pom.xml .
RUN mvn dependency:go-offline -q

COPY src ./src
RUN mvn clean package -DskipTests -q

# Etapa 2: Runtime
FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=build /app/target/Springboot-API-REST-DESPACHO-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 8081

ENV DB_ENDPOINT="" \
    DB_PORT="3306" \
    DB_NAME="" \
    DB_USERNAME="" \
    DB_PASSWORD=""

ENTRYPOINT ["java", "-jar", "app.jar"]