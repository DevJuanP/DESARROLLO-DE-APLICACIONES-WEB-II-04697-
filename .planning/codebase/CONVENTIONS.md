# CONVENTIONS

## Paquetes y capas (tabla capa → responsabilidad → ejemplo real con ruta)

Verificado con `grep "^package "` sobre 139 archivos `*/src/main/*.java` (excluidos `bin/`, `build/`, `target/`).

| Capa (subpaquete) | Responsabilidad | Ejemplo real con ruta |
|---|---|---|
| Raíz `*Application` | Bootstrapping Spring Boot | `backend/sales-services/src/main/java/com/cibertec/salesservices/SalesServicesApplication.java` |
| `rest` | `@RestController`, expone HTTP/JSON, delega a `negocio`, retorna `ResponseEntity` | `backend/sales-services/src/main/java/com/cibertec/salesservices/rest/SaleController.java` |
| `negocio` | Lógica de negocio, validaciones, lanza `ResponseStatusException` | `backend/sales-services/src/main/java/com/cibertec/salesservices/negocio/SaleService.java` |
| `repositorio` | Spring Data JPA (`JpaRepository`) | `backend/sales-services/src/main/java/com/cibertec/salesservices/repositorio/SaleRepository.java` |
| `entidades` | `@Entity` JPA, Lombok `@Getter/@Setter/@Builder` | `backend/sales-services/src/main/java/com/cibertec/salesservices/entidades/Sale.java` |
| `dto` | `record` inmutables Request/Response | `backend/sales-services/src/main/java/com/cibertec/salesservices/dto/SaleRequest.java` |
| `config` | Seeders `CommandLineRunner` (`*DataInitializer`) | `backend/sales-services/src/main/java/com/cibertec/salesservices/config/SaleDataInitializer.java` |
| `client` | Feign `ProductClient` / `NotificationClient` / `InventarioClient` | `practica-examen/ms-pedidos/src/main/java/com/cibertec/mspedidos/client/ProductClient.java` |
| `kafka` | `*Event` + `*Producer`/`*Consumer` + `*TopicConfig`/`*KafkaConfig` | `backend/sales-services/src/main/java/com/cibertec/salesservices/kafka/StockMovementProducer.java` |
| `rabbitmq` | `RabbitMQConfig` + `*Event` + `*Producer`/`*Consumer` | `backend/sales-services/src/main/java/com/cibertec/salesservices/rabbitmq/RabbitMQConfig.java` |
| `security` | Solo `jwt-sales-services`: JWT filter/service/config | `backend/jwt-sales-services/src/main/java/com/cibertec/jwtsalesservices/security/SecurityConfig.java` |
| `service` (solo resiliencia) | Variante en inglés de `negocio` | `resiliencia-demo/ms-pedidos/src/main/java/com/cibertec/resiliencia/pedidos/service/PedidoService.java` |

Paquetes base reales por módulo:

- `com.cibertec.jwtsalesservices.*` — `backend/jwt-sales-services` (subpaquetes `config/dto/entidades/negocio/repositorio/rest/security`)
- `com.cibertec.productsservices.*` — `backend/products-services` (`client/config/dto/entidades/kafka/negocio/rabbitmq/repositorio/rest`)
- `com.cibertec.salesservices.*` — `backend/sales-services` (`client/config/dto/entidades/kafka/negocio/rabbitmq/repositorio/rest`)
- `com.cibertec.mspedidos.*` — `practica-examen/ms-pedidos` (`client/config/dto/entidades/kafka/negocio/rabbitmq/repositorio/rest`)
- `com.cibertec.msnotificaciones.*` — `practica-examen/ms-notificaciones` (`client/config/dto/entidades/kafka/negocio/rabbitmq/repositorio/rest`)
- `com.cibertec.resiliencia.pedidos.*` — `resiliencia-demo/ms-pedidos` (`client/dto/rest/service`, sin `repositorio/entidades/negocio`)
- `com.cibertec.resiliencia.inventario.*` — `resiliencia-demo/ms-inventario` (`dto/rest`, estado en memoria con `ConcurrentHashMap`, sin JPA)
- `com.curso.zuul` — `spring_cloud/api-gateway` (solo `ApiGatewayApplication`, sin capas)
- `com.curso.eurekaserver` — `spring_cloud/eureka-server` (solo `EurekaServerApplication`, sin capas)
- `com.demo.eurekaclient` — `spring_cloud/ms-cuentas` (`CuentaController` + `MsCuentasApplication`, sin capas, saldo en `Map` estático)
- `com.edu.ms_notificaciones` — `spring_cloud/ms-notificaciones` (`NotificacionController` + `MsNotificacionesApplication`, sin capas)
- `com.curso.front` — `spring_cloud/ms-recargas` (`RecargaController` + `MsRecargasApplication`, sin capas)

