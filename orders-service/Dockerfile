# --- Etapa 1: Build ---
FROM golang:1.26.8-alpine AS builder

WORKDIR /app

# Cachear dependencias: solo se re-descargan si go.mod/go.sum cambian
COPY go.mod go.sum ./
RUN go mod download

COPY main.go config.go ./

# CGO_ENABLED=0: binario estático, sin dependencias dinámicas del SO
# -ldflags="-s -w": elimina símbolos de debug, reduce tamaño del binario
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o orders-service .

# --- Etapa 2: Runtime ---
# distroless: sin shell, sin package manager, sin herramientas extra.
FROM gcr.io/distroless/static-debian12:nonroot AS runtime

WORKDIR /app

COPY --from=builder /app/orders-service .

# La imagen distroless:nonroot ya corre como usuario no privilegiado 
USER nonroot:nonroot

EXPOSE 8080

ENTRYPOINT ["/app/orders-service"]
