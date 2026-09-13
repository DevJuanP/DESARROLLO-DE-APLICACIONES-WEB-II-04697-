# DESARROLLO DE APLICACIONES WEB II — 04697

Repositorio académico (monorepo) de la asignatura **Desarrollo de Aplicaciones Web II**.
Contiene varias **familias independientes de microservicios Spring Boot + un frontend Angular** para practicar, de forma progresiva:

- CRUD REST + JPA + MySQL
- Auth JWT + Spring Security
- Comunicación síncrona (OpenFeign / RestTemplate + Eureka + Gateway)
- Mensajería asíncrona (Kafka + RabbitMQ)
- Resiliencia (Circuit Breaker + fallback) y despliegue en K8s

> **Importante:** las familias son **independientes y no se ejecutan todas a la vez**.
> Hay colisiones de puertos entre familias (8081/8082/8083). Levanta **una familia a la vez**.

Documentación de análisis base en `.planning/codebase/` (`ARCHITECTURE.md`, `STACK.md`, `STRUCTURE.md`, `INTEGRATIONS.md`, `CONVENTIONS.md`, `CONCERNS.md`, `TESTING.md`).

---

## 1. Mapa general del repositorio

```
├── backend/              # Familia A: productos + ventas + auth JWT (sin Eureka ni gateway)
├── spring_cloud/         # Familia B: fintech con Eureka + Gateway (sin DB)
├── practica-examen/      # Familia C: pedidos ↔ notificaciones (con colas, doble Feign)
├── resiliencia-demo/     # Familia D: Feign + CircuitBreaker + K8s (sin colas ni Eureka)
├── frontend/hospital_web/# Plantilla Angular 21 "Cliniva" (mock local, sin backend real)
├── database/             # docker-compose MySQL:5510 + phpMyAdmin:3410
├── queue/                # docker-compose Kafka:9092 (+UI:8090) y RabbitMQ:5672 (+mgmt:15672)
└── postman/              # Collections raíz JWT + Flows + 2 environments
```

### Familias

| Familia | Carpeta | Idea en una línea | Discovery | Mensajería | BD |
|---|---|---|---|---|---|
| A | `backend/` | CRUD productos/ventas + auth JWT | No (Feign URL fija) | Kafka + Rabbit (products/sales) | MySQL `appdb` |
| B | `spring_cloud/` | Fintech tarjetas prepago: cuentas + recargas + gateway | Sí (Eureka 8761 + Gateway 8762) | No | No (memoria) |
| C | `practica-examen/` | Variante examen: pedidos + notificaciones + cancelaciones + correos | No (Feign URL/config) | Kafka 2 topics + Rabbit 2 exchanges | MySQL `appdb` |
| D | `resiliencia-demo/` | Demo `sin-resiliencia` vs `con-resiliencia` (CircuitBreaker + fallback) | No | No | No |
| Front | `frontend/hospital_web` | Template hospital Admin/Doctor/Patient, auth simulada | — | — | No |

---

## 2. Arquitectura por familia

### Familia A — `backend/*` (8081 / 8082 / 8083)

- `products-services (:8081)`: CRUD `/products`, `decreaseStock` sincronizado. Consume Kafka `stock-movements` (3 consumer-groups) y Rabbit `stock-exchange`, emite alertas `stock.low`.
- `sales-services (:8082)`: CRUD `/sales`, `GET /sales/{id}/details` (enriquece con producto vía Feign a `:8081`), `POST /sales/rabbit-reserve`. Publica `stock-movements` (Kafka) y `stock.reserve` (Rabbit).
- `jwt-sales-services (:8083)`: `POST /users` + `POST /auth/login` públicos, resto autenticado con JWT (`JwtService` + `JwtAuthenticationFilter` + `SecurityConfig`). `GET /sales` protegido. Sin Kafka/Rabbit ni llamadas salientes.

### Familia B — `spring_cloud/*` (8761 / 8762 / 8300 / 8100 / 8083)

```
cliente/postman → api-gateway (:8762) → lb://ms-cuentas | lb://ms-recargas | lb://ms-notificaciones
                                     ↘ eureka-server (:8761) (discovery)
ms-recargas (:8100) → RestTemplate @LoadBalanced → http://ms-cuentas/cuentas/saldo/{id} (:8300)
```

Sin MySQL/Kafka/Rabbit. Datos hardcodeados (cuentas 001/002/003). `ms-notificaciones` aquí es solo un stub `GET /notificaciones/enviar/{tipo}`.

### Familia C — `practica-examen/*` (8082 / 8081)

- `ms-pedidos (:8082)`: CRUD `/sales` + doble Feign (`ProductClient` + `NotificationClient` → `:8081`), publica Kafka `stock-movements` y `sale-cancellation-requests`, y Rabbit `purchase-email-queue` (stock solo si `messaging.rabbitmq.stock.enabled=true`, por defecto `false`).
- `ms-notificaciones (:8081)`: CRUD `/notificaciones...` + `/mensajes` + `/ventas/{saleId}/anulaciones/logs`. Consume cancelaciones Kafka, correos Rabbit y eventos stock.

