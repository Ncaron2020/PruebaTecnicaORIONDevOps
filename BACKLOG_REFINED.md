# Backlog Refinado — ORION Platform Engineering Challenge

Este documento refina las 8 historias de usuario de `BACKLOG.md`, siguiendo la estructura pedida en `REFINEMENT.md`: análisis, refinamiento, descomposición técnica, estimación y priorización por historia, y una justificación general al final.

**Convención de estimación:** XS (< 1h) · S (1-3h) · M (medio día) · L (1 día) · XL (> 1 día).

---

## HU-001 — Contenerización de la Solución

### 1. Análisis

**Dependencias:** ninguna — es la base de todo lo demás (CI/CD, Kubernetes y Helm dependen de que existan imágenes construibles).

**Riesgos:**
- `orders-service/go.mod` declara `go 1.26.4`, una versión que podía no estar disponible como imagen pública. Mitigado: validado que `golang:1.26.8-alpine` existe y es compatible (Go garantiza compatibilidad hacia adelante dentro de la misma línea mayor).
- Las imágenes `distroless` (sin shell) impiden usar `docker exec -it ... sh` para debugging interactivo. Mitigado: documentado el uso de `kubectl debug` con contenedores efímeros como alternativa en Kubernetes.
- `reception-service` corre bajo Gradle 9.2.1 vía wrapper (`gradlew`); en Windows el bit de ejecución puede perderse al clonar. Mitigado con `RUN chmod +x gradlew` dentro del Dockerfile.

**Ambigüedades:**
- El diagrama de arquitectura del `README.md` raíz (`Orders API → RabbitMQ → Orders Worker`) omite a Redis por completo y ubica a RabbitMQ como si fuera un componente interno de `orders-service`, cuando en realidad es un broker independiente y compartido entre ambos servicios.
- El README raíz menciona una carpeta `orders-worker/` que no existe en el código — el rol de "worker/consumer" lo cumple realmente `reception-service`.
- El proyecto Gradle completo de `reception-service` vive dentro de una subcarpeta llamada literalmente `test/` (`reception-service/test/build.gradle`, `.../test/src/...`), que no es una carpeta de pruebas sino la raíz real del proyecto (`settings.gradle` → `rootProject.name = 'test'`). Nomenclatura confusa heredada del repo original; no se renombró para no alterar la estructura entregada sin ser explícitamente pedido.

**Supuestos:**
- `reception-service` = el "Orders Worker" del diagrama original.
- Arquitectura real confirmada: `Cliente → orders-service (Go, Redis) → RabbitMQ → reception-service (Java, Postgres)`.
- Para el pipeline de esta prueba se usa GHCR (GitHub Container Registry) por no requerir infraestructura adicional. Para un entorno on-premise real, se recomienda **Docker Registry self-hosted** (decisión justificada en la sección de Justificación General).

### 2. Refinamiento

**Componentes involucrados:** `orders-service` (Go/Gin), `reception-service` (Java 17/Spring Boot 3.5.6/Gradle 9.2.1), RabbitMQ, Redis, PostgreSQL 16.

**Procesos de despliegue:** build multi-stage por servicio (build con toolchain completo → runtime mínimo); `docker compose up` para levantar el stack completo en local.

**Configuración requerida:** variables de entorno para hostnames de red internos de Docker (`rabbitmq`, `redis`, `postgres` en vez de `localhost`, que es como cada servicio fue programado por defecto para desarrollo aislado).

**Estrategia operacional:** local vía Compose; producción vía Kubernetes/Helm (HU-003).

**Consideraciones de seguridad:**
- Multi-stage build: la etapa de compilación (con compilador/SDK completo) se descarta, solo el artefacto final viaja a producción.
- Imagen final `distroless` en ambos servicios: sin shell, sin gestor de paquetes, superficie de ataque mínima.
- Usuario no-root (`nonroot`, uid 65532) en ambas imágenes.
- Binario estático en Go (`CGO_ENABLED=0`) — sin dependencias dinámicas del SO.
- Versiones de imágenes base pineadas exactas (`golang:1.26.8-alpine`, no `golang:1.26-alpine`) para builds reproducibles.

