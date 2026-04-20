FROM eclipse-temurin:17-jre-jammy
WORKDIR /app
COPY target/payment-container-1.0-SNAPSHOT.jar app.jar
EXPOSE 8182
ENTRYPOINT ["java", "-jar", "app.jar"]