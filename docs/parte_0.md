# Parte 0 — El Antes: La App Acoplada

**Duración**: 15 minutos  
**Objetivo**: Entender exactamente qué problema vamos a resolver y por qué importa

---

## El Punto de Partida

Antes de escribir una sola línea de Spring Modulith, necesitamos entender el problema que resuelve. No de forma abstracta — con código real.

La Bookstore que vas a ver a continuación es una aplicación Spring Boot completamente funcional. Los tests pasan, la API responde, el negocio está contento. Desde afuera todo se ve bien.

El problema está adentro.

---

## Package-by-Layer: El Enfoque Tradicional

La estructura con la que empezamos es la más común en proyectos Spring Boot:

```
com.geovannycode.bookstore/
├── BookstoreApplication.java
├── config/
│   ├── GlobalExceptionHandler.java
│   ├── FlywayConfig.java
│   └── RabbitMQConfig.java
├── entities/
│   ├── ProductEntity.java
│   ├── OrderEntity.java
│   └── InventoryEntity.java
├── exceptions/
│   ├── ProductNotFoundException.java
│   ├── OrderNotFoundException.java
│   └── InvalidOrderException.java
├── models/
│   ├── CreateOrderRequest.java
│   ├── CreateOrderResponse.java
│   ├── Customer.java
│   ├── OrderCreatedEvent.java
│   ├── OrderStatus.java
│   └── PagedResult.java
├── repositories/
│   ├── ProductRepository.java
│   ├── OrderRepository.java
│   └── InventoryRepository.java
├── services/
│   ├── OrderService.java
│   ├── ProductService.java
│   ├── InventoryService.java
│   └── OrderEventsInventoryHandler.java
└── web/
    ├── OrderRestController.java
    └── ProductRestController.java
```

Reconociste esta estructura. La has visto en tutoriales, en proyectos de trabajo, quizás en tu propio código. Organiza las clases **por capa técnica**: todos los controllers juntos, todos los servicios juntos, todos los repositorios juntos.

### ¿Por qué empieza así todo el mundo?

Tiene lógica al principio:

- Es la estructura que enseñan los tutoriales de Spring Boot
- Es fácil de navegar cuando el proyecto es pequeño
- Está alineada con las capas arquitectónicas (presentación, negocio, datos)
- Todos los desarrolladores la conocen

El problema aparece cuando el proyecto crece.

---

## Los Cuatro Problemas del Package-by-Layer

### Problema 1: La estructura no dice de qué trata la aplicación

Cuando abres el proyecto y ves `entities/`, `services/`, `repositories/` — ¿sabes de qué trata el negocio? No. Ves la tecnología, no el dominio.

Compara con una estructura que veremos más adelante:

```
// Package-by-layer: ¿qué hace esta app?
entities/ services/ repositories/ web/
→ "Usa JPA, tiene servicios y controllers" 😐

// Package-by-feature: ¿qué hace esta app?
catalog/ orders/ inventory/
→ "Gestiona un catálogo, órdenes e inventario" ✅
```

La estructura del código debería contar la historia del negocio, no de la tecnología.

### Problema 2: Todo tiene que ser public

Observa el repositorio en el código acoplado:

```java
// repositories/ProductRepository.java
public interface ProductRepository extends JpaRepository<ProductEntity, Long> {
    Optional<ProductEntity> findByCode(String code);
}
```

¿Por qué `public`? Porque `OrderService`, `InventoryService` y cualquier clase futura en cualquier paquete podría necesitar accederlo. No puedes controlarlo — Java solo te da visibilidad por paquete, y aquí todos están en paquetes diferentes.

Resultado: **todo es accesible desde todo**.

### Problema 3: El acoplamiento es invisible

Aquí está el `OrderService` del proyecto acoplado. Léelo con cuidado:

```java
// services/OrderService.java
@Service
@Transactional
public class OrderService {

    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;   // ← depende del repo de Catalog
    private final InventoryRepository inventoryRepository; // ← depende del repo de Inventory
    private final ApplicationEventPublisher eventPublisher;

    public CreateOrderResponse create(CreateOrderRequest request) {
        // Acceso directo al repositorio de Catalog — ¿debería?
        var product = productRepository.findByCode(request.productCode())
                .orElseThrow(() -> new ProductNotFoundException(request.productCode()));

        var orderNumber = "ORD-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase();
        var order = new OrderEntity(orderNumber, request, product);

        // Acceso directo al repositorio de Inventory — ¿y esto?
        var stock = inventoryRepository.findByProductCode(request.productCode())
                .orElseThrow(() -> new InvalidOrderException("Sin stock para: " + request.productCode()));

        if (stock.getStockLevel() < request.quantity()) {
            throw new InvalidOrderException("Stock insuficiente");
        }

        stock.decreaseStock(request.quantity());
        inventoryRepository.save(stock);

        var saved = orderRepository.save(order);
        eventPublisher.publishEvent(new OrderCreatedEvent(saved.getOrderNumber(), ...));

        return new CreateOrderResponse(orderNumber);
    }
}
```

`OrderService` conoce los repositorios de `Catalog` e `Inventory`. Está haciendo trabajo que debería pertenecer a esos dominios. Nadie lo notó porque todo está en el mismo paquete `services/` y es accesible.

!!! danger "El acoplamiento silencioso"
    Este es el tipo de acoplamiento más peligroso: **el que no se ve**. No hay ninguna alarma. El código compila, los tests pasan. El problema solo se manifiesta cuando:

    - Necesitas cambiar la lógica de inventario y no sabes qué más rompiste
    - Un desarrollador nuevo no entiende por qué `OrderService` maneja stock
    - Quieres extraer `inventory` como un servicio separado y descubres que está mezclado con `orders`

### Problema 4: Testing caro y frágil

¿Cómo testeas `OrderService`?

```java
@SpringBootTest
class OrderServiceTests {
    // Spring carga TODA la aplicación: todos los beans, Flyway, 
    // la conexión a Postgres, RabbitMQ...
    // Para testear 5 líneas de lógica de negocio.
}
```

O con mocks:
```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTests {
    @Mock ProductRepository productRepository;
    @Mock InventoryRepository inventoryRepository;
    @Mock OrderRepository orderRepository;
    @Mock ApplicationEventPublisher eventPublisher;
    // 4 mocks para un solo test.
    // Y si cambia la firma de algún método, el test falla aunque la lógica no cambiara.
}
```

Ninguna opción es buena a escala.

---

## La Visualización del Problema

Dibujemos las dependencias reales del proyecto acoplado:

```
┌─────────────────────────────────────────────────────────┐
│                   OrderService                          │
│                                                         │
│  ┌─────────────────┐  ┌───────────────────────────────┐ │
│  │  OrderRepository│  │  ProductRepository (Catalog!) │ │
│  └────────┬────────┘  └───────────────┬───────────────┘ │
│           │                           │                 │
│  ┌────────┴──────────────────────────┴───────────────┐  │
│  │         InventoryRepository (Inventory!)          │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

`OrderService` tiene líneas directas a los datos de `Catalog` e `Inventory`. Si mañana decides que el stock se valida diferente, o que el producto tiene un nuevo campo, tienes que tocar `OrderService`.

Esto es el **Big Ball of Mud**: un sistema donde todo está conectado con todo y nadie sabe con certeza el impacto de un cambio.

---

## ¿Cómo Debería Verse?

Antes de avanzar, hablemos de cómo debería ser el diseño correcto:

```
┌──────────────────┐    usa API pública    ┌──────────────────┐
│    orders/       │ ──────────────────────▶│    catalog/      │
│                  │                        │                  │
│  OrderService    │◀── evento ─────────────│  CatalogApi      │
└──────────────────┘    (no dep directa)    └──────────────────┘
         │
         │ publica evento
         ▼
┌──────────────────┐
│   inventory/     │
│                  │
│ InventoryHandler │ ← escucha el evento
└──────────────────┘
```

La diferencia clave:

- `orders/` usa la **API pública** de `catalog/` (no sus repositorios internos)
- `inventory/` no es llamado por `orders/` — **escucha un evento**
- Cada módulo controla sus propios datos y nadie más los toca directamente

Spring Modulith hace que estas reglas sean **verificables en tiempo de test**.

---

## Mirando el Código Acoplado

Antes de escribir una sola línea de refactoring, tómate 5 minutos para explorar el proyecto base:

```bash
# Revisa la estructura de paquetes
find src/main/java -name "*.java" | sort

