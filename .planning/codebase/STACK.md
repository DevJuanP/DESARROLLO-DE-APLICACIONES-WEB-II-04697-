# STACK

> Mapeo TECH verificado archivo por archivo. Todo lo afirmado cita su evidencia.
> Alcance: `backend/`, `frontend/hospital_web`, `spring_cloud/`, `practica-examen/`,
> `resiliencia-demo/`, `database/`, `queue/`, `postman/`. Ignorados `**/bin/**`,
> `**/build/**`, `*.class`, `*.jar`, `.git/` (incluye `resiliencia-demo/*/target/`).

## Lenguajes y versiones (con evidencia: archivo que lo prueba)

- **Java 17 (toolchain / sourceCompatibility) en TODOS los microservicios JVM.**
  - `backend/jwt-sales-services/build.gradle:10-12` → `sourceCompatibility = '17'`.
  - `backend/products-services/build.gradle:10-12` → `sourceCompatibility = '17'`.
  - `backend/sales-services/build.gradle:10-12` → `sourceCompatibility = '17'`.
  - `practica-examen/ms-pedidos/build.gradle:10-12` y
    `practica-examen/ms-notificaciones/build.gradle:10-12` → `sourceCompatibility = '17'`.
  - `spring_cloud/ms-notificaciones/build.gradle:10-14` → `toolchain {
    languageVersion = JavaLanguageVersion.of(17) }`.
  - `spring_cloud/{api-gateway,eureka-server,ms-cuentas,ms-recargas}/pom.xml:18` y
    `resiliencia-demo/{ms-pedidos,ms-inventario}/pom.xml` (`<java.version>17</java.version>`)
    → Java 17 vía `spring-boot-starter-parent:3.2.5`.
  - Runtime observado del entorno: `java -version` → 26.0.1 (NO es el target del proyecto;
    el target sigue siendo 17).
- **TypeScript ~5.9.3 (frontend).**
  - Evidencia: `frontend/hospital_web/package.json:88` → `"typescript": "~5.9.3"`.
  - Opciones estrictas en `frontend/hospital_web/tsconfig.json:9` (`"strict": true`,
    `"target": "ES2022"`, `"module": "ES2022"`, `"moduleResolution": "bundler"`).
- **JavaScript runtime del entorno:** `node --version` → v24.16.0, `npm --version` → 11.13.0
  (herramientas instaladas, no version fijada en repo; no hay `.nvmrc` ni `engines` en
  `frontend/hospital_web/package.json` — verificado, ese archivo no declara `engines`).
- **Bash (scripts ops):** `resiliencia-demo/deploy.sh:1` → `#!/usr/bin/env bash` con
  `set -euo pipefail` (requiere `docker`, `minikube`, `kubectl`, `curl`, y en modo gitops
  además `git` + `argocd`; ver `deploy.sh:132-135,159-160`).
- **YAML / JSON / XML / Docker Compose / K8s manifests** como lenguajes de config
  (ver secciones de config e infra).

## Frameworks backend (Spring Boot versión, starters: web, data-jpa, security, validation, eureka, etc.)

Versión dominante: **Spring Boot 3.2.5**. Excepción: **Spring Boot 4.0.6** solo en
`spring_cloud/ms-notificaciones`.

