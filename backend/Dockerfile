FROM eclipse-temurin:21-jre-jammy

WORKDIR /app

RUN groupadd --system spring && useradd --system --gid spring spring

COPY build/libs/*.jar app.jar

USER spring:spring

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
