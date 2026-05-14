# Parte 3 — CQRS en el Módulo Catalog

**Duración**: 30 minutos  
**Objetivo**: Implementar Command Query Responsibility Segregation dentro del módulo `catalog`

---

## ¿Qué hacemos aquí?

Implementamos CQRS **dentro del módulo `catalog`**. Todo lo que construimos en esta parte queda encapsulado — desde afuera solo existe `CatalogApi` y el DTO `Product`, igual que antes. Lo que cambia es la implementación interna.

---

## El problema que motiva CQRS

Sin CQRS, el módulo `catalog` tiene una sola tabla y un solo modelo para todo:

```sql
-- Una tabla que sirve para leer Y escribir
CREATE TABLE products (
    id          BIGINT,
    code        VARCHAR(50),
    name        VARCHAR(255),
    price       NUMERIC(10,2),
    category    VARCHAR(100),
    created_at  TIMESTAMP
);
```

Ahora llegan nuevos requerimientos:

- `GET /catalog/products/top-rated` → libros con `averageRating >= 4.5`
- `GET /catalog/products/{code}` → muestra rating promedio y número de reviews
- `GET /catalog/products/category/Arquitectura` → filtrado por categoría, ordenado por rating

Para responder desde una sola tabla necesitarías:

```sql
SELECT p.*,
       AVG(r.rating)  as average_rating,
       COUNT(r.id)    as review_count
FROM products p
LEFT JOIN reviews r ON r.product_code = p.code
WHERE p.category = 'Arquitectura'
GROUP BY p.id
ORDER BY average_rating DESC;
```

Con 10,000 productos y 500,000 reviews, esta consulta se vuelve lenta. CQRS resuelve el problema separando los modelos de lectura y escritura.

!!! tip "¿Cuándo aplicar CQRS?"
    CQRS agrega complejidad real — dos tablas, sincronización por eventos, eventual consistency. Vale la pena cuando los patrones de lectura y escritura son muy distintos y los datos calculados (como ratings promedio) se consultan mucho más de lo que se actualizan. No apliques CQRS por defecto a todos los módulos.

### La solución: modelos separados

```
Escritura (Command side)       Lectura (Query side)
─────────────────────────      ─────────────────────────
catalog.products               catalog.product_views
─────────────────────────      ─────────────────────────
id                             code         ← PK
code                           name
name                           price
description                    category
price                          average_rating  ← cacheado
category                       review_count    ← cacheado
created_at                     last_updated_at
```

`average_rating` y `review_count` se precalculan en `product_views`. Cuando alguien consulta, ya están listos — sin `AVG`, sin `JOIN`.

---

## La estructura que vamos a construir

```
catalog/
│
├── CatalogApi.java              ← actualizar (ahora delega a command Y query)
├── Product.java                 ← actualizar (agregar averageRating, reviewCount)
│
├── command/                     ← sub-paquete privado (escritura)
│   ├── CreateProductCommand.java   ← NUEVO
│   ├── UpdateProductCommand.java   ← NUEVO
│   ├── ProductEntity.java          ← ya existe
│   ├── ProductRepository.java      ← ya existe
│   ├── ProductCommandService.java  ← modificar (hacerlo public)
│   ├── ProductNotFoundException.java
│   └── ProductAlreadyExistsException.java
│
├── query/                       ← sub-paquete NUEVO (lectura)
│   ├── ProductView.java            ← NUEVO
│   ├── ProductViewRepository.java  ← NUEVO
│   └── ProductQueryService.java    ← NUEVO
│
├── internal/                    ← sub-paquete NUEVO (sincronización)
│   ├── CatalogEvents.java          ← NUEVO
│   └── CatalogEventHandler.java    ← NUEVO
│
└── web/
    ├── ProductRestController.java  ← modificar (delegar a CatalogApi)
    └── CatalogExceptionHandler.java
```

---

## Paso 1: Crear los directorios nuevos

```bash
cd src/main/java/com/geovannycode/bookstore/catalog
mkdir -p query internal
```

---

## Paso 2: Migración Flyway V3

Las migraciones V1 y V2 vienen del starter (tablas en schema `public`). La V3 crea el schema `catalog` con sus dos tablas CQRS. Primero necesita que el schema `catalog` exista.

Crea `src/main/resources/db/migration/V3__catalog_create_schema.sql`:

```sql
-- V3__catalog_create_schema.sql
-- Crea el schema aislado para el módulo catalog.
-- Cada módulo tendrá su propio schema — eso es la separación de datos
-- que complementa la separación de código que hace Spring Modulith.
CREATE SCHEMA IF NOT EXISTS catalog;
```

Crea `src/main/resources/db/migration/V4__catalog_create_tables.sql`:

```sql
-- V4__catalog_create_tables.sql
SET search_path TO catalog;

CREATE SEQUENCE product_id_seq START WITH 100 INCREMENT BY 50;

-- ── Write model (Command side) ────────────────────────────────────────
-- Modelo normalizado, optimizado para consistencia de escritura.
-- Solo ProductCommandService escribe aquí.
CREATE TABLE products (
    id          BIGINT          NOT NULL DEFAULT nextval('catalog.product_id_seq'),
    code        VARCHAR(50)     NOT NULL UNIQUE,
    name        VARCHAR(255)    NOT NULL,
    description TEXT,
    image_url   VARCHAR(500),
    price       NUMERIC(10, 2)  NOT NULL CHECK (price > 0),
    category    VARCHAR(100)    NOT NULL,
    created_at  TIMESTAMP       NOT NULL DEFAULT now(),
    updated_at  TIMESTAMP,
    PRIMARY KEY (id)
);

-- ── Read model (Query side) ───────────────────────────────────────────
-- Modelo desnormalizado, optimizado para consultas frecuentes.
-- Se sincroniza con el write model a través de eventos internos.
-- average_rating y review_count están cacheados — sin JOINs en consulta.
CREATE TABLE product_views (
    code            VARCHAR(50)      NOT NULL,
    name            VARCHAR(255)     NOT NULL,
    description     TEXT,
    image_url       VARCHAR(500),
    price           NUMERIC(10, 2)   NOT NULL,
    category        VARCHAR(100)     NOT NULL,
    average_rating  DOUBLE PRECISION NOT NULL DEFAULT 0.0,
    review_count    INT              NOT NULL DEFAULT 0,
    last_updated_at TIMESTAMP        NOT NULL DEFAULT now(),
    PRIMARY KEY (code)
);

CREATE INDEX idx_product_views_category ON product_views(category);
CREATE INDEX idx_product_views_rating   ON product_views(average_rating DESC);
```

!!! warning "Dos migraciones separadas para el schema y las tablas"
    No mezcles `CREATE SCHEMA` y `SET search_path` en el mismo archivo. Flyway ejecuta cada archivo en una transacción, y en PostgreSQL el `SET search_path` dentro de una transacción que acaba de crear el schema puede fallar. Dos archivos, sin problemas.

---

## Paso 3: Commands — los inputs de escritura

Los commands son los objetos que representan la intención de modificar el estado. No son entidades JPA ni DTOs de respuesta — solo datos de entrada validados.

### `command/CreateProductCommand.java`

```java
package com.geovannycode.bookstore.catalog.command;

import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

import java.math.BigDecimal;

/**
 * Command para crear un producto en el catálogo.
 *
 * En CQRS, los commands representan la intención de cambiar el estado.
 * Llevan las validaciones de entrada para que el servicio reciba
 * datos ya verificados.
 *
 * Es un record público porque ProductRestController lo recibe del HTTP body
 * y lo pasa a CatalogApi. Estar en el sub-paquete command no lo hace privado
 * en términos de Spring Modulith — los types internos son los que no deberían
 * cruzar la frontera del módulo, pero estos commands son parte del API de catalog.
 */
public record CreateProductCommand(
        @NotBlank(message = "El código del producto es obligatorio")
        String code,

        @NotBlank(message = "El nombre es obligatorio")
        String name,

        String description,
        String imageUrl,

        @NotNull(message = "El precio es obligatorio")
        @DecimalMin(value = "0.01", message = "El precio debe ser mayor a cero")
        BigDecimal price,

        @NotBlank(message = "La categoría es obligatoria")
        String category
) {}
```

### `command/UpdateProductCommand.java`

```java
package com.geovannycode.bookstore.catalog.command;

import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

import java.math.BigDecimal;

/**
 * Command para actualizar un producto existente.
 *
 * No incluye el code porque ese viene como PathVariable en el endpoint.
 * Solo los campos modificables — el code es inmutable una vez creado.
 */
public record UpdateProductCommand(
        @NotBlank(message = "El nombre es obligatorio")
        String name,

        String description,
        String imageUrl,

        @NotNull(message = "El precio es obligatorio")
        @DecimalMin(value = "0.01", message = "El precio debe ser mayor a cero")
        BigDecimal price,

        @NotBlank(message = "La categoría es obligatoria")
        String category
) {}
```

