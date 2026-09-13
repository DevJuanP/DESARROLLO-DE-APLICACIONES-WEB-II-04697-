# STRUCTURE

## Árbol top-level (con descripción de 1 línea por carpeta)

```
DESARROLLO-DE-APLICACIONES-WEB-II-04697-/
├── backend/                 # 3 Spring-Boot Gradle sin discovery: jwt-sales, products, sales
├── frontend/hospital_web/   # Plantilla Angular 21 "Cliniva" (mock local, sin backend real)
├── spring_cloud/            # Demo fintech Maven: eureka-server, api-gateway, ms-cuentas, ms-recargas, ms-notificaciones
├── practica-examen/         # Variante Gradle ms-pedidos + ms-notificaciones con Kafka/Rabbit y doble Feign
├── resiliencia-demo/        # Demo Maven Feign + Resilience4j: ms-pedidos + ms-inventario (+k8s, deploy.sh)
├── database/                # docker-compose MySQL:5510 + phpMyAdmin:3410 y .env
├── queue/                   # docker-compose Kafka:9092 (+UI:8090) y RabbitMQ:5672 (+mgmt:15672)
├── postman/                 # Collections raíz: Cibertec-JWT-Sales y Cibertec-Microservices-Flows + 2 envs
└── .planning/codebase/      # Documentos de mapa (ARCHITECTURE.md, STRUCTURE.md)
```

Se ignoran `**/bin/**`, `**/build/**`, `*.class`, `*.jar`, `.git/` (solo
compilados Gradle y wrappers).

## backend/ por servicio (árbol src hasta 3 niveles + entry point + configs)

### backend/jwt-sales-services (Gradle, puerto 8083)

```
backend/jwt-sales-services/
├── build.gradle / settings.gradle / gradlew(.bat) / gradle/wrapper/ / README.md
└── src/main/
    ├── java/com/cibertec/jwtsalesservices/
    │   ├── JwtSalesServicesApplication.java   # @SpringBootApplication (entry point)
    │   ├── rest/        # AuthController, SaleController, UserController
    │   ├── negocio/     # AuthService, SaleService, UserService
    │   ├── repositorio/ # AppUserRepository, SaleRepository
    │   ├── entidades/   # AppUser(app_users), Sale(secure_sales)
    │   ├── dto/         # LoginRequest, LoginResponse, UserRequest, UserResponse, SaleResponse
    │   ├── security/    # JwtService, JwtAuthenticationFilter, SecurityConfig
    │   └── config/      # SaleDataInitializer
    └── resources/application.yml  # server.port 8083, MySQL localhost:5510/appdb, app.jwt.secret/expiration-minutes:60
```

- Entry: `backend/jwt-sales-services/src/main/java/com/cibertec/jwtsalesservices/JwtSalesServicesApplication.java`
- Config: `backend/jwt-sales-services/src/main/resources/application.yml`

### backend/products-services (Gradle, puerto 8081)

```
backend/products-services/
├── build.gradle / settings.gradle / gradlew(.bat) / gradle/wrapper/
└── src/
    ├── main/
    │   ├── java/com/cibertec/productsservices/
    │   │   ├── ProductsServicesApplication.java  # entry point
    │   │   ├── rest/        # ProductController (/products CRUD)
    │   │   ├── negocio/     # ProductService (decreaseStock synchronized)
    │   │   ├── repositorio/ # ProductRepository
    │   │   ├── entidades/   # Product(products)
    │   │   ├── dto/         # ProductResponse
    │   │   ├── kafka/       # StockMovementEvent, StockUpdateConsumer, StockAuditConsumer, StockReportingConsumer
    │   │   ├── rabbitmq/    # RabbitMQConfig, StockReserveConsumer/Event, StockLowAlertProducer/Event
    │   │   ├── config/      # ProductDataInitializer
    │   │   └── client/      # package-info.java (sin Feign en este ms)
    │   └── resources/application.yml  # port 8081, MySQL appdb, kafka localhost:9092, rabbitmq localhost:5672
    └── test/java/com/cibertec/productsservices/ProductsServicesApplicationTests.java
```

