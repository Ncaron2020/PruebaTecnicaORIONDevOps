# RCA — Incidente de Cola Acumulada y Reinicios por OOMKilled

## Datos del incidente (según lo reportado)

```text
orders-service
- readiness probe failed
- connection refused rabbitmq:5672

orders-worker (reception-service)
- restart count: 14
- reason: OOMKilled

rabbitmq
- queue orders_ready: 12500 mensajes pendientes
- consumers: 1

kubernetes
- worker limit memory: 128Mi
- worker usage: 127Mi antes del reinicio
```

---

## 1. Diagnóstico Inicial

Organizando únicamente lo que los datos confirman, sin sacar conclusiones todavía:

- **`orders-worker` (`reception-service`)** tiene un límite de memoria de `128Mi` y llegó a `127Mi` de uso justo antes de cada reinicio — está prácticamente pegado al límite, sin margen.
- El `restart count: 14` indica que esto **no es un evento aislado**, es un patrón recurrente: el pod entra en un ciclo de crash-restart por la misma causa (`OOMKilled`).
- La cola `orders_ready` de RabbitMQ acumula `12.500 mensajes pendientes` con solo `1 consumer` activo — una acumulación masiva, señal de que el consumo está muy por debajo de la producción durante un periodo sostenido.
- `orders-service` reporta `readiness probe failed` con `connection refused rabbitmq:5672` — no logra establecer conexión con RabbitMQ. "Connection refused" es distinto de "timeout" o "lento": indica un rechazo activo de la conexión, no solo latencia.

**Dato que NO tenemos** (importante dejarlo explícito, no asumirlo): el escenario no reporta límite ni uso de memoria de `orders-service` ni de RabbitMQ mismo — solo del worker. Cualquier conclusión sobre el estado de memoria de esos dos componentes es hipótesis, no dato confirmado.

---

## 2. Hipótesis

**H1 — Límite de memoria insuficiente para el worker (hipótesis principal):**
El límite de `128Mi` es insuficiente para la carga real de `reception-service` (una aplicación Spring Boot/JVM, con overhead propio de la JVM más el procesamiento de mensajes). Al llegar sistemáticamente a ese límite, el kernel de Linux termina el proceso (`OOMKilled`). El pod se reinicia, pero vuelve a repetir el mismo patrón — de ahí el `restart count: 14`.

**H2 — La cola no se vacía porque el consumidor está inestable:**
Como consecuencia directa de H1: si el worker se reinicia constantemente, nunca llega a consumir mensajes de forma sostenida. Con la tasa de publicación de `orders-service` sin cambios, y el consumo colapsado, la cola crece sin control hasta los 12.500 mensajes reportados.

**H3 — RabbitMQ se protege activamente al saturarse de mensajes sin confirmar (explica el síntoma de `orders-service`):**
RabbitMQ tiene un mecanismo propio de protección, el *memory high watermark* (por defecto, ~40% de la memoria disponible del contenedor): al retener miles de mensajes sin consumir, su propio uso de memoria crece, y al cruzar ese umbral, RabbitMQ **bloquea/rechaza activamente nuevas conexiones de publicadores** para protegerse de quedarse sin memoria. Esto explicaría por qué `orders-service` recibe `connection refused` — no sería una falla de RabbitMQ, sino una respuesta de auto-protección deliberada.
*Pendiente de confirmar con métricas de memoria de RabbitMQ y sus logs (evento `memory alarm`), que no están disponibles en los datos de este escenario.*

**H4 — Hipótesis alternativa a descartar:**
Que el `connection refused` de `orders-service` sea un problema de red/DNS/configuración independiente del incidente del worker (coincidencia temporal, no relación causal). Se considera menos probable porque no explicaría por qué ambos síntomas ocurren en la misma ventana operativa, pero se documenta como hipótesis alternativa a validar/descartar con evidencia adicional (logs de red, eventos de Kubernetes del pod de RabbitMQ).

---

## 3. Causa Raíz (más probable, con la evidencia disponible)

Una **falla en cadena originada en un único punto**: el límite de memoria mal calibrado del worker (`reception-service`).

```
Límite de memoria insuficiente (128Mi) en reception-service
        │
        ▼
Ciclo de OOMKilled / restart en el worker (H1)
        │
        ▼
Consumo de la cola colapsado (consumers: 1, inestable) (H2)
        │
        ▼
Cola orders_ready acumula 12.500 mensajes sin consumir
        │
        ▼
RabbitMQ cruza su umbral de memoria y bloquea conexiones (H3)
        │
        ▼
orders-service recibe "connection refused" y falla su readiness probe
```

Un único ajuste mal calibrado (el límite de memoria del worker) se propaga en cascada hasta afectar a un componente que, en principio, no tenía ningún problema propio (`orders-service`).

---

## 4. Acciones de Mitigación Inmediata

Plan de acción para estabilizar el sistema mientras se investiga a fondo, priorizado en el orden en que debería ejecutarse:

