# Parte 4 — Eventos, Outbox Pattern y Externalización

**Duración**: 25 minutos  
**Objetivo**: Implementar comunicación event-driven entre módulos con garantías de entrega

---

## ¿Qué hacemos aquí?

La Parte 3 usó eventos **internos** al módulo `catalog` para sincronizar CQRS. Ahora implementamos eventos **entre módulos** — la comunicación entre `orders` e `inventory`.

1. La diferencia entre `@EventListener`, `@TransactionalEventListener` y `@ApplicationModuleListener`
2. El **problema del evento perdido** y por qué importa en producción
3. El **Event Publication Registry** — outbox pattern sin infraestructura extra
4. `@Externalized` — publicar eventos a RabbitMQ en una sola línea

---

## El escenario

Cuando se crea una orden necesitamos dos cosas:

1. **Dentro de la aplicación**: `inventory` descuenta el stock del producto
2. **Fuera de la aplicación**: sistemas externos (notificaciones, analytics, logística) se enteran

---

## Evolución 1: @EventListener — el básico

```java
@Component
class OrderEventsInventoryHandler {

    @EventListener  // ← el más simple
    void on(OrderCreatedEvent event) {
        inventoryService.decreaseStock(event.productCode(), event.quantity());
    }
}
```

El problema es que se ejecuta **dentro de la misma transacción** que creó la orden:

```
BEGIN TRANSACTION
  1. OrderService guarda la orden
  2. @EventListener se ejecuta INMEDIATAMENTE
  3. InventoryService descuenta stock
COMMIT
```

Si el paso 3 falla, el ROLLBACK también deshace la orden. Los módulos están acoplados transaccionalmente.

---

## Evolución 2: @TransactionalEventListener — separar las transacciones

```java
@Async
@Transactional(propagation = Propagation.REQUIRES_NEW)
@TransactionalEventListener  // ← se ejecuta DESPUÉS del commit
void on(OrderCreatedEvent event) {
    inventoryService.decreaseStock(event.productCode(), event.quantity());
}
```

Las transacciones ahora son independientes. Pero hay un problema nuevo.

### El problema del evento perdido

```
BEGIN TRANSACTION (orders)
  1. OrderService guarda la orden
  2. El evento queda en memoria RAM
COMMIT ✅

← la app se reinicia aquí, o el servidor falla

→ El evento nunca llega a inventory
→ La orden existe pero el stock no se descontó
→ Inconsistencia silenciosa
```

Nadie lo sabe. Sin logs de error, sin alertas. Simplemente el evento murió con el proceso.

---

## Evolución 3: @ApplicationModuleListener + Event Publication Registry

Spring Modulith persiste el evento en la base de datos **dentro de la misma transacción que lo publicó**. Si la app cae, el evento sigue en la BD y se reintenta al reiniciar.

### Paso 1: Agregar dependencias al pom.xml

```xml
<!-- Event Publication Registry (Outbox Pattern) -->
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-jdbc</artifactId>
</dependency>

<!-- Externalización de eventos a RabbitMQ -->
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-events-amqp</artifactId>
</dependency>
```

### Paso 2: Configurar application.properties

```properties
# Crea la tabla event_publication automáticamente al arrancar
spring.modulith.events.jdbc.schema-initialization.enabled=true

# "delete"  → borra el registro cuando el handler completa exitosamente
# "archive" → lo mueve a event_publication_archive (útil para auditoría)
spring.modulith.events.completion-mode=delete

# Si la app reinicia con eventos pendientes, los republica automáticamente
spring.modulith.events.republish-outstanding-events-on-restart=true
```

Con esto activado, el flujo queda:

```
BEGIN TRANSACTION (orders)
  1. OrderService guarda la orden en BD
  2. Spring Modulith persiste el evento en event_publication
     (misma transacción → si falla la orden, no hay evento)
COMMIT ✅

(el evento está en BD, garantizado)

(thread separado, transacción independiente)
BEGIN TRANSACTION (inventory)
  3. @ApplicationModuleListener ejecuta on()
  4. InventoryService descuenta stock
  5. Spring Modulith marca el evento como completado
COMMIT ✅
```

