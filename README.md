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

```text
Cliente
   │
   ▼
Orders API
   │
   ▼
RabbitMQ
   │
   ▼
Orders Worker
```

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

El repositorio contiene:
```text
orders-service/
orders-worker/
```
Ambos servicios son funcionales y pueden ejecutarse localmente.
- La responsabilidad del candidato NO es desarrollar nuevas funcionalidades de negocio.
- La responsabilidad principal es preparar la plataforma para ambientes productivos.
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

# Kubernetes

Todo el despliegue deberá realizarse utilizando Helm.
Se espera el uso de:
- Deployment
- Service
- ConfigMap
- Secret
Según aplique.
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
