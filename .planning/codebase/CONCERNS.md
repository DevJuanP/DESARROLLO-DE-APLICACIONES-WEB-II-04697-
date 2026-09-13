# CONCERNS

Auditoría estática del repo completo. Todo verificado por lectura directa de archivos y `git ls-files` en `D:\Cibertec\6to ciclo\DAW II\fork\DESARROLLO-DE-APLICACIONES-WEB-II-04697-`. No se reproduce ningún valor de secreto, solo nombres de variables y ubicación.

## Críticos (secrets en repo, auth débil, datos sensibles — con ruta y severidad)

### C1 — `database/.env` commiteado con credenciales reales (Alta)
- Ruta: `database/.env` (11 líneas).
- Contenido: nombres de variables commiteadas: `DB_IMAGE`, `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWD`, `DB_IMAGE_PHPMA`, `DB_HOST_PHPMA`, `DB_PORT_PHPMA`. Los valores están presentes en el archivo (no se reproducen aquí); la contraseña es trivial de diccionario y se reutiliza para `MYSQL_ROOT_PASSWORD`.
- Cómo se verificó: `read database/.env` + `read database/docker-compose.yml` (líneas 10-14: `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`, `MYSQL_ROOT_PASSWORD` consumen esas vars; `MYSQL_ROOT_PASSWORD` reutiliza la misma var que el usuario app) + `bash: git ls-files | grep -E "\.env"` → devuelve `database/.env` (trackeado).
- Agravante: el `.gitignore` raíz (líneas 8-10: `.env`, `.env.*`, `!.env.example`) declara que los `.env` no deben commitearse, pero el archivo ya estaba trackeado desde el commit inicial (`bash: git log --oneline -- database/.env` → `3b0df8c feat add initial commit`). El ignore no des-trackea lo ya commiteado. No existe `.env.example` en el repo (`glob **/*.env*` solo devuelve `database/.env`).

### C2 — JWT secret hardcodeado en `application.yml` commiteado (Alta)
- Ruta: `backend/jwt-sales-services/src/main/resources/application.yml` líneas 16-19 (`app.jwt.secret`, `app.jwt.expiration-minutes`).
- Hecho: `app.jwt.secret` es un literal auto-descrito y predecible commiteado en git (no se reproduce el valor). `app.jwt.expiration-minutes: 60` también hardcodeado.
- Cómo se verificó: `read` del yml citado + `bash: git ls-files | grep jwt-sales-services/src/main/resources/application.yml` (trackeado) + `read backend/jwt-sales-services/src/main/java/com/cibertec/jwtsalesservices/security/JwtService.java` líneas 21-27 (`@Value("${app.jwt.secret}")`, `Keys.hmacShaKeyFor(secret.getBytes(UTF_8))`, `signWith(secretKey)` sin algoritmo explícito → HMAC inferido de la longitud de la clave, sin `kid`, sin rotación).

### C3 — Credenciales DB/Kafka/Rabbit hardcodeadas en 5 `application.yml` (Alta)
- Rutas (todas con `spring.datasource.url/username/password` literales, valores no reproducidos):
  - `backend/jwt-sales-services/src/main/resources/application.yml` líneas 4-7
  - `backend/sales-services/src/main/resources/application.yml` líneas 4-7
  - `backend/products-services/src/main/resources/application.yml` líneas 4-7
  - `practica-examen/ms-pedidos/src/main/resources/application.yml` líneas 4-7
  - `practica-examen/ms-notificaciones/src/main/resources/application.yml` líneas 4-7
- Además `spring.rabbitmq.host/port/username/password` literales (credenciales por defecto del broker) en `backend/sales-services/...` líneas 17-21, `backend/products-services/...` líneas 22-26, `practica-examen/ms-pedidos/...` líneas 17-21, `practica-examen/ms-notificaciones/...` líneas 22-26.
- Cómo se verificó: `read` de los 5 yml + `glob **/application*.yml` (13 resultados, 5 con datasource) + `grep -rn ddl-auto/show-sql` confirma que los 5 apuntan al mismo `appdb` en `localhost:5510`.

### C4 — Registro abierto de usuarios: `POST /users` sin auth (Alta)
- Rutas:
  - `backend/jwt-sales-services/src/main/java/com/cibertec/jwtsalesservices/security/SecurityConfig.java` líneas 31-35: `requestMatchers(POST, "/users").permitAll()`, `requestMatchers(POST, "/auth/login").permitAll()`, `anyRequest().authenticated()`.
  - `backend/jwt-sales-services/src/main/java/com/cibertec/jwtsalesservices/rest/UserController.java` líneas 28-31: `POST /users` → `userService.createUser(request)` sin chequeo de rol.
  - `backend/jwt-sales-services/src/main/java/com/cibertec/jwtsalesservices/negocio/UserService.java` líneas 37/56: `passwordEncoder.encode(...)` (hash BCrypt correcto, verificado por `grep passwordEncoder`), pero el `role` viene del `UserRequest` del cliente sin whitelist visible → cualquier anónimo puede crear usuarios (potencialmente con rol elevado según lo que acepte `UserRequest`).
- Cómo se verificó: `read` de los tres archivos citados. No hay `@PreAuthorize`, ni `hasRole`, ni lista de endpoints de actuator restringida en este servicio.

