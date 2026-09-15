# Docker — guía genérica (agnóstica a cualquier repo)

> Nota para Obsidian: archivo autocontenido. Puedes copiarlo tal cual a tu vault y darle tu formato, tags y links.

## 1. Qué es Docker en una frase

Docker empaqueta una app con sus dependencias en una **imagen** y la corre aislada en un **contenedor**, igual en cualquier máquina.

Sin Docker: "en mi máquina sí funciona".
Con Docker: funciona igual si el contenedor corre.

## 2. Piezas: Engine, Desktop, CLI, Hub

- **Docker Engine / Daemon (`dockerd`):** el motor que crea y corre contenedores. Sin esto nada funciona. En Linux corre nativo, en Windows/Mac corre dentro de una VM ligera (WSL2 en Windows).
- **Docker CLI (`docker`):** el comando de consola que le habla al Engine. Ej: `docker ps`, `docker run`.
- **Docker Compose (`docker compose`):** herramienta para levantar varios contenedores juntos definidos en un YAML. Ej: base de datos + backend + mensajero con un solo `up -d`.
- **Docker Desktop:** paquete para Windows/Mac (y Linux) que trae Engine + CLI + Compose + interfaz gráfica. Es lo recomendado si eres principiante en Windows.
- **Docker Hub:** registro público de imágenes. `docker run mysql` descarga `mysql` desde ahí si no la tienes.

Aclaración clave: **"Docker de consola" no existe solo**. Siempre necesitas el Engine. La pregunta real es: ¿instalo Engine+CLI+GUI con Desktop, o instalo solo Engine+CLI a mano en Linux/WSL2? Para empezar, Desktop.

## 3. Conceptos base

### Imagen vs contenedor

- **Imagen:** plantilla inmutable. Ej: `mysql:8.0`, `rabbitmq:3-management`, `hello-world:latest`.
- **Contenedor:** instancia corriendo de una imagen. Puedes crear 5 contenedores de la misma imagen.

```bash
docker pull mysql:8.0      # descarga la imagen
docker run -d --name mi-mysql -p 5510:3306 -e MYSQL_ROOT_PASSWORD=password mysql:8.0
docker ps                  # ver contenedores corriendo
```

### Puerto `host:contenedor`

`-p 5510:3306` significa: puerto `5510` de tu PC → puerto `3306` del contenedor. Tu app se conecta a `localhost:5510` sin saber que dentro es `3306`.

### Volúmenes (datos que sobreviven)

Por defecto si borras un contenedor pierdes lo de dentro. Un **volumen** guarda datos fuera del contenedor:

```yaml
volumes:
  - my-db:/var/lib/mysql
```

`docker compose down` borra contenedores pero **conserva volúmenes**. `docker compose down -v` borra también los datos. Úsalo con cuidado.

### Redes

Compose crea una red privada para que los contenedores se hablen por nombre. Ej: phpMyAdmin se conecta al host `mysqldb`, no a `localhost`, porque ambos están en la red `net-mobile`.

### Dockerfile vs Compose

- **Dockerfile:** receta para construir tu propia imagen (FROM, COPY, RUN, CMD).
- **Compose (`docker-compose.yml`):** plano para correr N contenedores ya existentes (o construidos de un Dockerfile) con puertos, variables, volúmenes y redes.

Ejemplo mínimo de Compose:

```yaml
services:
  db:
    image: mysql:8.0
    ports:
      - "5510:3306"
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: appdb
    volumes:
      - my-db:/var/lib/mysql
volumes:
  my-db:
```

## 4. Instalación (resumen por SO)

### Windows (recomendado: Desktop + WSL2)

1. Activa virtualización en BIOS/UEFI (Intel VT-x / AMD SVM → Enabled). Verifica en `Administrador de tareas > Rendimiento > CPU`.
2. En PowerShell admin: `wsl --install`, `wsl --update`, reinicia.
3. Instala Docker Desktop con `Use WSL 2` marcado. Espera `Engine running`.
4. Verifica: `docker --version`, `docker compose version`, `docker run hello-world`.

### Linux (Ubuntu/Debian)

Engine nativo, sin Desktop obligatorio:

```bash
sudo apt update && sudo apt install -y docker.io docker-compose-plugin
sudo usermod -aG docker $USER
# cierra y reabre sesión
docker run hello-world
```

### Mac

Instala Docker Desktop desde docker.com (usa Virtualization framework). Los comandos son los mismos.

## 5. Ciclo de vida y comandos que sí o sí debes saber

```bash
docker --version
docker compose version

docker ps                 # contenedores corriendo
docker ps -a              # incluye detenidos
docker images             # imágenes descargadas

docker pull <imagen>      # descargar, ej: docker pull mysql:8.0
docker run hello-world    # prueba canónica: si dice Hello from Docker, todo ok
docker run -d --name web -p 8080:80 nginx
docker logs -f <nombre>   # ver logs en vivo
docker exec -it <nombre> sh  # entrar al contenedor
docker stop <nombre>      # detener
docker rm <nombre>        # borrar contenedor detenido
docker volume ls          # ver datos persistidos
```

Con Compose (siempre desde la carpeta del YAML):

```bash
docker compose up -d                          # levanta en segundo plano
docker compose ps                             # estado de ESTA carpeta
docker compose logs -f                        # logs de ESTA familia
docker compose logs -f <servicio>             # solo un servicio, ej: db, kafka
docker compose down                           # apaga y borra contenedores/red, conserva datos
docker compose down -v                        # apaga y BORRA datos (peligroso)
docker compose -f otro-archivo.yml up -d      # compose con nombre custom
```

Truco: `docker ps` es global (todo lo corriendo). `docker compose ps` es local (solo la carpeta donde estás parado).

## 6. Cómo diagnosticar cuando algo falla

1. ¿Está el Engine? Docker Desktop debe decir `Engine running`. Si no, `docker ps` da error de daemon.
2. ¿Está corriendo? `docker ps -a`, mira `STATUS` (Up, Exited, Restarting).
3. ¿Qué dice? `docker logs -f <nombre>` o `docker compose logs -f <servicio>`. El 90% de la respuesta está ahí.
4. ¿Puerto ocupado? `port is already allocated` → otro contenedor o programa usa ese puerto. `docker ps` para encontrarlo, o en Windows `netstat -ano | findstr :<puerto>`.
5. ¿Arrancando lento? Bases y Kafka tardan 20-60 s la primera vez. `ready for connections` en MySQL = listo.
6. ¿Limpieza? `docker system df` ve espacio, `docker system prune` borra contenedores/imágenes colgadas (no toca volúmenes salvo `-v`).

## 7. Buenas prácticas mínimas

- Un contenedor = un proceso (base por un lado, app por otro).
- Nunca subas contraseñas al repo en texto plano si es proyecto real; usa `.env` + `.gitignore`. Los `.env` de curso son de juguete.
- Fija versiones (`mysql:8.0`, no `mysql:latest`) para que no cambie solo.
- Usa `restart: always` o `unless-stopped` solo en infra local, no en jobs de una sola vez.
- Nombra contenedores y redes para reconocerlos en `docker ps`.
- Apaga lo que no uses: menos puertos ocupados, menos RAM.

## 8. Errores típicos de principiante

| Error | Qué significa |
|---|---|
| `Cannot connect to the Docker daemon` | Engine apagado, abre Docker Desktop |
| `port is already allocated` | Puerto host ocupado, cambia el mapeo o apaga el otro contenedor |
| `command not found: docker` | CLI no instalada o terminal abierta antes de instalar, reabre terminal |
| `permission denied` en Linux | Falta grupo `docker`, haz `usermod -aG docker` y reloguea |
| Datos desaparecieron tras `down -v` | `-v` borra volúmenes, era esperado |
| Contenedor `Exited (1)` en bucle | Mira `logs`, suele ser variable de entorno o volumen corrupto |
| WSL `0x80370102` en Windows | Virtualización desactivada o WSL desactualizado |

## 9. Checklist mental de 30 segundos

1. `docker ps` → ¿corre lo que espero?
2. No → `docker compose ps` + `docker compose logs -f`.
3. ¿Puertos? ¿Variables `.env`? ¿Volumen corrupto?
4. Arreglo → `up -d` de nuevo → verifico URL/puerto.
5. Termino → `down` (sin `-v` si quiero conservar datos).