### 3. Descomposición Técnica

| Actividad | Estado |
|---|---|
| `Dockerfile` `orders-service` (multi-stage, distroless, non-root) | ✅ Hecho |
| `Dockerfile` `reception-service` (multi-stage, distroless, non-root) | ✅ Hecho |
| `.dockerignore` en ambos servicios | ✅ Hecho |
| `docker-compose.yml` unificado en raíz (5 servicios) | ✅ Hecho |
| `pgAdmin` como servicio opcional (Compose `profiles`) | ✅ Hecho |
| Reubicación de composes parciales originales a `eliminados/` (evidencia, no borrado) | ✅ Hecho |
| Validación end-to-end real (Postman + consulta directa a Postgres) | ✅ Hecho |
| `README.md`: sección "Ejecución Local" y "Herramientas Opcionales" | ✅ Hecho |
| `README.md`: diagrama de arquitectura corregido | ⏳ Pendiente (se hace al cierre, junto con el resto de documentación) |

### 4. Estimación

Dockerfile por servicio: S cada uno · Compose unificado: M · Validación end-to-end: XS · Documentación: XS.

### 5. Priorización

**MVP** — toda la historia, es prerequisito de HU-002 y HU-003.

---

## HU-002 — Automatización CI/CD

### 1. Análisis

**Dependencias:** HU-001 (necesita Dockerfiles funcionales para construir imágenes).

**Riesgos:**
- Exponer credenciales del registry o del clúster si no se usan GitHub Secrets correctamente.
- Pipeline lento si no se cachean dependencias (`go mod`, dependencias de Gradle) entre corridas.
- No hay clúster de Kubernetes real accesible para el candidato — el deploy automatizado dentro del pipeline no puede validarse contra un entorno productivo real.

**Ambigüedades:** el README sugiere GitHub Actions **o** GitLab CI/CD, sin definir cuál usar.

**Supuestos:** se usa **GitHub Actions**, porque el fork ya vive en GitHub y permite usar `GITHUB_TOKEN` automático para autenticar contra GHCR sin gestionar credenciales adicionales.

### 2. Refinamiento

**Componentes:** workflows de GitHub Actions (`.github/workflows/`), uno por servicio o un workflow matriz.

**Procesos de despliegue:**
- En **Pull Request** hacia `main`: build + test + lint (gate de calidad, sin publicar nada).
- En **push/merge a `main`**: build + test + build de imagen + Trivy scan + push a GHCR con tag = SHA del commit (trazabilidad exacta commit ↔ imagen).

**Configuración requerida:** ninguna credencial adicional para GHCR (usa `GITHUB_TOKEN` del propio workflow). Si se automatiza el deploy a un clúster real, se necesitaría un `KUBE_CONFIG` como GitHub Secret.

**Estrategia operacional:** el pipeline es el único mecanismo válido para llevar cambios a producción — no se modifican contenedores en caliente (ver HU-005/RCA.md). Ante un fallo post-deploy, la mitigación es `helm rollback`, no un hotfix manual dentro del pod.

**Consideraciones de seguridad:** variables protegidas vía GitHub Secrets, ningún secreto en el repositorio, escaneo de vulnerabilidades de imagen (Trivy) como gate antes de publicar.

### 3. Descomposición Técnica

| Actividad | Estimación |
|---|---|
| Workflow: `go vet` + `go test` (orders-service) | S |
| Workflow: `./gradlew test` (reception-service) | S |
| Workflow: `docker build` multi-stage ambos servicios | S |
| Trivy scan de ambas imágenes | S |
| Push a GHCR con tag = SHA del commit | S |
| (Opcional) `helm upgrade` automatizado si hay clúster disponible | M |

### 4. Estimación total

M–L

