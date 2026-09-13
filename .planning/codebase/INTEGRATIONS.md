# INTEGRATIONS

> Mapeo TECH verificado archivo por archivo. Rutas reales, sin secretos copiados
> (solo nombres de propiedades / variables de entorno).

## Comunicación inter-servicios (REST, Feign/OpenFeign, gateway routes, eureka)

- **Dominio `backend/` — Feign directo sin Eureka.**
  - Emisor: `backend/sales-services/.../client/ProductClient.java:8-15` →
    `@FeignClient(name="products-services", url="http://localhost:8081")`,
    `GET /products/{id}`. Habilitado por `@EnableFeignClients` en
    `backend/sales-services/.../SalesServicesApplication.java:8`.
  - Receptor: `backend/products-services/.../rest/ProductController.java:26-31` expone
    `GET /products`, `GET /products/{id}` (+ POST/PUT/DELETE). Coherente con el Feign.
  - Orden de arranque documentado: products (8081) antes que sales (8082)
    (`backend/sales-services/README.md:20-23`, `backend/products-services/README.md:20-23`).
  - `backend/jwt-sales-services` (8083) es standalone: cero Feign, cero Eureka, cero
    llamadas a otros servicios (verificado por ausencia en su `build.gradle:18-28` y en
    `README.md:5` "No usa Kafka ni RabbitMQ" — tampoco usa REST saliente).
- **Dominio `practica-examen/` — doble Feign directo sin Eureka.**
  - `ms-pedidos/.../client/NotificationClient.java:8-16` →
    `@FeignClient(name="ms-notificaciones", contextId="notificationClient",
    url="${notificaciones.base-url:http://localhost:8081}")`,
    `GET /notificaciones/mensajes/{mensajeId}`. URL externa por
    `practica-examen/ms-pedidos/.../application.yml:26-28`
    (`notificaciones.base-url: http://localhost:8081`,
    `notificaciones.mensaje-confirmacion-id: 2`; consumida con default `:1` en
    `ms-pedidos/.../negocio/SaleService.java:45`).
  - `ms-pedidos/.../client/ProductClient.java:8-15` → `@FeignClient(name="ms-notificaciones",
    url="http://localhost:8081")`, `GET /products/{id}` → **ROTO** (ver Brechas: el
    receptor no expone `/products/*`).
  - `@EnableFeignClients` en `practica-examen/ms-pedidos/.../MsPedidosApplication.java:8`.
- **Dominio `spring_cloud/` — Eureka + Gateway + RestTemplate balanceado.**
  - Descubrimiento: `spring_cloud/eureka-server/.../application.yml:6-11`
    (`register-with-eureka: false, fetch-registry: false`, puerto 8761). Dashboard en
    `http://localhost:8761` (`spring_cloud/README.md:49-55`).
  - Clientes registrados: `ms-cuentas` (8300), `ms-recargas` (8100),
    `ms-notificaciones` (8083, `application.yaml`), `api-gateway` (8762) — todos con
    `register-with-eureka: true, fetch-registry: true, defaultZone:
    http://localhost:8761/eureka/` en sus `application.yml/yaml`.
  - Gateway: `spring_cloud/api-gateway/.../application.yml:8-21` →
    `lb://ms-cuentas` con `Path=/cuentas/**`, `lb://ms-recargas` con `Path=/recargas/**`,
    `lb://ms-notificaciones` con `Path=/notificaciones/**` (con resto `"-"` huérfano en
    línea 21 — ver Brechas).
  - Llamada interna: `spring_cloud/ms-recargas/.../front/RecargaController.java:22-26`
    hace `GET http://ms-cuentas/cuentas/saldo/{cuentaId}` con `RestTemplate` creado
    `@LoadBalanced` en `ms-recargas/.../front/MsRecargasApplication.java:16-19`.
    Lógica: aprueba si `saldo >= monto`, responde `recarga_aprobada/motivo/atendido_por`
    (`RecargaController.java:28-38`). Cuentas `001/002/003` hardcodeadas en `ms-cuentas`
    (`spring_cloud/README.md:12`).
  - Orden de arranque: eureka → cuentas → recargas → gateway (`spring_cloud/README.md:16-45`).
