# Flujos Kafka y RabbitMQ paso a paso (Familia A)

Guía para probar los mensajeros con la colección `postman/Cibertec-Microservices-Flows.postman_collection.json`
+ environment `postman/Cibertec-Local.postman_environment.json`.

## 0. Requisitos previos

1. Infra Docker arriba (ver [docs/docker-paso-a-paso.md](../docs/docker-paso-a-paso.md)):
   - MySQL `localhost:5510` + phpMyAdmin `:3410`
   - Kafka `:9092` + Kafka-UI `:8090`
   - RabbitMQ `:5672` + consola `:15672` (guest/guest)
2. Micros corriendo **en este orden**:
   - `products-services` `:8081` primero (los demás dependen de él)
   - `sales-services` `:8082` segundo
3. En Postman: importa la colección **y** el environment `Cibertec Local`, y **selecciónalo**
   arriba a la derecha. Sin environment activo, `{{last_sale_id}}` no se guarda y los
   `GET` posteriores fallan.

## 1. Health check (`0. Health And Seed`)

Corre `List Products` y `List Sales`. Ambos deben dar `200`.

- Anota si existe el producto `id: 1` y su `stock`. Los POST de los flujos usan
  `"productId": 1` (`quantity: 2` en Kafka, `quantity: 4` en Rabbit).
- Si no hay productos, crea uno con `4. Optional Product Setup > Create Product`.

## 2. Flujo Kafka (`1. Kafka Flow`)

Qué prueba: `sales` publica un evento en el topic `stock-movements` y `products` lo consume
con 3 consumer-groups (`stock-update-cg` descuenta stock, `reporting-cg`, `audit-cg`).

1. Abre Kafka-UI `http://localhost:8090` → cluster `ligo-local` → Topics. Déjalo abierto.
2. Corre `POST Create Sale - Kafka Stock Movement` → esperado `200`, body con
   `status: PENDING`. El test guarda `last_sale_id` en el environment.
3. Verifica el mensaje en 3 lugares:
   - **Kafka-UI:** Topics → `stock-movements` → Messages. 1 mensaje con key `1` y el JSON
     del evento (refresca si no aparece al instante; Kafka tarda unos segundos).
   - **Log de `sales-services`:** `Evento publicado en Kafka. topic=stock-movements, key=1...`
     (el productor).
   - **Log de `products-services`:** actividad de los 3 consumers del topic.
4. Corre `GET Get Last Sale` → trae la venta creada.
5. Corre de nuevo `List Products` → el stock del producto 1 bajó en 2. Ciclo completo:
   POST → Kafka → consumo → descuento.

Si el POST falla con error de conexión a `:9092`, el contenedor `ligo-kafka` no está arriba
(`docker ps` para verificar).

## 3. Flujo RabbitMQ (`2. RabbitMQ Flow`)

Qué prueba: `sales` publica `StockReserveEvent` al exchange `stock-exchange` (key `stock.reserve`,
cola `stock-reserve-queue`). Lo consume `products-services` (`StockReserveConsumer`). Si el stock
final queda menor a 5, `products` emite alerta a `stock-low-queue` y la consume `sales-services`
(`StockLowAlertConsumer`).

### 3.1 Ver el mensaje quieto en la cola (truco: detener al consumidor)

Si consumes con `products` corriendo, el mensaje desaparece al instante y no lo ves en la consola.
Para inspeccionar el payload:

1. En IntelliJ **detén solo `products-services` (`:8081`)**. Deja `sales-services` (`:8082`)
   corriendo: es quien expone el endpoint y publica. Si apagas `sales`, no puedes disparar nada.
2. Abre `http://localhost:15672` (guest/guest) → pestaña **Queues**. Verás
   `stock-reserve-queue` con 0 consumers.
3. Corre `POST Create Sale - Rabbit Reserve` → `200`, `PENDING`.
4. Vuelve a Queues → `stock-reserve-queue` tiene 1 mensaje en **Ready**. Entra a la cola →
   abajo **Get Message(s)** → verás el JSON del evento.
5. Repite el POST si quieres acumular 2-3 mensajes.

### 3.2 Ver el consumo

1. Levanta `products-services` de nuevo.
2. En la consola: Ready baja a 0 (los consume) y en su log verás el procesamiento.
   Procesará todo lo acumulado de golpe: es lo esperado, no un error.
3. Corre `GET Get Last Rabbit Sale` → trae la venta creada por este flujo.
4. Bonus: si el stock final quedó menor a 5, revisa el log de `sales-services`: llegó la alerta
   de `stock-low-queue`.

## 4. Flujo Feign (`3. Feign Flow`, opcional)

Con ambos micros arriba, corre `GET Get Sale Details Through Feign`. `sales-services` llama
síncronamente a `products-services` (`Feign GET /products/{id}`) y devuelve la venta enriquecida
con el producto. Si `products` está apagado, este flujo falla: es la diferencia clave con los
flujos async (Kafka/Rabbit siguen aceptando mensajes aunque el consumidor esté caído).