| Módulo | Boot / Cloud (evidencia) | Starters y libs clave (evidencia) |
|---|---|---|
| `backend/jwt-sales-services` | Boot `3.2.5` + dep-mgmt `1.1.4` (`build.gradle:3-4`) | `spring-boot-starter-data-jpa`, `-security`, `-validation`, `-web` (`build.gradle:19-22`); `jjwt-api/impl/jackson:0.12.5` (`build.gradle:23-25`); `mysql-connector-j` runtime (`build.gradle:26`); `spring-boot-starter-test`, `spring-security-test` (`build.gradle:27-28`) |
| `backend/products-services` | Boot `3.2.5` + dep-mgmt `1.1.4` (`build.gradle:3-4`) | `starter-web`, `starter-amqp`, `starter-data-jpa`, `spring-kafka` (`build.gradle:19-22`); `mysql-connector-j`; `lombok` compileOnly+annotationProcessor (`build.gradle:26-27`) |
| `backend/sales-services` | Boot `3.2.5`, Spring Cloud `2023.0.1` (`build.gradle:3-4,19`) | igual que products + `spring-cloud-starter-openfeign` (`build.gradle:27`); BOM `spring-cloud-dependencies` (`build.gradle:36-40`) |
| `spring_cloud/api-gateway` | Boot `3.2.5`, Cloud `2023.0.1` (`pom.xml:8,19`) | `spring-cloud-starter-netflix-eureka-client` + `spring-cloud-starter-gateway` (`pom.xml:23-30`) |
| `spring_cloud/eureka-server` | Boot `3.2.5`, Cloud `2023.0.1` (`pom.xml:8,19`) | `spring-boot-starter-web` + `spring-cloud-starter-netflix-eureka-server` (`pom.xml:24-37`) |
| `spring_cloud/ms-cuentas` | Boot `3.2.5`, Cloud `2023.0.1` (`pom.xml:8,19`) | `starter-web` + `eureka-client` (`pom.xml:23-30`); sin JPA/BD (ver `application.yml`, solo puerto 8300 + eureka) |
| `spring_cloud/ms-recargas` | Boot `3.2.5`, Cloud `2023.0.1` (`pom.xml:8,19`) | `starter-web` + `eureka-client` (`pom.xml:23-30`); `RestTemplate` + `@LoadBalanced` en código (`ms-recargas/.../MsRecargasApplication.java:16-19`) |
| `spring_cloud/ms-notificaciones` | **Boot `4.0.6`**, dep-mgmt `1.1.7`, Cloud `2025.1.1` (`build.gradle:3-4,21`) | `spring-boot-starter-webmvc` (nuevo artefacto Boot 4, no `starter-web`), `spring-cloud-starter-netflix-eureka-client` (`build.gradle:25-26`); test con `spring-boot-starter-webmvc-test` (`build.gradle:27`) |
| `practica-examen/ms-pedidos` | Boot `3.2.5`, Cloud `2023.0.1` (`build.gradle:3,19`) | `starter-web`, `starter-amqp`, `starter-data-jpa`, `spring-kafka`, `spring-cloud-starter-openfeign` (`build.gradle:23-27`); `mysql`, `lombok` |
| `practica-examen/ms-notificaciones` | Boot `3.2.5` (`build.gradle:3`) sin Cloud BOM | `starter-web`, `starter-amqp`, `starter-data-jpa`, `spring-kafka` (`build.gradle:19-22`); sin OpenFeign; `mysql`, `lombok` |
| `resiliencia-demo/ms-pedidos` | Boot `3.2.5`, Cloud `2023.0.1` (`pom.xml:10,21`) | `starter-web`, `starter-actuator`, `starter-aop`, `spring-cloud-starter-openfeign` (`pom.xml:25-40`); `resilience4j-spring-boot3` + `resilience4j-micrometer` **sin versión explícita** (heredada del BOM/transitivos, `pom.xml:41-48`) |
| `resiliencia-demo/ms-inventario` | Boot `3.2.5` (`pom.xml:10`), sin Cloud | `starter-web`, `starter-actuator` (`pom.xml:24-31`); sin Feign, sin JPA, sin mensajería |

Notas verificadas:
- `@EnableFeignClients` solo en emisores: `backend/sales-services/.../SalesServicesApplication.java:8`,
  `practica-examen/ms-pedidos/.../MsPedidosApplication.java:8`,
  `resiliencia-demo/ms-pedidos/.../MsPedidosApplication.java:7`. No existe en ningún receptor.
- `@CircuitBreaker(name="inventario", fallbackMethod=...)` solo en
  `resiliencia-demo/ms-pedidos/.../service/PedidoService.java:30`.
- Seguridad Spring solo en `backend/jwt-sales-services` (`SecurityConfig.java`,
  `JwtAuthenticationFilter.java`, `JwtService.java`). Ningún otro módulo declara
  `spring-boot-starter-security`.
- `spring_cloud` (salvo `ms-notificaciones` Boot 4) es deliberadamente mínimo según
  `spring_cloud/README.md:14`: "No se usa base de datos, Feign ni persistencia".

