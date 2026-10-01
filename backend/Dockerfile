# syntax=docker/dockerfile:1

########################################
# Build stage — compile and repackage the Spring Boot fat jar
########################################
FROM --platform=linux/amd64 eclipse-temurin:21-jdk-jammy AS build

WORKDIR /build

# Warm the dependency cache before copying sources so code-only changes
# don't re-download the world.
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN chmod +x mvnw && ./mvnw -B -q dependency:go-offline

COPY src/ src/
RUN ./mvnw -B package -DskipTests

########################################
# Runtime stage — JRE only, no build tooling
########################################
FROM --platform=linux/amd64 eclipse-temurin:21-jre-jammy

WORKDIR /app

COPY --from=build /build/target/backend-0.0.1-SNAPSHOT.jar /app/app.jar

# Pod runs with readOnlyRootFilesystem: true; /tmp is the only writable
# path (emptyDir), so pin the JVM temp dir there. HOME must also point
# somewhere writable for the JVM's user prefs.
ENV JAVA_TOOL_OPTIONS="-Djava.io.tmpdir=/tmp -XX:MaxRAMPercentage=75.0" \
    HOME=/tmp

# Runtime secrets (DB_HOST, DB_PORT, DB_NAME, POSTGRES_USER, POSTGRES_PASSWORD,
# GITHUB_CLIENT_ID/SECRET, GOOGLE_CLIENT_ID/SECRET) are injected via the pod
# spec — never baked into the image.

# Numeric uid so Kubernetes runAsNonRoot can verify the user.
USER 1000

EXPOSE 8080

# Base image has no curl/wget; use bash's /dev/tcp for a TCP liveness probe
# against whatever port the app was told to bind.
HEALTHCHECK --interval=30s --timeout=5s --start-period=60s --retries=3 \
  CMD ["bash", "-c", "exec 3<>/dev/tcp/127.0.0.1/${PORT:-8080}"]

# sh -c + exec so ${PORT} expands and java stays PID 1 for signal handling.
# Spring Boot reads --server.port; never hardcode it.
CMD ["sh", "-c", "exec java -jar /app/app.jar --server.port=${PORT:-8080}"]
