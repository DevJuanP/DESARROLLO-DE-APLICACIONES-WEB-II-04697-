# Docker paso a paso — Familias A y C (infra: MySQL + Kafka + RabbitMQ)

> Alcance: llegar hasta tener la infraestructura Docker corriendo.
> Los micros (servidor/cliente) se levantan después en IntelliJ IDEA con `bootRun`. Eso queda fuera de esta guía.

## 1. Qué vas a levantar y por qué

Docker aquí **no corre tu código Java**. Solo levanta infraestructura:

| Qué | Compose | Contenedores | Puertos host |
|---|---|---|---|
| Base de datos | `database/docker-compose.yml` | MySQL `mysqldb` + phpMyAdmin `tsl.phpmyadmin` | `5510` MySQL, `3410` phpMyAdmin |
| Mensajero Kafka | `queue/docker-compose-kafka.yml` | Zookeeper + Kafka + Kafka-UI | `9092` Kafka, `8090` Kafka-UI |
| Mensajero RabbitMQ | `queue/docker-compose-rabbitmq.yml` | RabbitMQ | `5672` AMQP, `15672` consola |

Config real de la base (`database/.env`):

```properties
DB_IMAGE=mysql:8.0
DB_HOST=mysqldb
DB_PORT=5510
DB_NAME=appdb
DB_USER=app
DB_PASSWD=password
DB_PORT_PHPMA=3410
```

Regla del repo: **una familia a la vez**. Hay colisiones en `8081`, `8082` (A ↔ C), `8083` (A ↔ B) y `8090` (Kafka-UI ↔ familia D). Apaga una antes de levantar otra.

## 2. Requisitos previos

