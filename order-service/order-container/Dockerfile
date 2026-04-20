FROM eclipse-temurin:17-jre-jammy
WORKDIR /app
COPY target/order-container-1.0-SNAPSHOT.jar app.jar
EXPOSE 8181
ENTRYPOINT ["java", "-jar", "app.jar"]