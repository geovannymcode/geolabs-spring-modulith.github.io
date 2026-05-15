# Parte 4 — Eventos, Outbox Pattern y Externalización

**Duración**: 25 minutos  
**Objetivo**: Implementar comunicación event-driven entre módulos con garantías de entrega

---

## ¿Qué hacemos aquí?

La Parte 3 usó eventos internos al módulo `catalog` para sincronizar CQRS. Ahora implementamos eventos entre módulos — la comunicación entre `orders` e `inventory`.

También aprovechamos esta parte para mejorar el diseño de `orders`: una orden real tiene múltiples ítems, no un solo producto. Hacemos ese refactor antes de agregar los eventos, para que todo quede coherente.

---

## Evolución 1: @EventListener — el básico

```java
@Component
class OrderEventsInventoryHandler {

    @EventListener
    void on(OrderCreatedEvent event) {
        inventoryService.decreaseStock(event.productCode(), event.quantity());
    }
}
```

Funciona, pero se ejecuta dentro de la misma transacción que creó la orden:

```
BEGIN TRANSACTION
  1. OrderService guarda la orden
  2. @EventListener ejecuta el handler inmediatamente
  3. InventoryService descuenta stock
COMMIT
```

Si el paso 3 falla, el ROLLBACK también deshace la orden. Los módulos están acoplados transaccionalmente aunque estén en paquetes separados.

---

## Evolución 2: @TransactionalEventListener — separar las transacciones

```java
@Async
@Transactional(propagation = Propagation.REQUIRES_NEW)
@TransactionalEventListener
void on(OrderCreatedEvent event) {
    inventoryService.decreaseStock(event.productCode(), event.quantity());
}
```

Las transacciones ahora son independientes. Si inventory falla, la orden quedó guardada. Mejor. Pero hay un problema nuevo.

### El problema del evento perdido

```
BEGIN TRANSACTION (orders)
  1. OrderService guarda la orden
  2. El evento queda en memoria RAM
COMMIT ✅

← el servidor se reinicia aquí por cualquier razón

→ El evento nunca llega a inventory
→ La orden existe, el stock no se descontó
→ Nadie sabe. Sin logs de error. Sin alertas.
```

---

## Evolución 3: @ApplicationModuleListener + Event Publication Registry

Spring Modulith guarda el evento en la base de datos dentro de la misma transacción que lo publicó. Si la app cae después del commit, el evento sigue en la BD y se reintenta al reiniciar.

### Agregar dependencias al pom.xml

```xml
<!-- Event Publication Registry: persiste los eventos en BD (Outbox Pattern) -->
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

### Configurar application.properties

```properties
# Crea la tabla event_publication automáticamente al arrancar
spring.modulith.events.jdbc.schema-initialization.enabled=true

# "delete"  → borra el registro cuando el handler completa exitosamente
# "archive" → lo mueve a event_publication_archive (útil para auditoría)
spring.modulith.events.completion-mode=delete

# Si la app reinicia con eventos pendientes, los republica automáticamente
spring.modulith.events.republish-outstanding-events-on-restart=true
```

El flujo con estas dependencias activas:

```
BEGIN TRANSACTION (orders)
  1. OrderService guarda la orden en BD
  2. Spring Modulith persiste el evento en event_publication
     (misma transacción — si falla la orden, no hay evento)
COMMIT ✅

(el evento está en BD, garantizado)

(thread separado, transacción independiente)
BEGIN TRANSACTION (inventory)
  3. @ApplicationModuleListener ejecuta on()
  4. InventoryService descuenta stock
  5. Spring Modulith marca el evento como completado