1. Windows 10 22H2 o Windows 11 64-bit actualizado.
2. Virtualización habilitada (ver paso 3).
3. WSL2 instalado.
4. Docker Desktop con backend WSL2.
5. Repo clonado en local, ej: `D:\...\DESARROLLO-DE-APLICACIONES-WEB-II-04697-`.
6. JDK 17 para los micros (no 21/25). Se aconseja descargarlo desde [adoptium.net → Temurin 17 LTS → Windows x64 (.msi)](https://adoptium.net/temurin/releases/?version=17) y configurar en IntelliJ Project SDK 17 + Gradle JVM 17.

Familias B y D **no necesitan** esta guía en local. D trae `Dockerfile` + `k8s/` solo para la demo de despliegue.

## 3. Verificar virtualización (antes de tocar la BIOS)

1. Abre `Administrador de tareas > Rendimiento > CPU`.
2. Mira `Virtualización`:
   - `Habilitada` → salta al paso 4.
   - `Deshabilitada` → entra a la BIOS/UEFI.

Si tienes que entrar a la BIOS, cambia **solo esto**:

- Intel: `Intel Virtualization Technology / VT-x` (y `VT-d` si aparece) → `Enabled`.
- AMD: `SVM Mode / AMD-V` → `Enabled`.

Rutas típicas: `Advanced > CPU Configuration`, Lenovo `Security > Virtualization`, HP `Security > System Security`, Dell `Virtualization Support`. Guarda con `F10` y reinicia.

Cómo entrar: `Configuración > Sistema > Recuperación > Inicio avanzado > Reiniciar ahora > Solucionar problemas > Opciones avanzadas > Configuración de firmware UEFI`, o al encender pulsar `Supr`, `F2`, `F10`, `Esc` según marca.

## 4. Instalar WSL2

En PowerShell **como administrador**:

```powershell
wsl --install
wsl --update
wsl --status
wsl -l -v
```

Lo último debe decir `VERSION 2`. Reinicia si te lo pide.

## 5. Instalar Docker Desktop

1. Descarga `Docker Desktop for Windows` desde docker.com.
2. En el instalador deja marcado `Use WSL 2 instead of Hyper-V`.
3. Abre Docker Desktop y espera a `Engine running` en verde abajo a la izquierda.
4. En `Settings > General` deja `Use WSL 2 based engine` activado.

Verifica en PowerShell normal:

```powershell
docker --version
docker compose version
docker run hello-world
docker ps
```

Esperado: `Docker version 29.x`, `Compose v5.x`, mensaje `Hello from Docker!`. `docker ps` vacío es normal porque `hello-world` termina y se apaga.

## 6. Levantar la base de datos (MySQL + phpMyAdmin)

```powershell
cd "D:\Cibertec\6to ciclo\DAW II\fork\DESARROLLO-DE-APLICACIONES-WEB-II-04697-\database"
docker compose up -d
docker ps
docker compose ps
docker compose logs -f db
```

Verificación:

1. `docker ps` muestra `mysqldb` y `tsl.phpmyadmin`.
2. Abre `http://localhost:3410`.
3. Login: Servidor `mysqldb`, usuario `app`, contraseña `password`. Debes ver la base `appdb`.
4. En Docker Desktop ves los 2 contenedores en verde.

Si `logs -f db` dice `ready for connections`, la base está lista para los micros.

## 7. Levantar Kafka

```powershell
cd "..\queue"
docker compose -f docker-compose-kafka.yml up -d
docker ps
```

Verificación:

1. Contenedores `ligo-zookeeper`, `ligo-kafka`, `ligo-kafka-ui` corriendo.
2. Abre `http://localhost:8090`, debes ver el cluster `ligo-local`.
3. Tu Spring Boot se conectará a `localhost:9092`.

Kafka tarda 20-40 s la primera vez. Si Kafka-UI sale vacío, espera y recarga.

## 8. Levantar RabbitMQ

```powershell
docker compose -f docker-compose-rabbitmq.yml up -d
docker ps
```

Verificación:

1. Contenedor `ligo-rabbitmq` corriendo.
2. Abre `http://localhost:15672`, usuario `guest` / contraseña `guest`.
3. Tu Spring Boot se conectará a `localhost:5672` por AMQP.

## 9. Estado final esperado

`docker ps` debe mostrar (nombres y puertos):

```text
mysqldb        0.0.0.0:5510->3306/tcp
tsl.phpmyadmin 0.0.0.0:3410->80/tcp
ligo-zookeeper 2181/tcp
ligo-kafka     0.0.0.0:9092->9092/tcp
ligo-kafka-ui  0.0.0.0:8090->8080/tcp
ligo-rabbitmq  0.0.0.0:5672->5672/tcp, 0.0.0.0:15672->15672/tcp
```

En este punto la infra está lista. Siguiente paso (fuera de esta guía): abrir cada micro en IntelliJ IDEA y levantarlo con `bootRun`, luego probar con Postman por HTTP/REST.

## 10. Apagar (importante)

Apaga siempre la familia antes de cambiar a otra:

```powershell
# estando en queue/
docker compose -f docker-compose-kafka.yml down
docker compose -f docker-compose-rabbitmq.yml down

# estando en database/
docker compose down
```

Comandos de consulta:

```powershell
docker ps              # qué contenedores están corriendo ahora
docker compose ps      # estado del compose de la carpeta donde estás parado
docker compose logs -f # logs en vivo para diagnosticar
docker volume ls       # volúmenes con datos persistidos (my-db, kafka_data, rabbitmq_data)
```

`down` apaga y borra contenedores/red, pero **no borra los volúmenes**, tus datos de MySQL/Kafka/Rabbit se conservan. Para borrar todo incluido datos: `docker compose down -v` (cuidado, borra la base).

## 11. Fallas típicas

| Síntoma | Causa probable | Qué hacer |
|---|---|---|
| `port is already allocated` en `5510/8090/15672` | Otra familia o programa usa ese puerto | `docker ps`, apaga la otra familia con `down`, o `netstat -ano \| findstr :5510` |
| `network net-mobile already exists` / conflicto | Red creada por otra familia | `docker network ls`, normalmente se reutiliza sin problema, no la borres si la otra familia la usa |
| MySQL se reinicia en bucle | `.env` modificado o volumen corrupto | `docker compose logs -f db`, verifica `.env`, último recurso `docker compose down -v` y `up -d` de nuevo |
| Kafka-UI vacío | Kafka aún arrancando | Espera 30 s, `docker compose logs -f kafka`, recarga `:8090` |
| `docker: command not found` | Docker Desktop cerrado | Abre Docker Desktop y espera `Engine running` |
| Error WSL `0x80370102` | Virtualización o WSL2 mal activado | Vuelve al paso 3, verifica BIOS + `wsl --update` |