### C5 — `JwtAuthenticationFilter` valida poco y falla abierto parcial (Alta/Media)
- Ruta: `backend/jwt-sales-services/src/main/java/com/cibertec/jwtsalesservices/security/JwtAuthenticationFilter.java` líneas 28-60.
- Hechos: solo extrae `Authorization: Bearer`, llama `jwtService.extractUsername()` + `isTokenValid()` (este último en `JwtService.java` líneas 46-48 solo compara `getExpiration().after(new Date())`; la firma se valida implícitamente en `parseClaims()` líneas 54-60 con `verifyWith(secretKey)`). No valida `issuer`, `audience`, ni `role`; el `catch (RuntimeException)` de líneas 55-57 hace `clearContext()` y **sigue la cadena** (`filterChain.doFilter` línea 59) sin devolver 401, delegando la decisión al `anyRequest().authenticated()` — correcto pero sin log ni respuesta de error propia, lo que dificulta detectar ataques de token-forging en logs. Además re-consulta `appUserRepository.findByUsername()` en cada request (sin caché) y construye `ROLE_ + user.getRole()` sin normalizar (línea 50).
- Cómo se verificó: `read` de `JwtAuthenticationFilter.java` + `JwtService.java` completos.

### C6 — Expiración JWT 60 min sin refresh/rotación (Media)
- Rutas: `backend/jwt-sales-services/src/main/resources/application.yml` línea 19 (`app.jwt.expiration-minutes`) + `JwtService.java` líneas 28-39 (`issuedAt`, `expiration = now + expirationMinutes*60`, claims `userId`, `role`, `subject=username`).
- Hecho: ventana de 60 min con secreto estático (C2) y sin refresh-token, blacklist ni `jti`. Si el secreto se filtra (ya está en git), todos los tokens son forjables hasta rotar el secreto y redeployar.
- Cómo se verificó: `read` de ambos archivos.

### C7 — `ddl-auto: update` + `show-sql: true` en 5 servicios (Media)
- Rutas: los 5 `application.yml` listados en C3, líneas `jpa.hibernate.ddl-auto: update` y `show-sql: true`.
- Cómo se verificó: `bash: grep -rn "ddl-auto" --include="*.yml" . | grep -v "/bin/|/build/|/target/"` → 5 hits; idem `show-sql` → 5 hits. Riesgo: Hibernate puede alterar el schema en arranque contra la DB compartida `appdb`; `show-sql:true` filtra SQL con datos por logs.

### C8 — Credenciales demo hardcodeadas en frontend (Media)
- Ruta: `frontend/hospital_web/src/app/core/service/login.service.ts` líneas 16-64 (array `users` con 3 cuentas demo y contraseñas triviales, valores no reproducidos) + líneas 66-100 (`login()` compara en memoria, genera token local vía `JWT.generate()` sin backend).
- Cómo se verificó: `read login.service.ts` completo. Es bypass total de cualquier backend: el login nunca sale a red.

### C9 — API keys de IA en `localStorage` + URLs hardcodeadas (Media)
- Ruta: `frontend/hospital_web/src/app/core/service/ai.service.ts` líneas 33-34 (`STORAGE_KEY='cliniva_ai_config'`), 41-43/48-52 (`localStorage.getItem/setItem` con `apiKey` en claro), 95 (`https://api.openai.com/v1/chat/completions`), 109 (`https://generativelanguage.googleapis.com/...`), 84-87/110-113 (envío de la key en header `Authorization: Bearer` / `X-goog-api-key` desde el browser, exponiéndola a DevTools y a cualquier XSS).
- Cómo se verificó: `read ai.service.ts` + `grep localhost|http://|https:// frontend/hospital_web/src` (63 matches; los únicos con URLs remotas reales fuera de assets son estos dos más `environment.apiUrl`).

## Deuda técnica (artefactos compilados en git, duplicación entre módulos, código muerto)

### D1 — 355 artefactos compilados commiteados pese a `.gitignore` (Alta)
- `.gitignore` raíz líneas 12-19 ignora `**/.gradle/`, `**/build/`, `**/bin/`, `*.class`, `*.jar`, `*.war`, `*.ear`, pero están trackeados desde el commit inicial.
- Conteo verificado con `bash: git ls-files | grep -E "(/bin/|/build/|\.class$|\.jar$)" | wc -l` → **355**; desglose verificado:
  - `git ls-files | grep -c "/bin/"` → **135**
  - `git ls-files | grep -c "/build/"` → **197**
  - `git ls-files | grep -c "\.class$"` → **273**
  - `git ls-files | grep -c "\.jar$"` → **14** (incluye 6 `gradle-wrapper.jar`: `backend/*/gradle/wrapper/`, `practica-examen/*/gradle/wrapper/`, `spring_cloud/ms-notificaciones/gradle/wrapper/` — estos últimos son convencionales de Gradle, pero los `build/*.jar`/`target/*.jar` no)
  - `git ls-files | grep -c "/target/"` → **24** (Maven `target/` de `resiliencia-demo/ms-inventario` y `ms-pedidos`: `.class`, `application.yml` duplicado en `target/classes/`, `*.jar` + `*.jar.original`, `maven-archiver/pom.properties`, `maven-status/.../createdFiles.lst|inputFiles.lst`).
- Hueco en `.gitignore`: no menciona `target/` (Maven) ni `dist/` del backend, por eso `resiliencia-demo/*/target/**` entra sin fricción. `frontend/hospital_web/.gitignore` sí ignora `/dist`, pero el raíz no cubre `target/`.
- Cómo se verificó: comandos citados + `read .gitignore` + `glob **/Dockerfile`, `**/pom.xml` para distinguir wrappers de build outputs.