## Frontend (framework, versión, UI libs, build tool)

- **Framework: Angular 21.0.x** (no 15). Evidencia: `frontend/hospital_web/package.json:13-25`
  (`@angular/core`, `common`, `compiler`, `forms`, `router`, `animations`,
  `platform-browser*`, `localize`, `service-worker`: `^21.0.3`; `cdk`/`material`: `^21.0.2`;
  `material-date-fns-adapter`: `^21.2.14`). CLI y build: `@angular/cli ~21.0.2`,
  `@angular/build ^21.0.2`, `@angular/compiler-cli ^21.0.3` (`package.json:72-74`).
  - `frontend/hospital_web/README.md:3` dice "Angular CLI 15.1.5" → **documentación
    desactualizada** (brecha doc vs código).
- **Build tool:** Application Builder esbuild (`@angular/build:application`) según
  `frontend/hospital_web/angular.json:18`; dev-server `@angular/build:dev-server`
  (`angular.json:93`); tests `@angular/build:karma` (`angular.json:111`); lint
  `@angular-eslint/builder:lint` (`angular.json:125`). Proyecto `cliniva`
  (`angular.json:6`), prefijo `app`, estilos `scss` (`angular.json:9-15`),
  salida `dist/cliniva` (`angular.json:20-22`), budgets prod 4 MB warning / 6 MB error
  (`angular.json:62-72`), PWA con `ngsw-config.json` + `manifest.webmanifest`
  (`angular.json:47,75`, `src/manifest.webmanifest` existe).
- **UI / libs principales** (`package.json:26-64`): Angular Material + CDK, Bootstrap
  `^5.3.7`, FullCalendar 6.1.19 (angular/core/daygrid/interaction/list/timegrid),
  ApexCharts 5.3.4 + `ng-apexcharts`, `ngx-echarts` + `echarts` 6.0.0, `chart.js` 4.5.0 +
  `ng2-charts`, `@swimlane/ngx-charts` + `ngx-datatable`, `ngx-scrollbar`, `ngx-mask`,
  `ngx-permissions`, `ngx-editor`, `ngx-gauge`, `ngx-dropzone-wrapper`, `angular-feather`,
  `@ngx-translate/core+http-loader` 17.0.0, `@ngx-loading-bar/core+router` 7.0.0,
  `sweetalert2`, `exceljs` + `file-saver`, `date-fns` 4.4.0, `rxjs ~7.8.2`, `zone.js ~0.15.1`,
  `core-js`, `tslib`, `subsink`, `base64-js`, `@popperjs/core`.
- **Dev/test:** `typescript ~5.9.3`, `eslint ^9.34.0` + `typescript-eslint`,
  `angular-eslint/* 21.0.1`, `jasmine-core ~5.9.0` + `karma ~6.4.4` + launchers
  (`package.json:66-90`); `tsconfig.spec.json` existe.
- **Estructura `src/`:** standalone bootstrap en `frontend/hospital_web/src/main.ts`
  (`bootstrapApplication(AppComponent, appConfig)`); módulos funcionales
  `src/app/{admin,authentication,doctor,patient,layout,shared,core,config,multilevel,extra-pages}`
  (verificado por listado de `src/app`); path aliases `@core/@shared/@config` en
  `tsconfig.json:23-42`; i18n legacy desactivado + templates estrictos
  (`tsconfig.json:44-49`).
- **Backend URL del front:** `src/environments/environment.ts:1-4` y
  `environment.development.ts:5-8` fijan `apiUrl: 'http://localhost:4200'` (sí mismo,
  sin wiring a ningún microservicio Java) → frontend **desconectado** de los backends.

## Build / package managers (gradle wrapper versión, maven, npm)

- **Gradle 8.7** (6 de 7 módulos Gradle): `gradle-wrapper.properties:3`
  `gradle-8.7-bin.zip` en `backend/{jwt,products,sales}-services` y
  `practica-examen/{ms-pedidos,ms-notificaciones}` (más `backend/sales-services`
  verificado; `products-services` y ambos de practica leídos idénticos).
  Cada uno con `gradlew` + `gradlew.bat` + `settings.gradle` (`rootProject.name = ...`).