## Nomenclatura (clases, DTOs, endpoints, archivos frontend — con 2-3 ejemplos reales cada una)

Clases — sufijos estrictos (12 `*Controller`, 10 `*Repository`, ~13 `*Service`):

- `XController` en `rest` (o raíz en spring_cloud): `backend/sales-services/.../rest/SaleController.java`, `backend/products-services/.../rest/ProductController.java`, `resiliencia-demo/ms-pedidos/.../rest/PedidoController.java`
- `XService` en `negocio` (excepción: `service` en resiliencia): `backend/sales-services/.../negocio/SaleService.java`, `practica-examen/ms-notificaciones/.../negocio/MensajeNotificacionService.java`, `resiliencia-demo/ms-pedidos/.../service/PedidoService.java`
- `XRepository` en `repositorio` (todos `extends JpaRepository`, resiliencia-demo no tiene ninguno): `backend/jwt-sales-services/.../repositorio/AppUserRepository.java`, `practica-examen/ms-notificaciones/.../repositorio/MensajeNotificacionRepository.java`, `practica-examen/ms-notificaciones/.../repositorio/SaleCancellationLogRepository.java`
- `XxxApplication` por módulo: `JwtSalesServicesApplication`, `MsPedidosApplication`, `MsInventarioApplication`
- Mensajería `*Event/*Producer/*Consumer/*Config`: `rabbitmq/StockReserveEvent.java`, `rabbitmq/StockReserveProducer.java`, `rabbitmq/StockLowAlertConsumer.java`, `kafka/SaleCancellationRequestedEvent.java`, `kafka/KafkaTopicConfig.java`

DTOs — `record` con sufijo `Request` (input) / `Response` (output), errores como `ErrorResponse(String message)`:

- `backend/sales-services/.../dto/SaleRequest.java` → `record SaleRequest(Long productId, Integer quantity, Long customerId)`
- `backend/sales-services/.../dto/SaleResponse.java` → `record SaleResponse(Long saleId, Long productId, Integer quantity, Long customerId, String status)`
- `backend/sales-services/.../dto/SaleWithProductResponse.java`, `practica-examen/ms-pedidos/.../dto/SaleWithNotificationResponse.java`, `practica-examen/ms-pedidos/.../dto/MensajeNotificacionResponse.java`
- `backend/jwt-sales-services/.../dto/LoginRequest.java`, `LoginResponse.java`, `UserRequest.java`, `UserResponse.java`
- `resiliencia-demo/ms-pedidos/.../dto/PedidoRequest.java` → `record PedidoRequest(Long productoId, Integer cantidad, Long clienteId)`; `PedidoResponse.java` agrega `estado, mensaje, inventario`

Endpoints — sin prefijo global de clase (salvo 2 excepciones); path completo en cada método; plural del recurso (`/sales`, `/products`, `/users`, `/pedidos`, `/inventario`):

- `backend/sales-services/.../rest/SaleController.java`: `GET /sales`, `GET /sales/{id}`, `GET /sales/{id}/details`, `POST /sales`, `POST /sales/rabbit-reserve`, `PUT /sales/{id}`, `DELETE /sales/{id}`
- `backend/products-services/.../rest/ProductController.java`: `GET /products`, `GET /products/{id}`, `POST /products`, `PUT /products/{id}`, `DELETE /products/{id}`
- `backend/jwt-sales-services/.../rest/UserController.java`: `POST /users`, `GET /users`, `PUT /users/{id}`, `DELETE /users/{id}`; `.../rest/AuthController.java`: `POST /auth/login`; `.../rest/SaleController.java`: solo `GET /sales`
- Excepciones con `@RequestMapping` a nivel de clase: `practica-examen/ms-notificaciones/.../rest/NotificacionesController.java` (`@RequestMapping("/notificaciones")` + `GET /`, `GET /{id}`, `POST /`, `POST /mensajes`, `GET /mensajes/{id}`, `GET /ventas/{saleId}/anulaciones/logs`, `PUT /{id}`, `DELETE /{id}`) y `spring_cloud/ms-notificaciones/.../NotificacionController.java` (`@RequestMapping("/notificaciones")` + `GET /enviar/{tipo}`)
- Demos sin persistencia: `spring_cloud/ms-cuentas/.../CuentaController.java` (`GET /cuentas/saldo/{id}`), `spring_cloud/ms-recargas/.../RecargaController.java` (`GET /recargas/procesar/{cuentaId}/{monto}`), `resiliencia-demo/ms-inventario/.../rest/InventarioController.java` (`GET /inventario/{productoId}`, `POST /inventario/demo/falla/{activo}`, `POST /inventario/demo/demora/{millis}`, `GET /inventario/demo/estado`), `resiliencia-demo/ms-pedidos/.../rest/PedidoController.java` (`POST /pedidos/sin-resiliencia`, `POST /pedidos/con-resiliencia`)