### 5. Priorización

**MVP:** build + test + build de imagen + push a registry. **Opcional:** deploy automatizado (depende de disponibilidad real de un clúster).

---

## HU-003 — Despliegue sobre Kubernetes (Helm)

### 1. Análisis

**Dependencias:** HU-001 (imágenes construibles y publicables).

**Riesgos:** sin clúster real disponible para probar en un entorno gestionado; sin definición de si se requiere exposición externa (Ingress).

**Ambigüedades:** no se especifica si se requiere autoescalado (HPA) ni Ingress.

**Supuestos:** se define un chart de Helm por servicio propio (`orders-service`, `reception-service`). Para RabbitMQ/Redis/Postgres se evalúan charts oficiales (ej. Bitnami) en vez de reinventar manifiestos — decisión a justificar según tiempo disponible.

### 2. Refinamiento

**Componentes:** `Deployment`, `Service`, `ConfigMap`, `Secret` por servicio propio; probes de `liveness`/`readiness`/`startup` reutilizando los endpoints ya existentes en el código (`/health`, `/actuator/health/liveness`, `/actuator/health/readiness`, `/actuator/health/startup`) — no requiere tocar código de aplicación, ya vienen implementados.

**Procesos de despliegue:** `helm install` / `helm upgrade`.

**Configuración requerida:** `ConfigMap` para valores no sensibles (nombre de cola, puertos); `Secret` para credenciales (RabbitMQ, Postgres, Redis).

**Estrategia operacional:** `resources.requests/limits` calibrados (directamente ligado al incidente de `RCA.md` — OOMKilled por límite insuficiente), `replicaCount >= 2` para disponibilidad, estrategia de `RollingUpdate`.

**Consideraciones de seguridad:** `securityContext` con `runAsNonRoot: true` (coherente con las imágenes distroless ya construidas), `readOnlyRootFilesystem` donde sea viable, `capabilities: drop: [ALL]`, `allowPrivilegeEscalation: false`.

### 3. Descomposición Técnica

| Actividad | Estimación |
|---|---|
| Chart Helm `orders-service` (Deployment, Service, ConfigMap, Secret, probes) | M |
| Chart Helm `reception-service` (ídem) | M |
| `values.yaml` parametrizado por ambiente | S |
| `securityContext` + resource requests/limits | S |
| Decisión y documentación: infraestructura (RabbitMQ/Redis/Postgres) vía Helm chart oficial vs. gestionado externo | S |
| Probes de liveness/readiness/startup en los charts | S |

### 4. Estimación total

L

### 5. Priorización

**MVP:** Deployment + Service + ConfigMap + Secret + probes para ambos servicios propios. **Opcional:** HPA, Ingress, PodDisruptionBudget.

---

## HU-004 — Configuración y Gestión Segura

### 1. Análisis

**Dependencias:** HU-003.

**Riesgos:** comitear por error un `values.yaml` con secretos reales en texto plano.

**Ambigüedades:** no se especifica un gestor de secretos externo (Vault, Sealed Secrets, SOPS).

**Supuestos:** para el alcance de esta prueba, `Secret` nativo de Kubernetes es suficiente (aunque solo esté codificado en base64, no cifrado en reposo por defecto). Se documenta como mejora futura el uso de un secret manager externo en producción real.

### 2. Refinamiento

**Componentes:** `ConfigMap` (configuración no sensible), `Secret` (credenciales), `values-<ambiente>.yaml` para separar configuración por ambiente sin duplicar templates.

**Configuración requerida:** las variables ya identificadas en HU-001 (`RABBITMQ_URL`/`RABBITMQ_HOST`, `DB_*`, `REDIS_*`).

**Consideraciones de seguridad:** ningún secreto en el repositorio (ya se usa `.env.template` en vez de `.env` real en `orders-service`), `Secret` de Kubernetes en vez de variables planas embebidas en el `Deployment`.

### 3. Descomposición Técnica