### D2 — 15 `.DS_Store` commiteados pese a estar ignorados (Baja)
- `bash: git ls-files | grep DS_Store` → 15 hits: raíz, `backend/`, `practica-examen/` (+2 submódulos), `resiliencia-demo/` (+`ms-pedidos/target/`), `spring_cloud/` (+5 submódulos y `src/`). El `.gitignore` raíz línea 2 (`.DS_Store`) y `frontend/hospital_web/.gitignore` línea 41 lo ignoran, pero ya estaban trackeados.
- Cómo se verificó: comando citado + `read .gitignore`.

### D3 — Duplicación `ms-pedidos`: `practica-examen/` vs `resiliencia-demo/` — mismo nombre, distinto dominio (Alta)
- Rutas: `practica-examen/ms-pedidos/src/main/java/com/cibertec/mspedidos/` (29 ficheros `.java` incl. test) vs `resiliencia-demo/ms-pedidos/src/main/java/com/cibertec/resiliencia/pedidos/` (7 ficheros).
- Verificación: `bash: find practica-examen/ms-pedidos/src -name "*.java" | sort` vs `find resiliencia-demo/ms-pedidos/src -type f | sort`. Paquetes distintos (`com.cibertec.mspedidos` vs `com.cibertec.resiliencia.pedidos`), sin código compartido.
- Divergencia funcional: practica = ventas + Feign a `ms-notificaciones` + Kafka (`SaleCancellationProducer`, `StockMovementProducer` con `@ConditionalOnProperty messaging.kafka.enabled`) + RabbitMQ (`PurchaseEmailProducer`, `StockReserveProducer`) **sin** Resilience4j (`grep -rn CircuitBreaker practica-examen/ms-pedidos/src` → 0 hits salvo comentarios); resiliencia = pedidos contra inventario + Feign con `connectTimeout/readTimeout: 1000` + `@CircuitBreaker(name="inventario", fallbackMethod="fallbackCrearPedido")` + Actuator, **sin** mensajería ni JPA. Puertos distintos: `practica-examen/ms-pedidos/.../application.yml` línea 24 `port: 8082` vs `resiliencia-demo/ms-pedidos/.../application.yml` línea 13 `port: 8090`.
- Riesgo operativo: ambas generan imagen Docker `ms-pedidos:${VERSION}` (`resiliencia-demo/deploy.sh` líneas 13-14) y el script hace `sed "s|image: .*|..."` sobre `k8s/ms-pedidos-deployment.yaml` (función `update_manifest_image` líneas 60-68) → confusión de artefactos y mutación del working tree.

### D4 — Duplicación `ms-notificaciones`: `practica-examen/` vs `spring_cloud/` — divergencia total (Alta)
- Rutas: `practica-examen/ms-notificaciones/src/main/java/com/cibertec/msnotificaciones/` (36 ficheros en `src/main`, verificado `find ... | wc -l` → 36: JPA `CorreoEnviado/MensajeNotificacion/Product/Sale/SaleCancellationLog`, Kafka `StockAudit/StockReporting/StockUpdate/SaleCancellation` consumers, RabbitMQ `CorreoEnviado/StockLowAlert/StockReserve`, `NotificacionesController` con `GET /notificaciones`, `GET /notificaciones/{id}`, `POST /notificaciones`, `POST /notificaciones/mensajes`, `GET /notificaciones/mensajes/{id}`, `GET /notificaciones/ventas/{saleId}/anulaciones/logs`) vs `spring_cloud/ms-notificaciones/src/main/java/com/edu/ms_notificaciones/` (3 ficheros en `src/main`: `MsNotificacionesApplication.java`, `NotificacionController.java`, `application.yaml`).
- `spring_cloud/ms-notificaciones/.../NotificacionController.java` (27 líneas): solo `GET /notificaciones/enviar/{tipo}` hardcodeado (`canal=SMS`, `estado=enviado`), sin DB/Kafka/Rabbit, puerto `8083` (`application.yaml` líneas 1-2) vs practica puerto `8081` (`application.yml` línea 33). Mismo `spring.application.name: ms-notificaciones` en ambos → colisión en Eureka si alguna vez se registra el de practica (hoy no usa Eureka).
- Cómo se verificó: `find` de ambos `src/main`, `read` de `NotificacionController.java` + ambos yml/yaml + `grep "@GetMapping|@PostMapping|@RequestMapping" .../NotificacionesController.java`.

### D5 — Feign `ProductClient` en practica apunta al servicio equivocado (Alta)
- Ruta: `practica-examen/ms-pedidos/src/main/java/com/cibertec/mspedidos/client/ProductClient.java` (9 líneas): `@FeignClient(name="ms-notificaciones", url="http://localhost:8081")` + `@GetMapping("/products/{id}")`. El path `/products/{id}` no existe en `practica-examen/ms-notificaciones` (ese micro no expone `/products`; su controller base es `/notificaciones` — verificado D4) ni en `spring_cloud/ms-notificaciones` (solo `/notificaciones/enviar/{tipo}`). Es copia-pega de `backend/sales-services/src/main/java/com/cibertec/salesservices/client/ProductClient.java` (mismo `url="http://localhost:8081"` + `/products/{id}`, correcto allí porque `backend/products-services` sí expone productos en 8081).
- Contraste: `NotificationClient.java` en el mismo paquete sí usa `url="${notificaciones.base-url:http://localhost:8081}"` + `/notificaciones/mensajes/{mensajeId}` (correcto contra practica ms-notificaciones). `notificaciones.base-url` está definido en `practica-examen/ms-pedidos/.../application.yml` líneas 26-28.
- Cómo se verificó: `read` de los 3 `*Client.java` + `grep @GetMapping .../NotificacionesController.java`.