- Entry: `backend/products-services/src/main/java/com/cibertec/productsservices/ProductsServicesApplication.java`
- Config: `backend/products-services/src/main/resources/application.yml`

### backend/sales-services (Gradle + OpenFeign, puerto 8082)

```
backend/sales-services/
├── build.gradle (suma spring-cloud-starter-openfeign 2023.0.1) / settings.gradle / gradlew(.bat)
└── src/
    ├── main/
    │   ├── java/com/cibertec/salesservices/
    │   │   ├── SalesServicesApplication.java  # @EnableFeignClients (entry point)
    │   │   ├── rest/        # SaleController (/sales, /sales/{id}/details, /sales/rabbit-reserve)
    │   │   ├── negocio/     # SaleService
    │   │   ├── repositorio/ # SaleRepository
    │   │   ├── entidades/   # Sale
    │   │   ├── dto/         # SaleRequest, SaleResponse, SaleWithProductResponse, ProductResponse, ErrorResponse
    │   │   ├── client/      # ProductClient (Feign products-services → http://localhost:8081)
    │   │   ├── kafka/       # KafkaTopicConfig(STOCK_MOVEMENTS_TOPIC=stock-movements), StockMovementProducer/Event
    │   │   ├── rabbitmq/    # RabbitMQConfig(stock-exchange/queues/keys), StockReserveProducer, StockLowAlertConsumer/Event
    │   │   └── config/      # SaleDataInitializer
    │   └── resources/application.yml  # port 8082, MySQL appdb, kafka producer, rabbitmq
    └── test/java/com/cibertec/salesservices/SalesServicesApplicationTests.java
```

- Entry: `backend/sales-services/src/main/java/com/cibertec/salesservices/SalesServicesApplication.java`
- Config: `backend/sales-services/src/main/resources/application.yml`

## frontend/hospital_web (árbol src clave)

```
frontend/hospital_web/
├── package.json (name cliniva, version 22.0.0, @angular/* ^21) / angular.json
└── src/
    ├── main.ts / index.html / styles.scss / manifest.webmanifest / favicon.ico
    ├── environments/            # environment.ts + environment.development.ts (apiUrl http://localhost:4200)
    ├── assets/
    └── app/
        ├── app.component.ts/html/scss/spec.ts / app.config.ts / app.routes.ts (APP_ROUTE)
        ├── authentication/      # auth.routes.ts + signin/signup/forgot-password/locked/two-factor/coming-soon/maintenance/page404/page500
        ├── admin/               # admin.routes.ts → dashboard/dashboard.routes.ts → main/dashboard2/nurse-dashboard
        ├── doctor/              # doctor.routes.ts → dashboard/
        ├── patient/             # patient.routes.ts → dashboard/
        ├── extra-pages/         # extra-pages.routes.ts → blank/
        ├── multilevel/          # multilevel.routes.ts → first1/first2/first3/secondlevel/thirdlevel
        ├── layout/              # app-layout/auth-layout+main-layout, header, sidebar(+metadata/service), right-sidebar, page-loader, components
        ├── core/                # guard/auth.guard+module-import.guard, interceptor/error.interceptor, models(config.interface/interface/role),
        │                        # service(auth/login/token/token-factory/JWT/startup/language/direction/app-directionality/rightsidebar/helpers/ai),
        │                        # providers/directionality.provider, initializers.ts, index.ts
        ├── shared/              # components/ (activity-list, appointment-*, attendance-chart, bed-occupancy, blood-bank-stock, ...),
        │                        # services/(index/storage.service), utils/, table*.ts, sub-sink.ts
        └── config/              # config.service.ts + index.ts
```

- Routing raíz: `frontend/hospital_web/src/app/app.routes.ts`
- Guard: `frontend/hospital_web/src/app/core/guard/auth.guard.ts`
- Login mock: `frontend/hospital_web/src/app/core/service/login.service.ts`
  (`login()` → `http.get('/user')`), `frontend/hospital_web/src/app/core/service/auth.service.ts`,
  `frontend/hospital_web/src/app/core/service/token.service.ts`
