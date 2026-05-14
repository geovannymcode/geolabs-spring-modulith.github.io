# Parte 1 — Migración a Package-by-Feature y Setup de Spring Modulith

**Duración**: 30 minutos  
**Objetivo**: Reorganizar el código, configurar Flyway, agregar Spring Modulith y hacer pasar el primer `ModularityTest`

---

## ¿Qué hacemos aquí?

Tres movimientos en esta parte:

1. Reorganizar el código de package-by-layer a **package-by-feature**
2. Configurar Flyway correctamente para Spring Boot 4.x
3. Agregar Spring Modulith, ejecutar el primer `ModularityTest` y resolver cada violación que encuentre

El test va a fallar la primera vez. Varias veces, de hecho. Eso es lo que queremos — cada error es un mensaje claro de qué está mal y dónde.

---

## ¿Qué es un módulo en Spring Modulith?

Un módulo es cualquier **paquete directo** bajo el paquete raíz de tu aplicación. Nada más — no hay XML, no hay anotación especial, no hay configuración adicional.

```
com.geovannycode.bookstore/      ← paquete raíz
├── catalog/                     ← MÓDULO
├── orders/                      ← MÓDULO
├── inventory/                   ← MÓDULO
└── common/                      ← MÓDULO
```

Los sub-paquetes dentro de cada módulo (`catalog/command/`, `catalog/web/`) son **internos** al módulo — privados por defecto. Ningún otro módulo puede acceder a lo que hay ahí, aunque las clases sean `public` en Java.

!!! tip "La visibilidad de módulo va más allá de Java"
    Java controla visibilidad con `public`, `protected`, `private`. Spring Modulith agrega una capa más: **visibilidad de módulo**. Una clase puede ser `public` en Java y ser invisible para otros módulos si está en un sub-paquete.

    Esto nos permite decir: "`ProductEntity` es pública dentro de `catalog`, pero invisible para `orders`". Sin esa distinción, tendríamos que hacer todo `package-private` en Java — lo cual es imposible de mantener en un monolito grande.

---

## La estructura objetivo

```
com.geovannycode.bookstore/
│
├── BookstoreApplication.java
│
├── config/
│   ├── FlywayConfig.java          ← NUEVO — necesario en Spring Boot 4.x
│   └── RabbitMQConfig.java
│
├── common/
│   ├── package-info.java          ← NUEVO — declara el módulo como OPEN
│   └── models/
│       └── PagedResult.java
│
├── catalog/
│   ├── CatalogApi.java            ← NUEVO — API pública del módulo
│   ├── Product.java               ← NUEVO — DTO público (nunca exponer ProductEntity)
│   ├── command/                   ← sub-paquete privado
│   │   ├── ProductEntity.java
│   │   ├── ProductRepository.java
│   │   ├── ProductCommandService.java
│   │   ├── CreateProductCommand.java
│   │   ├── ProductNotFoundException.java
│   │   └── ProductAlreadyExistsException.java
│   └── web/                       ← sub-paquete privado
│       ├── ProductRestController.java
│       └── CatalogExceptionHandler.java
│
├── orders/
│   ├── package-info.java          ← NUEVO — declara dependencias permitidas
│   ├── OrderCreatedEvent.java     ← en la RAÍZ del módulo (evento público)
│   ├── domain/                    ← sub-paquete privado
│   │   ├── OrderService.java
│   │   ├── OrderEntity.java
│   │   ├── OrderRepository.java
│   │   ├── CreateOrderRequest.java
│   │   ├── CreateOrderResponse.java
│   │   ├── OrderStatus.java
│   │   ├── OrderNotFoundException.java
│   │   └── InvalidOrderException.java
│   └── web/                       ← sub-paquete privado
│       ├── OrderRestController.java
│       └── OrdersExceptionHandler.java
│
└── inventory/
    ├── package-info.java          ← NUEVO — módulo sin dependencias externas
    ├── InventoryEntity.java
    ├── InventoryRepository.java
    ├── InventoryService.java
    └── OrderEventsInventoryHandler.java
```