# Lee el OrderService — identifica todas las dependencias
cat src/main/java/com/geovannycode/bookstore/services/OrderService.java

# Lee el GlobalExceptionHandler — ¿cuánto sabe sobre todos los módulos?
cat src/main/java/com/geovannycode/bookstore/config/GlobalExceptionHandler.java
```

### Observa el GlobalExceptionHandler

```java
// config/GlobalExceptionHandler.java — conoce TODAS las excepciones de TODOS los dominios
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(ProductNotFoundException.class)     // ← dominio Catalog
    ProblemDetail handle(ProductNotFoundException e) { ... }

    @ExceptionHandler(OrderNotFoundException.class)       // ← dominio Orders
    ProblemDetail handle(OrderNotFoundException e) { ... }

    @ExceptionHandler(InvalidOrderException.class)        // ← dominio Orders
    ProblemDetail handle(InvalidOrderException e) { ... }

    // Cuando mañana agregues InsufficientStockException de Inventory...
    // ¡también va aquí! Todo en un solo lugar que conoce todo.
}
```

Una sola clase que sabe de todos los dominios. Cuando el equipo crezca, este archivo tendrá conflictos de Git constantemente.

### Observa el OrderService completo

```java
// services/OrderService.java
@Service
@Transactional
public class OrderService {

    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;     // ← Catalog
    private final InventoryRepository inventoryRepository; // ← Inventory
    private final ApplicationEventPublisher eventPublisher;

    public CreateOrderResponse create(CreateOrderRequest request) {
        // Lógica de Catalog mezclada en Orders
        var product = productRepository.findByCode(request.productCode())
                .orElseThrow(() -> new ProductNotFoundException(request.productCode()));

        // Lógica de Inventory mezclada en Orders
        var stock = inventoryRepository.findByProductCode(request.productCode())
                .orElseThrow(() -> new InvalidOrderException("Sin stock"));

        if (stock.getStockLevel() < request.quantity()) {
            throw new InvalidOrderException("Stock insuficiente");
        }
        stock.decreaseStock(request.quantity());
        inventoryRepository.save(stock);

        // ¿Dónde termina Orders y dónde empieza Inventory?
        // No hay límite claro.

        var order = new OrderEntity(/*...*/);
        orderRepository.save(order);
        eventPublisher.publishEvent(new OrderCreatedEvent(/*...*/));

        return new CreateOrderResponse(order.getOrderNumber());
    }
}
```

¿Lo ves? `OrderService` hace el trabajo de tres módulos distintos al mismo tiempo.

---

## ¿Qué Aprenderemos a Hacer?

En las próximas partes vamos a:

1. **Reorganizar** el código a package-by-feature (Parte 1)
2. **Detectar** todas las violaciones de boundaries con Spring Modulith (Parte 1)
3. **Resolver** cada violación paso a paso (Partes 1 y 2)
4. **Agregar** CQRS al módulo `catalog` (Parte 3)
5. **Implementar** comunicación entre módulos via eventos con Outbox Pattern (Parte 4)
6. **Testear** cada módulo en aislamiento (Parte 5)
7. **Generar** documentación C4 automáticamente (Parte 6)

!!! note "Repositorio de referencia"
    El resultado final de todo el workshop está en el proyecto `bookstore-modulith` que ya tienes clonado. Úsalo como referencia cuando te quedes atascado — pero intenta construir el camino tú mismo.

---

## Checklist de la Parte 0

Antes de continuar, asegúrate de haber:

- [ ] Entendido por qué package-by-layer crea problemas con el tiempo
- [ ] Identificado las tres dependencias problemáticas en `OrderService`
- [ ] Visto que `GlobalExceptionHandler` tiene conocimiento de todos los dominios
- [ ] Comprendido la diferencia entre el modelo acoplado y el modular

---

**Siguiente**: [Parte 1 — Migración a Package-by-Feature y Setup de Spring Modulith](parte_1.md)
