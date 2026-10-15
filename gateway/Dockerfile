FROM maven:3.10.0-eclipse-temurin-17-alpine AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:17.0.20.1_1-jre-alpine-3.24
WORKDIR /app
COPY --from=build /app/target/gateway-0.0.1-SNAPSHOT.jar .
EXPOSE 8082
CMD [ "java", "-jar", "gateway-0.0.1-SNAPSHOT.jar" ]