- Envs: `frontend/hospital_web/src/environments/environment.ts`,
  `frontend/hospital_web/src/environments/environment.development.ts`

## spring_cloud / practica-examen / resiliencia-demo (qué contiene cada ms)

### spring_cloud/ (Maven salvo ms-notificaciones=Gradle; README raíz fintech)

```
spring_cloud/
├── README.md
├── postman/sistema-fintech-tarjetas-prepago.postman_collection.json
├── eureka-server/    # EurekaServerApplication (@EnableEurekaServer) + application.yml(8761)+application.properties(vacío) + pom(spring-cloud-starter-netflix-eureka-server) + DockerFile
├── api-gateway/      # ApiGatewayApplication + application.yml(8762, rutas /cuentas/**→lb://ms-cuentas, /recargas/**→lb://ms-recargas, /notificaciones/**→lb://ms-notificaciones) + pom(gateway+eureka-client)
├── ms-cuentas/       # MsCuentasApplication + CuentaController(GET /cuentas/saldo/{id}, hardcoded 001/002/003) + application.yml(8300) + pom(eureka-client)
├── ms-recargas/      # MsRecargasApplication(RestTemplate @LoadBalanced @Bean) + RecargaController(GET /recargas/procesar/{cuentaId}/{monto} → http://ms-cuentas/...) + application.yml(8100) + pom(eureka-client)
└── ms-notificaciones/ # MsNotificacionesApplication + NotificacionController(@RequestMapping /notificaciones, GET /enviar/{tipo}) + application.yaml(8083) + build.gradle(eureka-client) + settings.gradle
```

Entrys: `spring_cloud/eureka-server/src/main/java/com/curso/eurekaserver/EurekaServerApplication.java`,
`spring_cloud/api-gateway/src/main/java/com/curso/zuul/ApiGatewayApplication.java`,
`spring_cloud/ms-cuentas/src/main/java/com/demo/eurekaclient/MsCuentasApplication.java`,
`spring_cloud/ms-recargas/src/main/java/com/curso/front/MsRecargasApplication.java`,
`spring_cloud/ms-notificaciones/src/main/java/com/edu/ms_notificaciones/MsNotificacionesApplication.java`.

### practica-examen/ (Gradle; ms-pedidos :8082, ms-notificaciones :8081)

```
practica-examen/
├── ms-pedidos/  # MsPedidosApplication + rest/SaleController + negocio/SaleService+SaleReserveService + repositorio/SaleRepository + entidades/Sale
│                 # + dto(Error/SaleRequest/SaleResponse/SaleWithProduct/SaleWithNotification/Product/MensajeNotificacion)
│                 # + client/ProductClient+NotificationClient(Feign, url ${notificaciones.base-url})
│                 # + kafka(KafkaTopicConfig stock-movements+sale-cancellation-requests, StockMovementProducer, SaleCancellationProducer/Event)
│                 # + rabbitmq(RabbitMQConfig purchase-email-queue + stock condicional, PurchaseEmailProducer/Event, StockReserveProducer, StockLowAlertConsumer)
│                 # + config/SaleDataInitializer + application.yml(port 8082, notificaciones.base-url, messaging.* flags) + postman/ms-pedidos.postman_collection.json + README.md
└── ms-notificaciones/  # MsNotificacionesApplication + rest/NotificacionesController + negocio(Product/MensajeNotificacion/CorreoEnviado/SaleCancellation services)
                        # + repositorio(×5: CorreoEnviado/MensajeNotificacion/Product/SaleCancellationLog/Sale) + entidades(CorreoEnviado/MensajeNotificacion/Product/Sale/SaleCancellationLog)
                        # + kafka(SaleCancellationConsumer/Config/Event + StockAudit/Reporting/UpdateConsumers + StockMovementEvent)
                        # + rabbitmq(CorreoEnviadoConsumer/Event, RabbitMQConfig, StockLowAlertProducer, StockReserveConsumer)
                        # + dto(MensajeNotificacion/Product/SaleCancellationLog responses) + config/ProductDataInitializer
                        # + application.yml(port 8081, messaging.consumers.enabled) + postman/ms-notificaciones.postman_collection.json + README.md
```

