# Stage 1: Maven build environment
FROM maven:3.9.4-eclipse-temurin-17 AS build-env
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline

# Stage 2: Build Spring Boot application
FROM maven:3.9.4-eclipse-temurin-17 AS build-stage
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 3: JRE runtime
FROM eclipse-temurin:17-jre AS runtime
WORKDIR /app
COPY --from=build-stage /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]