# ORION Platform Engineering Challenge

## Introducción

Bienvenido a la prueba técnica para el cargo de DevOps & Platform Engineer.
El objetivo de esta evaluación es validar tus capacidades para diseñar, automatizar, desplegar y operar plataformas tecnológicas modernas basadas en contenedores y Kubernetes.
La prueba busca simular una situación real donde un equipo de desarrollo ha construido una solución funcional, pero aún no existe una estrategia de despliegue, automatización, seguridad y operación.
---

# Contexto de Negocio

La plataforma ORION soporta aplicaciones utilizadas en Sistemas Inteligentes de Transporte (ITS).
Actualmente el equipo de desarrollo ha construido una solución basada en microservicios que permite registrar órdenes y procesarlas mediante mensajería asíncrona.
La solución está compuesta por:
- Orders API
- reception-service
- Message Broker (RabbitMQ)
La aplicación actualmente funciona en entorno local de desarrollo.
Sin embargo, la organización necesita preparar la solución para ambientes empresariales utilizando prácticas modernas de DevOps y Platform Engineering.
ver README de cada servicio para mayor entendimiento del mismo. 
---
# Arquitectura Actual

El diagrama original de esta sección omitía Redis y ubicaba a RabbitMQ como si fuera un componente interno de `orders-service`. Se corrige aquí reflejando la arquitectura real del código entregado (ver justificación completa en `BACKLOG_REFINED.md`, HU-001):

```text
                    Cliente
                       │
                       ▼
        ┌──────────────────────────┐
        │      orders-service       │
        │   (Go, :8080) ── Redis    │
        └──────────────┬────────────┘
                       │ publica
                       ▼
                 ┌───────────┐
                 │  RabbitMQ  │  (broker independiente, compartido)
                 └───────────┘
                       │ consume
                       ▼
        ┌──────────────────────────┐
        │    reception-service      │
        │  (Java, :8081) ── Postgres│
        └──────────────────────────┘
```

**Nota sobre nomenclatura:** el backlog original y esta sección se refieren a "Orders Worker" — en el código entregado, ese rol lo cumple `reception-service` (no existe una carpeta `orders-worker/`). Ver "Código Entregado" abajo.

---
# Objetivo
Preparar la plataforma para su despliegue y operación empresarial mediante:
- Contenerización.
- CI/CD.
- Kubernetes.
- Helm.
- Configuración segura.
- Observabilidad.
- Buenas prácticas operativas.
---

# Código Entregado

El repositorio contiene (estructura real, corregida frente a la mencionada originalmente en esta sección):
```text
orders-service/          # Orders API (Go) - HTTP, Redis, publica en RabbitMQ
reception-service/test/  # Consumer (Java/Spring Boot) - consume RabbitMQ, persiste en Postgres
```
No existe una carpeta `orders-worker/` — ese rol lo cumple `reception-service`. Tampoco existe una carpeta `orders-service` bajo la raíz de `reception-service`: el proyecto Gradle completo vive en `reception-service/test/` (nomenclatura heredada del repositorio original, no modificada por no ser parte del alcance).

Ambos servicios son funcionales y pueden ejecutarse localmente.
- La responsabilidad del candidato NO es desarrollar nuevas funcionalidades de negocio.
- La responsabilidad principal es preparar la plataforma para ambientes productivos.

Ver el detalle completo de esta y otras inconsistencias/decisiones en [`BACKLOG_REFINED.md`](./BACKLOG_REFINED.md).

---

# Alcance

El backlog inicial se encuentra documentado en:
```text
BACKLOG.md
```
Antes de iniciar la implementación deberá realizar el ejercicio descrito en:
```text
REFINEMENT.md
```
---
# Tecnologías

Como mínimo se espera el uso de:
- Docker
- Kubernetes
Se recomienda el uso de:
- Helm
- GitHub Actions
- GitLab CI/CD
- Terraform
- Trivy
- SonarQube

Los candidatos podrán utilizar herramientas adicionales o alternativas siempre que justifiquen técnicamente su elección y la solución sea reproducible.
---

# Entregables

La solución debe incluir:

```text
Dockerfile(s)

docker-compose.yml

Pipeline CI/CD

Chart Helm (recomendado)

README.md actualizado

BACKLOG_REFINED.md

RCA.md
```
---
# Ejecución Local

La solución deberá poder ejecutarse localmente utilizando:
```bash
docker compose up
```
Esto levanta los 5 componentes core de la plataforma: `orders-service`, `reception-service`, `rabbitmq`, `redis` y `postgres`.

---

# Herramientas Opcionales

## pgAdmin

Interfaz web para administrar/inspeccionar la base de datos Postgres de `reception-service`. No se levanta con `docker compose up` por defecto (queda fuera del perfil activo), ya que es una herramienta de conveniencia para desarrollo, no un componente core de la plataforma.

Si un desarrollador la necesita, la levanta explícitamente por nombre:
```bash
docker compose up -d pgadmin
```

|Variable|Valor por defecto|
|---|---|
|URL|http://localhost:5050|
|Usuario|admin@admin.com|
|Contraseña|admin|