Entrys: `practica-examen/ms-pedidos/src/main/java/com/cibertec/mspedidos/MsPedidosApplication.java`,
`practica-examen/ms-notificaciones/src/main/java/com/cibertec/msnotificaciones/MsNotificacionesApplication.java`.

### resiliencia-demo/ (Maven; ms-pedidos :8090, ms-inventario :8091; sin colas/Eureka)

```
resiliencia-demo/
├── README.md / deploy.sh
├── postman/resiliencia-ms-pedidos-inventario.postman_collection.json
├── k8s/  # ms-pedidos-deployment.yaml + ms-pedidos-service.yaml(:8090) + ms-inventario-deployment.yaml + ms-inventario-service.yaml(:8091)
├── ms-pedidos/      # MsPedidosApplication(@EnableFeignClients) + rest/PedidoController(POST /pedidos/sin-resiliencia|con-resiliencia)
│                    # + service/PedidoService(@CircuitBreaker(name=inventario, fallbackMethod=fallbackCrearPedido))
│                    # + client/InventarioClient(Feign url ${inventario.base-url} GET /inventario/{productoId})
│                    # + dto(PedidoRequest/PedidoResponse/StockResponse) + application.yml(port 8090, feign timeouts 1000ms, resilience4j inventario, actuator) + pom.xml + Dockerfile
└── ms-inventario/   # MsInventarioApplication + rest/InventarioController(GET /inventario/{productoId}, POST /inventario/demo/falla|demora, GET /inventario/demo/estado)
                     # + dto(StockResponse/DemoStateResponse) + application.yml(port 8091, actuator) + pom.xml + Dockerfile
```

Entrys: `resiliencia-demo/ms-pedidos/src/main/java/com/cibertec/resiliencia/pedidos/MsPedidosApplication.java`,
`resiliencia-demo/ms-inventario/src/main/java/com/cibertec/resiliencia/inventario/MsInventarioApplication.java`.

## Soporte (database, queue, postman)

```
database/  # docker-compose.yml (db mysql ${DB_*} 5510:3306 + phpmyadmin 3410:80, red net-mobile, vol my-db) + .env (mysql:8.0, appdb/app/password)
queue/     # docker-compose-kafka.yml (zookeeper:2181, kafka:9092, kafka-ui 8090:8080) + docker-compose-rabbitmq.yml (5672 + 15672, guest/guest)
postman/   # Cibertec-JWT-Sales.postman_collection.json + Cibertec-JWT-Local.postman_environment.json + Cibertec-Microservices-Flows.postman_collection.json + Cibertec-Local.postman_environment.json
spring_cloud/postman/ + practica-examen/*/postman/ + resiliencia-demo/postman/  # collections por demo (ver secciones anteriores)
```

## Dónde agregar código nuevo (tabla: quiero X → voy a Y)