Si la app cae entre los pasos 2 y 3, al reiniciar Spring Modulith relee `event_publication` y reintenta la entrega.

---

## Tabla de comparación

| Mecanismo | ¿Cuándo ejecuta? | ¿Transacción propia? | ¿Resistente a caídas? |
|---|---|---|---|
| `@EventListener` | Dentro del publish | Misma que el publisher | ❌ No |
| `@TransactionalEventListener` | Después del commit | Nueva transacción | ❌ No (RAM) |
| `@ApplicationModuleListener` | Después del commit | Nueva transacción | ✅ Sí (BD) |

---

## Paso 3: Actualizar OrderCreatedEvent

El evento del starter tenía campos planos. Lo actualizamos para agregar `@Externalized` y un `Customer` embebido — esto hace que el mensaje de RabbitMQ sea más claro para consumidores externos que no necesitan consultar la app para obtener los datos del cliente.

```java
// orders/OrderCreatedEvent.java
package com.geovannycode.bookstore.orders;

import org.springframework.modulith.events.Externalized;
import java.math.BigDecimal;

/**
 * Evento publicado cuando una orden es creada.
 *
 * Es la API pública del módulo orders — está en la raíz del paquete
 * para que inventory (y cualquier otro módulo) pueda importarlo.
 *
 * @Externalized indica a Spring Modulith que publique este evento
 * también en RabbitMQ. El formato es: "exchangeName::routingKey"
 *
 * El record Customer va embebido para que los consumidores externos
 * no necesiten hacer una consulta adicional para obtener datos del cliente.
 */
@Externalized("bookstore.orders::order.created")
public record OrderCreatedEvent(
        String orderNumber,
        String productCode,
        String productName,
        BigDecimal productPrice,
        int quantity,
        Customer customer
) {
    public record Customer(
            String name,
            String email,
            String phone,
            String deliveryAddress
    ) {}
}
```

---

## Paso 4: Actualizar OrderService

`OrderService` necesita tres ajustes respecto a la Parte 1:
- Constructor de `OrderEntity` expandido con los campos individuales
- `eventPublisher.publishEvent()` actualizado con el nuevo `Customer` embebido
- Agregar `getByOrderNumber()` que usa `OrderRestController`

```java
// orders/domain/OrderService.java
package com.geovannycode.bookstore.orders.domain;

import com.geovannycode.bookstore.catalog.CatalogApi;
import com.geovannycode.bookstore.orders.OrderCreatedEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.UUID;

@Service
@Transactional
public class OrderService {

    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    private final OrderRepository orderRepository;
    private final CatalogApi catalogApi;
    private final ApplicationEventPublisher eventPublisher;

    public OrderService(OrderRepository orderRepository,
                        CatalogApi catalogApi,
                        ApplicationEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.catalogApi = catalogApi;
        this.eventPublisher = eventPublisher;
    }

    public CreateOrderResponse create(CreateOrderRequest request) {
        var product = catalogApi.getByCode(request.productCode())
                .orElseThrow(() -> new InvalidOrderException(
                        "Producto no encontrado: " + request.productCode()));

        var orderNumber = "ORD-" + UUID.randomUUID()
                .toString().substring(0, 8).toUpperCase();

        // Constructor expandido con los campos individuales de OrderEntity
        var order = new OrderEntity(
                orderNumber,
                request.customerName(), request.customerEmail(),
                request.customerPhone(), request.deliveryAddress(),
                product.code(), product.name(),
                product.price(), request.quantity()
        );

        var saved = orderRepository.save(order);
        log.info("Orden creada: orderNumber={}, product={}", orderNumber, product.code());

        // Spring Modulith persiste este evento en event_publication
        // dentro de esta misma transacción.
        // Si el COMMIT falla, el evento tampoco se persiste.
        // Si la app cae después del COMMIT, el evento queda en BD
        // y se reintenta al reiniciar.
        eventPublisher.publishEvent(new OrderCreatedEvent(
                saved.getOrderNumber(),
                product.code(),
                product.name(),
                product.price(),
                request.quantity(),
                new OrderCreatedEvent.Customer(
                        request.customerName(), request.customerEmail(),
                        request.customerPhone(), request.deliveryAddress()
                )
        ));

        return new CreateOrderResponse(orderNumber);
    }

    @Transactional(readOnly = true)
    public OrderEntity getByOrderNumber(String orderNumber) {
        return orderRepository.findByOrderNumber(orderNumber)
                .orElseThrow(() -> new OrderNotFoundException(orderNumber));
    }
}
```