COMMIT ✅
```

---

## Comparación de los tres mecanismos

| Mecanismo | ¿Cuándo ejecuta? | ¿Transacción propia? | ¿Resistente a caídas? |
|---|---|---|---|
| `@EventListener` | Dentro del publish | Misma que el publisher | ❌ No |
| `@TransactionalEventListener` | Después del commit | Nueva transacción | ❌ No (RAM) |
| `@ApplicationModuleListener` | Después del commit | Nueva transacción | ✅ Sí (BD) |

Para comunicación entre módulos en producción: siempre `@ApplicationModuleListener`.

---

## Paso 1: Refactorizar a órdenes multi-ítem

El diseño del starter tiene un solo producto por orden. Una orden real puede tener varios. Hacemos el refactor ahora, antes de implementar los eventos, para que todo quede alineado.

### `orders/domain/OrderItemEntity.java` — nueva entidad

Cada ítem de la orden vive en su propia tabla. El precio se guarda como snapshot — el precio que el cliente pagó en ese momento, independientemente de cambios futuros en el catálogo.

```java
package com.geovannycode.bookstore.orders.domain;

import jakarta.persistence.*;
import java.math.BigDecimal;

/**
 * Un ítem dentro de una orden: qué producto, a qué precio y cuántas unidades.
 *
 * El precio es un snapshot del momento de la compra.
 * Si el precio del catálogo cambia mañana, las órdenes antiguas
 * conservan el precio que el cliente efectivamente pagó.
 */
@Entity
@Table(name = "order_items")
public class OrderItemEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_item_seq")
    @SequenceGenerator(
            name = "order_item_seq",
            sequenceName = "order_item_id_seq",
            allocationSize = 50
    )
    private Long id;

    @Column(nullable = false)
    private String productCode;

    @Column(nullable = false)
    private String productName;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal productPrice;

    @Column(nullable = false)
    private int quantity;

    protected OrderItemEntity() {}

    public OrderItemEntity(String productCode, String productName,
                           BigDecimal productPrice, int quantity) {
        this.productCode = productCode;
        this.productName = productName;
        this.productPrice = productPrice;
        this.quantity = quantity;
    }

    public Long getId() { return id; }
    public String getProductCode() { return productCode; }
    public String getProductName() { return productName; }
    public BigDecimal getProductPrice() { return productPrice; }
    public int getQuantity() { return quantity; }
}
```

### `orders/domain/OrderEntity.java` — actualizar

Los campos `productCode`, `productName`, `productPrice`, `quantity` salen de `OrderEntity`. En su lugar entra la relación `@OneToMany` a `OrderItemEntity`.

```java
package com.geovannycode.bookstore.orders.domain;

import jakarta.persistence.*;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "orders")
public class OrderEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
    @SequenceGenerator(name = "order_seq", sequenceName = "order_id_seq", allocationSize = 50)
    private Long id;

    @Column(nullable = false, unique = true)
    private String orderNumber;

    @Column(nullable = false)
    private String customerName;

    @Column(nullable = false)
    private String customerEmail;

    @Column(nullable = false)
    private String customerPhone;

    @Column(nullable = false)
    private String deliveryAddress;

    /**
     * Los ítems de la orden.
     *
     * CascadeType.ALL: al guardar la orden, los ítems se guardan automáticamente.
     * orphanRemoval: si se elimina un ítem de la lista, se borra de BD.
     * @JoinColumn: la tabla order_items tiene la FK order_id que apunta aquí.
     */
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "order_id", nullable = false)
    private List<OrderItemEntity> items = new ArrayList<>();

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderStatus status;

    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    private Instant updatedAt;

    @PrePersist
    void onPrePersist() {
        this.createdAt = Instant.now();
        if (this.status == null) this.status = OrderStatus.NEW;
    }

    @PreUpdate
    void onPreUpdate() {
        this.updatedAt = Instant.now();
    }

    protected OrderEntity() {}

    public OrderEntity(String orderNumber, String customerName, String customerEmail,
                       String customerPhone, String deliveryAddress,
                       List<OrderItemEntity> items) {
        this.orderNumber = orderNumber;
        this.customerName = customerName;
        this.customerEmail = customerEmail;
        this.customerPhone = customerPhone;
        this.deliveryAddress = deliveryAddress;
        this.items.addAll(items);
        this.status = OrderStatus.NEW;
    }

    public Long getId() { return id; }
    public String getOrderNumber() { return orderNumber; }
    public String getCustomerName() { return customerName; }
    public String getCustomerEmail() { return customerEmail; }
    public String getCustomerPhone() { return customerPhone; }
    public String getDeliveryAddress() { return deliveryAddress; }
    public List<OrderItemEntity> getItems() { return List.copyOf(items); }
    public OrderStatus getStatus() { return status; }
    public Instant getCreatedAt() { return createdAt; }
    public Instant getUpdatedAt() { return updatedAt; }

    public void setStatus(OrderStatus status) { this.status = status; }
}
```

### `orders/domain/CreateOrderRequest.java` — actualizar

```java
package com.geovannycode.bookstore.orders.domain;

