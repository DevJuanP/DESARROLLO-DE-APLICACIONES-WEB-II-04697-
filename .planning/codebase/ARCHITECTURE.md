# ARCHITECTURE

## Vista general del sistema (diagrama ASCII: frontend → gateway/eureka → microservicios → DB/queue)

El repo contiene 4 familias independientes (no se ejecutan todas a la vez; hay colisiones
de puertos entre familias). Cada familia tiene su propio diagrama.

### Familia A — backend/* (productos + ventas + auth JWT, sin Eureka ni gateway)

```
                 +-------------------+
                 | frontend/hospital |
                 | _web (mock local, |
                 |  no consume estos |
                 |  backends)        |
                 +---------+---------+
                           | (sin wiring real: LoginService hace GET /user simulado)
                           v
  +----------------+------------------+------------------+
  | products-services (:8081)          | sales-services (:8082) | jwt-sales-services (:8083) |
  | CRUD /products                     | CRUD /sales +          | POST /auth/login           |
  |                                    |  GET /sales/{id}/      | CRUD /users                |
  |                                    |   details (Feign)      | GET /sales (protegido JWT) |
  +--------+---------------+-----------+----------+-----+------+--------+---------+---------+
           |               |                      |    |                |         |
           v               v                      v    v                v         v
   +-------+------+ +------+------+      +--------+--+ +---+---+  +-----+---+ +--+--------+
   | MySQL appdb  | | Kafka :9092 |      | MySQL appdb | | Kafka |  | MySQL   | | (sin cola)|
   | (:5510→3306) | | topic:      |      | (:5510→3306)| | topic |  | appdb   | |           |
   | tabla        | | stock-      |      | tabla sales | | stock-|  | tablas  | |           |
   | products     | | movements   |      |             | | move- |  | app_    | |           |
   |              | | (3 consumer |      |             | | ments |  | users,  | |           |
   |              | |  groups)    |      |             | | (prod)|  | secure_ | |           |
   |              | +------+------+      |             | |       |  | sales   | |           |
   |              | | RabbitMQ    |      |             | +---+---+  |         | |           |
   |              | | :5672       |      |             | | RabbitMQ|  |         | |           |
   |              | | stock-      |      |             | | stock-  |  |         | |           |
   |              | | exchange    |      |             | | exch.   |  |         | |           |
   +--------------+ +-------------+      +-------------+ +-------+--+  +---------+ +-----------+
```

Flujo async familia A: `sales-services` publica `stock-movements` (Kafka) y
`stock.reserve` (RabbitMQ `stock-exchange`); `products-services` consume ambos
(`StockUpdateConsumer`/`StockAuditConsumer`/`StockReportingConsumer`,
`StockReserveConsumer`) y publica alertas `stock.low`.

### Familia B — spring_cloud/* (fintech con Eureka + Gateway, sin DB)

```
                    +------------------+
                    | cliente / postman|
                    +--------+---------+
                             |
              +--------------+--------------+
              | api-gateway (:8762)         |
              |  /cuentas/**         -> lb://ms-cuentas
              |  /recargas/**        -> lb://ms-recargas
              |  /notificaciones/**  -> lb://ms-notificaciones
              +--------------+--------------+
                             | (service discovery)
              +--------------+--------------+
              | eureka-server (:8761)       |
              +------+------+-------+------+
                     |      |       |
                     v      v       v
              ms-cuentas ms-recargas ms-notificaciones
              (:8300)    (:8100)     (:8083)
              GET        GET /recar- GET /notifica-
              /cuentas/  gas/proce- ciones/enviar/
              saldo/{id} sar/...    {tipo}
              (hardcoded 001/002/003)
                     ^      |
                     |______|  RestTemplate @LoadBalanced
                               http://ms-cuentas/cuentas/saldo/{cuentaId}
```

Sin MySQL/Kafka/RabbitMQ. `ms-notificaciones` de esta familia es un stub
(`GET /notificaciones/enviar/{tipo}`) y comparte puerto 8083 con
`backend/jwt-sales-services` (no corren juntos).

### Familia C — practica-examen/* (pedidos ↔ notificaciones, con colas)