---

## Paso 5: Actualizar OrderEventsInventoryHandler

Reemplazamos las tres anotaciones de la Parte 1 por `@ApplicationModuleListener`. Esta anotación está disponible a partir de que tienes `spring-modulith-starter-core` en el classpath (que ya teníamos desde la Parte 1). La diferencia es que **ahora** con `spring-modulith-starter-jdbc` activo, también persiste el evento en `event_publication`.

```java
// inventory/OrderEventsInventoryHandler.java
package com.geovannycode.bookstore.inventory;

import com.geovannycode.bookstore.orders.OrderCreatedEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.modulith.events.ApplicationModuleListener;
import org.springframework.stereotype.Component;

/**
 * Handler que reduce el stock cuando se crea una orden.
 *
 * inventory no sabe nada de cómo funciona orders internamente.
 * Solo conoce el evento público OrderCreatedEvent.
 *
 * @ApplicationModuleListener garantiza:
 * - Se ejecuta DESPUÉS del commit de la transacción de orders
 * - En su propia transacción independiente (REQUIRES_NEW)
 * - Si falla, la orden ya está guardada (no hay rollback cruzado)
 * - Si la app cae antes de ejecutarse, Spring Modulith reintenta al reiniciar
 */
@Component
class OrderEventsInventoryHandler {

    private static final Logger log = LoggerFactory.getLogger(OrderEventsInventoryHandler.class);

    private final InventoryService inventoryService;

    OrderEventsInventoryHandler(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }

    @ApplicationModuleListener
    void on(OrderCreatedEvent event) {
        log.info("Actualizando stock → order={}, product={}, qty={}",
                event.orderNumber(), event.productCode(), event.quantity());

        inventoryService.decreaseStock(event.productCode(), event.quantity());
    }
}
```

!!! warning "Si @ApplicationModuleListener no resuelve"
    Verifica que el import sea exactamente:
    ```java
    import org.springframework.modulith.events.ApplicationModuleListener;
    ```
    Si el IDE sigue sin resolverlo, usa las tres anotaciones equivalentes mientras se resuelve la dependencia:
    ```java
    @Async
    @TransactionalEventListener
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    void on(OrderCreatedEvent event) { ... }
    ```
    El comportamiento es idéntico — `@ApplicationModuleListener` es un alias de las tres juntas.

---

## Paso 6: Actualizar RabbitMQConfig

`Jackson2JsonMessageConverter` está deprecada en Spring AMQP 4.x (que viene con Spring Boot 4.x). El reemplazo es `Jackson3JsonMessageConverter` — el mismo propósito, compatible con Jackson 3.

```java
// config/RabbitMQConfig.java
package com.geovannycode.bookstore.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.TopicExchange;
import org.springframework.amqp.support.converter.Jackson3JsonMessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * Configuración de RabbitMQ para externalización de eventos.
 *
 * Spring Modulith usa el formato "exchange::routingKey" en @Externalized:
 *   @Externalized("bookstore.orders::order.created")
 *   → exchange: "bookstore.orders"
 *   → routing key: "order.created"
 *
 * El Jackson3JsonMessageConverter serializa los eventos como JSON
 * al publicarlos en RabbitMQ.
 */
@Configuration
public class RabbitMQConfig {

    @Bean
    TopicExchange ordersExchange() {
        return new TopicExchange("bookstore.orders", true, false);
    }

    @Bean
    Queue orderCreatedQueue() {
        return new Queue("bookstore.order.created", true);
    }

    @Bean
    Binding orderCreatedBinding(Queue orderCreatedQueue, TopicExchange ordersExchange) {
        return BindingBuilder
                .bind(orderCreatedQueue)
                .to(ordersExchange)
                .with("order.created");
    }

    @Bean
    Jackson3JsonMessageConverter messageConverter() {
        return new Jackson3JsonMessageConverter();
    }
}
```