| Quiero X | Voy a Y |
|---|---|
| Nuevo endpoint auth/usuarios/ventas JWT | `backend/jwt-sales-services/src/main/java/com/cibertec/jwtsalesservices/rest/` + lógica en `.../negocio/` + repo en `.../repositorio/` + DTO en `.../dto/` |
| Nueva regla login/token/permiso JWT | `backend/jwt-sales-services/src/main/java/com/cibertec/jwtsalesservices/security/` (`JwtService`, `JwtAuthenticationFilter`, `SecurityConfig`) y `.../negocio/AuthService.java` |
| Nueva entidad/tabla del monolito JWT | `backend/jwt-sales-services/src/main/java/com/cibertec/jwtsalesservices/entidades/` + repo espejo + `config/SaleDataInitializer.java` para seed |
| Nuevo endpoint productos | `backend/products-services/src/main/java/com/cibertec/productsservices/rest/ProductController.java` + `negocio/ProductService.java` |
| Nuevo consumer/producer Kafka stock | `backend/products-services/.../kafka/` o `backend/sales-services/.../kafka/` (`*Event`, `*Consumer`, `*Producer`, `KafkaTopicConfig`) |
| Nueva cola/binding RabbitMQ stock | `.../rabbitmq/RabbitMQConfig.java` (+ `*Producer/*Consumer/*Event`) en products-services, sales-services o practica-examen |
| Nueva venta con producto (sync) | `backend/sales-services/.../rest/SaleController.java` + `negocio/SaleService.java` + `client/ProductClient.java` |
| Nuevo pedido con notificación/cancelación | `practica-examen/ms-pedidos/.../rest|negocio|kafka|rabbitmq|client/` y su contraparte en `practica-examen/ms-notificaciones/.../kafka|rabbitmq|rest|negocio/` |
| Nueva notificación/mensaje/correo | `practica-examen/ms-notificaciones/src/main/java/com/cibertec/msnotificaciones/rest/NotificacionesController.java` + `negocio/` + `repositorio/` + `entidades/` |
| Nuevo micro con discovery o ruta gateway | `spring_cloud/<nuevo-ms>/src/main/java/...` (eureka-client) + ruta en `spring_cloud/api-gateway/src/main/resources/application.yml` + alta en Eureka `:8761` |
| Nueva llamada recargas→cuentas | `spring_cloud/ms-recargas/src/main/java/com/curso/front/RecargaController.java` (RestTemplate `@LoadBalanced`) |
| Nuevo endpoint con fallback resiliente | `resiliencia-demo/ms-pedidos/src/main/java/com/cibertec/resiliencia/pedidos/rest/PedidoController.java` + `service/PedidoService.java` (`@CircuitBreaker` + `fallbackCrearPedido`) + tuning en `src/main/resources/application.yml` (`resilience4j.*`) |
| Nuevo comportamiento stock simulado | `resiliencia-demo/ms-inventario/src/main/java/com/cibertec/resiliencia/inventario/rest/InventarioController.java` |
| Nueva pantalla/rol/ruta hospital | `frontend/hospital_web/src/app/<admin|doctor|patient|...>/` + ruta en `app.routes.ts` o `*.routes.ts` + guard `core/guard/auth.guard.ts` + `core/models/role.ts` |
| Nuevo servicio HTTP real del frontend | `frontend/hospital_web/src/app/core/service/` (hoy mock: `login.service.ts`) + URL en `src/environments/environment*.ts` + interceptor `core/interceptor/error.interceptor.ts` |
| Nuevo widget/tabla compartido | `frontend/hospital_web/src/app/shared/components/` + `shared/services/` + `shared/utils/` |
| Nueva infra local (DB/cola) | `database/docker-compose.yml` + `database/.env` o `queue/docker-compose-*.yml` |
| Nuevo flujo Postman | `postman/` (raíz) o `spring_cloud/postman/`, `practica-examen/*/postman/`, `resiliencia-demo/postman/` según la familia |

## Archivos clave (entry points, configs — con rutas exactas)

- Entries Spring: `backend/jwt-sales-services/src/main/java/com/cibertec/jwtsalesservices/JwtSalesServicesApplication.java`,
  `backend/products-services/src/main/java/com/cibertec/productsservices/ProductsServicesApplication.java`,
  `backend/sales-services/src/main/java/com/cibertec/salesservices/SalesServicesApplication.java`,
  `spring_cloud/eureka-server/src/main/java/com/curso/eurekaserver/EurekaServerApplication.java`,
  `spring_cloud/api-gateway/src/main/java/com/curso/zuul/ApiGatewayApplication.java`,
  `spring_cloud/ms-cuentas/src/main/java/com/demo/eurekaclient/MsCuentasApplication.java`,
  `spring_cloud/ms-recargas/src/main/java/com/curso/front/MsRecargasApplication.java`,
  `spring_cloud/ms-notificaciones/src/main/java/com/edu/ms_notificaciones/MsNotificacionesApplication.java`,
  `practica-examen/ms-pedidos/src/main/java/com/cibertec/mspedidos/MsPedidosApplication.java`,
  `practica-examen/ms-notificaciones/src/main/java/com/cibertec/msnotificaciones/MsNotificacionesApplication.java`,
  `resiliencia-demo/ms-pedidos/src/main/java/com/cibertec/resiliencia/pedidos/MsPedidosApplication.java`,
  `resiliencia-demo/ms-inventario/src/main/java/com/cibertec/resiliencia/inventario/MsInventarioApplication.java`