### D6 — `spring_cloud/api-gateway` ruta fantasma + YAML con guion colgante (Media)
- Ruta: `spring_cloud/api-gateway/src/main/resources/application.yml` líneas 18-21: ruta `id: ms-notificaciones, uri: lb://ms-notificaciones, Path=/notificaciones/**` seguida de un `-` suelto en la línea 21 (`            -`), resto de una edición.
- Hecho: ningún `spring_cloud` README ni compose levanta un `ms-notificaciones` Eureka-compatible salvo `spring_cloud/ms-notificaciones` (Boot 4.0.6/webmvc, ver D8) que sí se registraría, pero el `README.md` de `spring_cloud` (líneas 1-14) solo documenta `ms-cuentas`/`ms-recargas` y dice "registra los tres servicios" sin mencionar notificaciones → ruta sin contrato documentado ni tests.
- Cómo se verificó: `read application.yml` del gateway + `read spring_cloud/README.md` + `read spring_cloud/postman/sistema-fintech-tarjetas-prepago.postman_collection.json` (sin requests a `/notificaciones`).

### D7 — Código muerto / a medio migrar (Media/Baja)
- `spring_cloud/eureka-server/src/main/resources/application.properties` vacío (1 línea, verificado `read`) mientras la config real vive en `application.yml` (15 líneas) → archivo muerto que confunde.
- `spring_cloud/eureka-server/DockerFile` (nombre con F mayúscula, verificado `ls`): `FROM java:8`, `ADD /target/...jar` — obsoleto (resto del repo usa Java 17, Boot 3.2.5) y nunca usado por `deploy.sh` (solo resiliencia).
- `backend/*/bin/` y `*/build/` duplican `src/main/resources/application.yml` compilado (`backend/jwt-sales-services/bin/main/application.yml`, `build/resources/main/application.yml`, etc.) → secretos también en artefactos compilados commiteados.
- `resiliencia-demo/ms-pedidos/target/.DS_Store` trackeado (ver D2).

### D8 — Mezcla de stacks sin matriz de compatibilidad (Media)
- Verificado: `backend/*`, `practica-examen/*`: Gradle + Boot `3.2.5` + Java 17 (`grep spring.boot.*version` en `build.gradle`); `resiliencia-demo/*`, `spring_cloud/{eureka-server,api-gateway,ms-cuentas,ms-recargas}`: Maven + Boot `3.2.5` + Spring Cloud `2023.0.1` + Java 17 (`grep spring-boot-starter-parent -A1 pom.xml`); excepción: `spring_cloud/ms-notificaciones/build.gradle`: Boot `4.0.6` + `spring-boot-starter-webmvc`/`webmvc-test` + Spring Cloud `2025.1.1` (`read build.gradle` 39 líneas). Boot 4.x requiere Spring Framework 7 / Jakarta EE 11 y rompe con el resto (Boot 3.x). No hay `settings.gradle` raíz ni BOM común.

## Inconsistencias (puertos/credenciales/versiones que no coinciden entre servicios y composes)

Tabla de puertos (verificado por `read` de cada `application.yml/yaml` + `grep -ho localhost:[0-9]* postman/*.json spring_cloud/postman/*.json resiliencia-demo/postman/*.json`):

| Servicio | Archivo | Puerto | Estado |
|---|---|---|---|
| `database` MySQL host | `database/.env` (`DB_PORT`) + `database/docker-compose.yml` línea 9 (`${DB_PORT}:3306`) | 5510 externo → 3306 | OK con los 5 JDBC `localhost:5510` |
| phpMyAdmin | `database/.env` (`DB_PORT_PHPMA`) + `docker-compose.yml` línea 22 | 3410 externo → 80 | OK interno, sin auth propia documentada |
| `backend/products-services` | `backend/products-services/.../application.yml` línea 29 | 8081 | **COLISIONA** con `practica ms-notificaciones` (8081) |
| `practica ms-notificaciones` | `practica-examen/ms-notificaciones/.../application.yml` línea 33 | 8081 | **COLISIONA** con anterior; además `spring.json.value.default.type` difiere (`...msnotificaciones.kafka.StockMovementEvent` vs `...productsservices.kafka...`) |
| `backend/sales-services` | `backend/sales-services/.../application.yml` línea 24 | 8082 | **COLISIONA** con `practica ms-pedidos` (8082) |
| `practica ms-pedidos` | `practica-examen/ms-pedidos/.../application.yml` línea 24 | 8082 | **COLISIONA** con anterior |
| `backend/jwt-sales-services` | `backend/jwt-sales-services/.../application.yml` línea 14 | 8083 | **COLISIONA** con `spring_cloud/ms-notificaciones` (8083) |
| `spring_cloud/ms-notificaciones` | `spring_cloud/ms-notificaciones/.../application.yaml` línea 2 | 8083 | **COLISIONA** con anterior |
| `spring_cloud/ms-cuentas` | `spring_cloud/ms-cuentas/.../application.yml` línea 2 | 8300 | OK |
| `spring_cloud/ms-recargas` | `spring_cloud/ms-recargas/.../application.yml` línea 2 | 8100 | OK |
| Eureka | `spring_cloud/eureka-server/.../application.yml` línea 5 | 8761 | OK |
| Gateway | `spring_cloud/api-gateway/.../application.yml` línea 2 | 8762 | OK |
| `resiliencia ms-pedidos` | `resiliencia-demo/ms-pedidos/.../application.yml` línea 13 | 8090 (+ NodePort 30082 en `k8s/ms-pedidos-service.yaml` línea 12) | OK aislado |
| `resiliencia ms-inventario` | `resiliencia-demo/ms-inventario/.../application.yml` línea 6 | 8091 (+ NodePort 30081 en `k8s/ms-inventario-service.yaml` línea 12) | OK aislado |
| Kafka / UI | `queue/docker-compose-kafka.yml` líneas 26/50 | 9092 / 8090 externo UI | **8090 de Kafka-UI colisiona nominalmente con `resiliencia ms-pedidos:8090`** (distinto host/proceso, pero confunde; verificado `read` ambos) |
| RabbitMQ / mgmt | `queue/docker-compose-rabbitmq.yml` líneas 13-14 | 5672 / 15672 (`guest/guest` líneas 17-18) | OK con los 4 yml que declaran `rabbitmq host:localhost port:5672 username/password` por defecto |
| Postman `Cibertec Local` | `postman/Cibertec-Local.postman_environment.json` | `sales_base_url=...:8082`, `products_base_url=...:8081` | Coherente con `backend/*`, **incompatible** con correr `practica-examen/*` a la vez por colisión 8081/8082 |
| Postman `Cibertec JWT Local` | `postman/Cibertec-JWT-Local.postman_environment.json` | `jwt_base_url=...:8083` | Coherente con `jwt-sales-services`, **colisiona** con `spring_cloud/ms-notificaciones:8083` |