### Familia D — `resiliencia-demo/*` (8090 / 8091)

- `ms-pedidos (:8090)`: `POST /pedidos/sin-resiliencia` (Feign directo, 500 si falla) vs `POST /pedidos/con-resiliencia` (`@CircuitBreaker(name="inventario", fallbackMethod=...)` → `RECIBIDO_SIN_VALIDAR_STOCK`).
- `ms-inventario (:8091)`: `GET /inventario/{productoId}` + simuladores `POST /inventario/demo/falla/{true|false}`, `POST /inventario/demo/demora/{millis}`, `GET /inventario/demo/estado`. Único con `Dockerfile` + `k8s/` + `deploy.sh` + Actuator.

### Frontend — `frontend/hospital_web` (Cliniva)

Angular 21 + Material, lazy por rol (`admin/doctor/patient`), `AuthGuard` por `localStorage`, `apiUrl: http://localhost:4200` (loopback). **No consume ningún backend Java** (`LoginService` hace `GET /user` simulado).

---

## 3. Stack

- **Java 17** en todos los microservicios. **Spring Boot 3.2.5** dominante (excepción: `spring_cloud/ms-notificaciones` con Boot **4.0.6** + Cloud 2025.1.1).
- **Build:** Gradle 8.7 (`backend/*`, `practica-examen/*`), Gradle 9.5.1 (`spring_cloud/ms-notificaciones`), Maven (`spring_cloud` resto + `resiliencia-demo`). Frontend: Angular CLI 21 / npm (sin lockfile).
- **Persistencia:** MySQL 8.0 (Docker, `localhost:5510/appdb`, `ddl-auto: update`, sin Flyway/Liquibase). Solo familias A y C.
- **Mensajería:** Kafka 7.4.0 (topics `stock-movements`, `sale-cancellation-requests`) + RabbitMQ 3-management (exchanges `stock-exchange`, `notification-exchange`; colas `stock-reserve-queue`, `stock-low-queue`, `purchase-email-queue`). Solo familias A y C.
- **Discovery/Gateway:** solo familia B (Eureka + Spring Cloud Gateway + `RestTemplate @LoadBalanced`).
- **Resiliencia:** solo familia D (Resilience4j + Actuator + timeouts Feign 1000ms).
- **Frontend:** TypeScript ~5.9, Angular 21 + Material + Bootstrap 5 + FullCalendar/ApexCharts/ECharts, Karma+Jasmine para tests.

Tabla completa módulo → stack en `.planning/codebase/STACK.md`.

---

## 4. Puertos

| Servicio | Puerto | Servicio | Puerto |
|---|---|---|---|
| `backend/products-services` | 8081 | `practica ms-notificaciones` | 8081 ⚠️ colisiona |
| `backend/sales-services` | 8082 | `practica ms-pedidos` | 8082 ⚠️ colisiona |
| `backend/jwt-sales-services` | 8083 | `spring_cloud/ms-notificaciones` | 8083 ⚠️ colisiona |
| `spring_cloud/eureka-server` | 8761 | `spring_cloud/api-gateway` | 8762 |
| `spring_cloud/ms-cuentas` | 8300 | `spring_cloud/ms-recargas` | 8100 |
| `resiliencia ms-pedidos` | 8090 | `resiliencia ms-inventario` | 8091 |
| MySQL / phpMyAdmin | 5510→3306 / 3410→80 | Kafka / Kafka-UI / ZK | 9092 / 8090 ⚠️ / 2181 |
| RabbitMQ / mgmt | 5672 / 15672 | Frontend dev | 4200 |

No levantar `backend/*` y `practica-examen/*` a la vez, ni `jwt-sales` con `spring_cloud/ms-notificaciones`, ni `kafka-ui` con `resiliencia ms-pedidos` sin remapear.

---

## 5. Requisitos

- JDK 17, Docker + Docker Compose, Node 24 + npm, `curl` / Postman.
- Maven solo necesario para `spring_cloud/*` y `resiliencia-demo/*` (4 módulos traen `mvnw`, resiliencia usa `maven:3.9` en Docker).
- Cada servicio Gradle trae su `gradlew`.

---

## 6. Cómo levantar cada familia

### Infra común (familias A y C)

```bash
cd database && docker compose up -d        # MySQL 5510 + phpMyAdmin 3410
cd ../queue && docker compose -f docker-compose-kafka.yml up -d
cd ../queue && docker compose -f docker-compose-rabbitmq.yml up -d
```

### Familia A — backend

```bash
cd backend/products-services && ./gradlew bootRun   # :8081 primero
cd ../sales-services && ./gradlew bootRun           # :8082
cd ../jwt-sales-services && ./gradlew bootRun       # :8083
```

Probar: `GET http://localhost:8081/products`, `GET http://localhost:8082/sales`, `POST http://localhost:8083/auth/login`.

### Familia B — spring_cloud

```bash
cd spring_cloud/eureka-server && ./mvnw spring-boot:run  # :8761
cd ../ms-cuentas && ./mvnw spring-boot:run               # :8300
cd ../ms-recargas && ./mvnw spring-boot:run              # :8100
cd ../api-gateway && ./mvnw spring-boot:run              # :8762
```