Archivos frontend (`frontend/hospital_web`, plantilla Angular "Cliniva") — kebab-case, cuarteto por componente:

- `src/app/authentication/signin/signin.component.ts` + `signin.component.html` + `signin.component.scss` + `signin.component.spec.ts`
- `src/app/layout/header/header.component.ts`, `src/app/layout/sidebar/sidebar.service.ts`, `src/app/admin/dashboard/main/main.component.spec.ts`
- Servicios `*.service.ts` en minúsculas con guiones: `src/app/core/service/auth.service.ts`, `src/app/core/service/login.service.ts`, `src/app/core/service/token.service.ts`, `src/app/config/config.service.ts`, `src/app/shared/services/storage.service.ts`

## Idioma del código (qué está en español vs inglés, con ejemplos)

Mixto sistemático: infraestructura y framework en inglés, dominio/negocio y mensajes de error en español.

- En inglés: capas y keywords (`rest`, `repositorio` es la excepción, `negocio` es español; `config`, `client`, `security`, `dto`, `kafka`, `rabbitmq`), clases (`SaleController`, `ProductService`, `SaleRepository`), anotaciones y tipos Spring/JPA/Lombok, DTOs de sales/products (`SaleRequest.productId/quantity/customerId/status`, `Product.name/price/stock`), endpoints sales/products/users (`/sales`, `/products`, `/auth/login`)
- En español: subpaquetes `negocio`, `entidades`, `repositorio`; entidades de notificaciones (`practica-examen/ms-notificaciones/.../entidades/MensajeNotificacion.java`: `nombre`, `activo`; tabla `mensajes_notificacion`; `CorreoEnviado`, `SaleCancellationLog`); DTOs de resiliencia (`PedidoRequest.productoId/cantidad/clienteId`, `PedidoResponse.estado/mensaje`); métodos (`crearPedidoSinResiliencia`, `crearPedidoConResiliencia`, `consultarStock`, `registrarMensaje`, `cambiarFalla`); mensajes de error (`"Venta no encontrada"`, `"Producto no encontrado"`, `"Credenciales invalidas"`, `"El username ya existe"`, `"Stock insuficiente para procesar el movimiento"`, `"Inventario no disponible para la demo"`); comentarios didácticos con analogía AWS (`"// @RestController expone endpoints HTTP JSON del servicio. // En AWS esto equivale a un API Gateway..."` en `backend/sales-services/.../rest/SaleController.java`)
- Frontend en inglés: `signin/signup/forgot-password`, `AuthService/LoginService/TokenService`, `AuthGuard`, `Role.Admin/Doctor/Patient`, rutas `admin/doctor/patient/authentication`

## Estilo API REST (prefijos, verbos, códigos de estado, manejo de errores)