### I1 — Misma DB `appdb` para 5 servicios sin prefijo de tablas (Media)
- Los 5 yml con datasource usan `jdbc:mysql://localhost:5510/appdb` + `ddl-auto:update`. Entidades distintas (`Sale`, `Product`, `AppUser`, `MensajeNotificacion`, `CorreoEnviado`...) comparten schema; `Sale` existe en 3 servicios (`backend/sales-services`, `jwt-sales-services`, `practica`) con mapeos potencialmente distintos → riesgo de alteración cruzada del DDL. Verificado por `grep ddl-auto` (5 hits) + `glob **/entidades/*.java`.

### I2 — `INVENTARIO_BASE_URL` vs `inventario.base-url` (Baja, frágil pero funciona)
- `resiliencia-demo/k8s/ms-pedidos-deployment.yaml` línea 20-21: `INVENTARIO_BASE_URL=http://ms-inventario-svc:8091`; `InventarioClient.java` línea 10: `url="${inventario.base-url}"`; `application.yml` línea 16: `inventario.base-url: http://localhost:8091`. Spring relaxed-binding resuelve `INVENTARIO_BASE_URL` → `inventario.base-url`, pero no está documentado en `README.md` ni en el deployment (sin `valueFrom`, sin Secret/ConfigMap). Verificado por `grep -rn "INVENTARIO_BASE_URL|inventario.base" resiliencia-demo/ --include="*.java" --include="*.yml"`.

### I3 — Versiones Spring Cloud divergentes (Media)
- `2023.0.1` en `backend/sales-services/build.gradle` (`ext springCloudVersion`), `practica-examen/ms-pedidos/build.gradle`, `resiliencia-demo/*/pom.xml` (`spring-cloud.version`), `spring_cloud/{eureka, gateway, cuentas, recargas}/pom.xml` vs `2025.1.1` solo en `spring_cloud/ms-notificaciones/build.gradle` línea 21. Verificado por `bash: for f in spring_cloud/*/build.gradle; grep springCloudVersion; done` + `grep spring-cloud.version pom.xml`. `2025.1.x` está alineado a Boot 4.x, incompatible con clientes Feign `2023.0.x`.

## Resiliencia/escalabilidad (qué falta: retries, circuit breaker, límites — según resiliencia-demo vs resto)

Referencia sana: `resiliencia-demo/ms-pedidos/.../application.yml` líneas 4-37 (Feign `connectTimeout/readTimeout: 1000`, `resilience4j.circuitbreaker.instances.inventario`: `slidingWindowSize:5`, `minimumNumberOfCalls:5`, `failureRateThreshold:50`, `waitDurationInOpenState:10s`, `permittedNumberOfCallsInHalfOpenState:2`, `automaticTransition...:true`, Actuator `health,info,circuitbreakers,circuitbreakerevents`) + `PedidoService.java` (`@CircuitBreaker(name="inventario", fallbackMethod="fallbackCrearPedido")` → estado degradado `RECIBIDO_SIN_VALIDAR_STOCK`) + `PedidoController.java` (endpoints `/pedidos/sin-resiliencia` y `/pedidos/con-resiliencia` para comparar). Verificado por `read` de los 4 archivos + `resiliencia-demo/README.md` (187 líneas, tabla comparativa líneas 182-185).

### R1 — `backend/*` y `practica-examen/*`: cero Resilience4j (Alta)
- `bash: grep -rn "CircuitBreaker|Retry|Bulkhead|RateLimiter|TimeLimiter|resilience4j" backend/*/build.gradle practica-examen/*/build.gradle backend/*/src practica-examen/*/src --include="*.gradle" --include="*.java" | grep -v "/bin/|/build/"` → 0 hits. Los Feign (`backend/sales-services/.../ProductClient.java`, `practica-examen/ms-pedidos/.../ProductClient.java|NotificationClient.java`) no tienen timeouts ni fallback: `practica .../SaleService.java` líneas 68-84 (`getSaleDetailsWithFeign` llama directo y multiplica precio) y `createSale` (guarda `PENDING` y luego llama a notificaciones sin try/catch visible) propagan la excepción Feign como 500. Sin `@EnableRetry`, sin `spring.cloud.openfeign.client.config.*` en ningún yml de backend/practica (verificado `grep -rn openfeign backend practica-examen --include="*.yml"` → 0 hits).