Probar: `curl http://localhost:8761`, `curl http://localhost:8300/cuentas/saldo/001`, `curl http://localhost:8762/recargas/procesar/001/50`.

### Familia C — practica-examen

```bash
cd practica-examen/ms-notificaciones && ./gradlew bootRun  # :8081
cd ../ms-pedidos && ./gradlew bootRun                      # :8082
```

### Familia D — resiliencia-demo

```bash
cd resiliencia-demo/ms-inventario && mvn spring-boot:run  # :8091
cd ../ms-pedidos && mvn spring-boot:run                   # :8090
# K8s alternativo: ver resiliencia-demo/deploy.sh (modos local / gitops con ArgoCD)
```

Demo: `POST :8091/inventario/demo/falla/true` → `POST :8090/pedidos/sin-resiliencia` (500) vs `POST :8090/pedidos/con-resiliencia` (fallback). Estado CB: `GET :8090/actuator/circuitbreakers`.

### Frontend

```bash
cd frontend/hospital_web && npm install && npm start  # http://localhost:4200
```

---

## 7. Postman

| Colección | Cubre |
|---|---|
| `postman/Cibertec-JWT-Sales...` + env `Cibertec-JWT-Local` | login JWT, `GET /sales` con/sin token, CRUD usuarios (:8083) |
| `postman/Cibertec-Microservices-Flows...` + env `Cibertec-Local` | flujos Kafka/Rabbit/Feign + CRUD productos (:8081/:8082) |
| `spring_cloud/postman/sistema-fintech...` | Eureka + saldos/recargas directas y vía gateway |
| `practica-examen/*/postman/` | ventas + Feign + anulación Kafka + logs |
| `resiliencia-demo/postman/resiliencia-...` | sin/con resiliencia + falla/demora + actuator CB |

---

## 8. Convenciones de código

- Capas ES: `rest/` (controller fino) → `negocio/` (reglas, lanza `ResponseStatusException`) → `repositorio/` (JpaRepository) → `entidades/` (JPA) + `dto/` (`record` `Request`/`Response`). Excepción: `spring_cloud/*` (1 controller, sin capas) y `resiliencia-demo` (`rest/service/client/dto`, sin JPA).
- Endpoints en plural sin versionado (`/sales`, `/products`, `/users`, `/pedidos`, `/inventario`). Sin `@ControllerAdvice` global.
- Mensajería tipada (`*Event`/`*Producer`/`*Consumer`/`*Config`), seeders `*DataInitializer`, DTOs nunca exponen entidades.
- Frontend: standalone components kebab-case, lazy por rol, guards `auth.guard`, aliases `@core/@shared`.
- Commits: 1 solo commit inicial (`feat add initial commit`).

Detalle en `.planning/codebase/CONVENTIONS.md` y `STRUCTURE.md` (incluye tabla “quiero X → voy a Y”).

---

## 9. Tests

- Backend: JUnit 5 + `spring-boot-starter-test`. 9/12 módulos solo tienen `contextLoads()`; sin tests `jwt-sales-services` ni `resiliencia-demo/*`. 0 tests de negocio/mocks/Testcontainers.
- Frontend: 83 `*.spec.ts` estilo `should create` (Karma+Jasmine, necesita Chrome).

```bash
cd backend/sales-services && ./gradlew test
cd spring_cloud/ms-cuentas && mvn test
cd frontend/hospital_web && npx ng test --watch=false --browsers=ChromeHeadless
```

Recomendación mínima y gaps en `.planning/codebase/TESTING.md`.

---

## 10. Advertencias conocidas

- **Puertos en colisión** (ver §4) y **misma DB `appdb` para 5 servicios** con `ddl-auto: update` (riesgo de DDL cruzado).
- **Secretos demo commiteados** (`database/.env`, `app.jwt.secret`, user/pass en `application.yml`) — solo uso local.
- **Feign roto en practica-examen:** `ProductClient` pide `GET :8081/products/{id}` a un servicio que no expone `/products`; mensajería stock desactivada por defecto (`messaging.rabbitmq.stock.enabled: false`).
- **Sin resiliencia fuera de `resiliencia-demo`** (Feign sin timeouts/fallback), gateway con `-` huérfano en YAML, `ms-notificaciones` Boot 4 vs resto Boot 3, frontend desconectado + auth solo-cliente.
- **Deuda git:** 355 artefactos `bin/build/target/*.class/*.jar` + 15 `.DS_Store` commiteados pese a `.gitignore`.

Auditoría completa en `.planning/codebase/CONCERNS.md` e `INTEGRATIONS.md` (brechas).

---

## 11. Dónde seguir

- READMEs por módulo: `backend/*/README.md`, `spring_cloud/README.md`, `practica-examen/*/README.md`, `resiliencia-demo/README.md`.
- Mapa detallado: `.planning/codebase/ARCHITECTURE.md` (diagramas ASCII + flujos auth/ventas/pedidos/recargas).