| Actividad | Estimación |
|---|---|
| `ConfigMap` con configuración no sensible | S |
| `Secret` con credenciales | S |
| `values.yaml` separado por ambiente | S |
| Documentar política de no comitear secretos | XS |

### 4. Estimación total

S

### 5. Priorización

**MVP.**

---

## HU-005 — Resiliencia Operativa

### 1. Análisis

**Dependencias:** HU-003.

**Riesgos:** el escenario de incidente descrito en `REFINEMENT.md` (OOMKilled, cola acumulada) es evidencia directa de qué pasa cuando los límites de recursos no están bien calibrados — ver `RCA.md`.

**Ambigüedades:** no se especifica el volumen de tráfico esperado, necesario para dimensionar límites y réplicas con precisión.

**Supuestos:** se dimensiona de forma conservadora y se deja configurable vía `values.yaml`, documentando explícitamente que son valores de partida, no una capacidad validada con carga real.

### 2. Refinamiento

**Componentes:** probes ya definidos en HU-003, `resources.requests/limits`, `replicaCount >= 2`, `restartPolicy: Always` (comportamiento por defecto de Kubernetes).

**Estrategia operacional:** ante un incidente en producción, la mitigación es `helm rollback` a la última revisión estable — nunca modificar un pod en caliente (rompe trazabilidad e infraestructura inmutable).

### 3. Descomposición Técnica

| Actividad | Estimación |
|---|---|
| Calibrar `resources.requests/limits` (ligado al RCA) | S |
| `replicaCount >= 2` en ambos servicios | XS |
| Ajustar thresholds/timeouts de los probes | S |
| (Opcional) HPA basado en CPU/memoria | M |
| (Opcional) `PodDisruptionBudget` | XS |

### 4. Estimación total

S–M

### 5. Priorización

**MVP:** límites calibrados + réplicas + probes. **Opcional:** HPA, PDB.

---

## HU-006 — Análisis de Incidentes

Cubierta en detalle en [`RCA.md`](RCA.md) (documento separado, según lo pedido explícitamente en `REFINEMENT.md`).

---

## HU-007 — Seguridad de Plataforma (Opcional)

### 1. Análisis

**Dependencias:** HU-001, HU-003.

**Riesgos:** sin esto, el tráfico entre pods queda sin restricción (cualquier pod del clúster podría hablarle a RabbitMQ/Postgres si comparten namespace).

### 2. Refinamiento

Gran parte de las recomendaciones de seguridad de `REFINEMENT.md` (multi-stage, usuario no-root, imágenes livianas) ya quedaron cubiertas en HU-001/HU-003 — esta historia cubre lo que falta a nivel de red y runtime del clúster.

### 3. Descomposición Técnica

| Actividad | Estimación |
|---|---|
| `NetworkPolicy` básica (solo permitir tráfico necesario entre servicios) | M |
| Revisión de `capabilities` del contenedor (`drop: [ALL]`) | S |
| (Fuera de alcance, mejora futura) mTLS / service mesh | XL |

### 4. Estimación total

M

### 5. Priorización

**Opcional**, tal como está marcada en `BACKLOG.md`.

---

## HU-008 — Observabilidad (Opcional)

### 1. Análisis

**Dependencias:** HU-003.

**Riesgos:** sin observabilidad, un incidente como el descrito en el escenario (cola de 12.500 mensajes acumulándose) sería mucho más difícil de detectar a tiempo — se hubiera notado solo cuando ya era crítico.

**Ambigüedad importante y decisión tomada:** instrumentar métricas de negocio dentro de `orders-service` (Go) requeriría **modificar código de la aplicación** (agregar un exportador de métricas). El README raíz indica explícitamente que *"la responsabilidad del candidato NO es desarrollar nuevas funcionalidades de negocio"*. Se interpreta que instrumentación de métricas de aplicación cae en ese límite, por lo que **no se modifica el código fuente de los servicios**. En su lugar, esta historia se resuelve a nivel de plataforma: observabilidad de infraestructura (recursos, salud de pods) sin tocar la lógica de negocio, aprovechando los endpoints de Actuator que `reception-service` ya expone de fábrica.