---

## Paso 1: Crear los directorios

```bash
cd src/main/java/com/geovannycode/bookstore

mkdir -p common/models
mkdir -p catalog/{command,web}
mkdir -p orders/{domain,web}
mkdir -p inventory
```

---

## Paso 2: Mover las clases

Mueve cada clase a su nuevo paquete y actualiza la declaración `package` en la primera línea de cada archivo. IntelliJ puede hacer este refactor automáticamente con clic derecho → Refactor → Move.

=== "common/"

    ```
    models/PagedResult.java  →  common/models/PagedResult.java
    ```
    Cambiar:
    ```java
    package com.geovannycode.bookstore.models;
    ```
    Por:
    ```java
    package com.geovannycode.bookstore.common.models;
    ```

=== "catalog/command/"

    ```
    entities/ProductEntity.java              →  catalog/command/ProductEntity.java
    repositories/ProductRepository.java      →  catalog/command/ProductRepository.java
    services/ProductService.java             →  catalog/command/ProductCommandService.java
    models/CreateProductRequest.java         →  catalog/command/CreateProductCommand.java
    exceptions/ProductNotFoundException.java →  catalog/command/ProductNotFoundException.java
    exceptions/ProductAlreadyExistsException →  catalog/command/ProductAlreadyExistsException.java
    ```

    Renombra `ProductService` → `ProductCommandService` en nombre de clase y nombre de archivo.

=== "catalog/web/"

    ```
    web/ProductRestController.java  →  catalog/web/ProductRestController.java
    ```

    El `GlobalExceptionHandler` no lo muevas — lo dividimos en el Paso 5.

=== "orders/"

    ```
    models/OrderCreatedEvent.java     →  orders/OrderCreatedEvent.java   ← raíz del módulo
    entities/OrderEntity.java         →  orders/domain/OrderEntity.java
    repositories/OrderRepository.java →  orders/domain/OrderRepository.java
    services/OrderService.java        →  orders/domain/OrderService.java
    models/CreateOrderRequest.java    →  orders/domain/CreateOrderRequest.java
    models/CreateOrderResponse.java   →  orders/domain/CreateOrderResponse.java
    models/OrderStatus.java           →  orders/domain/OrderStatus.java
    exceptions/OrderNotFoundException →  orders/domain/OrderNotFoundException.java
    exceptions/InvalidOrderException  →  orders/domain/InvalidOrderException.java
    web/OrderRestController.java      →  orders/web/OrderRestController.java
    ```

=== "inventory/"

    ```
    entities/InventoryEntity.java              →  inventory/InventoryEntity.java
    repositories/InventoryRepository.java       →  inventory/InventoryRepository.java
    services/InventoryService.java             →  inventory/InventoryService.java
    services/OrderEventsInventoryHandler.java  →  inventory/OrderEventsInventoryHandler.java
    ```

!!! warning "OrderCreatedEvent va en la raíz de orders/, no en domain/"
    Este es el error más común. `OrderCreatedEvent` debe estar en `orders/` directamente — no en `orders/domain/`. La raíz del módulo es la API pública: lo que vive ahí es visible para todos los módulos. Lo que está en sub-paquetes es privado.

    El módulo `inventory` necesita escuchar `OrderCreatedEvent`. Si el evento está en `orders/domain/`, Spring Modulith lo tratará como un tipo interno y el handler de inventory generará una violación.

    Verifica que quedó bien:
    ```bash
    find src -name "OrderCreatedEvent.java"
    # Debe mostrar: .../bookstore/orders/OrderCreatedEvent.java
    ```

---

## Paso 3: Limpiar OrderRestController

El `OrderRestController` del starter tiene un método que hay que eliminar antes de seguir:

```java
// ELIMINA este método de OrderRestController
@GetMapping
PagedResult<OrderEntity> getAll(
        @RequestParam(defaultValue = "1") int page,
        @RequestParam(defaultValue = "10") int size) {
    return orderService.getAll(page, size);
}
```

Y el método correspondiente en `OrderService`:

```java
// ELIMINA también este método de OrderService
@Transactional(readOnly = true)
public PagedResult<OrderEntity> getAll(int page, int size) {
    var pageable = PageRequest.of(
            Math.max(0, page - 1), size,
            Sort.by(Sort.Direction.DESC, "createdAt")
    );
    return PagedResult.of(orderRepository.findAll(pageable));
}
```

### ¿Por qué eliminarlo?

Hay dos razones.

La primera es de diseño. `OrderEntity` es la entidad JPA del módulo `orders` — tiene su ID de base de datos, sus timestamps de Hibernate, sus anotaciones de persistencia. Exponer eso directamente desde un endpoint REST mezcla la capa de persistencia con la capa de presentación. Si mañana cambias el nombre de un campo en la base de datos, el contrato del API cambia también. Son dos cosas que deberían poder evolucionar independientemente.

La segunda es de consistencia con la arquitectura que estamos construyendo. En la Parte 3 vamos a agregar CQRS al módulo `catalog`, que separa explícitamente el modelo de escritura (entidades JPA) del modelo de lectura (vistas optimizadas para consultas). En la Parte 5 veremos cómo testear estos módulos en aislamiento. Tener un endpoint que retorna `OrderEntity` directamente va en contra de ese diseño.

!!! info "¿Dónde queda la consulta de órdenes entonces?"
    En la Parte 5 del workshop, cuando implementemos los tests, agregaremos un `OrderView` como DTO de lectura y un endpoint `GET /api/orders/{orderNumber}` que lo use. Por ahora los endpoints que necesitas para seguir el workshop son `POST /api/orders` (crear orden) y `GET /api/orders/{orderNumber}` (obtener por número).

    El `OrderRestController` que queda después de la limpieza:

    ```java
    @RestController
    @RequestMapping("/api/orders")
    class OrderRestController {

        private final OrderService orderService;

        OrderRestController(OrderService orderService) {
            this.orderService = orderService;
        }

        @PostMapping
        @ResponseStatus(HttpStatus.CREATED)
        CreateOrderResponse create(@Valid @RequestBody CreateOrderRequest request) {
            return orderService.create(request);
        }

        @GetMapping("/{orderNumber}")
        OrderEntity getByOrderNumber(@PathVariable String orderNumber) {
            return orderService.getByOrderNumber(orderNumber);
        }
    }
    ```

---

## Paso 4: FlywayConfig — el fix de Spring Boot 4.x

### ¿Cuál es el problema?

En Spring Boot 4.x con `spring-boot-docker-compose` activo, el orden de inicialización cambió. Lo que pasa es:

```
1. Spring Boot detecta compose.yml y levanta los contenedores
2. Spring configura el datasource usando las credenciales del compose.yml
3. JPA intenta inicializar el EntityManagerFactory
4. Hibernate valida que las tablas existan → ERROR: relation "orders" does not exist
5. Flyway nunca tuvo oportunidad de correr las migraciones
```

El error parece un problema de configuración pero es un problema de orden de inicialización.

### La solución

Al registrar Flyway como `@Bean` explícito con `initMethod = "migrate"`, Spring garantiza que las migraciones corren antes de que cualquier bean que use el `DataSource` — incluido el `EntityManagerFactory` de JPA — sea inicializado.

Crea `config/FlywayConfig.java`:

```java
package com.geovannycode.bookstore.config;

import org.flywaydb.core.Flyway;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.sql.DataSource;

/**
 * Configuración manual de Flyway para Spring Boot 4.x.
 *
 * En Boot 3.x, Flyway se autoconfigura y corre antes que JPA sin hacer nada.
 * En Boot 4.x ese orden cambió cuando spring-boot-docker-compose está activo.
 * Este bean fuerza que Flyway migre antes de que el EntityManagerFactory
 * intente validar el schema.
 */
@Configuration
public class FlywayConfig {

    @Bean(initMethod = "migrate")
    public Flyway flyway(DataSource dataSource) {
        return Flyway.configure()
                .dataSource(dataSource)
                .locations("classpath:db/migration")
                .load();
    }
}
```

### Actualizar application.properties

```properties
# FlywayConfig.java se encarga de esto — desactivamos la autoconfiguración
spring.flyway.enabled=false
```

!!! note "Spring Boot 3.x"
    Si usas Spring Boot 3.x, elimina `FlywayConfig.java` y cambia la propiedad a `spring.flyway.enabled=true`. La autoconfiguración de Boot 3.x funciona bien sin intervención.

!!! tip "¿Por qué no usar spring.jpa.defer-datasource-initialization?"
    Esa propiedad existe en Boot 3.x y resuelve un problema similar pero diferente. En Boot 4.x con docker-compose activo no es confiable. El bean explícito es más directo y predecible — Spring no puede reordenar la inicialización de un bean que ya tiene `initMethod`.

---

## Paso 5: Dividir el GlobalExceptionHandler

El `GlobalExceptionHandler` del starter conoce las excepciones de todos los dominios en un solo archivo. Si en el futuro extraes `catalog` como un servicio separado, ¿qué pasa con `ProductNotFoundException` que sigue en ese handler monolítico?

La solución: cada módulo maneja sus propias excepciones. `basePackages` en `@RestControllerAdvice` garantiza que el handler solo intercepta errores que vienen de ese módulo.

**Elimina** `GlobalExceptionHandler.java`.

**Crea** `catalog/web/CatalogExceptionHandler.java`:

```java
package com.geovannycode.bookstore.catalog.web;

import com.geovannycode.bookstore.catalog.command.ProductNotFoundException;
import com.geovannycode.bookstore.catalog.command.ProductAlreadyExistsException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler;

import java.time.Instant;

@RestControllerAdvice(basePackages = "com.geovannycode.bookstore.catalog")
class CatalogExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(ProductNotFoundException.class)
    ProblemDetail handle(ProductNotFoundException ex) {
        var problem = ProblemDetail
                .forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Producto no encontrado");
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }

    @ExceptionHandler(ProductAlreadyExistsException.class)
    ProblemDetail handle(ProductAlreadyExistsException ex) {
        var problem = ProblemDetail
                .forStatusAndDetail(HttpStatus.CONFLICT, ex.getMessage());
        problem.setTitle("Producto ya existe");
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }
}
```

**Crea** `orders/web/OrdersExceptionHandler.java`:

```java
package com.geovannycode.bookstore.orders.web;

import com.geovannycode.bookstore.orders.domain.OrderNotFoundException;
import com.geovannycode.bookstore.orders.domain.InvalidOrderException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.servlet.mvc.method.annotation.ResponseEntityExceptionHandler;

import java.time.Instant;

@RestControllerAdvice(basePackages = "com.geovannycode.bookstore.orders")
class OrdersExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    ProblemDetail handle(OrderNotFoundException ex) {
        var problem = ProblemDetail
                .forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Orden no encontrada");
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }

    @ExceptionHandler(InvalidOrderException.class)
    ProblemDetail handle(InvalidOrderException ex) {
        var problem = ProblemDetail
                .forStatusAndDetail(HttpStatus.BAD_REQUEST, ex.getMessage());
        problem.setTitle("Orden inválida");
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }
}
```

---

## Paso 6: Agregar Spring Modulith al pom.xml