```
  cliente/postman
      |
      +---> ms-pedidos (:8082) ---------------------------+
      |      POST/GET/PUT/DELETE /sales                   |
      |      GET /sales/{id}/details (Feign → products*)  |
      |      Feign → ms-notificaciones (:8081)            |
      |        GET /notificaciones/mensajes/{id}          |
      |      publica Kafka: stock-movements,              |
      |        sale-cancellation-requests                 |
      |      publica Rabbit: purchase-email-queue         |
      |        (+ stock.* si messaging.rabbitmq.stock     |
      |           .enabled=true; por defecto false)       |
      +---> ms-notificaciones (:8081) <-------------------+
             GET|POST|PUT|DELETE /notificaciones...       |
             consume Kafka stock-movements (3 groups) +   |
               sale-cancellation-requests (group          |
               sale-cancellation-cg → sale_cancellation_  |
               logs)                                      |
             consume/produce Rabbit purchase-email-queue, |
               stock-reserve-queue, stock-low-queue       |
                    |                |                   |
                    v                v                   v
             MySQL appdb (:5510)  Kafka :9092      RabbitMQ :5672
```

(*) `ProductClient` de ms-pedidos apunta por defecto a `http://localhost:8081`,
es decir, a ms-notificaciones de esta misma familia en el puerto que en la
familia A ocupa products-services. Ver "Decisiones y variaciones".

### Familia D — resiliencia-demo/* (Feign + CircuitBreaker, sin colas ni Eureka)

```
  postman/kubectl
      |
      +---> ms-pedidos (:8090)
      |      POST /pedidos/sin-resiliencia  (Feign directo, sin protección)
      |      POST /pedidos/con-resiliencia  (@CircuitBreaker(name="inventario",
      |                                       fallbackMethod="fallbackCrearPedido"))
      |      Feign InventarioClient → ${inventario.base-url} (defecto http://localhost:8091)
      |        GET /inventario/{productoId}
      |
      +---> ms-inventario (:8091)
             GET /inventario/{productoId}
             POST /inventario/demo/falla/{activo}   (simula error)
             POST /inventario/demo/demora/{millis}  (simula latencia)
             GET  /inventario/demo/estado
             (despliegue K8s: ms-pedidos-svc:8090, ms-inventario-svc:8091,
              INVENTARIO_BASE_URL inyectada por env en deployment)
```

Atención: `queue/docker-compose-kafka.yml` expone `kafka-ui` en host `8090`,
mismo puerto host que `ms-pedidos` de esta demo. No levantar ambos a la vez
sin remapear.

## Mapa de servicios (tabla: servicio | puerto | responsabilidad | DB | depende de)