import jakarta.validation.Valid;
import jakarta.validation.constraints.*;
import java.util.List;

/**
 * Request para crear una orden con múltiples ítems.
 *
 * Los datos del cliente van en el nivel raíz.
 * Cada ítem especifica qué producto y cuántas unidades.
 */
public record CreateOrderRequest(
        @NotBlank(message = "El nombre del cliente es obligatorio")
        String customerName,

        @NotBlank(message = "El email es obligatorio")
        @Email(message = "El email no tiene un formato válido")
        String customerEmail,

        @NotBlank(message = "El teléfono es obligatorio")
        String customerPhone,

        @NotBlank(message = "La dirección de entrega es obligatoria")
        String deliveryAddress,

        @NotEmpty(message = "La orden debe tener al menos un ítem")
        @Valid
        List<Item> items
) {
    /**
     * Qué producto y cuántas unidades.
     * La validación de que el producto existe ocurre en OrderService
     * al consultar CatalogApi — no aquí.
     */
    public record Item(
            @NotBlank(message = "El código del producto es obligatorio")
            String productCode,

            @Min(value = 1, message = "La cantidad mínima es 1")
            @Max(value = 100, message = "La cantidad máxima por ítem es 100")
            int quantity
    ) {}
}
```

---

## Paso 2: Actualizar OrderCreatedEvent

El evento ahora lleva una lista de ítems y un `Customer` embebido. `@Externalized` indica a Spring Modulith que este evento debe publicarse también en RabbitMQ además de entregarse internamente a `inventory`.

```java
// orders/OrderCreatedEvent.java
package com.geovannycode.bookstore.orders;

import org.springframework.modulith.events.Externalized;
import java.math.BigDecimal;
import java.util.List;

/**
 * Evento publicado cuando una orden es creada.
 *
 * Está en la raíz del módulo orders para que inventory
 * (y cualquier otro módulo) pueda importarlo.
 *
 * @Externalized le dice a Spring Modulith:
 * "publica este evento en RabbitMQ además de entregarlo internamente".
 * El formato es "exchangeName::routingKey".
 *
 * Customer e Item van embebidos para que los consumidores externos
 * tengan toda la información sin necesitar consultas adicionales.
 */
@Externalized("bookstore.orders::order.created")
public record OrderCreatedEvent(
        String orderNumber,
        List<Item> items,
        Customer customer
) {
    public record Item(
            String productCode,
            String productName,
            BigDecimal productPrice,
            int quantity
    ) {}

    public record Customer(
            String name,
            String email,
            String phone,
            String deliveryAddress
    ) {}
}
```

---

## Paso 3: Actualizar OrderService

`OrderService` ahora itera la lista de ítems, valida cada producto contra `CatalogApi` y construye las listas de entidades y eventos en paralelo.

```java
package com.geovannycode.bookstore.orders.domain;