---

## Paso 4: El write model — ProductEntity y ProductCommandService

### `command/ProductEntity.java`

`ProductEntity` es el modelo de escritura. Esta clase existe desde la Parte 1 — aquí la actualizamos para que apunte al schema `catalog` en lugar del schema `public` del starter.

```java
package com.geovannycode.bookstore.catalog.command;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.Instant;

/**
 * Modelo de escritura (Command side) del patrón CQRS.
 *
 * Optimizado para consistencia: normalizado, sin campos calculados.
 * Solo ProductCommandService lo usa dentro del paquete catalog.command.
 *
 * La clase es package-private: nadie fuera de catalog.command puede
 * instanciarla directamente. JPA funciona bien con clases package-private.
 */
@Entity
@Table(name = "products", schema = "catalog")
class ProductEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "catalog_product_seq")
    @SequenceGenerator(
            name = "catalog_product_seq",
            sequenceName = "catalog.product_id_seq",
            allocationSize = 50
    )
    private Long id;

    @Column(nullable = false, unique = true)
    private String code;

    @Column(nullable = false)
    private String name;

    @Column(columnDefinition = "TEXT")
    private String description;

    private String imageUrl;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    @Column(nullable = false)
    private String category;

    @Column(nullable = false, updatable = false)
    private Instant createdAt;

    private Instant updatedAt;

    @PrePersist
    void onPrePersist() {
        this.createdAt = Instant.now();
    }

    @PreUpdate
    void onPreUpdate() {
        this.updatedAt = Instant.now();
    }

    protected ProductEntity() {}

    ProductEntity(String code, String name, String description,
                  String imageUrl, BigDecimal price, String category) {
        this.code = code;
        this.name = name;
        this.description = description;
        this.imageUrl = imageUrl;
        this.price = price;
        this.category = category;
    }

    // Getters package-private — solo accesibles dentro de catalog.command
    Long getId() { return id; }
    String getCode() { return code; }
    String getName() { return name; }
    String getDescription() { return description; }
    String getImageUrl() { return imageUrl; }
    BigDecimal getPrice() { return price; }
    String getCategory() { return category; }

    void setName(String name) { this.name = name; }
    void setDescription(String description) { this.description = description; }
    void setImageUrl(String imageUrl) { this.imageUrl = imageUrl; }
    void setPrice(BigDecimal price) { this.price = price; }
    void setCategory(String category) { this.category = category; }
}
```

### `command/ProductCommandService.java`

!!! warning "ProductCommandService debe ser public"
    En la Parte 1 lo dejamos package-private (`class ProductCommandService`). Eso funciona mientras solo lo usa código dentro del mismo paquete `catalog.command`. Ahora `CatalogApi` — que vive en el paquete raíz `catalog` — necesita importarlo y usarlo. Java no permite acceder a una clase package-private desde otro paquete, aunque sea del mismo módulo.

    La solución: hacer `ProductCommandService` **public**. Spring Modulith seguirá protegiéndolo correctamente — los tipos en sub-paquetes no son accesibles desde otros módulos aunque sean `public`. La protección de Spring Modulith opera a nivel de módulo, no de modificador Java.