### R2 — `spring_cloud/ms-recargas → ms-cuentas` vía `RestTemplate @LoadBalanced` sin timeout/CB (Media)
- `spring_cloud/README.md` línea 12: "`ms-recargas` ... usa un `RestTemplate` con `@LoadBalanced` para llamar a `http://ms-cuentas/cuentas/saldo/{cuentaId}`". No hay `resilience4j`, `spring-retry` ni `HttpComponentsClientHttpRequestFactory` con timeouts en `spring_cloud/ms-recargas/pom.xml` (solo `starter-web`, `eureka-client`, verificado `grep starter pom.xml`) ni en su `application.yml` (13 líneas, solo puerto + eureka). El gateway tampoco tiene `CircuitBreaker` filter ni `Retry` en `application.yml` (solo `Path` predicates).

### R3 — Mensajería sin DLQ/reintentos configurados (Media)
- `practica-examen/ms-pedidos/.../application.yml` líneas 30-36 (`messaging.kafka.enabled:true`, `messaging.rabbitmq.enabled:true`, `messaging.rabbitmq.stock.enabled:false`) + `@ConditionalOnProperty` en `KafkaTopicConfig`, `SaleCancellationProducer`, `StockMovementProducer`, `RabbitMQConfig`, `StockLowAlertConsumer` (verificado `grep -rn ConditionalOnProperty .../src/main/java`). No hay `spring.kafka.consumer/producer.retries`, `ack-mode`, `dlt`, ni `spring.rabbitmq.listener.*.retry` en ningún yml (verificado `grep -rn "retry|dlt|dead|ack-mode" --include="*.yml" backend practica-examen`). `SaleController.deleteSale` publica a Kafka y devuelve `202` sin confirmación (`SaleService.java` líneas de `deleteSale`: `producer.publish(...)` + `buildSaleResponse(sale)` sin tx).

### R4 — K8s de resiliencia-demo no es escalable ni observable (Media)
- Rutas: `resiliencia-demo/k8s/ms-pedidos-deployment.yaml` (23 líneas), `ms-inventario-deployment.yaml` (20 líneas), ambos `Service.yaml` (12 líneas c/u).
- Hechos verificados por `read` + `grep -rn "resources|limits|liveness|readiness|replicas|hpa|pdb" resiliencia-demo/k8s/`: `replicas: 1` sin HPA/PDB, sin `resources.requests/limits`, sin `liveness/readinessProbe` (aunque hay `/actuator/health`), `imagePullPolicy: Never` (solo minikube local), `type: NodePort` con `nodePort: 30081/30082` fijos, sin `namespace`, sin `ConfigMap/Secret` (la URL de inventario va en `env.value` plano), sin `NetworkPolicy`, sin TLS. `deploy.sh` líneas 142-146 construye con `minikube docker-env` (acoplado a minikube) y líneas 151-153 mutan los manifests con `sed` (ensucia `git diff`, choca con el chequeo gitops de líneas 46-48 que exige working tree limpio).

### R5 — Sin rate-limiting ni bulkhead en ningún servicio (Baja/Media)
- `grep -rn "RateLimiter|Bulkhead|gateway.*filters.*RequestRateLimiter|spring.cloud.gateway.*redis" --include="*.yml" --include="*.java" --include="pom.xml" --include="*.gradle" . | grep -v "/bin/|/build/|/target/"` → 0 hits. El demo de inventario permite `POST /inventario/demo/demora/{millis}` sin tope (`InventarioController.java`: `demoraMillis.set(Math.max(0, millis))`, `Thread.sleep(demoraMillis.get())` en el request thread) → un cliente puede fijar demoras arbitrarias y agotar threads de `ms-pedidos` sin CB (endpoint `/sin-resiliencia`) — útil para la demo, peligroso si se expone.

## Riesgos frontend (URLs hardcodeadas, falta de guards, deps)

### F1 — `apiUrl` apunta a sí mismo en prod y dev (Alta)
- Rutas: `frontend/hospital_web/src/environments/environment.ts` línea 3 y `environment.development.ts` línea 7: `apiUrl: 'http://localhost:4200'` en ambas (prod `production:true`, dev `production:false`). `angular.json` líneas 82-86 reemplaza `environment.ts` por `environment.development.ts` en dev, pero ambos valores son idénticos → el build de producción llama a localhost. Ningún `environment.prod.ts` separado.
- Cómo se verificó: `read` de ambos environments + `read angular.json` + `grep localhost frontend/hospital_web/src` (hits en environments).

### F2 — Auth 100% cliente, falsificable desde DevTools (Alta)
- Rutas: `src/app/core/service/login.service.ts` (usuarios demo en memoria, C8), `src/app/core/service/JWT.ts` líneas 40-50 (`createToken`: `base64(JSON({typ:JWT,alg:HS256})).base64(JSON({exp,user})).base64(literal)` — firma simétrica fake verificable por cualquiera que lea el bundle), `src/app/core/guard/auth.guard.ts` líneas 18-40 (solo lee `LocalStorageService.get('currentUser')` y compara `roles[0].name` contra `route.data['role']`; nunca valida firma ni `exp`), `src/app/core/service/auth.service.ts` líneas 25-62 (guarda `token` y `currentUser` en `localStorage` vía `store.set`), `src/app/shared/services/storage.service.ts` líneas 24-42 (persistencia en `localStorage` con prefijo `dark_/light_ + ltr_/rtl_` derivado del theme → cambiar theme "pierde" la sesión; `clear()` líneas 54-56 hace `localStorage.clear()` global).
- Cómo se verificó: `read` de los 5 archivos. Efecto: basta `localStorage.setItem('light_ltr_currentUser', '{"roles":[{"name":"ADMIN"}]}')` para entrar como admin.

