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
| `Dockerfile` `orders-service` (multi-stage, distroless, non-root) | Hecho |
| `Dockerfile` `reception-service` (multi-stage, distroless, non-root) | Hecho |
| `.dockerignore` en ambos servicios | Hecho |
| `docker-compose.yml` unificado en raíz (5 servicios) | Hecho |
| `pgAdmin` como servicio opcional (Compose `profiles`) | Hecho |
| Reubicación de composes parciales originales a `eliminados/` (evidencia, no borrado) | Hecho |
| Validación end-to-end real (Postman + consulta directa a Postgres) | Hecho |
| `README.md`: sección "Ejecución Local" y "Herramientas Opcionales" | Hecho |
| `README.md`: diagrama de arquitectura corregido | Pendiente (se hace al cierre, junto con el resto de documentación) |

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

**Ambigüedades:**
- El README no es consistente respecto a la herramienta de CI/CD: en "Tecnologías" ofrece GitHub Actions y GitLab CI/CD como opciones equivalentes, y en "Herramientas y Libertad Tecnológica" deja explícito que la elección es libre siempre que se justifique — pero en la sección "Evaluación" el bullet dice literalmente `GitLab CI/CD`, sin mencionar GitHub Actions. Mismo patrón de inconsistencia de documentación que el de `orders-worker` (HU-001).

**Supuestos y decisión tomada:**
- Se usa **GitHub Actions** como pipeline principal, funcional y demostrable: el fork ya vive en GitHub, permite usar `GITHUB_TOKEN` automático para autenticar contra GHCR, sin gestionar credenciales adicionales ni infraestructura extra.
- Adicionalmente, se entrega un **`gitlab-ci.yml` equivalente** por completitud frente a la mención explícita en "Evaluación", replicando la misma lógica de jobs (build, test, build de imagen, scan, push). Se documenta honestamente que **no pudo validarse en ejecución real**, al no contar con un repositorio GitLab con runners disponibles para esta prueba — su corrección se basa en la sintaxis y estructura equivalente al workflow de GitHub Actions, que sí está probado y funcionando.

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

**Supuestos y decisión tomada:** se define un chart de Helm por servicio propio (`orders-service`, `reception-service`), más un tercer chart propio (`infra`) para RabbitMQ/Redis/Postgres con manifiestos simples (`Deployment`/`Service`/`PVC`), en vez de traer charts oficiales de terceros (ej. Bitnami). Justificación: el foco de la evaluación son los dos servicios propios (Deployment, Service, ConfigMap, Secret, probes, seguridad) — escribir los manifiestos de infraestructura a mano demuestra ese conocimiento de forma más directa que importar un chart ya armado por alguien más. En un entorno de producción real con alta disponibilidad/backups/replicación de por medio, un Operator o chart maduro (Bitnami, o el oficial de cada proyecto) sería la recomendación — para el alcance de esta prueba, manifiestos propios son apropiados.

**Clúster de prueba:** `kind` (Kubernetes in Docker) — corre como contenedores Docker en vez de una VM completa, arranca en segundos, y reutiliza el Docker Desktop que ya usa todo el proyecto, sin instalar un driver adicional.

### 2. Refinamiento

**Componentes:** `Deployment`, `Service`, `ConfigMap`, `Secret` por servicio propio; probes de `liveness`/`readiness`/`startup` reutilizando los endpoints ya existentes en el código (`/health`, `/actuator/health/liveness`, `/actuator/health/readiness`, `/actuator/health/startup`) — no requiere tocar código de aplicación, ya vienen implementados.

**Procesos de despliegue:** `helm install` / `helm upgrade`.

**Configuración requerida:** `ConfigMap` para valores no sensibles (nombre de cola, puertos); `Secret` para credenciales (RabbitMQ, Postgres, Redis).

**Estrategia operacional:** `resources.requests/limits` calibrados (directamente ligado al incidente de `RCA.md` — OOMKilled por límite insuficiente), `replicaCount >= 2` para disponibilidad, estrategia de `RollingUpdate`.

**Persistencia — PVC no solo en Postgres:** inicialmente se planeó PVC únicamente para Postgres (la base de datos "obvia"). Al analizar el flujo completo, se identificó que **Redis también necesita PVC**: `orders-service` usa Redis como bitácora de todos los eventos recibidos (`RPush` a `events_list`, expuesto vía `GET /api/v1/events`), y mientras un evento espera sin ser consumido en RabbitMQ, existe simultáneamente en Redis y en RabbitMQ — si el pod de Redis se reinicia sin persistencia, esa copia se pierde. Combinado con el hallazgo de abajo (mensajes de RabbitMQ no marcados como persistentes), un reinicio de ambos pods en la ventana entre publicación y consumo podría perder el evento **sin dejar rastro en ningún lado**. Se agrega PVC también a Redis por esta razón.