```java
package com.geovannycode.bookstore.catalog.command;

import com.geovannycode.bookstore.catalog.Product;
import com.geovannycode.bookstore.catalog.internal.CatalogEvents;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * Servicio de escritura del módulo Catalog (Command side).
 *
 * Dos responsabilidades:
 * 1. Persistir el cambio en el write model (ProductEntity → catalog.products)
 * 2. Publicar el evento que sincronizará el read model
 *
 * Este servicio nunca toca product_views directamente.
 * Eso es trabajo de CatalogEventHandler, que reacciona al evento.
 */
@Service
public class ProductCommandService {

    private static final Logger log = LoggerFactory.getLogger(ProductCommandService.class);

    private final ProductRepository productRepository;
    private final ApplicationEventPublisher eventPublisher;

    public ProductCommandService(ProductRepository productRepository,
                                 ApplicationEventPublisher eventPublisher) {
        this.productRepository = productRepository;
        this.eventPublisher = eventPublisher;
    }

    @Transactional
    public Product create(CreateProductCommand cmd) {
        if (productRepository.existsByCode(cmd.code())) {
            throw new ProductAlreadyExistsException(cmd.code());
        }

        var entity = new ProductEntity(
                cmd.code(), cmd.name(), cmd.description(),
                cmd.imageUrl(), cmd.price(), cmd.category()
        );
        var saved = productRepository.save(entity);
        log.info("Producto creado en write model: code={}", saved.getCode());

        // Publicamos el evento dentro de la transacción.
        // CatalogEventHandler lo recibirá después del commit y actualizará
        // product_views en su propia transacción independiente.
        eventPublisher.publishEvent(new CatalogEvents.ProductCreated(
                saved.getCode(), saved.getName(), saved.getDescription(),
                saved.getImageUrl(), saved.getPrice(), saved.getCategory()
        ));

        return toProduct(saved);
    }

    @Transactional
    public Product update(String code, UpdateProductCommand cmd) {
        var entity = productRepository.findByCode(code)
                .orElseThrow(() -> new ProductNotFoundException(code));

        entity.setName(cmd.name());
        entity.setDescription(cmd.description());
        entity.setImageUrl(cmd.imageUrl());
        entity.setPrice(cmd.price());
        entity.setCategory(cmd.category());

        var saved = productRepository.save(entity);
        log.info("Producto actualizado en write model: code={}", saved.getCode());

        eventPublisher.publishEvent(new CatalogEvents.ProductUpdated(
                saved.getCode(), saved.getName(), saved.getDescription(),
                saved.getImageUrl(), saved.getPrice(), saved.getCategory()
        ));

        return toProduct(saved);
    }

    public java.util.Optional<ProductEntity> findEntityByCode(String code) {
        return productRepository.findByCode(code);
    }

    public java.util.List<ProductEntity> findEntitiesByCategory(String category) {
        return productRepository.findByCategory(category);
    }

    private Product toProduct(ProductEntity e) {
        // En el command side devolvemos rating 0.0 porque el read model
        // aún no procesó el evento. El cliente debería hacer un GET
        // para obtener el producto con rating actualizado.
        return new Product(
                e.getCode(), e.getName(), e.getDescription(),
                e.getImageUrl(), e.getPrice(), e.getCategory(),
                0.0, 0
        );
    }
}
```

---

## Paso 5: Los eventos internos — CatalogEvents

```java
package com.geovannycode.bookstore.catalog.internal;

import java.math.BigDecimal;

/**
 * Eventos internos del módulo Catalog para sincronización CQRS.
 *
 * Al vivir en catalog/internal/ (sub-paquete privado), estos eventos
 * son invisibles para otros módulos. Solo CatalogEventHandler los consume,
 * y lo hace dentro del mismo módulo catalog.
 *
 * Sealed interface: si agregamos ProductDeleted y olvidamos manejarlo
 * en CatalogEventHandler, el compilador lo detecta.
 * Es una garantía de exhaustiveness que ninguna anotación puede dar.
 */
public sealed interface CatalogEvents {

    record ProductCreated(
            String code,
            String name,
            String description,
            String imageUrl,
            BigDecimal price,
            String category
    ) implements CatalogEvents {}

    record ProductUpdated(
            String code,
            String name,
            String description,
            String imageUrl,
            BigDecimal price,
            String category
    ) implements CatalogEvents {}
}
```

---

## Paso 6: El read model — ProductView, ProductViewRepository y ProductQueryService

### `query/ProductView.java`

