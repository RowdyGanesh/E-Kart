# Use lightweight Java runtime image
FROM eclipse-temurin:8-jre

# Set working directory inside container
WORKDIR /app

# Copy Spring Boot jar into container
COPY target/ecart-service-0.0.1-SNAPSHOT.jar app.jar

# Application runs on port 8080
EXPOSE 9090

# Start the application
ENTRYPOINT ["java","-jar","app.jar"]