**Límite de esta solución — falta un ajuste a nivel de desarrollo:** dar persistencia a nivel de infraestructura (PVC en Redis y Postgres) garantiza que los **datos que ya se escribieron a disco** sobrevivan un reinicio de pod. Pero esto por sí solo **no es suficiente** para RabbitMQ: como se documenta abajo, los mensajes se publican sin `DeliveryMode: Persistent`, por lo que RabbitMQ los mantiene solo en memoria sin importar qué tan bien configuremos su almacenamiento a nivel de infraestructura — ese es un ajuste que debe hacerse en el código de `orders-service` (Go), fuera del alcance de esta HU por tratarse de código de aplicación, no de plataforma.

**Precisión importante sobre qué protege realmente la persistencia de Redis:** Redis y RabbitMQ son sistemas completamente independientes, sin ninguna conexión entre sí en el código — no existe ningún mecanismo de reconciliación que detecte "este evento está en Redis pero nunca llegó a `reception-service`" y lo vuelva a publicar en RabbitMQ. Por lo tanto, la persistencia de Redis **preserva evidencia/bitácora para investigación manual posterior** (saber qué se recibió), pero **no recupera automáticamente** un evento que RabbitMQ haya perdido — esa recuperación real solo la da corregir `DeliveryMode: Persistent` en el origen. Ambos ajustes son complementarios, no sustitutos entre sí.

**Consideraciones de seguridad:** `securityContext` con `runAsNonRoot: true` (coherente con las imágenes distroless ya construidas), `readOnlyRootFilesystem` donde sea viable, `capabilities: drop: [ALL]`, `allowPrivilegeEscalation: false`.

### 3. Descomposición Técnica

| Actividad | Estimación |
|---|---|
| Chart Helm `orders-service` (Deployment, Service, ConfigMap, Secret, probes) | M |
| Chart Helm `reception-service` (ídem) | M |
| Chart Helm `infra` (RabbitMQ, Redis, Postgres — Deployment/Service/PVC propios) | M |
| PVC para Postgres y Redis | S |
| `values.yaml` parametrizado por ambiente | S |
| `securityContext` + resource requests/limits | S |
| Probes de liveness/readiness/startup en los charts | S |

### 4. Estimación total

L

### 5. Priorización

**MVP:** Deployment + Service + ConfigMap + Secret + probes para ambos servicios propios. **Opcional:** HPA, Ingress, PodDisruptionBudget.

### 6. Validación real y hallazgos operativos (chart `infra`)

El chart `infra` (Postgres, Redis, RabbitMQ) se desplegó y probó en un clúster local real (`kind`), no solo se escribió a ciegas. Esa validación encontró 4 problemas reales que no eran evidentes solo leyendo el YAML — quedan documentados porque son justo el tipo de detalle operativo que distingue "un manifiesto que parece correcto" de "un manifiesto que realmente funciona":

1. **Postgres en `CrashLoopBackOff` por permisos del volumen.** El PVC recién creado pertenece a `root` por defecto; `fsGroup` da acceso de lectura/escritura por grupo pero no autoriza `chmod` (eso requiere ser el dueño). El propio entrypoint de Postgres intenta corregir permisos de su directorio de datos y fallaba con `Operation not permitted`. **Fix:** un `initContainer` que corre como root una sola vez (`chown -R 999:999`) antes de que arranque el contenedor principal, no-root.

2. **El `initContainer` fue rechazado por Kubernetes.** El pod hereda `runAsNonRoot: true` a todos sus contenedores por defecto, incluidos los `initContainers` — al pedirle `runAsUser: 0` (root) sin también overridear `runAsNonRoot: false` en ese contenedor puntual, Kubernetes lo rechazaba por contradicción ("debe ser no-root" + "usuario root" a la vez). **Fix:** `securityContext` propio en el `initContainer`, con `runAsNonRoot: false` explícito.

3. **RabbitMQ reiniciándose en bucle (falso positivo del liveness probe).** Los logs confirmaron que RabbitMQ arranca correctamente en ~23-25s, pero el `livenessProbe` (con `initialDelaySeconds: 20`) lo evaluaba y mataba justo antes de que terminara de arrancar — un pod sano, muerto por una probe mal calibrada, no por un problema real de la aplicación. **Fix:** `startupProbe` dedicado a la fase de arranque (reintenta varias veces antes de darse por vencido); mientras no pase, `liveness`/`readiness` ni siquiera empiezan a evaluarse. Se aplicó también a Postgres/Redis por el mismo riesgo, aunque no llegaron a fallar en la práctica.