```java
package com.geovannycode.bookstore.catalog.query;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.Instant;

/**
 * Modelo de lectura (Query side) del patrón CQRS.
 *
 * Desnormalizado para que las consultas más frecuentes sean directas:
 * sin JOINs, sin AGGREGATEs en tiempo de consulta.
 *
 * average_rating y review_count se actualizan cuando llegan eventos
 * de CatalogEventHandler. Es eventual consistency: puede haber un instante
 * entre que se crea un producto y que aparece en las consultas de lectura.
 */
@Entity
@Table(name = "product_views", schema = "catalog")
public class ProductView {

    @Id
    private String code;

    @Column(nullable = false)
    private String name;

    @Column(columnDefinition = "TEXT")
    private String description;

    private String imageUrl;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    @Column(nullable = false)
    private String category;

    @Column(nullable = false)
    private double averageRating = 0.0;

    @Column(nullable = false)
    private int reviewCount = 0;

    private Instant lastUpdatedAt;

    protected ProductView() {}

    public ProductView(String code, String name, String description,
                       String imageUrl, BigDecimal price, String category) {
        this.code = code;
        this.name = name;
        this.description = description;
        this.imageUrl = imageUrl;
        this.price = price;
        this.category = category;
        this.lastUpdatedAt = Instant.now();
    }

    /**
     * Recalcula el rating promedio al agregar un review.
     *
     * Esta lógica vive en el read model porque es lógica de proyección:
     * cómo almacenar eficientemente datos para consulta futura.
     * No es lógica de dominio (el dominio solo sabe que hubo un review).
     */
    public void addReview(double rating) {
        double totalRating = this.averageRating * this.reviewCount + rating;
        this.reviewCount++;
        this.averageRating = totalRating / this.reviewCount;
        this.lastUpdatedAt = Instant.now();
    }

    public void updateFrom(String name, String description,
                           String imageUrl, BigDecimal price, String category) {
        this.name = name;
        this.description = description;
        this.imageUrl = imageUrl;
        this.price = price;
        this.category = category;
        this.lastUpdatedAt = Instant.now();
    }

    public String getCode() { return code; }
    public String getName() { return name; }
    public String getDescription() { return description; }
    public String getImageUrl() { return imageUrl; }
    public BigDecimal getPrice() { return price; }
    public String getCategory() { return category; }
    public double getAverageRating() { return averageRating; }
    public int getReviewCount() { return reviewCount; }
}
```

### `query/ProductViewRepository.java`

```java
package com.geovannycode.bookstore.catalog.query;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;
import java.util.Optional;

/**
 * Repositorio de lectura del módulo Catalog.
 *
 * Solo lee de catalog.product_views — nunca toca catalog.products.
 * Esa separación es la esencia del Query side en CQRS.
 */
public interface ProductViewRepository extends JpaRepository<ProductView, String> {

    Optional<ProductView> findByCode(String code);

    Page<ProductView> findAll(Pageable pageable);

    List<ProductView> findByCategory(String category);

    /**
     * Retorna productos con rating mayor o igual al mínimo,
     * ordenados de mayor a menor rating.
     * Sin CQRS, esta consulta requeriría un AVG + GROUP BY en cada llamada.
     * Con el read model, es un simple WHERE + ORDER BY.
     */
    List<ProductView> findByAverageRatingGreaterThanEqualOrderByAverageRatingDesc(
            double minRating);
}
```

### `query/ProductQueryService.java`

```java
package com.geovannycode.bookstore.catalog.query;

import com.geovannycode.bookstore.catalog.Product;
import com.geovannycode.bookstore.common.models.PagedResult;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Optional;

/**
 * Servicio de lectura del módulo Catalog (Query side).
 *
 * Solo consulta product_views. Nunca toca catalog.products.
 * Si en el futuro necesitas escalar lectura independientemente de escritura,
 * este servicio puede apuntar a una réplica de PostgreSQL sin tocar
 * una sola línea de ProductCommandService.
 */
@Service
@Transactional(readOnly = true)
public class ProductQueryService {

    private final ProductViewRepository viewRepository;

    public ProductQueryService(ProductViewRepository viewRepository) {
        this.viewRepository = viewRepository;
    }

    public Optional<Product> findByCode(String code) {
        return viewRepository.findByCode(code).map(this::toProduct);
    }

    public PagedResult<Product> findAll(int page, int size) {
        var pageable = PageRequest.of(
                Math.max(0, page - 1), size,
                Sort.by("name")
        );
        return PagedResult.of(viewRepository.findAll(pageable).map(this::toProduct));
    }

    public List<Product> findByCategory(String category) {
        return viewRepository.findByCategory(category)
                .stream()
                .map(this::toProduct)
                .toList();
    }

    public List<Product> findByMinRating(double minRating) {
        return viewRepository
                .findByAverageRatingGreaterThanEqualOrderByAverageRatingDesc(minRating)
                .stream()
                .map(this::toProduct)
                .toList();
    }

    private Product toProduct(ProductView view) {
        return new Product(
                view.getCode(), view.getName(), view.getDescription(),
                view.getImageUrl(), view.getPrice(), view.getCategory(),
                view.getAverageRating(), view.getReviewCount()
        );
    }
}
```

---

## Paso 7: La sincronización — CatalogEventHandler

Este componente mantiene sincronizado el read model con el write model. Vive en `catalog/internal/` — nadie fuera del módulo sabe que existe.

