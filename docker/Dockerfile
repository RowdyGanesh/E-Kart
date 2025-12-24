# Use lightweight Java runtime image
FROM openjdk:8-jre-slim

# Set working directory inside container
WORKDIR /app

# Copy Spring Boot jar into container
COPY target/shopping-cart-0.0.1-SNAPSHOT.jar app.jar

# Application runs on port 8080
EXPOSE 9090

# Start the application
ENTRYPOINT ["java","-jar","app.jar"]