### F3 — Rutas `extra-pages` y `multilevel` sin guard (Media)
- Ruta: `src/app/app.routes.ts` líneas 42-55 (`extra-pages` y `multilevel` con `loadChildren` pero **sin** `canActivate`), vs `admin/doctor/patient` (líneas 15-41, con `AuthGuard` + `data.role`). Parte de `extra-pages` son páginas de error/blank/invoice/pricing/faqs, pero al no tener guard heredan solo el del layout padre (`path:''` línea 12) — si ese padre se relaja, quedan abiertas; además `authentication/*` (signin/signup/forgot/locked/page404/page500/two-factor/maintenance/coming-soon, verificado `read src/app/authentication/auth.routes.ts`) es público por diseño pero incluye `signup` sin backend real.
- Cómo se verificó: `read app.routes.ts` + `grep -rn canActivate|AuthGuard src/app --include="*.routes.ts"` (solo 4 hits, todos en `app.routes.ts` para el layout y 3 roles).

### F4 — Dependencias pesadas/obsoletas + `legacy-peer-deps` (Media)
- Ruta: `frontend/hospital_web/package.json` (91 líneas), `.npmrc` (`legacy-peer-deps=true`, verificado `read`), `angular.json` líneas 26-44 (`allowedCommonJsDependencies` con 18 libs CJS: `echarts`, `chart.js`, `apexcharts`, `exceljs`, `file-saver`, `sweetalert2`, etc. — penalizan bundle; budgets prod `4mb warn/6mb error` líneas 62-67 ya holgados).
- Puntos concretos (verificar con `npm audit` / `npm outdated`; aquí solo hechos del manifest): `exceljs@^4.4.0` (serie 4.x sin mantenimiento, último release 2023, historial de ReDoS/prototype-pollution — auditar), `karma@~6.4.4` + `karma-chrome-launcher@~3.2.0`/`karma-coverage@~2.2.1` (stack de test viejo; Angular 21 usa Vitest/Jest por defecto), `browser-sync@^3.0.4` (solo dev, abre puerto extra), `subsink@^1.0.2` (abandonado; el repo ya tiene `UnsubscribeOnDestroyAdapter`), mezcla `@angular/*@^21.0.x` con `@angular/material-date-fns-adapter@^21.2.14` (minor desalineado) y `rxjs@~7.8.2` + `zone.js@~0.15.1` pinnned con `~`. `legacy-peer-deps=true` oculta conflictos de peers en vez de resolverlos.
- Cómo se verificó: `read package.json` + `read .npmrc` + `read angular.json`.

### F5 — Tokens y PII en `localStorage`, sin `HttpOnly`/`CSP` visibles (Media)
- `TokenService` (`src/app/core/service/token.service.ts` línea 23: `key='redstar-token'`) + `AuthService` guardan `token`, `currentUser` (incluye `email`, `avatar`, `permissions`) y `roleNames` en `localStorage` (no `sessionStorage`, no cookie `HttpOnly`/`Secure`/`SameSite`). `src/index.html` líneas 10-11 carga Google Fonts externo sin `integrity`/`CSP` visible; `ngsw-config.json` (service worker, referenciado `angular.json` línea 75) cachea la app pero no hay `Content-Security-Policy` ni `interceptor` que inyecte `Authorization` de forma centralizada (verificado `glob src/app/**/*.ts`: no hay `*.interceptor.ts` entre los 100 primeros; `grep -rn "HttpInterceptor|provideHttpClient.*withInterceptors" src/app --include="*.ts"` → 0 hits fuera de comentarios).

## Priorización (tabla: problema | severidad Alta/Media/Baja | esfuerzo | archivo(s))

