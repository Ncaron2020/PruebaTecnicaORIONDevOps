# --- Etapa 1: Build ---
FROM eclipse-temurin:17-jdk-alpine AS builder

WORKDIR /app

# Cachear dependencias: solo se re-descargan si el wrapper o build.gradle cambian
COPY gradlew gradlew.bat build.gradle settings.gradle ./
COPY gradle ./gradle
RUN chmod +x gradlew
RUN ./gradlew dependencies --no-daemon

COPY src ./src

# -x test: las pruebas ya corren como paso independiente en el pipeline de CI,
# no se repiten aquí para no acoplar el build de la imagen a su resultado.
RUN ./gradlew bootJar --no-daemon -x test

# --- Etapa 2: Runtime ---
# distroless: sin shell, sin package manager, sin herramientas extra.
FROM gcr.io/distroless/java17-debian12:nonroot AS runtime

WORKDIR /app

COPY --from=builder /app/build/libs/*.jar app.jar

# La imagen distroless:nonroot ya corre como usuario no privilegiado 
USER nonroot:nonroot

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