```xml
<properties>
    <java.version>25</java.version>
    <spring-modulith.version>1.3.1</spring-modulith.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.modulith</groupId>
            <artifactId>spring-modulith-bom</artifactId>
            <version>${spring-modulith.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- ... dependencias existentes ... -->

    <!-- Modularidad, verificación de boundaries, endpoint /actuator/modulith -->
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-starter-core</artifactId>
    </dependency>

    <!-- @ApplicationModuleTest, AssertablePublishedEvents, Scenario -->
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## Paso 7: Crear los package-info.java

Los `package-info.java` son la forma de declararle a Spring Modulith las reglas de cada módulo. Son archivos Java válidos — solo contienen la declaración del paquete con sus anotaciones. Sin ellos, Spring Modulith usa la configuración por defecto: módulo CLOSED sin restricciones declaradas.

!!! info "¿Por qué en package-info.java y no en una clase?"
    La anotación `@ApplicationModule` va sobre el **paquete**, no sobre una clase. En Java, la única forma de anotar un paquete es con `package-info.java`. Es un archivo especial que el compilador procesa como metadato del paquete — no genera una clase `.class` por sí solo.

    El archivo debe llamarse exactamente `package-info.java` y vivir en la raíz del paquete que quieres anotar.

### `common/package-info.java`

```java
// src/main/java/com/geovannycode/bookstore/common/package-info.java

/**
 * Módulo common: utilidades compartidas.
 *
 * Al ser OPEN, expone todo su contenido incluyendo sub-paquetes.
 * Sin esta declaración, PagedResult estaría en common/models/ y
 * Spring Modulith lo trataría como privado — otros módulos no podrían usarlo.
 *
 * Solo los módulos de utilidades genuinamente transversales deberían ser OPEN.
 * Un módulo de dominio nunca debería serlo.
 */
@ApplicationModule(type = ApplicationModule.Type.OPEN)
package com.geovannycode.bookstore.common;

import org.springframework.modulith.ApplicationModule;
```

### `orders/package-info.java`

```java
// src/main/java/com/geovannycode/bookstore/orders/package-info.java

/**
 * Módulo orders: gestión del ciclo de vida de las órdenes.
 *
 * allowedDependencies declara con exactitud los módulos con los que
 * orders puede hablar. Si alguien agrega una dependencia a inventory
 * desde aquí, ModularityTest falla con este mensaje:
 * "Module 'orders' depends on module 'inventory'. Allowed targets: catalog, common."
 *
 * Incluimos "common" porque OrderRestController usa PagedResult.
 */
@ApplicationModule(allowedDependencies = {"catalog", "common"})
package com.geovannycode.bookstore.orders;

import org.springframework.modulith.ApplicationModule;
```

### `inventory/package-info.java`

```java
// src/main/java/com/geovannycode/bookstore/inventory/package-info.java

/**
 * Módulo inventory: gestión de stock.
 *
 * Sin allowedDependencies: inventory no llama a nadie.
 * Solo escucha eventos. Es el módulo más independiente del sistema.
 *
 * Puede importar OrderCreatedEvent de orders porque ese evento está
 * en la raíz del módulo orders — su API pública.
 */
@ApplicationModule
package com.geovannycode.bookstore.inventory;

import org.springframework.modulith.ApplicationModule;
```

!!! warning "catalog no necesita package-info.java en esta parte"
    `catalog` es CLOSED por defecto, que es lo que queremos. No tiene restricciones especiales que declarar. En la Parte 3 cuando implementemos CQRS tampoco lo necesitará.

---

## Paso 8: Crear el DTO público Product.java

`ProductEntity` vive en `catalog/command/` — un sub-paquete privado. El módulo `orders` necesita datos del producto para crear una orden, pero no puede importar `ProductEntity` porque es un tipo interno de otro módulo.

La solución correcta es un DTO en la raíz de `catalog/` con solo los campos que el exterior necesita.

!!! warning "Nunca expongas entidades JPA como API pública de un módulo"
    Si `CatalogApi` retornara `ProductEntity`, otros módulos sabrían que `catalog` usa JPA, que tiene una tabla `products`, que esa entidad tiene un campo `id` de tipo `Long`. Ese es el acoplamiento implícito que los módulos buscan eliminar.

    Cuando mañana decides migrar `catalog` a un modelo diferente — o extraerlo como microservicio — todos los módulos que importaban `ProductEntity` tendrían que cambiar también.

Crea `catalog/Product.java`:

```java
package com.geovannycode.bookstore.catalog;