4. **Probes fallando por timeout demasiado corto.** Kubernetes usa `timeoutSeconds: 1` por defecto si no se especifica — `rabbitmq-diagnostics` (corre sobre Erlang) a veces tardaba más de eso, sobre todo con varios pods compitiendo por CPU en el mismo nodo local. **Fix:** `timeoutSeconds: 5` explícito en los probes de los 3 servicios de `infra`.

**Por qué esto importa más allá de "se corrigió un bug":** los hallazgos 3 y 4 son variantes directas de la misma lección del incidente en `RCA.md` — una probe/límite mal calibrado puede matar un proceso sano y generar una falla en cadena que parece "el servicio está roto" cuando en realidad el servicio nunca tuvo un problema real. Reforzó por qué las recomendaciones del RCA (calibrar límites y probes con datos reales, no con valores adivinados) aplican en la práctica, no solo en teoría.

### 7. Chart `orders-service` — decisiones de configuración y secretos

**Por qué `RABBITMQ_URL` completa va en el `Secret`, no en el `ConfigMap`:** `orders-service` espera una sola variable con formato `amqp://usuario:password@host:puerto/` — usuario y contraseña quedan incrustados en la misma cadena. Aunque host/puerto no son sensibles por sí solos, no se pueden separar del resto de la URL sin cambiar el código de la aplicación (fuera de alcance) — por lo tanto, la URL completa se trata como sensible y vive en el `Secret`, no en el `ConfigMap`.

**Duplicación de datos entre `charts/infra` y `charts/orders-service`:** al ser charts de Helm separados (sin un chart "paraguas" que comparta `values.yaml` entre ellos), las credenciales de RabbitMQ (`user`/`password`) están declaradas en **dos lugares** — una vez en `charts/infra/values.yaml` (quien las define, vía `RABBITMQ_DEFAULT_USER`/`PASS`) y otra vez en `charts/orders-service/values.yaml` (quien las consume, deben coincidir manualmente). Es un trade-off real y consciente de la decisión de usar charts separados por servicio en vez de uno compartido — documentado, no oculto.

**Credenciales versionadas en `values.yaml` — decisión de alcance, no un descuido.** Ambos `values.yaml` (`infra` y `orders-service`) llevan un comentario explícito señalando que, en una situación empresarial real con infraestructura ya definida, esto se abordaría distinto:
- **(a)** inyectando el valor en el momento del despliegue (`--set`, o un values file de secretos generado ahí mismo por el pipeline, nunca comiteado), o
- **(b)** el chart no define el `Secret` en absoluto, solo lo **referencia por nombre** (`secretRef`) — el `Secret` se crea por un proceso completamente aparte (manual, o sincronizado desde un gestor de secretos externo como Vault), sin que el desarrollador del chart llegue a ver ni gestionar el valor real.

Un matiz importante discutido: llamar a un recurso `Secret` en Kubernetes **no lo hace automáticamente seguro en git** — el tipo `Secret` solo protege cómo Kubernetes lo almacena/controla el acceso **dentro del clúster** (base64 en `etcd`, RBAC), no de dónde viene el valor. Si ese valor sale de un `values.yaml` versionado, sigue estando expuesto en el historial de git sin importar que el recurso final se llame `Secret`.

### 8. Validación real — charts `orders-service` y `reception-service`

Ambos charts se desplegaron y probaron end-to-end contra el clúster `kind`, incluyendo el flujo completo de negocio (no solo "el pod arrancó"):

- `orders-service`: conectó a Redis y RabbitMQ desde el arranque (sin reintentos ni errores), publicó un evento de prueba real (`POST /api/v1/events`), y se confirmó tanto en `GET /api/v1/events` (Redis) como directamente en la cola de RabbitMQ (`rabbitmqctl list_queues`).
- `reception-service`: consumió automáticamente ese mismo evento en cuanto arrancó (sin que nadie lo disparara manualmente), lo persistió en Postgres, y se confirmó con una consulta SQL directa — reproduciendo exactamente el mismo flujo que ya habíamos validado en local con `docker-compose.yml` en HU-001, ahora sobre Kubernetes real.

**Hallazgo adicional, específico de `reception-service`:** el `README.md` propio de `reception-service` lista `/actuator/health/startup` como endpoint disponible junto a `/liveness` y `/readiness`. Probado directamente (`curl` contra la IP del pod desde dentro del clúster), **`/actuator/health/startup` devuelve `404` real** — no es un problema de timing, el endpoint no existe con la configuración actual de `application.properties` (probablemente requeriría una configuración adicional de Spring Boot Actuator no presente en el código entregado). Como el `startupProbe` del chart dependía de ese endpoint, el pod quedaba permanentemente en `0/1 Ready`, sin llegar nunca a pasar. **Fix:** se cambió el `startupProbe` para usar `/actuator/health/readiness` en su lugar (sí funciona, confirmado con `200`), sin tocar código ni configuración de la aplicación — ajuste contenido enteramente en el chart de Helm.

