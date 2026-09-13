# Familia C — `practica-examen/` (pedidos ↔ notificaciones)

Bloque de **práctica / examen**: variante completa de ventas con doble llamada Feign, anulaciones por Kafka y correos por Rabbit. Es la familia A llevada a escenario de examen.

## Objetivo de la clase

1. CRUD `/sales` que enriquece con **doble Feign** (`ProductClient` + `NotificationClient`).
2. Anulación async: `sale-cancellation-requests` (Kafka) → `sale_cancellation_logs` consultables.
3. Correos async: `purchase-email-queue` (Rabbit) → `correos_enviados`.
4. Flags `messaging.*` + `@ConditionalOnProperty` para activar/desactivar mensajería.

## Servicios

| Servicio | Puerto | Responsabilidad |
|---|---|---|
| `ms-pedidos` | 8082 | CRUD `/sales`, `POST /sales/rabbit-reserve`, detalle con producto+notificación; publica movimientos, cancelaciones y correos |
| `ms-notificaciones` | 8081 | CRUD `/notificaciones...` + `/mensajes` + `GET /ventas/{saleId}/anulaciones/logs`; consume cancelaciones, correos y stock |

Stack: Java 17, Boot 3.2.5, Gradle 8.7, MySQL `appdb`, Kafka 2 topics, Rabbit 2 exchanges, OpenFeign (solo pedidos).

## Docker

Misma infra que familia A (ver README raíz § Infra): MySQL + Kafka + Rabbit. **Apagar familia A antes** (colisionan 8081/8082). Sin `Dockerfile`: con `./gradlew bootRun`.

## Cómo levantar (en orden)

```bash
cd practica-examen/ms-notificaciones && ./gradlew bootRun  # :8081 primero
cd ../ms-pedidos && ./gradlew bootRun                      # :8082
```

Nota: `messaging.rabbitmq.stock.enabled: false` por defecto → el flujo `rabbit-reserve` de stock está inactivo hasta activar el flag.

## Postman

- `ms-pedidos/postman/ms-pedidos.postman_collection.json` → CRUD ventas, `GET /sales/{id}/details`, `POST /sales/rabbit-reserve`, anulación (`DELETE`-semántico) y verificación en `GET {notif}/notificaciones/ventas/{saleId}/anulaciones/logs`.
- `ms-notificaciones/postman/ms-notificaciones.postman_collection.json` → mensajes y logs.

## Protocolos

Postman todo **HTTP/REST**. Entre servicios: HTTP (doble Feign a `http://localhost:8081`), Kafka **TCP binario** (topics `stock-movements` + `sale-cancellation-requests`, `:9092`), Rabbit **AMQP/TCP** (`purchase-email-queue`, `stock-*`, `:5672`), MySQL **TCP** (`:5510`). Nada UDP.

## Endpoints clave

```http
POST http://localhost:8082/sales
GET  http://localhost:8082/sales/{id}/details
POST http://localhost:8082/sales/rabbit-reserve
GET  http://localhost:8081/notificaciones/mensajes/{id}
GET  http://localhost:8081/notificaciones/ventas/{saleId}/anulaciones/logs
```

Advertencia: el `ProductClient` de pedidos apunta a `GET :8081/products/{id}`, path que notificaciones no expone (herencia de familia A) → ese detalle falla con 404. Ver `.planning/codebase/INTEGRATIONS.md` (Brecha 1).