!!! warning "@ApplicationModuleListener no está disponible aún"
    `@ApplicationModuleListener` viene del artefacto `spring-modulith-starter-jdbc` que agregaremos en la **Parte 4** cuando implementemos el Outbox Pattern. En esta parte usamos las tres anotaciones equivalentes por separado:

    - `@TransactionalEventListener` — el handler corre DESPUÉS del commit de la transacción del command
    - `@Transactional(propagation = REQUIRES_NEW)` — en su propia transacción independiente
    - `@Async` — en un thread separado, sin bloquear la respuesta HTTP

    En la Parte 4 reemplazaremos las tres por `@ApplicationModuleListener` cuando agreguemos la dependencia correcta. El comportamiento es idéntico.

```java
package com.geovannycode.bookstore.catalog.internal;

import com.geovannycode.bookstore.catalog.query.ProductView;
import com.geovannycode.bookstore.catalog.query.ProductViewRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Propagation;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.transaction.event.TransactionalEventListener;

/**
 * Sincronización CQRS: mantiene product_views actualizado.
 *
 * El orden de ejecución es:
 * 1. ProductCommandService.create() guarda en catalog.products y publica el evento
 * 2. La transacción del command hace COMMIT
 * 3. @TransactionalEventListener recibe el evento (post-commit)
 * 4. @Async lo ejecuta en un thread separado
 * 5. @Transactional(REQUIRES_NEW) abre su propia transacción
 * 6. ProductView se crea en catalog.product_views y hace COMMIT
 *
 * Si el paso 6 falla, el paso 1 ya está persistido — no hay rollback.
 * En la Parte 4 agregaremos el Event Publication Registry para garantizar
 * que el reintento ocurra automáticamente en caso de fallo.
 */
@Component
class CatalogEventHandler {

    private static final Logger log = LoggerFactory.getLogger(CatalogEventHandler.class);

    private final ProductViewRepository viewRepository;

    CatalogEventHandler(ProductViewRepository viewRepository) {
        this.viewRepository = viewRepository;
    }

    @Async
    @TransactionalEventListener
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    void on(CatalogEvents.ProductCreated event) {
        log.info("CQRS sync → creando ProductView: code={}", event.code());

        var view = new ProductView(
                event.code(), event.name(), event.description(),
                event.imageUrl(), event.price(), event.category()
        );
        viewRepository.save(view);
    }

    @Async
    @TransactionalEventListener
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    void on(CatalogEvents.ProductUpdated event) {
        log.info("CQRS sync → actualizando ProductView: code={}", event.code());

        viewRepository.findByCode(event.code()).ifPresent(view -> {
            view.updateFrom(event.name(), event.description(),
                    event.imageUrl(), event.price(), event.category());
            viewRepository.save(view);
        });
    }
}
```

También necesitas habilitar `@Async` en la aplicación. Agrega `@EnableAsync` en la clase principal:

```java
// BookstoreApplication.java
@SpringBootApplication
@EnableAsync
public class BookstoreApplication {
    public static void main(String[] args) {
        SpringApplication.run(BookstoreApplication.class, args);
    }
}
```

---

## Paso 8: Actualizar Product.java — agregar averageRating y reviewCount

En la Parte 1 el record `Product` tenía solo los campos básicos. Ahora que tenemos el read model con datos calculados, el DTO público los incluye:

```java
package com.geovannycode.bookstore.catalog;

import java.math.BigDecimal;

/**
 * DTO público del módulo Catalog.
 *
 * El contrato del módulo con el resto del sistema.
 * Incluye averageRating y reviewCount del read model
 * sin que el consumidor sepa que hay dos tablas detrás.
 */
public record Product(
        String code,
        String name,
        String description,
        String imageUrl,
        BigDecimal price,
        String category,
        double averageRating,   ← nuevo
        int reviewCount          ← nuevo
) {}
```

---

## Paso 9: Actualizar CatalogApi — delegar a command Y query

`CatalogApi` en la Parte 1 solo delegaba a `ProductCommandService`. Ahora que existe el query side, las escrituras van al command y las lecturas van al query:

```java
package com.geovannycode.bookstore.catalog;

import com.geovannycode.bookstore.catalog.command.CreateProductCommand;
import com.geovannycode.bookstore.catalog.command.ProductCommandService;
import com.geovannycode.bookstore.catalog.command.UpdateProductCommand;
import com.geovannycode.bookstore.catalog.query.ProductQueryService;
import com.geovannycode.bookstore.common.models.PagedResult;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Optional;

/**
 * API pública del módulo Catalog.
 *
 * Quien usa CatalogApi no necesita saber que hay CQRS detrás:
 *   - Las escrituras van a ProductCommandService → catalog.products
 *   - Las lecturas van a ProductQueryService  → catalog.product_views
 *
 * ProductCommandService y ProductQueryService son public para que
 * CatalogApi pueda importarlos desde el paquete raíz del módulo.
 * Spring Modulith sigue protegiendo el boundary — otros módulos no
 * pueden importarlos directamente (están en sub-paquetes).
 */
@Service
public class CatalogApi {

    private final ProductCommandService commandService;
    private final ProductQueryService queryService;

    public CatalogApi(ProductCommandService commandService,
                      ProductQueryService queryService) {
        this.commandService = commandService;
        this.queryService = queryService;
    }

    // ── Comandos (van al write model) ─────────────────────────────────

    public Product create(CreateProductCommand command) {
        return commandService.create(command);
    }

    public Product update(String code, UpdateProductCommand command) {
        return commandService.update(code, command);
    }

    // ── Consultas (van al read model) ──────────────────────────────────

    public Optional<Product> getByCode(String code) {
        return queryService.findByCode(code);
    }

    public PagedResult<Product> getAll(int page, int size) {
        return queryService.findAll(page, size);
    }

    public List<Product> getByCategory(String category) {
        return queryService.findByCategory(category);
    }

    public List<Product> getTopRated(double minRating) {
        return queryService.findByMinRating(minRating);
    }
}
```

---

## Paso 10: Actualizar ProductRestController

El controller de la Parte 1 usaba `ProductCommandService` directamente. Ahora todo pasa por `CatalogApi`. También agregamos los endpoints de consulta que aprovechan el read model:

```java
package com.geovannycode.bookstore.catalog.web;

import com.geovannycode.bookstore.catalog.CatalogApi;
import com.geovannycode.bookstore.catalog.Product;
import com.geovannycode.bookstore.catalog.command.CreateProductCommand;
import com.geovannycode.bookstore.catalog.command.UpdateProductCommand;
import com.geovannycode.bookstore.common.models.PagedResult;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/catalog/products")
class ProductRestController {

    private final CatalogApi catalogApi;

    ProductRestController(CatalogApi catalogApi) {
        this.catalogApi = catalogApi;
    }

    @GetMapping
    PagedResult<Product> getAll(
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "10") int size) {
        return catalogApi.getAll(page, size);
    }

    @GetMapping("/{code}")
    ResponseEntity<Product> getByCode(@PathVariable String code) {
        return catalogApi.getByCode(code)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @GetMapping("/category/{category}")
    List<Product> getByCategory(@PathVariable String category) {
        return catalogApi.getByCategory(category);
    }

    @GetMapping("/top-rated")
    List<Product> getTopRated(
            @RequestParam(defaultValue = "4.0") double minRating) {
        return catalogApi.getTopRated(minRating);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    Product create(@Valid @RequestBody CreateProductCommand command) {
        return catalogApi.create(command);
    }

    @PutMapping("/{code}")
    Product update(@PathVariable String code,
                   @Valid @RequestBody UpdateProductCommand command) {
        return catalogApi.update(code, command);
    }
}
```

---

## Paso 11: Probar el flujo completo

```bash
mvn spring-boot:run
```

### Crear un producto

```http
POST http://localhost:8080/api/catalog/products
Content-Type: application/json

{
  "code": "P011",
  "name": "Effective Java",
  "description": "La biblia de Java moderno por Joshua Bloch.",
  "price": 48.99,
  "category": "Lenguajes"
}
```

Respuesta:
```json
{
    "code": "P011",
    "name": "Effective Java",
    "description": "La biblia de Java moderno por Joshua Bloch.",
    "imageUrl": null,
    "price": 48.99,
    "category": "Lenguajes",
    "averageRating": 0.0,
    "reviewCount": 0
}
```

### Consultar el producto

```http
GET http://localhost:8080/api/catalog/products/P011
```

Los datos vienen de `catalog.product_views`, no de `catalog.products`.

### Verificar en PostgreSQL

```bash
docker exec -it bookstore-modulith-postgres-1 \
  psql -U bookstore -d bookstore
```