- Sin versionado (`/v1` no existe) y sin prefijo `/api`; recurso en plural directo (`/sales`, `/products`, `/users`, `/notificaciones`, `/pedidos`, `/inventario`). Solo `NotificacionesController` (practica + spring_cloud) usa `@RequestMapping` de clase.
- Verbos correctos: `GET` lectura (incluye agregados `GET /sales/{id}/details`, `GET /ventas/{saleId}/anulaciones/logs`), `POST` creación y acciones (`POST /sales/rabbit-reserve`, `POST /pedidos/con-resiliencia`, `POST /inventario/demo/falla/{activo}`), `PUT /{recurso}/{id}`, `DELETE /{recurso}/{id}`. Inconsistencia didáctica: `GET /recargas/procesar/{cuentaId}/{monto}` y `GET /notificaciones/enviar/{tipo}` mutan/ejecutan con GET.
- Códigos: `200 OK` vía `ResponseEntity.ok(...)` (mayoría, incluso `POST /sales` y `POST /pedidos/*` que deberían ser 201), `201 CREATED` solo en `POST /users` (`UserController.createUser`), `204 No Content` en todos los DELETE (`ResponseEntity.noContent().build()`). Errores vía `ResponseStatusException` en capa `negocio`/`service`: `404 NOT_FOUND` (`"Venta no encontrada"`, `"Producto no encontrado"`, `"Usuario no encontrado"`, `"Mensaje no encontrado"`), `401 UNAUTHORIZED` (`AuthService`: `"Credenciales invalidas"`), `409 CONFLICT` (`"El username ya existe"`, `"El email ya existe"`, `"Stock insuficiente..."`), `503 SERVICE_UNAVAILABLE` (demo inventario con falla activa).
- Sin `@ControllerAdvice` ni `@ExceptionHandler` en ningún módulo (verificado por grep). `ErrorResponse(String message)` existe como `record` en `backend/sales-services/.../dto/ErrorResponse.java` y `practica-examen/ms-pedidos/.../dto/ErrorResponse.java` pero **no se referencia** desde ningún controller/service (grep solo encuentra su propia declaración). `ProductController` mezcla estilos: `GET/POST/PUT` retornan el DTO crudo y solo `DELETE` usa `ResponseEntity`.
- Feign clients replican el path del proveedor: `ProductClient` (`@GetMapping("/products/{id}")`), `NotificationClient` (`@GetMapping("/notificaciones/mensajes/{mensajeId}")`), `InventarioClient` (`@GetMapping("/inventario/{productoId}")`).

## Commits y .gitignore (estado real + qué falta ignorar)

- `git log --oneline` = **1 solo commit**: `3b0df8c feat add initial commit`. Rama `main` sincronizada con `origin/main`, `git status --short` limpio. Sin historial por módulo ni convención observable más allá de ese mensaje.
- `.gitignore` (27 líneas, raíz) cubre: `.DS_Store`, `.idea/`, `*.iml`, `.vscode/`, `.env*`, `**/.gradle/`, `**/build/`, `**/bin/`, `*.class`, `*.jar`, `*.war`, `*.ear`, `*.log`, `*.db/*.sqlite*`. No ignora `target/` (Maven: `resiliencia-demo/*/target`, `spring_cloud/*/target`), ni `node_modules/` / `dist/` (Angular `frontend/hospital_web`), ni `*.env` ya cubierto pero falta `.angular/`.
- Deuda real verificada: `git ls-files | grep -cE "\.class$|\.jar$"` = **287 archivos `.class`** ya trackeados (ej. `backend/jwt-sales-services/bin/main/.../JwtSalesServicesApplication.class`), más `bin/` commiteado, porque el `.gitignore` se añadió en el mismo único commit posterior a los binarios. Falta `git rm -r --cached` de `**/bin **/build **/target *.class *.jar` y agregar `target/`, `node_modules/`, `dist/`, `.angular/` al `.gitignore`.

## Convenciones frontend (estructura, naming, estilo)

- Angular ~21 (paquete `cliniva@22.0.0`), standalone components con `imports: [...]` por componente, `inject()` + signals (`AuthService.currentUser = signal(...)`), `providedIn: 'root'`, lazy loading por rol: `app.routes.ts` (`APP_ROUTE` con `MainLayoutComponent` + `AuthGuard`, `loadChildren` → `ADMIN_ROUTE`/`DOCTOR_ROUTE`/`PATIENT_ROUTE`, `data: { role: Role.Admin }`), alias de paths `@core/*` y `@shared/*` (ej. `import { User } from '@core/models/interface'`).
- Estructura `src/app/`: por rol (`admin/doctor/patient`), `authentication/` (signin/signup/locked/page404/page500/...), `layout/` (app-layout/header/sidebar/right-sidebar/page-loader), `core/` (`guard/auth.guard.ts`, `guard/module-import.guard.ts`, `interceptor/error.interceptor.ts`, `models/interface.ts`, `service/*`), `shared/` (`components/` con ~56 specs: `breadcrumb`, `chat-widget`, `bed-occupancy`...; `services/storage.service.ts`, `utils/`), `config/config.service.ts`, `extra-pages/`, `multilevel/`. Estilo global SCSS por capas en `src/assets/scss/` (`common/`, `components/`, `ui/`, `theme/`, `pages/`, `fonts/`, `plugins/`) + `src/styles.scss`.
- Naming: componentes `kebab-case.component.{ts,html,scss,spec.ts}` (137 `.component.ts`, 127 `.html`, 125 `.scss`); servicios `*.service.ts` (ej. `token-factory.service.ts`, `direction.service.ts`); guards `*.guard.ts`, interceptores `*.interceptor.ts`, modelos `*.interface.ts`/`role.ts`; rutas `*.routes.ts` (`admin.routes`, `doctor.routes`).
