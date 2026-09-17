# Archivos reemplazados (no eliminados)

Este directorio conserva los `docker-compose` originales de cada servicio,
tal como llegaron en el repositorio antes de unificarlos.

## Por qué se movieron aquí en vez de borrarlos

Por integridad y trazabilidad del proceso de evaluación: se prefiere dejar
evidencia explícita de qué existía originalmente y qué se reemplazó, en vez
de eliminarlo directamente. El historial de `git` también preserva este
movimiento como un rename (`git mv`), no como un delete + create.

## Qué los reemplazó

Ambos quedaron consolidados en un único [`docker-compose.yml`](../docker-compose.yml)
en la raíz del repositorio, que levanta los 5 componentes de la plataforma
(`orders-service`, `reception-service`, `rabbitmq`, `redis`, `postgres`) con
`docker compose up`, sin necesidad de correr composes separados por servicio.

## Por qué fue necesario unificarlos

- `orders-service/docker-compose.yaml` declaraba su propio RabbitMQ.
- `reception-service/test/docker-compose.yml` traía RabbitMQ **comentado**,
  para evitar chocar con el anterior si ambos se levantaban a la vez (mismo
  puerto 5672 mapeado al host).

Esto reflejaba que cada servicio se desarrolló de forma aislada. El compose
unificado resuelve esto con un único RabbitMQ compartido, referenciado por
ambos servicios mediante el hostname interno de Docker (`rabbitmq`), en vez
de `localhost`.
