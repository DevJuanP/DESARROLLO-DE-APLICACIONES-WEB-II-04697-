# TESTING

## Frameworks detectados (JUnit/Mockito/AssertJ, Testcontainers, Jest/Vitest/Cypress — con evidencia de archivo o "no detectado")

- JUnit 5 + Spring Boot Test: **detectado** en los 12 módulos backend. Evidencia: `testImplementation 'org.springframework.boot:spring-boot-starter-test'` en `backend/*/build.gradle` (3), `practica-examen/*/build.gradle` (2), `spring_cloud/ms-notificaciones/build.gradle` (`spring-boot-starter-webmvc-test` + `junit-platform-launcher`), y `<artifactId>spring-boot-starter-test</artifactId>` en `resiliencia-demo/ms-*/pom.xml` (2) y `spring_cloud/{api-gateway,eureka-server,ms-cuentas,ms-recargas}/pom.xml` (4). Todos los tests usan `org.junit.jupiter.api.Test` + `@SpringBootTest`. `tasks.named('test') { useJUnitPlatform() }` en los 6 `build.gradle`.
- Mockito / AssertJ: **no detectado** — `grep -rn "Mockito|MockBean|@Mock|assertThat|@DataJpaTest|@WebMvcTest"` sobre `backend spring_cloud practica-examen resiliencia-demo` devuelve 0 resultados. Vienen transitivamente con `spring-boot-starter-test` pero ningún archivo los importa.
- Testcontainers: **no detectado** — 0 referencias en `*.java`, `build.gradle` y `pom.xml`.
- spring-security-test: **detectado** solo en `backend/jwt-sales-services/build.gradle` (`testImplementation 'org.springframework.security:spring-security-test'`), sin ningún test que lo use.
- Frontend: Karma + Jasmine (no Jest/Vitest/Cypress/Playwright). Evidencia: `frontend/hospital_web/package.json` (`karma ~6.4.4`, `karma-jasmine ~5.1.0`, `karma-coverage ~2.2.1`, `karma-chrome-launcher`, `jasmine-core ~5.9.0`, `@types/jasmine`, script `"test": "ng test"`), `frontend/hospital_web/angular.json` (`"test": { "builder": "@angular/build:karma", "tsConfig": "tsconfig.spec.json" }`), `frontend/hospital_web/tsconfig.spec.json`. **No detectado**: `jest.config*`, `vitest*`, `cypress*`, `playwright*`, `karma.conf.js` (no existe archivo de config en la raíz de `hospital_web`).

## Dónde están los tests (rutas o "sin tests en X")

Backend — 9 archivos `*ApplicationTests.java`, todos con el único test `contextLoads()` (conteo con `find ... -path "*/test/*"`, excluidos `bin/build/target`):

- `backend/products-services/src/test/java/com/cibertec/productsservices/ProductsServicesApplicationTests.java` — `contextLoads` OK
- `backend/sales-services/src/test/java/com/cibertec/salesservices/SalesServicesApplicationTests.java` — `contextLoads` OK
- `practica-examen/ms-notificaciones/src/test/java/com/cibertec/msnotificaciones/MsNotificacionesApplicationTests.java` — `contextLoads` OK
- `practica-examen/ms-pedidos/src/test/java/com/cibertec/mspedidos/MsPedidosApplicationTests.java` — `contextLoads` OK
- `spring_cloud/api-gateway/src/test/java/com/curso/zuul/ApiGatewayApplicationTests.java` — `contextLoads` OK
- `spring_cloud/eureka-server/src/test/java/com/curso/eurekaserver/EurekaServerApplicationTests.java` — `contextLoads` OK
- `spring_cloud/ms-cuentas/src/test/java/com/demo/eurekaclient/MsCuentasApplicationTests.java` — `contextLoads` OK
- `spring_cloud/ms-notificaciones/src/test/java/com/edu/ms_notificaciones/MsNotificacionesApplicationTests.java` — `contextLoads` OK
- `spring_cloud/ms-recargas/src/test/java/com/curso/front/MsRecargasApplicationTests.java` — `contextLoads` OK
- Sin tests en `backend/jwt-sales-services` (no existe `src/test/`), sin tests en `resiliencia-demo/ms-inventario` (no existe `src/test/`), sin tests en `resiliencia-demo/ms-pedidos` (no existe `src/test/`). Total: 9/12 módulos con test, 3/12 sin test, 0 tests unitarios/de integración reales (solo arranque de contexto).

Frontend — **83 archivos `*.spec.ts`** (`find frontend -name "*.spec.ts" | wc -l`), todos estilo `should create` / `should create the app`:

- `src/app/app.component.spec.ts` (3 tests: crea app, título `cliniva`, render)
- `src/app/authentication/*/*.spec.ts` (6: signin, signup, locked, forgot-password, page404, page500)
- `src/app/admin/dashboard/*/*.spec.ts` (3), `doctor/dashboard` (1), `patient/dashboard` (1), `extra-pages/blank` (1), `layout/*` (7), `multilevel/*` (6), `core` (1), `shared/components/*` (56)

## Cómo correrlos (comandos exactos por módulo: ./gradlew test, mvn test, npm test — según lo encontrado)

Requiere JDK 17 (Gradle `sourceCompatibility = '17'`, Maven `<java.version>17</java.version>`) y Node + ChromeHeadless para el frontend. Ningún módulo necesita broker/DB externos para su test actual (`contextLoads` con H2/config por defecto; los `@SpringBootTest` de eureka/gateway levantan su propio contexto).