### 2. Refinamiento

**Componentes recomendados:** stack Prometheus + Grafana (o alternativa liviana) para métricas de infraestructura del clúster; alertas sugeridas, por ejemplo sobre profundidad de cola en RabbitMQ (directamente relacionado con el incidente del RCA).

### 3. Descomposición Técnica

| Actividad | Estimación |
|---|---|
| Documentar estrategia de observabilidad recomendada | S |
| (Opcional, si el tiempo alcanza) desplegar `kube-prometheus-stack` vía Helm | L |

### 4. Estimación total

S (documentación) – L (implementación completa)

### 5. Priorización

**Opcional / mejora futura** — dado el plazo de 2 días de la prueba, se prioriza documentar la estrategia sobre desplegar el stack completo.

---

## Justificación General

### Qué actividades se agregaron (no pedidas explícitamente, pero necesarias)

- Validación end-to-end real del flujo de mensajería (POST vía Postman → verificación en Redis → verificación en `reception-service` → consulta directa en Postgres), no solo "que los contenedores levanten".
- Carpeta `eliminados/` con README explicando por qué los `docker-compose` parciales originales no se borraron, sino que se preservaron como evidencia junto con el historial de `git mv`.
- Documentación de dos hallazgos de código encontrados durante la validación (ver más abajo).

### Qué actividades se dejaron fuera de alcance (y por qué)

- Gestor de secretos externo (Vault, Sealed Secrets, SOPS): `Secret` nativo de Kubernetes es suficiente para el alcance de esta prueba; se documenta como mejora futura en HU-004.
- mTLS / service mesh: fuera de alcance por tiempo y complejidad, mencionado como mejora futura en HU-007.
- Instrumentación de métricas dentro del código de `orders-service`/`reception-service`: el README indica explícitamente que no es responsabilidad del candidato desarrollar funcionalidad de negocio nueva; HU-008 se resuelve a nivel de plataforma en su lugar.

### Qué herramientas se eligieron y por qué

- **GitHub Actions** para CI/CD: el fork ya vive en GitHub, se integra sin infraestructura ni credenciales adicionales.
- **GHCR** como registry para el pipeline de esta prueba (sin infraestructura que levantar). Para un entorno on-premise real se recomienda **Docker Registry self-hosted**, por simplicidad operativa y por ser la herramienta con la que el equipo tiene mayor familiaridad — compensando la falta de escaneo de vulnerabilidades integrado (que sí tiene Harbor) con Trivy corriendo como paso independiente en el pipeline.
- **Imágenes `distroless` + usuario no-root** en ambos servicios: postura de seguridad consistente entre `orders-service` y `reception-service`, no solo en uno de los dos.
- **GitHub Flow (sin rama `develop`)**: no existe necesidad real de múltiples ambientes de integración en el alcance de esta prueba; se prioriza un flujo simple (`feature → Pull Request → main`) sobre replicar Git Flow sin una razón concreta que lo justifique.

### Qué riesgos se identificaron

- Inconsistencias en la documentación original del repositorio (diagrama de arquitectura incompleto, referencia a una carpeta `orders-worker` inexistente, carpeta `reception-service/test/` mal nombrada).
- **`reception-service` no conserva el `id` original del evento** — genera un UUID nuevo al persistir en Postgres, descartando el `id` enviado por `orders-service`. Detectado durante la validación end-to-end de HU-001.
- La columna `name` en `reception_data` tiene restricción `UNIQUE` — un segundo evento con el mismo texto de `message` fallaría al insertarse. Riesgo funcional a tener en cuenta si el volumen real de eventos incluye mensajes repetidos.
- `go.mod` declarando una versión de Go que podía no estar disponible públicamente al momento del build — mitigado validando la imagen exacta antes de pinearla.