---

## Paso 7: Demo completa — el flujo de una orden

Levanta la aplicación:

```bash
mvn spring-boot:run
```

### Crear la orden

Desde Postman, Insomnia o cualquier cliente HTTP:

```http
POST http://localhost:8080/api/orders
Content-Type: application/json

{
  "productCode": "P001",
  "quantity": 2,
  "customerName": "Geovanny Mendoza",
  "customerEmail": "geo@barranquillajug.com",
  "customerPhone": "+57 300 1234567",
  "deliveryAddress": "Calle 72 #45-10, Barranquilla"
}
```

Respuesta esperada:
```json
{
  "orderNumber": "ORD-5A3133F2"
}
```

Anota el `orderNumber` — lo usarás en las verificaciones siguientes.

### Observar los logs

En la consola deberías ver:

```
INFO  OrderService               : Orden creada: orderNumber=ORD-5A3133F2, product=P001
INFO  OrderEventsInventoryHandler: Actualizando stock → order=ORD-5A3133F2, product=P001, qty=2
INFO  InventoryService            : Stock disminuido: product=P001, quantity=2, remaining=596
```

Los logs de `OrderEventsInventoryHandler` aparecen **después** del de `OrderService` porque `@ApplicationModuleListener` ejecuta post-commit, en un thread separado.

---

## Verificar en PostgreSQL

Conéctate a la base de datos:

```bash
docker exec -it bookstore-modulith-postgres-1 \
  psql -U bookstore -d bookstore
```

Las tablas del starter están en el schema `public`. Reemplaza `ORD-5A3133F2` con tu orderNumber real:

```sql
-- La orden existe en la tabla orders (schema public del starter)
SELECT order_number, status, product_code, quantity
FROM orders
WHERE order_number = 'ORD-5A3133F2';

-- El stock se descontó en la tabla stock (schema public del starter)
SELECT product_code, stock_level
FROM stock
WHERE product_code = 'P001';
-- Debe mostrar 596 (el starter carga 598 para P001;
-- creaste una orden de 2 unidades → 598 - 2 = 596)

-- El event_publication debe estar vacío si completion-mode=delete
-- y los handlers ya procesaron el evento
SELECT * FROM event_publication;
-- 0 rows = los eventos se entregaron y se borraron exitosamente

-- Si hay filas con completion_date = NULL, el handler aún no procesó
SELECT event_type, publication_date, completion_date
FROM event_publication
WHERE completion_date IS NULL;
```

!!! tip "Si ves filas en event_publication con completion_date = NULL"
    El handler no procesó el evento todavía (puede estar en cola asíncrona) o falló. Espera 2-3 segundos y vuelve a consultar. Si persiste con `completion_date IS NULL`, revisa los logs de la aplicación para ver si hay un error en `OrderEventsInventoryHandler`.

---

## Verificar en RabbitMQ

El evento `OrderCreatedEvent` anotado con `@Externalized` también llega a RabbitMQ. Así lo verificas:

**1.** Abre [http://localhost:15672](http://localhost:15672) con `guest` / `guest`

**2.** Ve a **Queues and Streams** → haz clic en `bookstore.order.created`

**3.** En la página de la cola, baja hasta la sección **Get messages**

**4.** Haz clic en el título **Get messages** para expandirla

**5.** Deja el campo *Count* en `1` y haz clic en el botón **Get Message(s)**

**6.** Verás el mensaje con su contenido JSON:

```json
{
  "orderNumber": "ORD-5A3133F2",
  "productCode": "P001",
  "productName": "Clean Code",
  "productPrice": 45.99,
  "quantity": 2,
  "customer": {
    "name": "Geovanny Mendoza",
    "email": "geo@barranquillajug.com",
    "phone": "+57 300 1234567",
    "deliveryAddress": "Calle 72 #45-10, Barranquilla"
  }
}
```

!!! info "¿Por qué el mensaje no desaparece de la cola?"
    El admin de RabbitMQ tiene el modo `nack` por defecto al leer mensajes de esta forma. El mensaje vuelve a la cola después de verlo. En producción, un consumer real (otro microservicio, por ejemplo) consumiría y confirmaría el mensaje. En el workshop no tenemos consumer externo configurado — solo verificamos que el mensaje llegó.

---

## Verificar el Event Publication Registry antes del procesamiento

Si quieres ver el evento mientras aún está en proceso (antes de que el handler termine), puedes consultar inmediatamente después de crear la orden:

```sql
-- Ejecuta esto justo después de POST /api/orders
SELECT
    id,
    event_type,
    publication_date,
    completion_date,
    serialized_event
FROM event_publication
ORDER BY publication_date DESC
LIMIT 5;
```

Con `completion-mode=delete`, los eventos desaparecen cuando se procesan. Si quieres verlos siempre, cambia a:

```properties
spring.modulith.events.completion-mode=archive
```

Y consulta el historial:

```sql
SELECT event_type, publication_date, completion_date
FROM event_publication_archive
ORDER BY publication_date DESC;
```

---

## Resumen del flujo completo

```
POST /api/orders
      │
      ▼
 OrderService.create()
      │
      ├─→ orders (tabla) ──────────────────────────────┐
      └─→ event_publication (tabla) ─────────── misma transacción
                                                       │
                                                  COMMIT
                                                       │
                        ┌──────────────────────────────┤
                        │                              │
               ┌────────▼────────┐          ┌──────────▼──────────┐
               │    inventory    │          │      RabbitMQ        │
               │                 │          │                      │
               │ decreaseStock() │          │ bookstore.orders     │
               │ (stock tabla)   │          │ → order.created      │
               └─────────────────┘          └──────────────────────┘
           @ApplicationModuleListener              @Externalized
```

---

## Comparación final

| Aspecto | Sin Spring Modulith Events | Con Spring Modulith Events |
|---|---|---|
| Garantía de entrega | ❌ Evento en RAM | ✅ Persiste en BD |
| Reintentos automáticos | ❌ Manual | ✅ Al reiniciar |
| Visibilidad de pendientes | ❌ Sin visibilidad | ✅ `event_publication` |
| Externalización a MQ | ❌ Implementación manual | ✅ Una anotación |
| Código en el handler | 3 anotaciones | 1 anotación |

---

## Checklist de la Parte 4

- [ ] `spring-modulith-starter-jdbc` agregado al `pom.xml`
- [ ] `spring-modulith-events-amqp` agregado al `pom.xml`
- [ ] `application.properties` con Event Publication Registry configurado
- [ ] `OrderCreatedEvent` actualizado con `@Externalized` y `Customer` inner record
- [ ] `OrderService` con constructor de `OrderEntity` expandido, `Customer` embebido y `getByOrderNumber()`
- [ ] `OrderEventsInventoryHandler` con `@ApplicationModuleListener` y logger
- [ ] `RabbitMQConfig` con `Jackson3JsonMessageConverter` (no `Jackson2JsonMessageConverter`)
- [ ] POST a `/api/orders` retorna `orderNumber` ✅
- [ ] Logs muestran inventory handler ejecutando post-commit ✅
- [ ] SQL de verificación en `orders` y `stock` funciona ✅
- [ ] RabbitMQ admin muestra mensaje en `bookstore.order.created` ✅
- [ ] `ModularityTest` sigue pasando ✅

---

**Anterior**: [Parte 3 — CQRS en Catalog](parte_3.md) &nbsp;&nbsp;&nbsp;&nbsp; **Siguiente**: [Parte 5 — Testing en Aislamiento](parte_5.md)