import java.math.BigDecimal;

/**
 * DTO público del módulo catalog.
 *
 * Es el único contrato que catalog expone al resto del sistema.
 * ProductEntity permanece invisible dentro de catalog/command/.
 *
 * En la Parte 3, cuando implementemos CQRS, este record también
 * incluirá averageRating y reviewCount del read model.
 */
public record Product(
        String code,
        String name,
        String description,
        String imageUrl,
        BigDecimal price,
        String category
) {}
```

---

## Paso 9: Crear CatalogApi

`CatalogApi` es la puerta de entrada al módulo `catalog`. Vive en la raíz del módulo para que sea accesible desde fuera. En esta parte delega a `ProductCommandService`. En la Parte 3 también delegará a `ProductQueryService`.

El método `toProduct()` es el mapper que convierte `ProductEntity` → `Product`. Esta conversión ocurre dentro del módulo `catalog`, donde es legítima. Ningún módulo externo sabe que `ProductEntity` existe.

Crea `catalog/CatalogApi.java`:

```java
package com.geovannycode.bookstore.catalog;

import com.geovannycode.bookstore.catalog.command.ProductCommandService;
import com.geovannycode.bookstore.catalog.command.ProductEntity;
import com.geovannycode.bookstore.common.models.PagedResult;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Optional;

/**
 * API pública del módulo catalog.
 *
 * Único punto de entrada para que otros módulos interactúen con catalog.
 * Retorna Product (DTO público), nunca ProductEntity (tipo interno).
 *
 *   orders → CatalogApi.getByCode()     ✅ correcto
 *   orders → ProductCommandService      ❌ tipo interno de catalog
 *   orders → ProductRepository          ❌ implementación interna de catalog
 *   orders → ProductEntity              ❌ tipo interno de catalog
 */
@Service
public class CatalogApi {

    private final ProductCommandService productService;

    public CatalogApi(ProductCommandService productService) {
        this.productService = productService;
    }

    public Optional<Product> getByCode(String code) {
        return productService.getByCode(code).map(this::toProduct);
    }

    public List<Product> getByCategory(String category) {
        return productService.getByCategory(category)
                .stream()
                .map(this::toProduct)
                .toList();
    }