**Metodología de diagnóstico, para que quede como referencia:** el primer intento de probar los endpoints vía `kubectl port-forward` desde la máquina local dio `000` (sin conexión) para los 4 endpoints por igual — una pista de que el problema no era el endpoint puntual, sino la ruta de red usada para probar. Cambiar a un pod temporal *dentro* del clúster, apuntando primero al `Service` (también falló, porque el `Service` no enruta a pods que aún no están `Ready`) y finalmente directo a la **IP del pod** (bypasseando el `Service`), aisló el problema real: 3 de los 4 endpoints devolvían `200`, solo `/startup` daba `404`. Sin este orden de descarte, hubiera sido fácil confundir un problema de ruta de red con un problema de la aplicación.

### 9. Evidencia de validación end-to-end vía Postman (post-corrección)

Con el `startupProbe` ya corregido y los 5 pods (`orders-service`, `reception-service`, `postgres`, `redis`, `rabbitmq`) en `1/1 Running`, se repitió manualmente desde Postman la misma prueba end-to-end de HU-001 (antes hecha contra `docker-compose`), esta vez contra el clúster de Kubernetes real (vía `kubectl port-forward` a los `Service` de cada uno):

1. `POST http://localhost:8080/api/v1/events` → `202 Accepted`, `{"status": "published & cached", ...}`.
2. `GET http://localhost:8080/api/v1/events` → `200 OK`, `count: 2` — incluye el evento de esta prueba y el de la validación anterior (`k8s-test-01`), confirmando que Redis conserva el historial entre despliegues (PVC funcionando).
3. `GET http://localhost:8081/api/v1/messages` → `200 OK` — ambos eventos aparecen persistidos en Postgres, cada uno con un UUID generado por `reception-service` (reproduce el hallazgo ya documentado: el `id` original no se conserva).

Confirma que el comportamiento del sistema en Kubernetes es idéntico al validado en local con `docker-compose.yml` — la migración de plataforma no introdujo ninguna regresión funcional.

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

### Convención de mensajes de commit

Los primeros commits de la rama `feature/hu-001-containerization` usan un formato de etiqueta libre (`[Categoría]: descripción`, en algunos casos con la HU referenciada aparte, en otros no). A partir de `feature/hu-002-ci-cd` se adoptó el estándar `[HU-XXX-Categoría]: descripción`, para que la trazabilidad entre cada commit y su historia de usuario correspondiente quede explícita y consistente de cara a las historias venideras. No se reescribió el historial de los commits ya realizados para preservar su integridad; el ajuste aplica desde este punto en adelante.

### Qué riesgos se identificaron

- Inconsistencias en la documentación original del repositorio (diagrama de arquitectura incompleto, referencia a una carpeta `orders-worker` inexistente, carpeta `reception-service/test/` mal nombrada).
- **`reception-service` no conserva el `id` original del evento** — genera un UUID nuevo al persistir en Postgres, descartando el `id` enviado por `orders-service`. Detectado durante la validación end-to-end de HU-001.
- La columna `name` en `reception_data` tiene restricción `UNIQUE` — un segundo evento con el mismo texto de `message` fallaría al insertarse. Riesgo funcional a tener en cuenta si el volumen real de eventos incluye mensajes repetidos.
- `go.mod` declarando una versión de Go que podía no estar disponible públicamente al momento del build — mitigado validando la imagen exacta antes de pinearla.
- **`orders-service` publica en RabbitMQ sin `DeliveryMode: Persistent`** (`main.go`, función `publishEvent`) — el mensaje solo vive en memoria del broker, no se escribe a disco, sin importar qué tan robusta sea la configuración de almacenamiento de RabbitMQ a nivel de infraestructura. Si el pod de RabbitMQ se reinicia antes de que `reception-service` consuma el mensaje, se pierde. Detectado durante el análisis de persistencia de HU-003. Requiere un ajuste de código (agregar el flag al `amqp.Publishing`), fuera de alcance de las historias de infraestructura — documentado, no corregido.
- **La lista `events_list` de Redis crece sin límite ni expiración** — `publishEvent` solo hace `RPush`, nunca hay un `LPop`/`LTrim`/`EXPIRE` en ningún lado del código. En un despliegue de larga duración, esto es consumo de memoria no acotado (Redis podría quedarse sin memoria) y hace que `GET /api/v1/events` sea cada vez más pesado, al traer siempre el historial completo. Además, como Redis y RabbitMQ no están conectados entre sí (no hay reconciliación automática), tener el evento registrado en Redis no garantiza que haya sido procesado por `reception-service` — solo sirve como bitácora para investigación manual, no como mecanismo de recuperación. Requiere ajuste de código (TTL, trim periódico, o límite de tamaño), fuera de alcance de las historias de infraestructura.