- **Gradle 9.5.1** solo en `spring_cloud/ms-notificaciones`
  (`spring_cloud/ms-notificaciones/gradle/wrapper/gradle-wrapper.properties:3`).
- **Maven:** wrappers `mvnw` + `mvnw.cmd` SOLO en
  `spring_cloud/{eureka-server,ms-cuentas,ms-recargas,api-gateway}/` (4 módulos;
  verificado por glob `**/mvnw*` — no hay `.mvn/` ni wrapper en `resiliencia-demo/`).
  `resiliencia-demo/*/Dockerfile` usa imagen `maven:3.9-eclipse-temurin-17` multistage +
  runtime `eclipse-temurin:17-jre` (puertos 8090/8091 expuestos). `mvn` del entorno:
  **no instalado** (`mvn: command not found`).
- **npm:** `frontend/hospital_web/package.json` scripts `ng/start/build/test/lint`
  (`package.json:3-10`); lockfile: **no existe** `package-lock.json`/`yarn.lock`/`pnpm-lock`
  en `frontend/hospital_web/` (verificado por glob — brecha de reproducibilidad).
  Node/npm del entorno: v24.16.0 / 11.13.0.

## Bases de datos y migraciones

- **Motor: MySQL 8.0** vía Docker. Evidencia: `database/.env:1` (`DB_IMAGE`, valor de
  imagen MySQL 8.0), `database/docker-compose.yml:3` (`image: ${DB_IMAGE}`), volumen
  `my-db:/var/lib/mysql`, red `net-mobile`, servicio `phpmyadmin` (`DB_IMAGE_PHPMA`,
  `DB_HOST_PHPMA`, `DB_PORT_PHPMA` en `.env:9-11`).
- **Quién usa MySQL:** `backend/{jwt,products,sales}-services` y
  `practica-examen/{ms-pedidos,ms-notificaciones}` — los 5 apuntan al MISMO
  `jdbc:mysql://localhost:5510/appdb` con `ddl-auto: update`, `show-sql: true`
  (evidencia: sus 5 `application.yml:4-11`). Credenciales por defecto usuario `app`
  (var `DB_USER`); ver INTEGRATIONS.md para nombres de vars (sin valores).
- **Migraciones: NO existen.** Sin Flyway/Liquibase en ningún `build.gradle`/`pom.xml`
  (búsqueda negativa verificada); el schema se crea con `hibernate.ddl-auto: update`
  → solo apto para demo local, no versionado.
- **Sin BD:** `spring_cloud/{eureka-server,api-gateway,ms-cuentas,ms-recargas,
  ms-notificaciones}` y `resiliencia-demo/{ms-pedidos,ms-inventario}` no declaran
  datasource (sus `application.yml/yaml` solo tienen puerto + eureka o resiliencia;
  `ms-cuentas`/`ms-recargas` usan datos hardcodeados en memoria según
  `spring_cloud/README.md:12`).
- **Admin UI:** phpMyAdmin en puerto `${DB_PORT_PHPMA}` (`docker-compose.yml:22`);
  volúmenes `my-db`, `zookeeper_data`, `kafka_data`, `rabbitmq_data` en sus composes.

## Infra / messaging / service discovery

- **Service discovery — Eureka:** `spring_cloud/eureka-server` puerto **8761**
  (`eureka-server/.../application.yml:4-5`, `register-with-eureka: false`,
  `fetch-registry: false`). Clientes: `api-gateway` (8762), `ms-cuentas` (8300),
  `ms-recargas` (8100), `ms-notificaciones`/Boot4 (8083) — todos con `defaultZone:
  http://localhost:8761/eureka/` en sus `application.yml/yaml`. Ningún módulo de
  `backend/`, `practica-examen/` o `resiliencia-demo/` usa Eureka (cero `eureka-client`
  en sus builds — verificado).
- **API Gateway:** `spring_cloud/api-gateway/.../application.yml:8-21` con 3 rutas
  `lb://ms-cuentas (/cuentas/**)`, `lb://ms-recargas (/recargas/**)`,
  `lb://ms-notificaciones (/notificaciones/**)`. Detalle: línea 21 trae un resto
  `"-"` huérfano tras la ruta de notificaciones (posible typo YAML a revisar).