- **Dominio `resiliencia-demo/` — Feign directo + Circuit Breaker, sin Eureka.**
  - `resiliencia-demo/ms-pedidos/.../client/InventarioClient.java:8-15` →
    `@FeignClient(name="ms-inventario", url="${inventario.base-url}")`,
    `GET /inventario/{productoId}`. URL por `resiliencia-demo/ms-pedidos/.../application.yml:15-16`
    (`inventario.base-url: http://localhost:8091`). `@EnableFeignClients` en
    `.../MsPedidosApplication.java:7`.
  - Protección: `@CircuitBreaker(name="inventario", fallbackMethod="fallbackCrearPedido")`
    en `.../service/PedidoService.java:30`; fallback responde estado
    `RECIBIDO_SIN_VALIDAR_STOCK` (ver `resiliencia-demo/README.md:141-152`).
    Timeouts Feign 1000/1000 ms (`application.yml:5-10`). Endpoints v1
    `POST /pedidos/sin-resiliencia` (sin protección, 500 si falla inventario) y v2
    `POST /pedidos/con-resiliencia` (ver `README.md:31-44,86+`).
  - `ms-inventario` expone `GET /inventario/{productoId}` + simuladores
    `POST /inventario/demo/falla/{true|false}` y `.../demora/{ms}` más
    `GET /inventario/demo/estado` (ver colección Postman § Postman).
- **Frontend:** sin integración con ningún backend. `frontend/hospital_web/src/environments/
  environment.ts:3` y `environment.development.ts:7` → `apiUrl: 'http://localhost:4200'`
  (loopback al propio dev-server). No hay interceptores/gateways hacia :8081/:8082/:8083.

## Messaging (Kafka / RabbitMQ: exchanges, topics, queues según compose + código)

Infra (ver `queue/`):
- `queue/docker-compose-kafka.yml`: `confluentinc/cp-zookeeper:7.4.0` (2181) +
  `confluentinc/cp-kafka:7.4.0` (9092 externo / 29092 interno,
  `KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"`) + `provectuslabs/kafka-ui:latest`
  (`kafka-ui:8090→8080`, cluster `ligo-local`). Contenedores `ligo-zookeeper/kafka/kafka-ui`.
- `queue/docker-compose-rabbitmq.yml`: `rabbitmq:3-management` (AMQP 5672, consola 15672;
  usuario default por compose). Contenedor `ligo-rabbitmq`, volumen `rabbitmq_data`.

Dominio `backend/` (siempre activo, sin flags):
- **Kafka topic `stock-movements`** (3 particiones, 1 réplica):
  `backend/sales-services/.../kafka/KafkaTopicConfig.java:13-23` (productor) y consumo en
  `backend/products-services/.../kafka/`: `StockUpdateConsumer.java:24`
  (`topics="stock-movements"`, `groupId="stock-update-cg"`),
  `StockReportingConsumer.java:17` (`groupId="reporting-cg"`),
  `StockAuditConsumer.java:17` (`groupId="audit-cg"`) — fan-out por grupos.
  Producer: `backend/sales-services/.../kafka/StockMovementProducer.java:26`
  (`kafkaTemplate.send(STOCK_MOVEMENTS_TOPIC, key, event)`). Consumer deserializa con
  `JsonDeserializer`, `trusted.packages: "com.cibertec.productsservices.kafka,
  com.cibertec.salesservices.kafka"`, `use.type.headers: false`, default type
  `...productsservices.kafka.StockMovementEvent`
  (`backend/products-services/.../application.yml:12-21`); bootstrap `localhost:9092`.
