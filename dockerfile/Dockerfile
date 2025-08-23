FROM openjdk:11-jre-slim

WORKDIR /app

# Install curl for health checks
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# Copy all available JAR files
COPY *.jar /app/

# Default to countries service
ENV JAR_FILE="countries-assembly-1.0.1.jar"
ENV SERVICE_NAME="countries-service"
ENV SERVICE_PORT="8080"

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:${SERVICE_PORT}/health || exit 1

# Set Java options
ENV JAVA_OPTS="-Xmx512m -Xms256m"

# Create entrypoint script
RUN echo '#!/bin/bash\n\
echo "Starting service: $SERVICE_NAME"\n\
echo "Using JAR: $JAR_FILE"\n\
echo "Port: $SERVICE_PORT"\n\
echo "Available JAR files:"\n\
ls -la /app/*.jar\n\
echo "Starting application..."\n\
java $JAVA_OPTS -jar /app/$JAR_FILE --server.port=$SERVICE_PORT\n\
' > /app/entrypoint.sh && chmod +x /app/entrypoint.sh

# Run application via entrypoint
CMD ["/app/entrypoint.sh"]