| Servicio | Puerto | Responsabilidad | DB | Depende de |
|---|---|---|---|---|
| backend/jwt-sales-services | 8083 | Auth JWT (`POST /auth/login`), ABM usuarios (`/users`), lectura ventas protegidas (`GET /sales`) | MySQL `appdb` (`localhost:5510`), tablas `app_users`, `secure_sales` | Solo MySQL |
| backend/products-services | 8081 | CRUD `/products`, `decreaseStock` sincronizado; consume eventos stock (Kafka+Rabbit) y emite alertas stock bajo | MySQL `appdb`, tabla `products` | MySQL, Kafka `localhost:9092` topic `stock-movements`, RabbitMQ `localhost:5672` exchange `stock-exchange` |
| backend/sales-services | 8082 | CRUD `/sales`, `GET /sales/{id}/details` (enriquece con producto vía Feign), `POST /sales/rabbit-reserve`; publica movimientos stock | MySQL `appdb`, tabla sales | MySQL, `products-services` (`http://localhost:8081`, Feign `products-services`), Kafka (produce `stock-movements`), RabbitMQ (produce `stock.reserve`, consume `stock.low`) |
| spring_cloud/eureka-server | 8761 | Service discovery Eureka (`@EnableEurekaServer`, `register-with-eureka:false`) | Ninguna | Ninguna |
| spring_cloud/api-gateway | 8762 | Spring Cloud Gateway: `/cuentas/**→lb://ms-cuentas`, `/recargas/**→lb://ms-recargas`, `/notificaciones/**→lb://ms-notificaciones` | Ninguna | eureka-server (`defaultZone http://localhost:8761/eureka/`) |
| spring_cloud/ms-cuentas | 8300 | `GET /cuentas/saldo/{id}`, saldos hardcodeados cuentas 001/002/003 | Ninguna (memoria) | eureka-server |
| spring_cloud/ms-recargas | 8100 | `GET /recargas/procesar/{cuentaId}/{monto}`, aprueba/rechaza consultando ms-cuentas | Ninguna | eureka-server, `ms-cuentas` vía `RestTemplate @LoadBalanced` (`http://ms-cuentas/cuentas/saldo/{cuentaId}`) |
| spring_cloud/ms-notificaciones | 8083 | Stub `GET /notificaciones/enviar/{tipo}` | Ninguna | eureka-server |
| practica-examen/ms-pedidos | 8082 | CRUD `/sales` + `POST /sales/rabbit-reserve`, detalle con producto y con notificación (Feign); emite cancelaciones y correos | MySQL `appdb` | MySQL, Kafka (produce `stock-movements`, `sale-cancellation-requests`), RabbitMQ (`purchase-email-queue`; stock solo si `messaging.rabbitmq.stock.enabled=true`), `ms-notificaciones` (`${notificaciones.base-url}` defecto `http://localhost:8081`) |
| practica-examen/ms-notificaciones | 8081 | CRUD `/notificaciones...` + `/mensajes` + `/ventas/{saleId}/anulaciones/logs`; consume cancelaciones y stock; registra correos | MySQL `appdb` (tablas `mensajes_notificacion`, `correos_enviados`, `sale_cancellation_logs`, más `products`/`sale` locales) | MySQL, Kafka (consume `stock-movements`, `sale-cancellation-requests`), RabbitMQ (`purchase-email-queue`, `stock-reserve-queue`, `stock-low-queue`) |
| resiliencia-demo/ms-pedidos | 8090 | `POST /pedidos/sin-resiliencia` vs `POST /pedidos/con-resiliencia` (CircuitBreaker `inventario` + fallback) | Ninguna | `ms-inventario` vía Feign (`${inventario.base-url}` defecto `http://localhost:8091`) |
| resiliencia-demo/ms-inventario | 8091 | `GET /inventario/{productoId}` + endpoints demo `falla/demora/estado` | Ninguna | Ninguna |
| frontend/hospital_web (Cliniva) | no declarado (dev server; `environment.apiUrl=http://localhost:4200`) | Plantilla hospital Angular: admin/doctor/patient, auth, layout | Ninguna (login simulado) | Ninguna real (mock `/user`) |
| Infra MySQL+phpMyAdmin | 5510→3306, 3410→80 | `database/docker-compose.yml` + `database/.env` (`appdb`/`app`/`password`) | — | — |
| Infra Kafka | 9092 (+ UI 8090→8080, ZK 2181) | `queue/docker-compose-kafka.yml` | — | — |
| Infra RabbitMQ | 5672 (+ mgmt 15672) | `queue/docker-compose-rabbitmq.yml` (guest/guest) | — | — |

Puertos marcados "no declarado" solo si no aparecen en configs: no hay casos en
backends/micros (todos verificados en `application.yml/yaml`). El frontend no
declara `server.port` propio (usa el dev-server de Angular y `apiUrl` mock).

## Flujos clave (auth JWT login, ventas, productos, pedidos→notificaciones, recargas/cuentas)

### 1. Auth JWT login (backend/jwt-sales-services :8083)

1. `POST /users` crea usuario (`UserController.createUser` → `UserService`, sin
   auth por `SecurityConfig`: `POST /users` y `POST /auth/login` en `permitAll`).
2. `POST /auth/login` (`AuthController.login` con `LoginRequest`) →
   `AuthService` valida credenciales y `JwtService` firma el token
   (`app.jwt.secret`, `app.jwt.expiration-minutes=60`).
3. Respuesta `LoginResponse` con token. Cliente lo reenvía como
   `Authorization: Bearer <jwt>`.
4. `JwtAuthenticationFilter` (filtro `OncePerRequestFilter`) extrae el Bearer,
   valida con `JwtService` y puebla el `SecurityContext`.
5. `GET /sales` (`SaleController` → `SaleService`) exige autenticación
   (`anyRequest().authenticated()`); el seed lo provee `SaleDataInitializer`.
   Entidades: `AppUser(app_users)`, `Sale(secure_sales)`; repos
   `AppUserRepository`, `SaleRepository`.

### 2. Productos (backend/products-services :8081)

- `ProductController`: `GET /products`, `GET /products/{id}`,
  `POST /products`, `PUT /products/{id}`, `DELETE /products/{id}` →
  `ProductService` → `ProductRepository` → `Product(products)`; seed
  `ProductDataInitializer`; lectura DTO `ProductResponse`.