- **RabbitMQ `DirectExchange stock-exchange`** declarado en AMBOS lados
  (`backend/*/rabbitmq/RabbitMQConfig.java:17-21`): colas `stock-reserve-queue`
  (routing `stock.reserve`) y `stock-low-queue` (routing `stock.low`),
  converter `Jackson2JsonMessageConverter` (`RabbitMQConfig.java:63-65`).
  Flujo: `sales-services/.../rabbitmq/StockReserveProducer.java:22-24`
  (`convertAndSend(STOCK_EXCHANGE, STOCK_RESERVE_ROUTING_KEY, event)`)
  → `products-services/.../rabbitmq/StockReserveConsumer.java:26`
  (`@RabbitListener(queues=STOCK_RESERVE_QUEUE)`) → ante stock bajo
  `products-services/.../rabbitmq/StockLowAlertProducer.java:22-24`
  (`convertAndSend(STOCK_EXCHANGE, STOCK_LOW_ROUTING_KEY, ...)`)
  → `sales-services/.../rabbitmq/StockLowAlertConsumer.java:17`
  (`@RabbitListener(queues=STOCK_LOW_QUEUE)`, solo log warn). Conexión
  `localhost:5672` guest/guest en ambos `application.yml:22-26 / 17-21`.

Dominio `practica-examen/` (activable por flags, ver Config externa):
- **Kafka topics `stock-movements` + `sale-cancellation-requests`** (3 particiones c/u):
  `practica-examen/ms-pedidos/.../kafka/KafkaTopicConfig.java:15-34` (bean condicionado a
  `messaging.kafka.enabled`). Consumers en `ms-notificaciones/.../kafka/`:
  `StockUpdateConsumer:26` (`stock-update-cg`), `StockReportingConsumer:19`
  (`reporting-cg`), `StockAuditConsumer:19` (`audit-cg`) sobre `stock-movements`, más
  `SaleCancellationConsumer.java:22-26` (`topics="sale-cancellation-requests"`,
  `groupId="sale-cancellation-cg"`, factory dedicada) → persiste vía
  `SaleCancellationService` y se expone en `GET /notificaciones/ventas/{saleId}/anulaciones/logs`
  (`NotificacionesController.java:64`).
- **RabbitMQ `stock-exchange` + `notification-exchange`** (`DirectExchange`):
  `practica-examen/ms-pedidos/.../rabbitmq/RabbitMQConfig.java:19-26` →
  colas `stock-reserve-queue`, `stock-low-queue`, `purchase-email-queue`;
  keys `stock.reserve`, `stock.low`, `email.purchase.thanks`. Producers
  `StockReserveProducer` (flag `messaging.rabbitmq.stock.enabled`) y
  `PurchaseEmailProducer` (flag `messaging.rabbitmq.enabled`); consumers
  `StockReserveConsumer:28`, `CorreoEnviadoConsumer:23` (cola purchase-email) y
  `StockUpdateConsumer` del lado notificaciones. OJO: con la config actual
  `messaging.rabbitmq.stock.enabled: false` en ms-pedidos, los beans de stock NO se
  crean en el emisor (ver Brechas).