    private Product toProduct(ProductEntity entity) {
        return new Product(
                entity.getCode(),
                entity.getName(),
                entity.getDescription(),
                entity.getImageUrl(),
                entity.getPrice(),
                entity.getCategory()
        );
    }
}
```

---

## Paso 10: Actualizar OrderService

`OrderService` del starter tenía tres problemas: inyectaba `ProductRepository` directamente, inyectaba `InventoryRepository` directamente y manipulaba el stock de inventory desde el módulo orders.

Los tres salen. Así queda después de la migración:

```java
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

        var order = new OrderEntity(
                orderNumber,
                request.customerName(), request.customerEmail(),
                request.customerPhone(), request.deliveryAddress(),
                product.code(), product.name(),
                product.price(), request.quantity()
        );

        var saved = orderRepository.save(order);
        log.info("Orden creada: {}", orderNumber);

        // El descuento de stock lo hace inventory cuando reciba este evento.
        // En la Parte 4 mejoraremos esto con @ApplicationModuleListener
        // y el Event Publication Registry para garantizar la entrega.
        eventPublisher.publishEvent(new OrderCreatedEvent(
                saved.getOrderNumber(),
                product.code(),
                product.name(),
                product.price(),
                request.quantity(),
                request.customerName(),
                request.customerEmail()
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

!!! warning "Qué eliminar exactamente del OrderService del starter"
    Busca y elimina estas líneas en el método `create()` del starter:

    ```java
    // ELIMINA el campo y el parámetro del constructor
    private final InventoryRepository inventoryRepository;

    // ELIMINA del constructor
    InventoryRepository inventoryRepository,

    // ELIMINA las 5 líneas de lógica de inventory en create()
    var stock = inventoryRepository.findByProductCode(request.productCode())...
    if (stock.getStockLevel() < request.quantity()) { ... }
    stock.decreaseStock(request.quantity());
    inventoryRepository.save(stock);
    ```

    Después limpia los imports — ningún import del paquete `inventory` debe quedar en `OrderService`.

    Verifica:
    ```bash
    grep -n "inventory" src/main/java/com/geovannycode/bookstore/orders/domain/OrderService.java
    # No debe mostrar nada
    ```

---

## Paso 11: Crear el ModularityTest

```java
// src/test/java/com/geovannycode/bookstore/ModularityTest.java
package com.geovannycode.bookstore;

import org.junit.jupiter.api.Test;
import org.springframework.modulith.core.ApplicationModules;

/**
 * Guardián de la arquitectura modular.
 *
 * No necesita Spring, no necesita base de datos, no necesita Docker.
 * Analiza el bytecode estáticamente y verifica que se cumplan
 * todas las reglas declaradas en los package-info.java.
 *
 * Si alguien agrega una dependencia incorrecta entre módulos,
 * este test falla en segundos con el mensaje exacto.
 */
class ModularityTest {

    static final ApplicationModules modules =
            ApplicationModules.of(BookstoreApplication.class);

    @Test
    void verifiesModularStructure() {
        modules.verify();
    }

    @Test
    void printsModuleStructure() {
        // Muestra módulos detectados, sus beans y dependencias.
        // Corre este método primero para entender qué vio Spring Modulith.
        modules.forEach(System.out::println);
    }
}
```

### Ejecutar

```bash
mvn test -Dtest=ModularityTest
```

---

## Paso 12: Leer y resolver las violaciones

Cuando el test falla, el output lista cada violación con la clase exacta que la causa. Vamos una por una.

!!! tip "Cómo leer los mensajes de Spring Modulith"
    Cada violación tiene la forma:
    ```
    Module 'A' depends on non-exposed type
    com.geovannycode.bookstore.B.SomeInternalClass
    within module 'B'!
    ```
    La lectura es: alguna clase en el módulo `A` importa o referencia `SomeInternalClass` que está en un sub-paquete privado del módulo `B`. Abre la clase del módulo `A`, busca cualquier referencia a `SomeInternalClass` o a su paquete y corrígela.

### Violación: ciclo `orders ↔ inventory`

```
Cycle detected: Slice inventory → Slice orders → Slice inventory
```

Existe porque `OrderService` todavía tiene `InventoryRepository` (orders → inventory) y `OrderEventsInventoryHandler` usa `OrderCreatedEvent` (inventory → orders). Con las dos dependencias juntas hay un ciclo.

El Paso 10 ya elimina `InventoryRepository` de `OrderService`. Verifica que no quedó ningún rastro:

```bash
grep -r "InventoryRepository\|InventoryEntity" \
  src/main/java/com/geovannycode/bookstore/orders/
# No debe mostrar nada
```

### Violación: `orders` usa `common` sin declararlo

```
Module 'orders' depends on module 'common' via OrderRestController → PagedResult.
Allowed targets: catalog.
```

`OrderRestController` usa `PagedResult` de `common` pero el `package-info.java` de `orders` solo declaró `catalog`. Ya está corregido en el Paso 7 con `allowedDependencies = {"catalog", "common"}`.

### Violación: `orders` accede a `ProductEntity` (tipo interno)

```
Module 'orders' depends on module 'catalog' via OrderService → ProductEntity.
Allowed targets: catalog.
```

`catalog` está en `allowedDependencies` pero `ProductEntity` vive en `catalog/command/` — un sub-paquete privado. Estar en `allowedDependencies` permite usar lo que está en la raíz del módulo, no sus internos.

El Paso 9 y 10 resuelven esto: `CatalogApi` retorna `Product` (DTO público) y `OrderService` usa `Product` en lugar de `ProductEntity`.

Verifica:

```bash
grep -n "ProductEntity" \
  src/main/java/com/geovannycode/bookstore/orders/domain/OrderService.java
# No debe mostrar nada
```

### Ejecutar el test de nuevo

Con todos los fixes aplicados:

```bash
mvn test -Dtest=ModularityTest
```

Resultado esperado:

```
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
BUILD SUCCESS
```

---

## ¿Qué significa que el test pase?

Spring Modulith analizó todo el bytecode del proyecto y verificó que:

- Ningún módulo accede a tipos internos de otro módulo
- No hay dependencias circulares
- Las dependencias declaradas en `allowedDependencies` son las únicas que existen
- Los módulos OPEN están correctamente configurados

Cada vez que alguien hace un commit, este test corre en el pipeline. Si introduce una dependencia incorrecta, el build falla con el mensaje exacto de qué rompió. Sin revisión manual de código, sin linters externos, sin reglas en un wiki.

---

## Resumen de conceptos

| Concepto | Qué aprendiste |
|---|---|
| Package-by-feature | El código se organiza por dominio de negocio, no por capa técnica |
| Módulo OPEN | `@ApplicationModule(type = OPEN)` expone todos sus sub-paquetes |
| `allowedDependencies` | Declara exactamente qué módulos puede usar el tuyo |
| API pública del módulo | Clases en la raíz — el único contrato visible desde afuera |
| DTO público | Nunca expongas entidades JPA como API pública; crea un record |
| `package-info.java` | Archivo Java para anotar un paquete con reglas modulares |
| `ModularityTest` | Verifica toda la arquitectura en cada build |
| `FlywayConfig` | Bean explícito que garantiza orden de inicialización en Boot 4.x |

---

## Checklist

- [ ] Directorios creados: `catalog/command/`, `catalog/web/`, `orders/domain/`, `orders/web/`, `inventory/`, `common/models/`
- [ ] Clases movidas con `package` declarations actualizados
- [ ] Método `getAll` eliminado de `OrderRestController` y de `OrderService`
- [ ] `FlywayConfig.java` creado en `config/`
- [ ] `spring.flyway.enabled=false` en `application.properties`
- [ ] Spring Modulith agregado al `pom.xml` (core + test)
- [ ] `common/package-info.java` con `@ApplicationModule(type = OPEN)`
- [ ] `orders/package-info.java` con `allowedDependencies = {"catalog", "common"}`
- [ ] `inventory/package-info.java` con `@ApplicationModule`
- [ ] `GlobalExceptionHandler` eliminado
- [ ] `CatalogExceptionHandler` en `catalog/web/` con `basePackages`
- [ ] `OrdersExceptionHandler` en `orders/web/` con `basePackages`
- [ ] `Product.java` record creado en la raíz de `catalog/`
- [ ] `CatalogApi.java` creado retornando `Product`, no `ProductEntity`
- [ ] `InventoryRepository` eliminado de `OrderService` incluyendo imports
- [ ] `ProductEntity` eliminado de `OrderService` incluyendo imports
- [ ] `OrderService` actualizado para usar `CatalogApi` y tipo `Product`
- [ ] `OrderCreatedEvent` en la raíz de `orders/` verificado con `find`
- [ ] `ModularityTest` pasa ✅

---

**Anterior**: [Parte 0 — El Antes](parte_0.md) &nbsp;&nbsp;&nbsp;&nbsp; **Siguiente**: [Parte 2 — Boundaries y Reglas Modulares](parte_2.md)