Para conectarse a Postgres desde pgAdmin, usar como host `postgres` (nombre del servicio en la red interna de Docker Compose), puerto `5432`.

---

# CI/CD

Pipeline principal: [`.github/workflows/ci-cd.yml`](.github/workflows/ci-cd.yml) (GitHub Actions).

- **En Pull Request hacia `main`**: `test-orders-service` (`go vet` + `go test`), `test-reception-service` (`./gradlew test`, con Postgres/RabbitMQ como servicios efímeros del job) y `code-quality` (SonarQube Cloud) corren en paralelo — gate de validación, sin publicar nada.
- **En push/merge a `main`**: adicionalmente corre `build-and-push` — construye ambas imágenes, las escanea con Trivy (falla si hay vulnerabilidades `CRITICAL` con fix disponible) y las publica en GHCR (`ghcr.io/ncaron2020/orders-service`, `ghcr.io/ncaron2020/reception-service`), taggeadas con el SHA del commit y `latest`.
- Los merges a `main` corren en cola (nunca en paralelo ni se cancelan entre sí) — cada uno construye y publica su propia imagen de forma completa.

Análisis de calidad de código, público: [SonarQube Cloud](https://sonarcloud.io/summary/new_code?id=Ncaron2020_PruebaTecnicaORIONDevOps).

También se entrega [`.gitlab-ci.yml`](.gitlab-ci.yml), equivalente, por completitud frente a la mención de "GitLab CI/CD" en la sección de Evaluación — no se pudo validar en ejecución real (sin repositorio GitLab con runners disponible para esta prueba). Ver justificación completa en `BACKLOG_REFINED.md`, HU-002.

---

# Kubernetes

Todo el despliegue se realiza utilizando Helm. Estructura de charts en [`charts/`](./charts):

```text
charts/
├── infra/              # RabbitMQ, Redis, Postgres (Deployment, Service, PVC)
├── orders-service/     # Deployment, Service, ConfigMap, Secret
└── reception-service/  # Deployment, Service, ConfigMap, Secret
```

## Requisitos previos

Un clúster de Kubernetes accesible vía `kubectl`/`helm`. Validado en desarrollo con [`kind`](https://kind.sigs.k8s.io/) local — no es un requisito del proyecto en sí, cualquier clúster real sirve igual.

## Instalación (orden importa: `infra` primero)

```bash
helm install infra ./charts/infra
helm install orders-service ./charts/orders-service
helm install reception-service ./charts/reception-service
```

Para actualizar cualquiera tras un cambio:
```bash
helm upgrade <release> ./charts/<chart>
```

## Configuración por ambiente

`orders-service` y `reception-service` incluyen `values-dev.yaml` (réplica única, recursos livianos, para clústeres de un solo nodo como `kind`). El `values.yaml` base de cada uno queda con configuración apropiada para producción (`replicaCount: 2`):
```bash
helm install orders-service ./charts/orders-service \
  -f charts/orders-service/values.yaml -f charts/orders-service/values-dev.yaml
```

## Qué incluyen los charts

- `Deployment`, `Service`, `ConfigMap`, `Secret` por servicio propio.
- `securityContext` (`runAsNonRoot`, UID explícito) coherente con las imágenes `distroless` de HU-001.
- Probes de `liveness`/`readiness`/`startup` — `httpGet` (imágenes sin shell) para los servicios propios, usando los endpoints de Spring Boot Actuator en `reception-service`.
- `PersistentVolumeClaim` para Postgres y Redis (justificación de por qué Redis también lo necesita en `BACKLOG_REFINED.md`, HU-003).
- `resources.requests/limits` calibrados — el de `reception-service` está directamente ligado al incidente analizado en `RCA.md`.
- `replicaCount: 2` + estrategia `RollingUpdate` explícita (`maxUnavailable: 0`) para cero downtime en actualizaciones.

Todo el proceso de refinamiento, decisiones de diseño, hallazgos encontrados durante el despliegue real (con evidencia de logs) y su corrección están documentados en detalle en [`BACKLOG_REFINED.md`](./BACKLOG_REFINED.md), sección HU-003.

---

# Troubleshooting

Durante la prueba se incluye un escenario de incidente que deberá ser analizado.
El análisis deberá documentarse en:
```text
RCA.md
```

---
# Evaluación

Se evaluarán:
- Docker.
- Kubernetes.
- Helm.
- GitLab CI/CD.
- Seguridad.
- Troubleshooting.
- Observabilidad.
- Documentación.
- Criterio técnico.
---
# Herramientas y Libertad Tecnológica

La evaluación busca validar conocimientos y criterio técnico, no el uso de una herramienta específica.
El repositorio de la prueba será entregado mediante GitHub, sin embargo, el candidato podrá utilizar las herramientas que considere apropiadas para resolver el desafío.
Se valorará especialmente la capacidad de justificar las decisiones tomadas y la aplicación de buenas prácticas de automatización, seguridad, observabilidad y operación.

---
# Consideraciones Finales

No existe una única solución correcta.
Se valorará especialmente la capacidad para justificar decisiones técnicas y operativas.
En caso de asumir comportamientos o configuraciones no especificadas, documente claramente dichos supuestos.