- **Kafka:** broker Confluent `7.4.0` + Zookeeper + kafka-ui en
  `queue/docker-compose-kafka.yml` (puertos 9092 broker, 2181 zk, 8090 kafka-ui;
  `KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"`). Clientes: `spring-kafka` en
  `backend/products+salse-services` y `practica-examen` ambos. Topics en código:
  `stock-movements` (3 particiones/1 réplica, ambos dominios) y
  `sale-cancellation-requests` (solo practica: `ms-pedidos/.../KafkaTopicConfig.java:15-16`).
  Consumer groups verificados: `stock-update-cg`, `reporting-cg`, `audit-cg`,
  `sale-cancellation-cg`.
- **RabbitMQ:** `rabbitmq:3-management` en `queue/docker-compose-rabbitmq.yml`
  (AMQP 5672 + consola 15672, credenciales default por compose). Clientes:
  `spring-boot-starter-amqp` en los mismos 4 módulos con `spring-kafka`.
  Exchanges/colas `DirectExchange` declarados en código (`RabbitMQConfig.java` de cada
  dominio): `stock-exchange`, `notification-exchange` (solo practica),
  `stock-reserve-queue`, `stock-low-queue`, `purchase-email-queue` (solo practica),
  routing keys `stock.reserve`, `stock.low`, `email.purchase.thanks` (solo practica).
- **Resiliencia/observabilidad:** Resilience4j + Actuator + AOP solo en
  `resiliencia-demo/ms-pedidos` (`pom.xml:30-48`, `application.yml:18-37` con CB
  `inventario`: ventana 5, mínimo 5 llamadas, umbral 50 %, espera 10 s, 2 llamadas
  half-open; endpoints `health,info,circuitbreakers,circuitbreakerevents`). Feign
  timeouts 1000/1000 ms (`application.yml:5-10`).
- **K8s/GitOps (solo resiliencia):** `resiliencia-demo/k8s/` (deployments + services
  `NodePort` 30081/30082 → 8091/8090), `Dockerfile`s multistage ya citados,
  `deploy.sh` modos `local` (kubectl apply) y `gitops` (ArgoCD `--path .../k8s`,
  `--sync-policy automated`). Ningún otro dominio tiene manifests K8s/Dockerfile.

## Herramientas (postman, docker)

- **Postman (6 archivos, todos verificados por parseo):**
  - `postman/Cibertec-JWT-Sales.postman_collection.json` ("Cibertec JWT Sales"):
    usuarios públicos + login + ventas/usuarios protegidos.
  - `postman/Cibertec-Microservices-Flows.postman_collection.json`
    ("Cibertec Microservices Flows"): health/seed + flujos Kafka/Rabbit/Feign + CRUD productos.
  - `postman/Cibertec-JWT-Local.postman_environment.json`: vars `jwt_base_url`,
    `jwt_token`, `jwt_user_id`.
  - `postman/Cibertec-Local.postman_environment.json`: vars `sales_base_url`,
    `products_base_url`, `product_id`, `feign_sale_id`, `last_sale_id`, `last_rabbit_sale_id`.
  - `spring_cloud/postman/sistema-fintech-tarjetas-prepago.postman_collection.json`:
    Eureka dashboard + registro + saldos/recargas directas y vía gateway.
  - `resiliencia-demo/postman/resiliencia-ms-pedidos-inventario.postman_collection.json`:
    demo sin/con resiliencia + CB endpoints + simulación de falla/demora.
  - `practica-examen/ms-pedidos/postman/ms-pedidos.postman_collection.json`: ventas +
    integración Feign/Kafka-anulación contra ms-notificaciones.
- **Docker:** `database/docker-compose.yml` (MySQL+phpMyAdmin), `queue/docker-compose-kafka.yml`
  (zk+kafka+kafka-ui), `queue/docker-compose-rabbitmq.yml` (rabbitmq), Dockerfiles solo en
  `resiliencia-demo/ms-*/Dockerfile`. Sin compose raíz que levante todo junto.