| Problema | Severidad | Esfuerzo | Archivo(s) |
|---|---|---|---|
| C1 `database/.env` con credenciales en git + `MYSQL_ROOT_PASSWORD` reutilizada | Alta | S (git rm --cached + rotar credenciales + `.env.example`) | `database/.env`, `database/docker-compose.yml`, `.gitignore` |
| C2 `app.jwt.secret` hardcodeado + expiración fija | Alta | S (env `APP_JWT_SECRET`, rotar, validar longitud ≥256bit) | `backend/jwt-sales-services/src/main/resources/application.yml`, `.../security/JwtService.java` |
| C3 user/pass/rabbitmq/kafka literales en 5 yml | Alta | S (env + perfiles `local/prod`) | 5 `application.yml` de `backend/*` y `practica-examen/*` |
| D1 355 artefactos (`bin/135`, `build/197`, `class/273`, `jar/14`, `target/24`) en git + falta `target/` en ignore | Alta | M (git rm --cached masivo + `git-filter-repo` histórico + ampliar ignore) | `.gitignore`, `backend/*/bin|build/**`, `resiliencia-demo/*/target/**` |
| D3/D4 colisión de nombres `ms-pedidos`/`ms-notificaciones` + D5 Feign a `/products` contra servicio equivocado | Alta | M (renombrar artefactos/imágenes, corregir `ProductClient`, alinear `notificaciones.base-url`) | `practica-examen/ms-pedidos/.../ProductClient.java`, `.../NotificationClient.java`, `.../application.yml`, `resiliencia-demo/deploy.sh`, `resiliencia-demo/k8s/*`, ambos `ms-notificaciones` |
| C4 `POST /users` abierto (registro sin auth/roles) | Alta | S (restringir a `ADMIN` o captcha/aprobación + whitelist de `role`) | `backend/jwt-sales-services/.../security/SecurityConfig.java`, `.../rest/UserController.java`, `.../negocio/UserService.java` |
| F1 `apiUrl=localhost:4200` en prod + F2 auth solo-cliente falsificable | Alta | M (env por stage + backend real o eliminar login fake; validar JWT en guard/interceptor) | `frontend/hospital_web/src/environments/*`, `angular.json`, `.../core/service/login.service.ts`, `.../core/service/JWT.ts`, `.../core/guard/auth.guard.ts`, `.../core/service/auth.service.ts` |
| R1 sin Resilience4j/timeouts/fallback en `backend`+`practica` (Feign desnudo → 500/cascada) | Alta | M (portar patrón `resiliencia-demo`: timeouts + CB + fallback + Actuator) | `backend/sales-services/.../ProductClient.java`, `practica-examen/ms-pedidos/.../SaleService.java`, `.../client/*.java`, yml de ambos |
| Puertos 8081/8082/8083 en colisión triple (no co-ejecutables) | Alta | S (reasignar: p.ej. practica 8084/8085, spring-notif 8086; documentar matriz) | 5 `application.yml/yaml` implicados, `postman/*.json`, `spring_cloud/README.md`, `resiliencia-demo/README.md` |
| C5 filtro JWT sin issuer/audience/401 propio + C6 60min sin refresh | Media | S (validar claims, 401 JSON, `jti`/refresh o lifetimes cortos) | `.../security/JwtAuthenticationFilter.java`, `.../security/JwtService.java` |
| C7 `ddl-auto:update` + `show-sql:true` en 5 servicios sobre `appdb` compartida (I1) | Media | S (Flyway/Liquibase + `validate`, `show-sql:false` fuera de local, DB por servicio o prefijos) | 5 `application.yml`, `database/docker-compose.yml` |
| C8/C9 creds demo + apikey IA en `localStorage` y llamadas directas a OpenAI/Gemini desde browser | Media | S (proxy backend, Secret Manager, CSP) | `frontend/hospital_web/src/app/core/service/login.service.ts`, `.../ai.service.ts` |
| R2 `RestTemplate @LoadBalanced` sin timeout + gateway sin Retry/CB (D6 ruta fantasma con `-` colgante) | Media | S (timeouts + `Retry`/`CircuitBreaker` filters, quitar ruta o documentarla, fix YAML) | `spring_cloud/api-gateway/.../application.yml`, `spring_cloud/ms-recargas/**`, `spring_cloud/README.md` |
| R3 Kafka/Rabbit sin retries/DLQ/ack explícito | Media | M (configurar `retries`, `dlt`, `ack-mode`, idempotencia) | `practica-examen/ms-pedidos/.../kafka/*`, `.../rabbitmq/*`, `.../application.yml`, `backend/*/src/**/kafka|rabbitmq/*` |
| R4 K8s `replicas:1`, sin probes/limits/HPA, `NodePort` fijo, `imagePullPolicy:Never`, `sed` muta manifests | Media | M (probes, limits, HPA/PDB, ConfigMap/Secret, build CI, no-mutación) | `resiliencia-demo/k8s/*.yaml`, `resiliencia-demo/deploy.sh`, `resiliencia-demo/ms-*/Dockerfile` |
| D8 Boot 4.0.6/webmvc + Cloud 2025.1.1 solo en `spring_cloud/ms-notificaciones` (incompatible con Boot 3.2.5/2023.0.1 del resto) | Media | S (pinear a Boot 3.2.5/Cloud 2023.0.1 o aislar módulo) | `spring_cloud/ms-notificaciones/build.gradle` |
| I3/I2 Kafka-UI `:8090` vs `ms-pedidos:8090`; `INVENTARIO_BASE_URL` plano sin ConfigMap | Baja/Media | XS (cambiar UI a 8099, documentar relaxed-binding o renombrar var) | `queue/docker-compose-kafka.yml`, `resiliencia-demo/k8s/ms-pedidos-deployment.yaml`, `.../client/InventarioClient.java` |
| F3 `extra-pages`/`multilevel` sin `canActivate` explícito; F5 PII/token en `localStorage`, sin interceptor/CSP | Media | S (guards explícitos, `sessionStorage`/cookies, interceptor, CSP/nonces) | `frontend/hospital_web/src/app/app.routes.ts`, `.../extra-pages/*.routes.ts`, `.../shared/services/storage.service.ts`, `.../core/service/token.service.ts`, `src/index.html` |
| F4 `exceljs@4.4.0`/karma viejos, 18 CJS en bundle, `legacy-peer-deps=true` | Media | M (`npm audit/outdated`, sustituir `exceljs`/`subsink`, quitar flag) | `frontend/hospital_web/package.json`, `.npmrc`, `angular.json` |
| D2 15 `.DS_Store` + D7 `application.properties` vacío + `eureka DockerFile` (`java:8`) + `gateway` `-` colgante | Baja | XS (git rm + fix) | `.DS_Store` (15 rutas), `spring_cloud/eureka-server/src/main/resources/application.properties`, `spring_cloud/eureka-server/DockerFile`, `spring_cloud/api-gateway/.../application.yml:21` |
| R5 `POST /inventario/demo/demora/{millis}` sin tope + `Thread.sleep` en thread de request | Baja (demo) | XS (cap + validación + endpoint solo perfil `demo`) | `resiliencia-demo/ms-inventario/.../rest/InventarioController.java` |
