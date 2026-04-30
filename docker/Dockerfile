# Use lightweight Java runtime
FROM eclipse-temurin:8-jre

# Set working directory
WORKDIR /app

# Copy the built JAR file
COPY target/*.jar app.jar

# Default port (can be overridden at runtime)
ENV SERVER_PORT=9090

# Expose application port
EXPOSE 9090

# Run the application
ENTRYPOINT ["sh","-c","java -jar app.jar --server.port=${SERVER_PORT}"]