```sql
-- Write model
SELECT code, name, price FROM catalog.products WHERE code = 'P011';

-- Read model
SELECT code, name, average_rating, review_count
FROM catalog.product_views WHERE code = 'P011';
```

Ambas tablas deben tener el producto. Esa es la sincronización CQRS.

### Lo que ocurrió internamente

```
1. POST /api/catalog/products → ProductRestController
2. ProductRestController → CatalogApi.create(command)
3. CatalogApi → ProductCommandService.create(command)
4. ProductCommandService guarda ProductEntity en catalog.products ✅
5. ProductCommandService publica CatalogEvents.ProductCreated (en memoria)
6. Transacción hace COMMIT
7. HTTP 201 regresa al cliente ← el cliente recibe respuesta aquí
8. (asíncrono) @TransactionalEventListener ejecuta CatalogEventHandler.on()
9. CatalogEventHandler guarda ProductView en catalog.product_views ✅
```

El cliente recibe el 201 en el paso 7, antes de que el read model se actualice. Hay un intervalo brevísimo de eventual consistency entre los pasos 7 y 9.

---

## Diagrama del flujo CQRS

```
         catalog/
┌──────────────────────────────────────────────────┐
│                                                  │
│  POST /products                                  │
│       │                                          │
│       ▼                                          │
│  CatalogApi.create()                             │
│       │                                          │
│       ▼                                          │
│  ProductCommandService → ProductEntity           │
│       │                  (catalog.products)      │
│       │                                          │
│       │ publica CatalogEvents.ProductCreated     │
│       ▼                                          │
│  [COMMIT] → HTTP 201 al cliente                  │
│                                                  │
│  (async, post-commit)                            │
│  CatalogEventHandler → ProductView               │
│                         (catalog.product_views)  │
│                                                  │
│  GET /products/{code}                            │
│       │                                          │
│       ▼                                          │
│  CatalogApi.getByCode()                          │
│       │                                          │
│       ▼                                          │
│  ProductQueryService → ProductView → Product DTO │
│                        (catalog.product_views)   │
└──────────────────────────────────────────────────┘
         (encapsulado — invisible desde afuera)
```

---

## Antes y después

| Antes (sin CQRS) | Después (con CQRS) |
|---|---|
| Una tabla para leer y escribir | `catalog.products` + `catalog.product_views` |
| `AVG` + `COUNT` en cada consulta de rating | Rating cacheado en `product_views` |
| Cambiar el modelo de escritura rompe lecturas | Modelos independientes |
| Escalar lectura obliga a escalar todo | Puedes apuntar el query side a una réplica |

Todo encapsulado en `catalog`. Para `orders`, solo existe `CatalogApi`. El CQRS es un detalle interno.

---

## Checklist de la Parte 3

- [ ] Directorios `catalog/query/` y `catalog/internal/` creados
- [ ] V3 de Flyway crea el schema `catalog`
- [ ] V4 de Flyway crea `catalog.products` y `catalog.product_views`
- [ ] `CreateProductCommand.java` creado en `catalog/command/`
- [ ] `UpdateProductCommand.java` creado en `catalog/command/`
- [ ] `ProductEntity` actualizado para apuntar a `schema = "catalog"`
- [ ] `ProductCommandService` cambiado a `public class`
- [ ] `CatalogEvents` con `sealed interface` creado en `catalog/internal/`
- [ ] `ProductView.java` creado en `catalog/query/`
- [ ] `ProductViewRepository.java` creado en `catalog/query/`
- [ ] `ProductQueryService.java` creado en `catalog/query/`
- [ ] `CatalogEventHandler` con `@TransactionalEventListener` + `@Async` + `@Transactional(REQUIRES_NEW)` en `catalog/internal/`
- [ ] `@EnableAsync` agregado en `BookstoreApplication`
- [ ] `Product.java` actualizado con `averageRating` y `reviewCount`
- [ ] `CatalogApi` actualizado para delegar a command Y query
- [ ] `ProductRestController` actualizado para delegar a `CatalogApi`
- [ ] `ModularityTest` sigue pasando ✅
- [ ] Crear un producto via HTTP y verificar ambas tablas en PostgreSQL

---

**Anterior**: [Parte 2 — Boundaries y Reglas](parte_2.md) &nbsp;&nbsp;&nbsp;&nbsp; **Siguiente**: [Parte 4 — Eventos, Outbox y Externalización](parte_4.md)