- Configs puertos spring: `backend/jwt-sales-services/src/main/resources/application.yml` (8083),
  `backend/products-services/src/main/resources/application.yml` (8081),
  `backend/sales-services/src/main/resources/application.yml` (8082),
  `spring_cloud/eureka-server/src/main/resources/application.yml` (8761),
  `spring_cloud/api-gateway/src/main/resources/application.yml` (8762 + rutas),
  `spring_cloud/ms-cuentas/src/main/resources/application.yml` (8300),
  `spring_cloud/ms-recargas/src/main/resources/application.yml` (8100),
  `spring_cloud/ms-notificaciones/src/main/resources/application.yaml` (8083),
  `practica-examen/ms-pedidos/src/main/resources/application.yml` (8082),
  `practica-examen/ms-notificaciones/src/main/resources/application.yml` (8081),
  `resiliencia-demo/ms-pedidos/src/main/resources/application.yml` (8090),
  `resiliencia-demo/ms-inventario/src/main/resources/application.yml` (8091)
- Seguridad JWT: `backend/jwt-sales-services/src/main/java/com/cibertec/jwtsalesservices/security/SecurityConfig.java`,
  `.../security/JwtAuthenticationFilter.java`, `.../security/JwtService.java`
- Clientes sync: `backend/sales-services/src/main/java/com/cibertec/salesservices/client/ProductClient.java`,
  `practica-examen/ms-pedidos/src/main/java/com/cibertec/mspedidos/client/NotificationClient.java`,
  `practica-examen/ms-pedidos/src/main/java/com/cibertec/mspedidos/client/ProductClient.java`,
  `resiliencia-demo/ms-pedidos/src/main/java/com/cibertec/resiliencia/pedidos/client/InventarioClient.java`
- Topologías colas: `backend/sales-services/src/main/java/com/cibertec/salesservices/kafka/KafkaTopicConfig.java`,
  `backend/sales-services/src/main/java/com/cibertec/salesservices/rabbitmq/RabbitMQConfig.java`,
  `backend/products-services/src/main/java/com/cibertec/productsservices/rabbitmq/RabbitMQConfig.java`
- Frontend: `frontend/hospital_web/src/app/app.routes.ts`, `frontend/hospital_web/src/main.ts`,
  `frontend/hospital_web/src/app/app.config.ts`, `frontend/hospital_web/src/app/core/guard/auth.guard.ts`,
  `frontend/hospital_web/src/environments/environment.ts`, `frontend/hospital_web/package.json`
- Infra: `database/docker-compose.yml`, `database/.env`, `queue/docker-compose-kafka.yml`,
  `queue/docker-compose-rabbitmq.yml`, `resiliencia-demo/k8s/ms-pedidos-deployment.yaml`,
  `resiliencia-demo/k8s/ms-pedidos-service.yaml`, `resiliencia-demo/k8s/ms-inventario-deployment.yaml`,
  `resiliencia-demo/k8s/ms-inventario-service.yaml`, `resiliencia-demo/deploy.sh`
- Docs/contratos: `spring_cloud/README.md`, `resiliencia-demo/README.md`,
  `practica-examen/ms-pedidos/README.md`, `practica-examen/ms-notificaciones/README.md`,
  `postman/Cibertec-JWT-Sales.postman_collection.json`, `postman/Cibertec-Microservices-Flows.postman_collection.json`