- **`.gitignore`:** raíz ignora `.DS_Store`, `.idea/`, `.vscode/`, `.env*` (salvo
  `.env.example`), `**/.gradle/`, `**/build/`, `**/bin/`, `*.class`, `*.jar/.war/.ear`,
  `*.log`, `*.db/sqlite*` (verificado `/.gitignore:1-27`). Cada servicio Spring/Maven trae
  además su `.gitignore` propio generado por Initializr.
- **READMEs por módulo (todos existen y fueron leídos):** `backend/*/README.md`,
  `spring_cloud/README.md`, `practica-examen/*/README.md`, `resiliencia-demo/README.md`,
  `frontend/hospital_web/README.md` (+ 2 READMEs internos de componentes del front).
  No hay README raíz (verificado: no existe `/README.md`).

## Tabla resumen módulo → stack

| Módulo | Lenguaje | Boot / FE | Build | BD | Mensajería | Discovery / Gateway | Puerto |
|---|---|---|---|---|---|---|---|
| `backend/jwt-sales-services` | Java 17 | Boot 3.2.5 | Gradle 8.7 | MySQL 5510/appdb (`update`) | — | — | 8083 |
| `backend/products-services` | Java 17 | Boot 3.2.5 | Gradle 8.7 | MySQL 5510/appdb (`update`) | Kafka CG ×3 + RabbitMQ | — | 8081 |
| `backend/sales-services` | Java 17 | Boot 3.2.5 + Cloud 2023.0.1 | Gradle 8.7 | MySQL 5510/appdb (`update`) | Kafka producer + RabbitMQ prod/cons + Feign | — | 8082 |
| `practica-examen/ms-pedidos` | Java 17 | Boot 3.2.5 + Cloud 2023.0.1 | Gradle 8.7 | MySQL 5510/appdb (`update`) | Kafka 2 topics + RabbitMQ 2 exchanges + Feign ×2 | — | 8082 |
| `practica-examen/ms-notificaciones` | Java 17 | Boot 3.2.5 | Gradle 8.7 | MySQL 5510/appdb (`update`) | Kafka CG ×4 + RabbitMQ 2 colas | — | 8081 |
| `spring_cloud/eureka-server` | Java 17 | Boot 3.2.5 + Cloud 2023.0.1 | Maven (mvnw) | — | — | Eureka server | 8761 |
| `spring_cloud/api-gateway` | Java 17 | Boot 3.2.5 + Cloud 2023.0.1 | Maven (mvnw) | — | — | Eureka client + Gateway 3 rutas | 8762 |
| `spring_cloud/ms-cuentas` | Java 17 | Boot 3.2.5 + Cloud 2023.0.1 | Maven (mvnw) | memoria | — | Eureka client | 8300 |
| `spring_cloud/ms-recargas` | Java 17 | Boot 3.2.5 + Cloud 2023.0.1 | Maven (mvnw) | memoria | — | Eureka client + RestTemplate LB | 8100 |
| `spring_cloud/ms-notificaciones` | Java 17 | **Boot 4.0.6** + Cloud 2025.1.1 | **Gradle 9.5.1** | — | — | Eureka client | 8083 |
| `resiliencia-demo/ms-pedidos` | Java 17 | Boot 3.2.5 + Cloud 2023.0.1 | Maven 3.9 (Docker, sin wrapper) | — | — | Feign + Resilience4j + Actuator | 8090 |
| `resiliencia-demo/ms-inventario` | Java 17 | Boot 3.2.5 | Maven 3.9 (Docker, sin wrapper) | — | — | Actuator | 8091 |
| `frontend/hospital_web` | TS 5.9 / ES2022 | Angular 21 + Material | Angular CLI 21 / npm (sin lockfile) | — | — | `apiUrl` localhost:4200 (loopback) | 4200 |
| `database/` | Compose+YAML | — | docker compose | MySQL 8.0 + phpMyAdmin | — | red `net-mobile` | 5510/3410 |
| `queue/` | Compose+YAML | — | docker compose | — | Kafka 7.4.0 (9092/8090 UI) + RabbitMQ 3-mgmt (5672/15672) | — | ver celdas |