```bash
# Gradle (6 módulos, cada uno con su wrapper; ejecutar desde la carpeta del módulo)
cd backend/sales-services       && ./gradlew test
cd backend/products-services    && ./gradlew test
cd backend/jwt-sales-services   && ./gradlew test   # declara spring-security-test pero hoy no tiene src/test
cd practica-examen/ms-pedidos         && ./gradlew test
cd practica-examen/ms-notificaciones  && ./gradlew test
cd spring_cloud/ms-notificaciones     && ./gradlew test

# Maven (6 módulos; usan mvnw solo en api-gateway, resto con mvn del PATH)
cd spring_cloud/api-gateway   && ./mvnw test
cd spring_cloud/eureka-server && mvn test
cd spring_cloud/ms-cuentas    && mvn test
cd spring_cloud/ms-recargas   && mvn test
cd resiliencia-demo/ms-inventario && mvn test   # hoy: "No tests to run"
cd resiliencia-demo/ms-pedidos    && mvn test   # hoy: "No tests to run"

# Frontend (desde frontend/hospital_web; Karma necesita Chrome)
cd frontend/hospital_web && npm test            # = ng test (watch)
cd frontend/hospital_web && npx ng test --watch=false --browsers=ChromeHeadless  # CI, una pasada
```

## Cobertura y gaps (qué módulos no tienen tests, riesgos)

- Cobertura real ≈ 0%: los 9 tests backend solo verifican que el contexto Spring arranca; los 83 specs frontend solo verifican `expect(component).toBeTruthy()` (más 2 asserts de título en `app.component.spec.ts`). No hay asserts de negocio, ni mocks, ni tests de controllers/services/repositories/mensajería. `karma-coverage` está declarado pero sin umbrales ni reportes publicados.
- Gaps críticos por módulo:
  - `backend/jwt-sales-services` (auth + usuarios + ventas protegidas): **sin `src/test`** pese a declarar `spring-boot-starter-test` + `spring-security-test`. Riesgo alto: `AuthService` (401 credenciales), `UserService` (409 duplicados, 404), `JwtService`/`JwtAuthenticationFilter`/`SecurityConfig` sin cobertura.
  - `resiliencia-demo/ms-pedidos` + `ms-inventario`: **sin `src/test`** aunque declaran `spring-boot-starter-test`. Riesgo alto: `PedidoService.crearPedidoSinResiliencia/crearPedidoConResiliencia` (circuit breaker/retry/fallback) y `InventarioController` (falla/demora simuladas, 503/404) son el corazón de la demo y no tienen prueba.
  - `backend/sales-services`, `backend/products-services`, `practica-examen/ms-pedidos`, `practica-examen/ms-notificaciones`: solo `contextLoads`. Riesgo medio-alto: `SaleService` (404 venta, reserva Rabbit, publicación Kafka, llamadas Feign), `ProductService` (404, 409 stock insuficiente), `SaleReserveService`, `MensajeNotificacionService`, consumers/producers Rabbit+Kafka sin prueba; un cambio de tópico/cola/exchange rompe en runtime.
  - `spring_cloud/*` (5 módulos): solo `contextLoads`, aceptable para eureka/gateway, pero `CuentaController` (saldos hardcodeados), `RecargaController` (`RestTemplate` a `http://ms-cuentas`, parseo `saldo` sin null-check → NPE si la cuenta no responde) y `NotificacionController` no tienen prueba de contrato.
  - Frontend: 83 specs existen pero ninguno prueba servicios con HTTP (`auth.service.ts`, `login.service.ts` con usuarios hardcodeados), guards (`auth.guard.ts`), interceptor (`error.interceptor.ts`) ni formularios de `signin`.
- Riesgo transversal: servicios escriben en MySQL + RabbitMQ + Kafka en el mismo método (ej. `SaleService.createSaleWithRabbitReserve`) sin tests de integración ni Testcontainers; los únicos `ErrorResponse` están sin usar y no hay `@ControllerAdvice`, por lo que el formato de error depende del default de Spring.

## Recomendación mínima (qué test agregar primero por servicio)

1. `backend/jwt-sales-services` → `AuthServiceTest` (Mockito + `@WebMvcTest(AuthController)`): login válido devuelve token, login inválido → 401 `"Credenciales invalidas"`; `POST /users` duplicado → 409. Es el módulo con más riesgo y cero tests.
2. `resiliencia-demo/ms-pedidos` → `PedidoServiceTest` (Mockito del `InventarioClient` + WireMock/stub): con falla activa, `crearPedidoSinResiliencia` propaga 503 y `crearPedidoConResiliencia` aplica fallback/timeout; más `InventarioControllerTest` (`@WebMvcTest`): `GET /inventario/999` → 404, con `falla=true` → 503.
3. `backend/sales-services` y `practica-examen/ms-pedidos` → `SaleServiceTest` (Mockito de `SaleRepository`, `ProductClient`, `StockReserveProducer`): `getSaleById(999)` → 404 `"Venta no encontrada"`; `createSale` publica evento y retorna `SaleResponse`; `SaleControllerTest` (`@WebMvcTest`): `POST /sales` → 200 y `DELETE /sales/{id}` → 204.
4. `backend/products-services` y `practica-examen/ms-notificaciones` → `ProductServiceTest`: `updateProduct` inexistente → 404, movimiento con stock insuficiente → 409; `MensajeNotificacionServiceTest`: `getMensajeActivoById` inexistente → 404.
5. `spring_cloud/ms-recargas` → `RecargaControllerTest` (`@WebMvcTest` + `@MockBean RestTemplate`): saldo suficiente → `recarga_aprobada=true`, saldo insuficiente → `false`, respuesta sin campo `saldo` no lanza NPE.
6. `frontend/hospital_web` → `auth.service.spec.ts` + `auth.guard.spec.ts` con `HttpTestingController`: login guarda token/roles, guard bloquea sin sesión; luego `signin.component.spec.ts` con envío de formulario inválido.