import com.geovannycode.bookstore.catalog.CatalogApi;
import com.geovannycode.bookstore.catalog.Product;
import com.geovannycode.bookstore.orders.OrderCreatedEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.ArrayList;
import java.util.List;
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
        // Validar cada producto en el catálogo y construir los ítems
        List<OrderItemEntity> orderItems = new ArrayList<>();
        List<OrderCreatedEvent.Item> eventItems = new ArrayList<>();

        for (CreateOrderRequest.Item item : request.items()) {
            Product product = catalogApi.getByCode(item.productCode())
                    .orElseThrow(() -> new InvalidOrderException(
                            "Producto no encontrado: " + item.productCode()));

            orderItems.add(new OrderItemEntity(
                    product.code(), product.name(),
                    product.price(), item.quantity()
            ));

            eventItems.add(new OrderCreatedEvent.Item(
                    product.code(), product.name(),
                    product.price(), item.quantity()
            ));
        }

        var orderNumber = "ORD-" + UUID.randomUUID()
                .toString().substring(0, 8).toUpperCase();

        var order = new OrderEntity(
                orderNumber,
                request.customerName(), request.customerEmail(),
                request.customerPhone(), request.deliveryAddress(),
                orderItems
        );

        var saved = orderRepository.save(order);
        log.info("Orden creada: orderNumber={}, items={}",
                orderNumber, orderItems.size());

        // Spring Modulith persiste este evento en event_publication
        // dentro de esta transacción. Si falla el commit, no hay evento.
        // Si la app cae después del commit, el evento queda en BD
        // y se reintenta al reiniciar.
        eventPublisher.publishEvent(new OrderCreatedEvent(
                saved.getOrderNumber(),
                eventItems,
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

## Paso 4: Actualizar OrderEventsInventoryHandler

El handler ahora itera la lista de ítems del evento para descontar stock de cada producto:

```java
package com.geovannycode.bookstore.inventory;

import com.geovannycode.bookstore.orders.OrderCreatedEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.modulith.events.ApplicationModuleListener;
import org.springframework.stereotype.Component;

/**
 * Descuenta stock cuando se crea una orden.
 *
 * inventory no sabe nada de cómo funciona orders internamente.
 * Solo conoce el evento público OrderCreatedEvent.
 *
 * @ApplicationModuleListener garantiza:
 * - Se ejecuta DESPUÉS del commit de la transacción de orders
 * - En su propia transacción independiente
 * - Si falla, la orden ya está guardada — no hay rollback cruzado
 * - Si la app cae antes de ejecutarse, Spring Modulith reintenta al reiniciar
 */
@Component
class OrderEventsInventoryHandler {

    private static final Logger log =
            LoggerFactory.getLogger(OrderEventsInventoryHandler.class);

    private final InventoryService inventoryService;

    OrderEventsInventoryHandler(InventoryService inventoryService) {
        this.inventoryService = inventoryService;
    }

    @ApplicationModuleListener
    void on(OrderCreatedEvent event) {
        for (OrderCreatedEvent.Item item : event.items()) {
            log.info("Actualizando stock → order={}, product={}, qty={}",
                    event.orderNumber(), item.productCode(), item.quantity());
            inventoryService.decreaseStock(item.productCode(), item.quantity());
        }
    }
}
```

!!! warning "Si @ApplicationModuleListener no resuelve en el IDE"
    Verifica que el import sea exactamente:
    ```java
    import org.springframework.modulith.events.ApplicationModuleListener;
    ```
    Si sigue sin resolver, usa las tres anotaciones equivalentes mientras se descarga la dependencia:
    ```java
    @Async
    @TransactionalEventListener
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    void on(OrderCreatedEvent event) { ... }
    ```
    El comportamiento es idéntico — `@ApplicationModuleListener` es un alias de las tres juntas.

---

## Paso 5: Configurar RabbitMQConfig

Para que el evento llegue a RabbitMQ necesitamos declarar dónde publicarlo y en qué formato.

El exchange recibe los mensajes y los distribuye. La queue es donde aterrizan. El routing key es la dirección que le dice al exchange a qué queue mandar cada mensaje.

```
OrderCreatedEvent
      │
      ▼
exchange: "bookstore.orders"        ← @Externalized("bookstore.orders::order.created")
      │                                                 ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
      │ routing key: "order.created"                                ↑↑↑↑↑↑↑↑↑↑↑↑↑
      ▼
queue: "bookstore.order.created"
```

`JacksonJsonMessageConverter` convierte el `OrderCreatedEvent` a JSON al publicarlo. Sin este bean, RabbitMQ recibiría bytes sin formato y los consumidores no podrían deserializar el mensaje.

```java
package com.geovannycode.bookstore.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.Queue;
import org.springframework.amqp.core.TopicExchange;
import org.springframework.amqp.support.converter.JacksonJsonMessageConverter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQConfig {

    @Bean
    TopicExchange ordersExchange() {
        // Coincide con la parte izquierda de @Externalized:
        // @Externalized("bookstore.orders::order.created")
        //                ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
        return new TopicExchange("bookstore.orders", true, false);
    }

    @Bean
    Queue orderCreatedQueue() {
        return new Queue("bookstore.order.created", true);
    }

    @Bean
    Binding orderCreatedBinding(Queue orderCreatedQueue, TopicExchange ordersExchange) {
        // Coincide con la parte derecha de @Externalized:
        // @Externalized("bookstore.orders::order.created")
        //                                  ↑↑↑↑↑↑↑↑↑↑↑↑↑
        return BindingBuilder
                .bind(orderCreatedQueue)
                .to(ordersExchange)
                .with("order.created");
    }

    @Bean
    JacksonJsonMessageConverter messageConverter() {
        // Spring Modulith AMQP detecta este bean automáticamente
        // y lo usa al externalizar eventos con @Externalized
        return new JacksonJsonMessageConverter();
    }
}
```

---

## Paso 6: Migración Flyway V5

Esta migración crea la tabla `order_items`, migra los datos de las órdenes existentes y elimina las columnas de producto que ya no pertenecen a `orders`.

Crea `src/main/resources/db/migration/V5__orders_multi_item.sql`:

```sql
-- V5__orders_multi_item.sql
--
-- Antes: cada orden tenía un solo producto (product_code, product_name,
--        product_price, quantity directamente en la tabla orders).
--
-- Ahora: una orden puede tener N ítems. Los datos del producto se
--        mueven a la nueva tabla order_items.

-- ── Secuencia para order_items ──────────────────────────────────────
CREATE SEQUENCE order_item_id_seq START WITH 100 INCREMENT BY 50;

-- ── Nueva tabla de ítems ─────────────────────────────────────────────
CREATE TABLE order_items (
    id            BIGINT          NOT NULL DEFAULT nextval('order_item_id_seq'),
    order_id      BIGINT          NOT NULL,
    product_code  VARCHAR(50)     NOT NULL,
    product_name  VARCHAR(255)    NOT NULL,
    product_price NUMERIC(10, 2)  NOT NULL,
    quantity      INT             NOT NULL CHECK (quantity > 0),
    PRIMARY KEY (id),
    CONSTRAINT fk_order_items_order
        FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE
);

CREATE INDEX idx_order_items_order_id ON order_items(order_id);

-- ── Migrar datos existentes de orders → order_items ──────────────────
INSERT INTO order_items (id, order_id, product_code, product_name, product_price, quantity)
SELECT nextval('order_item_id_seq'), id, product_code, product_name, product_price, quantity
FROM orders
WHERE product_code IS NOT NULL;

-- ── Eliminar columnas de producto de orders ──────────────────────────
ALTER TABLE orders DROP COLUMN product_code;
ALTER TABLE orders DROP COLUMN product_name;
ALTER TABLE orders DROP COLUMN product_price;
ALTER TABLE orders DROP COLUMN quantity;

DROP INDEX IF EXISTS idx_orders_product_code;
```

---

## Paso 7: Prerequisito — crear productos en el catálogo

!!! warning "Los productos de la BD del starter no están en el catálogo CQRS"
    El starter cargó los productos directamente en la tabla `public.products` (schema público). Ahora el módulo `catalog` tiene sus propias tablas `catalog.products` y `catalog.product_views` — y es ahí donde `CatalogApi.getByCode()` busca.

    Antes de crear una orden, tienes que crear el producto en el catálogo nuevo. Si intentas crear una orden con `P001` sin haber creado el producto primero, recibirás:
    ```json
    {"title": "Orden inválida", "detail": "Producto no encontrado: P001"}
    ```
    Eso es correcto — no es un bug.

Crea primero el producto:

```http
POST http://localhost:8080/api/catalog/products
Content-Type: application/json

{
  "code": "P001",
  "name": "Clean Code",
  "description": "Un manual para crear software ágil y de calidad.",
  "price": 45.99,
  "category": "Ingeniería de Software"
}
```

Y si quieres probar multi-ítem, crea también P003:

```http
POST http://localhost:8080/api/catalog/products
Content-Type: application/json

{
  "code": "P003",
  "name": "Designing Data-Intensive Applications",
  "description": "Principios y paradigmas para sistemas de datos a escala.",
  "price": 59.99,
  "category": "Sistemas Distribuidos"
}
```

---

## Paso 8: Demo completa

Levanta la aplicación:

```bash
mvn spring-boot:run
```

### Crear la orden (multi-ítem)

```http
POST http://localhost:8080/api/orders
Content-Type: application/json

{
  "customerName": "Geovanny Mendoza",
  "customerEmail": "geo@barranquillajug.com",
  "customerPhone": "+57 300 1234567",
  "deliveryAddress": "Calle 72 #45-10, Barranquilla",
  "items": [
    { "productCode": "P001", "quantity": 2 },
    { "productCode": "P003", "quantity": 1 }
  ]
}
```

Respuesta:
```json
{
  "orderNumber": "ORD-5A3133F2"
}
```

Anota el `orderNumber` — lo usas en las verificaciones.

### Observar los logs

```
INFO  OrderService               : Orden creada: orderNumber=ORD-5A3133F2, items=2
INFO  OrderEventsInventoryHandler: Actualizando stock → order=ORD-5A3133F2, product=P001, qty=2
INFO  InventoryService            : Stock disminuido: product=P001, quantity=2, remaining=596
INFO  OrderEventsInventoryHandler: Actualizando stock → order=ORD-5A3133F2, product=P003, qty=1
INFO  InventoryService            : Stock disminuido: product=P003, quantity=1, remaining=298
```

Los logs del handler aparecen después del de `OrderService` — confirma que `@ApplicationModuleListener` ejecuta post-commit.

---

## Verificar en PostgreSQL

Conéctate a la base de datos:

```bash
docker exec -it bookstore-modulith-postgres-1 \
  psql -U bookstore -d bookstore
```

Reemplaza `ORD-5A3133F2` con tu `orderNumber` real:

```sql
-- La orden existe
SELECT order_number, status, customer_name
FROM orders
WHERE order_number = 'ORD-5A3133F2';

-- Los ítems de la orden (la tabla orders ya no tiene product_code ni quantity)
SELECT oi.product_code, oi.product_name, oi.product_price, oi.quantity
FROM order_items oi
JOIN orders o ON o.id = oi.order_id
WHERE o.order_number = 'ORD-5A3133F2';

-- El stock se descontó para cada producto
SELECT product_code, stock_level
FROM stock
WHERE product_code IN ('P001', 'P003');
-- P001: 596 (estaba 598, restamos 2)
-- P003: 298 (estaba 299, restamos 1)

-- Event Publication Registry — debe estar vacío si completion-mode=delete
SELECT * FROM event_publication;
-- 0 rows = los eventos se entregaron y se borraron exitosamente
```

!!! tip "Si ves filas con completion_date = NULL"
    El handler no terminó de procesar todavía. Espera 2-3 segundos y vuelve a consultar. Si persiste, revisa los logs de la app para ver si `OrderEventsInventoryHandler` lanzó una excepción.

---

## Verificar en RabbitMQ

**1.** Abre [http://localhost:15672](http://localhost:15672) con `guest` / `guest`

**2.** Ve a **Queues and Streams** → haz clic en `bookstore.order.created`

**3.** Baja hasta la sección **Get messages** → haz clic para expandirla

**4.** Deja el campo *Count* en `1` → haz clic en **Get Message(s)**

Verás el mensaje con el `OrderCreatedEvent` serializado como JSON:

```json
{
  "orderNumber": "ORD-5A3133F2",
  "items": [
    {
      "productCode": "P001",
      "productName": "Clean Code",
      "productPrice": 45.99,
      "quantity": 2
    },
    {
      "productCode": "P003",
      "productName": "Designing Data-Intensive Applications",
      "productPrice": 59.99,
      "quantity": 1
    }
  ],
  "customer": {
    "name": "Geovanny Mendoza",
    "email": "geo@barranquillajug.com",
    "phone": "+57 300 1234567",
    "deliveryAddress": "Calle 72 #45-10, Barranquilla"
  }
}
```

!!! info "El mensaje vuelve a la cola después de verlo"
    Al leer desde el admin de RabbitMQ el mensaje se reencola (modo `nack` por defecto). En producción un consumer real lo consumiría y confirmaría. En el workshop no tenemos consumer externo — solo verificamos que el mensaje llegó.

---

## Diagrama del flujo completo

```
POST /api/orders
      │
      ▼
 OrderService.create()
      │
      ├─→ order_items (BD)   ─────────────────────────────┐
      ├─→ orders (BD)        ─────────────────────────────┤
      └─→ event_publication (BD) ─────────── misma transacción
                                                          │
                                                     COMMIT
                                                          │
                         ┌────────────────────────────────┤
                         │                                │
              ┌──────────▼──────────┐       ┌─────────────▼────────────┐
              │      inventory      │       │         RabbitMQ          │
              │                     │       │                           │
              │  itera event.items()│       │  bookstore.orders         │
              │  decreaseStock()    │       │  → bookstore.order.created│
              └─────────────────────┘       └───────────────────────────┘
          @ApplicationModuleListener               @Externalized
```

---

## Checklist de la Parte 4

- [ ] `spring-modulith-starter-jdbc` y `spring-modulith-events-amqp` agregados al `pom.xml`
- [ ] `application.properties` con Event Publication Registry configurado
- [ ] `OrderItemEntity.java` creado en `orders/domain/`
- [ ] `OrderEntity` actualizado — quitados campos de producto, agregado `@OneToMany items`
- [ ] `CreateOrderRequest` actualizado — `List<Item> items` con sub-record
- [ ] `OrderCreatedEvent` actualizado — `@Externalized`, `List<Item>`, `Customer` inner record
- [ ] `OrderService` actualizado — itera ítems, valida cada producto, construye listas
- [ ] `OrderEventsInventoryHandler` con `@ApplicationModuleListener`, logger e iteración de ítems
- [ ] `RabbitMQConfig` con `JacksonJsonMessageConverter`
- [ ] V5 de Flyway creada — tabla `order_items`, migración de datos, columnas eliminadas de `orders`
- [ ] Productos creados en el catálogo antes de crear la orden ✅
- [ ] POST a `/api/orders` con `items[]` retorna `orderNumber` ✅
- [ ] Logs muestran handler ejecutando post-commit por cada ítem ✅
- [ ] SQL de verificación en `order_items` y `stock` funciona ✅
- [ ] RabbitMQ admin muestra mensaje con la lista de ítems ✅
- [ ] `ModularityTest` sigue pasando ✅

---

**Anterior**: [Parte 3 — CQRS en Catalog](parte_3.md) &nbsp;&nbsp;&nbsp;&nbsp; **Siguiente**: [Parte 5 — Testing en Aislamiento](parte_5.md)
