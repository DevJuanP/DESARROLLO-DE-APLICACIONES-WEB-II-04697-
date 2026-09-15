# Familia A — `backend/` (CRUD + JWT + mensajería base)

Bloque de clases de **fundamentos**: CRUD REST con JPA/MySQL, autenticación JWT y primera mensajería Kafka + Rabbit con llamada síncrona Feign.

## Objetivo de la clase

1. CRUD `/products` y `/sales` con capas `rest → negocio → repositorio → entidades` + DTOs `record`.
2. Auth JWT real: registro/login, filtro `OncePerRequestFilter`, rutas protegidas.
3. Comunicación sync (Feign `sales → products`) y async (Kafka `stock-movements`, Rabbit `stock-exchange`).

## Servicios

| Servicio | Puerto | Responsabilidad |
|---|---|---|
| `products-services` | 8081 | CRUD `/products`, `decreaseStock` sincronizado; consume stock (Kafka+Rabbit), emite `stock.low` |
| `sales-services` | 8082 | CRUD `/sales`, `GET /sales/{id}/details` (Feign), `POST /sales/rabbit-reserve`; publica movimientos |
| `jwt-sales-services` | 8083 | `POST /users` + `POST /auth/login` públicos, `GET /sales` y resto protegidos con JWT |

Stack: Java 17, Boot 3.2.5, Gradle 8.7, MySQL `appdb`, `spring-kafka`, `spring-amqp`, OpenFeign (solo sales), `jjwt` (solo jwt).

## Docker

Necesita infra (ver README raíz § Infra):

```bash
cd database && docker compose up -d
cd ../queue && docker compose -f docker-compose-kafka.yml up -d
cd ../queue && docker compose -f docker-compose-rabbitmq.yml up -d
```

Sin `Dockerfile` para estos micros.

## Cómo levantar (en orden)

```bash
cd backend/products-services && ./gradlew bootRun   # :8081 primero
cd ../sales-services && ./gradlew bootRun           # :8082
cd ../jwt-sales-services && ./gradlew bootRun       # :8083
```

## Postman

- `postman/Cibertec-JWT-Sales.postman_collection.json` + env `Cibertec-JWT-Local` → `POST /users`, `POST /auth/login`, `GET /sales` con/sin token (`:8083`).
- `postman/Cibertec-Microservices-Flows.postman_collection.json` + env `Cibertec-Local` → health, `POST /sales` (Kafka), `POST /sales/rabbit-reserve` (Rabbit), `GET /sales/{id}/details` (Feign), CRUD productos.

> Paso a paso para probar los flujos Kafka y Rabbit (ver mensajes en Kafka-UI y en la consola de Rabbit): [flujos-kafka-rabbit.md](flujos-kafka-rabbit.md).

## Protocolos

Postman todo **HTTP/REST**. Entre servicios: HTTP (controllers + Feign `sales → http://localhost:8081`), Kafka **TCP binario** (topic `stock-movements`, 3 consumer-groups, `:9092`), Rabbit **AMQP/TCP** (exchange `stock-exchange`, colas `stock-reserve-queue`/`stock-low-queue`, keys `stock.reserve`/`stock.low`, `:5672`), MySQL **TCP** (`localhost:5510/appdb`). Nada UDP.

## Endpoints clave

```http
GET  http://localhost:8081/products
POST http://localhost:8082/sales
GET  http://localhost:8082/sales/{id}/details
POST http://localhost:8083/auth/login
GET  http://localhost:8083/sales   # con Authorization: Bearer <jwt>
```

Detalle por servicio en `products-services/README.md`, `sales-services/README.md`, `jwt-sales-services/README.md`. Mapa en `.planning/codebase/ARCHITECTURE.md` (Familia A).
