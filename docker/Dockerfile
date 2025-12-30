# Use lightweight Java runtime image
FROM eclipse-temurin:8-jre

# Set working directory inside container
WORKDIR /app

# Copy Spring Boot jar into container
COPY target/ecart-service-0.0.1-SNAPSHOT.jar app.jar

# Make Spring Boot listen on 9090
ENV SERVER_PORT=9090

# Expose the correct port
EXPOSE 9090

# Start the application with correct port + health enabled
ENTRYPOINT ["java","-jar","app.jar","--server.port=9090","--management.endpoints.web.exposure.include=*","--management.server.port=9090"]