Dominios sin mensajería: `spring_cloud/*` (cero `spring-kafka`/`amqp` en sus
`pom.xml`/`build.gradle`), `resiliencia-demo/*` (README.md:3 "No usa Eureka, RabbitMQ
ni Kafka"), `backend/jwt-sales-services` (README.md:5), `frontend/`.

## Persistencia (qué DB usa cada servicio, credenciales vía .env — NO copies secrets, solo nombres de vars)

- **MySQL 8.0 compartido `appdb` en `localhost:5510`** para 5 servicios:
  `backend/jwt-sales-services`, `backend/products-services`, `backend/sales-services`,
  `practica-examen/ms-pedidos`, `practica-examen/ms-notificaciones`.
  Evidencia DSN: `jdbc:mysql://localhost:5510/appdb?...serverTimezone=America/Lima` en los
  5 `application.yml:5` respectivos, con `jpa.hibernate.ddl-auto: update` + `show-sql: true`.
  Las 5 apps comparten schema físico (colisiones de tablas posibles entre dominios —
  ver Brechas). Sin Flyway/Liquibase en ningún build (migraciones inexistentes).
- **Variables (nombres, sin valores):** compose `database/docker-compose.yml` parametriza
  con `DB_IMAGE`, `DB_HOST` (container_name), `DB_PORT` (host:5510→3306), `DB_NAME`,
  `DB_USER`, `DB_PASSWD` (también como `MYSQL_ROOT_PASSWORD`), `DB_IMAGE_PHPMA`,
  `DB_HOST_PHPMA`, `DB_PORT_PHPMA`; definidos en `database/.env:1-11`. Los `application.yml`
  de las apps repiten literales equivalentes (host/puerto/base/usuario) en claro en lugar
  de referenciar el `.env` (acoplamiento por duplicación).
- **phpMyAdmin:** servicio `phpmyadmin` (`database/docker-compose.yml:18-26`), puerto
  host `${DB_PORT_PHPMA}`, red `net-mobile`, volumen `my-db:/var/lib/mysql`.
- **Sin persistencia:** `spring_cloud/*` (memoria hardcodeada en ms-cuentas;
  `README.md:12-14`), `resiliencia-demo/*` (sin datasource en sus `application.yml`),
  `frontend/` (sin BD local; artefactos ignorados por `.gitignore`: `*.db/sqlite*`).

## Auth (JWT: dónde se emite, dónde se valida, filtros)

Ámbito exclusivo: `backend/jwt-sales-services` (puerto 8083). Ningún otro módulo emite o
valida JWT (cero `jjwt`/security en sus builds — verificado).

- **Emisión:** `.../security/JwtService.java:28-39` (`generateToken(AppUser)`: `subject`
  = username, claims `userId`/`role`, `issuedAt`/`expiration` = ahora + `expiration-minutes`,
  firma HMAC con `Keys.hmacShaKeyFor(secret)`). Secret y expiración inyectados de
  `app.jwt.secret` / `app.jwt.expiration-minutes`
  (`JwtService.java:21-27`, definidos en `.../application.yml:16-19` con expiración 60).
  Endpoint `POST /auth/login` (público) + `POST /users` (público); resto autenticado —
  ver `.../security/SecurityConfig.java:31-35`.
- **Validación:** `.../security/JwtAuthenticationFilter.java:18-60`
  (`extends OncePerRequestFilter`): lee `Authorization: Bearer <token>`, `extractUsername`
  + `isTokenValid` (`JwtService.java:42-48`, parse con `verifyWith(secretKey)`), carga
  usuario de `AppUserRepository` y fija `UsernamePasswordAuthenticationToken` con
  `ROLE_<role>` en `SecurityContextHolder`; ante token inválido limpia el contexto y
  continúa la cadena (el `SecurityFilterChain` deniega después).
- **Cadena:** `SecurityConfig.java:27-38` (`@EnableWebSecurity`, CSRF off, sesión
  `STATELESS`, filtro JWT antes de `UsernamePasswordAuthenticationFilter`) + beans
  `AuthenticationManager` y `BCryptPasswordEncoder` (`SecurityConfig.java:40-48`).
  Endpoints protegidos verificados en `README.md:11-14`:
  `GET /users`, `PUT /users/{id}`, `DELETE /users/{id}`, `GET /sales`.
- **Brecha de alcance:** el JWT de :8083 NO es aceptado por :8081/:8082 (products/sales y
  practica-examen no tienen filtro JWT); cada dominio tiene su propio puerto y su propio
  modelo de auth (o ninguno).

## Config externa (application.yml por servicio, variables de entorno)

| Servicio (archivo) | Puerto | Config externa relevante (claves, no valores sensibles) |
|---|---|---|
| `backend/jwt-sales-services/.../application.yml` | 8083 | `spring.datasource.url/username/password`, `spring.jpa.*`, `server.port`, `app.jwt.secret`, `app.jwt.expiration-minutes` |
| `backend/products-services/.../application.yml` | 8081 | `spring.datasource.*`, `spring.jpa.*`, `spring.kafka.bootstrap-servers` + consumer `JsonDeserializer`/`trusted.packages`/`default.type`, `spring.rabbitmq.host/port/username/password`, `server.port` |
| `backend/sales-services/.../application.yml` | 8082 | `spring.datasource.*`, `spring.jpa.*`, `spring.kafka.producer` (serializers), `spring.rabbitmq.*`, `server.port` |
| `practica-examen/ms-pedidos/.../application.yml` | 8082 | anterior + `notificaciones.base-url`, `notificaciones.mensaje-confirmacion-id`, `messaging.kafka.enabled`, `messaging.rabbitmq.enabled`, `messaging.rabbitmq.stock.enabled` (flags `@ConditionalOnProperty`, ver `RabbitMQConfig.java:16`, `KafkaTopicConfig.java:12`) |
| `practica-examen/ms-notificaciones/.../application.yml` | 8081 | `spring.datasource.*`, `spring.jpa.*`, consumer Kafka `trusted.packages/default.type`, `spring.rabbitmq.*`, `messaging.consumers.enabled` (condiciona `SaleCancellationConsumer.java:11`), `server.port` |
| `spring_cloud/eureka-server/.../application.yml` (+ `application.properties` vacío 1 línea) | 8761 | `spring.application.name`, `server.port`, `eureka.client.*`, `eureka.instance.*` |
| `spring_cloud/api-gateway/.../application.yml` | 8762 | `spring.application.name`, `spring.cloud.gateway.routes[3]` (`id/uri lb://*/predicates Path`), `eureka.client.service-url.defaultZone` |
| `spring_cloud/ms-cuentas/.../application.yml` | 8300 | `server.port`, `spring.application.name`, `eureka.client.*` |
| `spring_cloud/ms-recargas/.../application.yml` | 8100 | idem cuentas (nombre/puerto propios) |
| `spring_cloud/ms-notificaciones/.../application.yaml` (nótese **.yaml**, no .yml) | 8083 | idem cuentas (nombre/puerto propios) |
| `resiliencia-demo/ms-pedidos/.../application.yml` | 8090 | `spring.application.name`, `spring.cloud.openfeign.client.config.default.connectTimeout/readTimeout`, `inventario.base-url`, `management.health/endpoints` (CB), `resilience4j.circuitbreaker.instances.inventario.*`, `server.port` |
| `resiliencia-demo/ms-inventario/.../application.yml` | 8091 | `spring.application.name`, `server.port`, `management.endpoints.web.exposure.include` |
| `frontend/hospital_web/src/environments/environment{,.development}.ts` | 4200 | `production`, `apiUrl` (loopback, sin backend) |
| `database/.env` + `docker-compose.yml` | — | `DB_IMAGE/DB_HOST/DB_PORT/DB_NAME/DB_USER/DB_PASSWD/DB_IMAGE_PHPMA/DB_HOST_PHPMA/DB_PORT_PHPMA` → `MYSQL_DATABASE/MYSQL_USER/MYSQL_PASSWORD/MYSQL_ROOT_PASSWORD`, volúmenes, red `net-mobile` |
| `queue/docker-compose-*.yml` | — | Kafka: `KAFKA_BROKER_ID/ZOOKEEPER_CONNECT/LISTENERS/ADVERTISED_LISTENERS/.../AUTO_CREATE_TOPICS_ENABLE`, `KAFKA_CLUSTERS_0_*` (UI); Rabbit: `RABBITMQ_DEFAULT_USER/RABBITMQ_DEFAULT_PASS` |

Sin Config Server / Vault / `.env.example` (el `.gitignore:8-10` contempla `!.env.example`
pero ese archivo **no existe** — verificado por glob). Todo secreto/demo vive en claro en
`application.yml` o `database/.env` (ambos commiteados).

## Postman (qué flujos cubre cada colección)

- `postman/Cibertec-JWT-Sales.postman_collection.json` (**"Cibertec JWT Sales"**,
  env `postman/Cibertec-JWT-Local.postman_environment.json` con `jwt_base_url`,
  `jwt_token`, `jwt_user_id`): ① usuarios públicos `POST {{jwt_base_url}}/users` +
  `POST /auth/login` (obtener JWT); ② ventas protegidas `GET /sales` sin token (debe
  fallar 401/403) y con JWT; ③ usuarios protegidos `GET /users`, `PUT /users/{{jwt_user_id}}`.
  Cubre emisión + validación del § Auth contra :8083.
- `postman/Cibertec-Microservices-Flows.postman_collection.json`
  (**"Cibertec Microservices Flows"**, env `postman/Cibertec-Local.postman_environment.json`
  con `sales_base_url`, `products_base_url`, `product_id`, `feign_sale_id`, `last_sale_id`,
  `last_rabbit_sale_id`): ⓪ health/seed (`GET /products`, `GET /sales`); ① Kafka
  (`POST /sales` → `GET /sales/{{last_sale_id}}`); ② RabbitMQ
  (`POST /sales/rabbit-reserve` → `GET /sales/{{last_rabbit_sale_id}}`); ③ Feign
  (`GET /sales/{{feign_sale_id}}/details`); ④ CRUD productos (`POST/GET /products`).
  Cubre los 3 mecanismos del § Messaging + Feign del dominio `backend/` (:8081/:8082).
- `spring_cloud/postman/sistema-fintech-tarjetas-prepago.postman_collection.json`
  (**"Sistema Fintech Tarjetas Prepago"**, 7 requests): Eureka dashboard
  (`GET {{eurekaUrl}}`), apps registradas (`GET {{eurekaUrl}}/eureka/apps`), saldos
  directos (`{{cuentasUrl}}/cuentas/saldo/001`), recargas aprobada/rechazada directas
  (`{{recargasUrl}}/recargas/procesar/001/100`, `.../003/50`) y equivalentes vía gateway
  (`{{gatewayUrl}}/cuentas/saldo/002`, `{{gatewayUrl}}/recargas/procesar/001/50`).
  Cubre § Eureka/Gateway/RestTemplate (puertos 8761/8300/8100/8762).
- `resiliencia-demo/postman/resiliencia-ms-pedidos-inventario.postman_collection.json`
  (**"Resiliencia - ms-pedidos y ms-inventario"**): ⓪ estado
  (`GET /inventario/demo/estado`); ① inventario sano (falla/false, demora/0, stock
  directo); ② v1 sin resiliencia (pedido OK → activar falla → pedido 500); ③ v2 con
  resiliencia (pedido fallback, repetir ×5 para abrir CB, `GET /actuator/circuitbreakers`,
  `GET /actuator/circuitbreakerevents`); ④ timeout (demora 2000 → pedido degradado →
  demora 0). Cubre § Feign+CB (:8090/:8091).
- `practica-examen/ms-pedidos/postman/ms-pedidos.postman_collection.json`
  (**"ms-pedidos"**, vars `baseUrl`, `notificacionesBaseUrl`, `mensajeId`, `saleId`):
  preparación (`POST/GET /notificaciones/mensajes`), ventas CRUD
  (`POST/GET /sales`, `GET /sales/{{saleId}}`, `GET /sales/{{saleId}}/details`,
  `POST /sales/rabbit-reserve`, `PUT /sales/{{saleId}}`), anulación Kafka
  (`DELETE`-semántico `.../sales/{{saleId}}` → evento `sale-cancellation-requests`) y
  verificación (`GET {{notificacionesBaseUrl}}/notificaciones/ventas/{{saleId}}/anulaciones/logs`).
  Cubre Feign + Kafka-anulación + Rabbit del dominio practica.

## Brechas (integraciones rotas o sin configurar)

1. **Feign roto en practica-examen (404 garantizado).**
   `practica-examen/ms-pedidos/.../client/ProductClient.java:8-15` pide
   `GET http://localhost:8081/products/{id}`, pero `ms-notificaciones` solo expone
   `/notificaciones`, `/notificaciones/{id}`, `/notificaciones/mensajes*`,
   `/notificaciones/ventas/*/anulaciones/logs`
   (`practica-examen/ms-notificaciones/.../rest/NotificacionesController.java:24-64`).
   No existe `@RequestMapping("/products")` en ese servicio (grep negativo verificado).
   Además duplica `name="ms-notificaciones"` con `NotificationClient` (dos Feigns con
   mismo nombre, distinto `contextId` solo en uno).
2. **Mensajería stock desactivada por defecto en el emisor de practica.**
   `practica-examen/ms-pedidos/.../application.yml:35-36`
   (`messaging.rabbitmq.stock.enabled: false`) + `@ConditionalOnProperty(...stock.enabled)`
   en `RabbitMQConfig.java:31-73`, `StockReserveProducer.java:12`,
   `StockLowAlertConsumer.java:12`, `SaleReserveService.java:12` → con la config
   commiteada NO se crean exchange/colas/producers de stock en ms-pedidos, mientras el
   lado notificaciones sí declara sus consumers sin ese flag (p. ej.
   `StockReserveConsumer.java:28`). Flujo `rabbit-reserve` de practica inoperante hasta
   activar el flag.
3. **Colisión de puertos 8081/8082 entre dominios independientes.**
   `backend/products-services` y `practica-examen/ms-notificaciones` usan **8081**;
   `backend/sales-services` y `practica-examen/ms-pedidos` usan **8082** (sus 4
   `application.yml:server.port`). No pueden levantarse los dos dominios a la vez.
   Colisión adicional: `backend/jwt-sales-services` y `spring_cloud/ms-notificaciones`
   comparten **8083**; y `kafka-ui` (`queue/docker-compose-kafka.yml:50`, host 8090)
   colisiona con `resiliencia-demo/ms-pedidos` (**8090**).
4. **Cinco servicios contra la MISMA base `appdb` con `ddl-auto: update`.**
   `backend/*` ×3 + `practica-examen/*` ×2 comparten `localhost:5510/appdb`
   (ver § Persistencia). Entidades homónimas (`Sale`, `Product`, `Mensaje*`) de dominios
   distintos comparten schema → riesgo de tablas/columnas mezcladas y pérdida de datos
   al alternar dominios. Sin migraciones que lo ordenen.
5. **Divergencia Spring Boot 4 vs 3 en `spring_cloud`.**
   `spring_cloud/ms-notificaciones/build.gradle` (Boot **4.0.6**, Cloud **2025.1.1**,
   Gradle **9.5.1**, artefacto `starter-webmvc`, config `application.yaml`) frente al resto
   del mismo dominio (Boot 3.2.5, Cloud 2023.0.1, Maven, `application.yml`). Riesgo de
   incompatibilidad de starters/transitivos y de confusión de build (dos sistemas en un
   mismo "proyecto"). El gateway además enruta a este servicio (`/notificaciones/**`)
   cuyo controlador real (`.../NotificacionController.java:12`) sí coincide en path, pero
   el servicio Boot 4 nunca fue validado contra el gateway Boot 3 en este repo.
6. **Gateway con resto YAML sospechoso + eureka con config duplicada vacía.**
   `spring_cloud/api-gateway/.../application.yml:21` termina la lista de rutas con un
   `"-"` huérfano (`Path=/notificaciones/**            -`) — revisar parseo.
   `spring_cloud/eureka-server/.../application.properties` existe pero está vacío
   (1 línea en blanco) duplicando a `application.yml` (fuente real).
7. **Frontend desconectado + README desactualizado + sin lockfile.**
   `apiUrl: localhost:4200` (loopback), `README.md:3` declara CLI 15.1.5 (real: 21),
   y no hay `package-lock.json` → builds npm no reproducibles.
8. **Secretos demo commiteados + `.env` commiteado + sin `.env.example`.**
   `app.jwt.secret` en claro en `backend/jwt-sales-services/.../application.yml:18`,
   credenciales literales en los 5 `application.yml` (datasource + rabbitmq guest/guest),
   y `database/.env` commiteado pese a que `/.gitignore:8` ignora `.env`
   (fue forzado o anterior al ignore). `.env.example` no existe aunque el ignore lo
   contempla (`/.gitignore:10`).
9. **Sin compose/K8s unificado y warrants de arranque frágiles.**
   No hay compose raíz ni script que levante MySQL+Kafka+Rabbit+servicios en orden;
   cada README indica orden manual distinto. `resiliencia-demo/` es el único con
   Dockerfiles + `k8s/` + `deploy.sh`, pero sin wrapper Maven (`mvnw` ausente, `mvn` no
   instalado en este entorno) y con dependencia de `minikube/kubectl/argocd`.
10. **Observabilidad y tolerancia a fallos ausentes fuera de resiliencia-demo.**
    Solo `resiliencia-demo/ms-pedidos` tiene Actuator + Resilience4j + timeouts Feign.
    `backend/sales-services` y `practica-examen/ms-pedidos` llaman por Feign sin
    timeouts ni fallback (respuestas `{"message":"... no disponible"}` artesanales en
    controladores, no CB). `spring_cloud/ms-recargas` llama con `RestTemplate` sin
    timeout/reintento/CB (NPE si `ms-cuentas` responde inesperado:
    `RecargaController.java:28` desreferencia sin null-check).