- `decreaseStock` es `synchronized` (control de concurrencia en-memoria/JVM).
- Consumers Kafka (topic `stock-movements`): `StockUpdateConsumer`
  (`stock-update-cg`), `StockAuditConsumer` (`audit-cg`),
  `StockReportingConsumer` (`reporting-cg`).
- RabbitMQ (`stock-exchange` DirectExchange): `StockReserveConsumer`
  (`stock-reserve-queue`, key `stock.reserve`), `StockLowAlertProducer`
  (`stock-low-queue`, key `stock.low`).

### 3. Ventas (backend/sales-services :8082)

- `SaleController`: `GET /sales`, `GET /sales/{id}`,
  `GET /sales/{id}/details`, `POST /sales`, `POST /sales/rabbit-reserve`,
  `PUT /sales/{id}`, `DELETE /sales/{id}` → `SaleService`.
- `POST /sales`: `savePendingSale` + `StockMovementProducer` publica
  `StockMovementEvent` en `stock-movements` (`KafkaTopicConfig`).
- `POST /sales/rabbit-reserve`: mismo guardado + `StockReserveProducer`
  publica `stock.reserve` en `stock-exchange`.
- `GET /sales/{id}/details`: `getSaleDetailsWithFeign` llama a
  `ProductClient.getProductById` (Feign `products-services`,
  URL fija `http://localhost:8081`, `GET /products/{id}`) y arma
  `SaleWithProductResponse`. Si products cae, el detalle falla (sin fallback).
- `StockLowAlertConsumer` escucha `stock-low-queue`.

### 4. Pedidos → notificaciones (practica-examen ms-pedidos :8082 → ms-notificaciones :8081)

- ms-pedidos `SaleService.createSale` devuelve `SaleWithNotificationResponse`:
  guarda venta, publica movimiento/cancelación en Kafka y correo en Rabbit, y
  enriquece vía doble Feign: `ProductClient` + `NotificationClient`
  (`GET /notificaciones/mensajes/{mensajeId}`, configurable por
  `notificaciones.base-url`, `mensaje-confirmacion-id: 2`).
- `SaleReserveService` + `POST /sales/rabbit-reserve` + flags
  `messaging.kafka.enabled` / `messaging.rabbitmq.enabled` /
  `messaging.rabbitmq.stock.enabled` (este último `false` por defecto:
  los beans stock de este ms llevan `@ConditionalOnProperty`).
- ms-notificaciones: `SaleCancellationConsumer` (topic
  `sale-cancellation-requests`, group `sale-cancellation-cg` →
  `SaleCancellationService` → `sale_cancellation_logs`, consultable en
  `GET /ventas/{saleId}/anulaciones/logs`); `CorreoEnviadoConsumer`
  (`purchase-email-queue` → `CorreoEnviadoService` → `correos_enviados`);
  `NotificacionesController` expone `/{id}`, `/mensajes`, `/mensajes/{id}`,
  `/ventas/{saleId}/anulaciones/logs`.
- Reutiliza los 3 consumers de `stock-movements` y la pareja
  `StockReserveConsumer`/`StockLowAlertProducer` de products-services
  (mismos nombres de clase y colas).

### 5. Recargas/cuentas (spring_cloud: gateway :8762 → recargas :8100 → cuentas :8300)

1. Cliente llama `GET http://localhost:8762/recargas/procesar/{cuentaId}/{monto}`
   → gateway matchea `Path=/recargas/**` → `lb://ms-recargas` (resuelto por Eureka).
2. `RecargaController.procesarRecarga` usa `RestTemplate @LoadBalanced`
   (`@Bean` en `MsRecargasApplication`) para
   `GET http://ms-cuentas/cuentas/saldo/{cuentaId}`.