**M1 — Aumentar temporalmente el límite de memoria del worker.**
Vía `kubectl`/`helm upgrade --set` (nunca editando el pod en caliente) — esto rompe el ciclo de `OOMKilled`, dejando que el pod se mantenga vivo el tiempo suficiente para empezar a consumir mensajes. Es la acción de mayor prioridad porque ataca directamente el origen de la cadena causal.

**M2 — Verificar el estado real de memoria de RabbitMQ.**
Con `rabbitmqctl status` o la UI de administración, confirmar si efectivamente cruzó su *memory high watermark* (validar H3, no darla por sentada).

**M3 — Monitorear que el worker empiece a drenar la cola.**
Una vez estable (sin más `OOMKilled`), confirmar que `consumers` vuelve a ser estable (no fluctuando por reinicios) y que `orders_ready` empieza a **bajar**, no solo a dejar de crecer.

**M4 — Evaluar escalar réplicas del worker temporalmente.**
Solo después de M1: escalar réplicas con un límite de memoria insuficiente solo multiplica el problema, no lo resuelve. Con el límite ya corregido, más réplicas ayudan a drenar el backlog más rápido.

**M5 — `orders-service` se recupera solo, sin intervención directa.**
Como su `connection refused` es consecuencia de que RabbitMQ está bajo presión (H3), en cuanto la cola empiece a bajar y RabbitMQ salga de su umbral de protección, `orders-service` debería volver a conectar y pasar su readiness probe sin necesidad de tocar nada de él directamente.

## 5. Acciones Preventivas

> Nota: estas acciones se implementan de forma concreta en las historias de usuario correspondientes (HU-003 Kubernetes/Helm, HU-005 Resiliencia Operativa), no quedan solo documentadas aquí. Cada una se comitea en su propia historia a medida que se avanza.

**Calibrar correctamente los `resources.requests/limits` del worker (HU-003/HU-005).**
`128Mi` fue insuficiente para una aplicación Spring Boot/JVM bajo carga. El valor definitivo debe salir de medir el consumo real (heap de la JVM + overhead del proceso), no de un número arbitrario, y dejarse con margen razonable sobre el uso observado.

**Configurar correctamente los probes de liveness/readiness/startup (HU-003).**
`reception-service` ya expone `/actuator/health/liveness`, `/readiness` y `/startup` — hay que usarlos con umbrales/tiempos de espera apropiados, en particular el `startup probe`, para no matar el pod durante el arranque normal de la JVM (que es más lento que un binario nativo como el de `orders-service`).

**Revisar el `prefetch count` del consumidor RabbitMQ en `reception-service` (HU-003/HU-005).**
Si el consumidor precarga más mensajes de los que puede procesar de forma segura dentro del límite de memoria asignado, contribuye directamente a saturar la memoria del pod. Ajustarlo a un valor acorde al límite de memoria definido.

**Réplicas mínimas >= 2 para el worker (HU-005).**
Así un solo pod en problemas no deja el consumo de la cola completamente en cero, como ocurrió en este incidente (`consumers: 1`).

**Manejo de fallos al publicar en `orders-service` (hallazgo de código, no cubierto por ninguna HU actual — se documenta como riesgo).**
Revisando `orders-service/main.go`, `publishEvent` guarda primero en Redis y **después** intenta publicar en RabbitMQ, sin reintento (`retry`) si la publicación falla. Si RabbitMQ rechaza la conexión como en este incidente (H3), el cliente recibe un error 500 y el evento **no llega a `reception-service`** — solo queda en la caché de Redis. Se recomienda evaluar un mecanismo de reintento con backoff, o un patrón *outbox*, para no perder eventos durante ventanas de backpressure de RabbitMQ.

## 6. Recomendaciones de Monitoreo

> Nota: la implementación completa de un stack de métricas/alertas corresponde a HU-008 (Observabilidad, opcional según `BACKLOG.md`). Lo que sigue son las señales mínimas que, de haber existido, habrían detectado este incidente antes de llegar a un estado crítico.

- **Profundidad de la cola `orders_ready`**: alertar al cruzar un umbral bajo (ej. > 1.000 mensajes), no esperar a los 12.500 reportados — para cuando se llega a ese número, el incidente ya lleva tiempo activo.
- **Uso de memoria del worker vs. su límite**: alertar al acercarse al límite (ej. > 80%), **antes** del `OOMKilled`, no después. Esta señal habría anticipado el incidente completo.
- **Restart count elevado en ventana corta**: alertar ante varios reinicios en pocos minutos — es la señal más directa de un crash loop, y en este caso ya iba en 14 reinicios sin que aparentemente se hubiera generado ninguna alerta.
- **Memory high watermark de RabbitMQ**: RabbitMQ expone esta métrica/alarma directamente — monitorearla habría confirmado o descartado H3 en tiempo real, en vez de tener que investigarlo después del hecho.
- **Dashboard de correlación**: tener memoria del worker, profundidad de cola y restart count en una sola vista — este incidente es un ejemplo claro de una falla en cadena que, vista por separado en distintos dashboards, es mucho más difícil de diagnosticar rápido.