3. `CuentaController` (`GET /cuentas/saldo/{id}`,(Modelo en memoria, incluye
   `server.port` en la respuesta) devuelve saldo hardcodeado (001/002/003).
4. ms-recargas decide `recarga_aprobada` y responde el mapa
   (`cuenta`, `recarga_aprobada`, ...). Acceso directo alternativo:
   `http://localhost:8100/...` y `http://localhost:8300/...`; Eureka UI en
   `http://localhost:8761`.

## Patrones (layered rest/negocio/repositorio, DTOs, filtros JWT, service discovery)

- **Layered `rest/negocio/repositorio/entidades`**: las 3 familias Spring-Boot
  (`backend/*`, `practica-examen/*`, `resiliencia-demo/ms-pedidos`) separan
  controlador REST fino (`rest/`: mapea HTTP↔DTO) → servicio (`negocio/` o
  `service/`: reglas, orquestación Feign + productores) → repositorio Spring
  Data (`repositorio/`: `SaleRepository`, `ProductRepository`,
  `AppUserRepository`, `MensajeNotificacionRepository`,
  `CorreoEnviadoRepository`, `SaleCancellationLogRepository`) →
  `entidades/` JPA (`@Entity/@Table`). Excepción: `spring_cloud/*` colapsa a
  1 controlador + `Application` (sin capas, sin JPA).
- **DTOs request/response**: nunca se exponen entidades. Ejemplos:
  `LoginRequest/LoginResponse`, `UserRequest/UserResponse`,
  `SaleRequest/SaleResponse/SaleWithProductResponse/SaleWithNotificationResponse`,
  `ProductResponse`, `MensajeNotificacionResponse`,
  `SaleCancellationLogResponse`, `PedidoRequest/PedidoResponse/StockResponse`,
  `ErrorResponse`. Eventos de cola también tipados
  (`StockMovementEvent`, `StockReserveEvent`, `StockLowAlertEvent`,
  `SaleCancellationRequestedEvent`, `PurchaseEmailEvent/CorreoEnviadoEvent`).
- **Filtros JWT**: solo `backend/jwt-sales-services` usa Spring Security +
  `JwtAuthenticationFilter extends OncePerRequestFilter` registrado en
  `SecurityConfig` (stateless, CSRF off, `POST /users` y `POST /auth/login`
  públicos). Ningún otro micro valida JWT.
- **Service discovery**: solo `spring_cloud/*`. `eureka-server`
  (`@EnableEurekaServer`); clientes con `spring-cloud-starter-netflix-eureka-client`
  (`register/fetch=true`, `defaultZone http://localhost:8761/eureka/`);
  gateway con `spring-cloud-starter-gateway` + rutas `lb://`; llamada
  balanceada vía `RestTemplate @LoadBalanced` con nombre lógico
  (`http://ms-cuentas/...`). El resto usa URLs fijas en Feign
  (`http://localhost:8081`, `${notificaciones.base-url}`,
  `${inventario.base-url}`) sin discovery.
- **Mensajería dual**: Kafka (eventos broadcast `stock-movements`,
  `sale-cancellation-requests`, 1 productor N consumer-groups) + RabbitMQ
  (tareas dirigidas por routing-key en `stock-exchange`: `stock.reserve`,
  `stock.low`, `email.purchase.thanks`/`purchase-email-queue`). Topics/colas
  se autocrean (`KafkaTopicConfig`, `RabbitMQConfig` con `Queue/Exchange/Binding`,
  `KAFKA_AUTO_CREATE_TOPICS_ENABLE=true`).
- **Seeders `*DataInitializer`**: `SaleDataInitializer`
  (jwt + sales + ms-pedidos), `ProductDataInitializer` (products +
  ms-notificaciones practica) precargan datos demo (`ddl-auto:update`).
- **Resiliencia aislada**: solo `resiliencia-demo/ms-pedidos` usa
  Resilience4j (`@CircuitBreaker(name="inventario", fallbackMethod=..._`,
  `slidingWindowSize=5/minimumNumberOfCalls=5/failureRateThreshold=50/
  waitDurationInOpenState=10s`, timeouts Feign 1000ms, actuator
  `health,info,circuitbreakers,circuitbreakerevents`). El endpoint
  `sin-resiliencia` es el control sin protección.

## Frontend (módulos, routing, cómo consume APIs)

- **Stack**: Angular 21 (`package.json` `name: cliniva`, `version 22.0.0`),
  Material 21, `src/main.ts` → `app.component.ts` + `app.config.ts`;
  estilos `styles.scss`; `environments/environment*.ts` con
  `apiUrl: 'http://localhost:4200'` (ambos ficheros iguales: sin backend real).
- **Módulos por rol** (lazy): `app/admin/` (`admin.routes.ts` →
  `dashboard/dashboard.routes.ts` → `main`, `dashboard2`, `nurse-dashboard`),
  `app/doctor/` (`doctor.routes.ts` → `dashboard`), `app/patient/`
  (`patient.routes.ts` → `dashboard`), más `extra-pages/`, `multilevel/`,
  `authentication/` (signin, signup, forgot-password, locked, two-factor,
  coming-soon, maintenance, page404, page500).
- **Routing** (`app/app.routes.ts` → `APP_ROUTE`): `'' → MainLayoutComponent
  + AuthGuard` con hijos `admin/doctor/patient/extra-pages/multilevel`
  (redirect `''→/authentication/signin`); `'authentication' →
  AuthLayoutComponent` lazy (`auth.routes.ts` → `AUTH_ROUTE`); `'**' →
  Page404Component`. Roles vía `data.role` (`Role.Admin/Doctor/Patient` en
  `core/models/role.ts`).
- **Guards**: `core/guard/auth.guard.ts` (`AuthGuard.canActivate` lee
  `LocalStorageService.get('currentUser')`, exige `roles[0].name` y compara
  con `route.data['role']`; si falla → `/authentication/signin`);
  `core/guard/module-import.guard.ts` (anti-reimportación).
- **Consumo APIs**: **mock local, no consume ningún backend del repo**.
  `core/service/login.service.ts#login` hace `http.get<User>('/user')`
  (comentario "Simulate a login API call"); `auth.service.ts` delega en él y
  guarda `token/roleArray/permissionArray` en `token.service.ts`/`JWT.ts`;
  `error.interceptor.ts` maneja errores HTTP; `config/config.service.ts`
  carga `config.interface.ts`. `shared/services/` solo aporta
  `storage.service.ts` (localStorage). No hay `HttpClient` contra `:8081–8083`,
  `:8090/8091` ni `:8100/8300/8762`.

## Decisiones y variaciones (diferencias entre backend/* vs spring_cloud/* vs practica-examen/* vs resiliencia-demo/*)

| Eje | backend/* | spring_cloud/* | practica-examen/* | resiliencia-demo/* |
|---|---|---|---|---|
| Build | Gradle (`build.gradle`, Boot 3.2.5, Java 17, wrapper `gradlew`) | Maven (`pom.xml`) salvo `ms-notificaciones` (Gradle); Boot 3.x | Gradle (Boot 3.2.5, Java 17) | Maven (Boot 3.2.5, Java 17, `Dockerfile` + `k8s/`) |
| Discovery/gateway | No | Sí (Eureka 8761 + Gateway 8762, `lb://`, `@LoadBalanced`) | No (Feign URL fija/config) | No (Feign URL fija/config) |
| Seguridad | Solo jwt-sales (Security+JWT jjwt 0.12.5) | Ninguna | Ninguna | Ninguna (actuator expuesto) |
| Persistencia | JPA+MySQL `appdb` en los 3 | Ninguna (datos en memoria) | JPA+MySQL `appdb` en los 2 | Ninguna |
| Colas | products+sales: Kafka+Rabbit completos; jwt: ninguna | Ninguna | Superset: añade `sale-cancellation-requests` y `purchase-email-queue`, con flags `messaging.*` y `@ConditionalOnProperty` | Ninguna |
| Llamadas sync | Feign solo sales→products (URL fija) | `RestTemplate @LoadBalanced` recargas→cuentas | Doble Feign pedidos→productos/notificaciones | Feign pedidos→inventario |
| Resiliencia | Ninguna | Ninguna | Ninguna | Solo aquí: Resilience4j CircuitBreaker + fallback + endpoints demo falla/demora + K8s |
| Paquetes | `com.cibertec.{jwtsalesservices,productsservices,salesservices}` | `com.curso.*`, `com.demo.eurekaclient`, `com.edu.ms_notificaciones` | `com.cibertec.{mspedidos,msnotificaciones}` | `com.cibertec.resiliencia.{pedidos,inventario}` |
| Estilo código | Capas ES (`rest/negocio/repositorio/entidades/dto`) + comentarios didácticos ES en products/sales | Mínimo (1 controller por ms) | Mismo layered ES + más entidades/consumers | `rest/service/client/dto` (service en singular), sin JPA |
| Puertos | 8081/8082/8083 | 8761/8762/8300/8100/8083 | 8082/8081 (**colisionan** con backend si corren juntos) | 8090/8091 (8090 **colisiona** con `kafka-ui` host) |
| Postman | `postman/Cibertec-*.json` (JWT + Flows + envs Local) | `spring_cloud/postman/sistema-fintech-tarjetas-prepago.*` + `spring_cloud/README.md` | `ms-pedidos/postman/`, `ms-notificaciones/postman/` + READMEs por ms | `postman/resiliencia-ms-pedidos-inventario.*` + `README.md` + `deploy.sh` |
