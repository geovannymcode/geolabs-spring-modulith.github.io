# Guía CQRS Spring Modulith - Parte 2: Implementación Detallada

## 5. ProductRepository - La Puerta de Acceso a la Base de Datos

### ¿Qué es un Repository?

**Definición simple**: Una "caja" que guarda y busca objetos en la base de datos
**Analogía**: Como un archivero - puedes guardar documentos y buscarlos después
**En código**: Provides métodos como `save()`, `findById()`, `findAll()`

### ¿Por qué crear una interface en lugar de una clase?

- Spring Data JPA **genera automáticamente** la implementación
- Tú solo defines QUÉ quieres hacer, Spring hace el CÓMO
- Es más fácil hacer testing (puedes crear versiones falsas)

### Implementación paso a paso

```java
// src/main/java/com/example/store/products/command/ProductRepository.java
package com.example.store.products.command;

import com.example.store.products.command.Product.ProductIdentifier;
import org.springframework.data.repository.CrudRepository;

/**
 * Repository para el agregado Product.
 * 
 * ¿Por qué interface y no class?
 * - Spring Data JPA genera automáticamente la implementación
 * - Proporciona métodos como save(), findById(), deleteById() gratis
 * - Podemos agregar métodos personalizados si necesitamos
 * 
 * ¿Por qué package-private (sin 'public')?
 * - Solo ProductCommandService debería usar este repository
 * - Otros módulos deben ir a través de la API pública ProductService
 * - Esto mantiene el encapsulamiento del módulo
 * 
 * ¿Qué es CrudRepository<Product, ProductIdentifier>?
 * - Product: El tipo de objeto que guardamos
 * - ProductIdentifier: El tipo del ID
 * - CRUD: Create, Read, Update, Delete
 */
interface ProductRepository extends CrudRepository<Product, ProductIdentifier> {
    
    // Spring Data JPA genera automáticamente estos métodos:
    // - save(Product product)
    // - findById(ProductIdentifier id)
    // - findAll()
    // - deleteById(ProductIdentifier id)
    // - count()
    // - existsById(ProductIdentifier id)
    
    // Si necesitáramos métodos personalizados, los agregaríamos aquí:
    // List<Product> findByCategory(String category);
    // List<Product> findByPriceBetween(BigDecimal min, BigDecimal max);
}
```

### ¿Cómo sabe Spring qué base de datos usar?

- Lee la configuración de `application.yml`
- Usa el driver de PostgreSQL que agregamos en `pom.xml`
- Genera automáticamente el SQL necesario

## 6. ProductCommandService - El Cerebro de las Operaciones

### ¿Qué hace un Service?

**Definición simple**: Contiene la lógica de negocio
**Ejemplo**: "Para crear un producto, validar datos + guardar + notificar"
**Regla**: Los Controllers son delgados, los Services contienen la lógica

### Conceptos importantes que vamos a usar:

#### ¿Qué es @Service?
- Le dice a Spring: "Esta clase contiene lógica de negocio"
- Spring la crea automáticamente (no necesitas `new ProductCommandService()`)
- Permite que otras clases la usen con `@Autowired` o constructor

#### ¿Qué es @Transactional?
**Definición simple**: "Todo o nada" para operaciones de base de datos

**Ejemplo práctico**:
```java
@Transactional
public void transferirDinero(String origen, String destino, BigDecimal cantidad) {
    // Paso 1: Quitar dinero de cuenta origen
    cuentaRepository.restar(origen, cantidad);
    
    // Paso 2: Agregar dinero a cuenta destino  
    cuentaRepository.sumar(destino, cantidad);
    
    // Si CUALQUIER paso falla, se deshace TODO
    // No queda dinero "en el aire"
}
```

**¿Por qué es importante?**
- Garantiza que la base de datos siempre esté consistente
- Si algo falla, automáticamente deshace todos los cambios
- No tienes que manejar manualmente las fallas

#### ¿Qué es ApplicationEventPublisher?
**Definición simple**: El "mensajero" que envía eventos a otros módulos

**¿Cómo funciona?**
1. Publicas: `eventPublisher.publishEvent(new ProductCreated(...))`
2. Spring busca quién quiere escuchar ese evento
3. Spring se lo entrega automáticamente

**Analogía**: Como WhatsApp - envías un mensaje a un grupo y todos los miembros lo reciben

#### ¿Qué es @RequiredArgsConstructor?
- Lombok genera automáticamente un constructor con todos los campos `final`
- Es la forma moderna de hacer dependency injection
- Más limpio que `@Autowired` en cada campo

### Implementación paso a paso

```java
// src/main/java/com/example/store/products/command/ProductCommandService.java
package com.example.store.products.command;

import com.example.store.products.command.Product.ProductIdentifier;
import com.example.store.products.command.ProductEvents.ProductCreated;
import com.example.store.products.command.ProductEvents.ProductReviewed;
import com.example.store.products.command.ProductEvents.ProductUpdated;
import lombok.RequiredArgsConstructor;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;

/**
 * Servicio para operaciones de comando de productos.
 * 
 * ¿Por qué package-private (class sin public)?
 * - Solo el controller de este módulo debería usarlo directamente
 * - Otros módulos usan la API pública ProductService
 * - Mantiene el encapsulamiento del módulo
 */
@Service                    // Spring maneja esta clase automáticamente
@Transactional              // Todas las operaciones son "todo o nada"
@RequiredArgsConstructor    // Lombok genera constructor con campos final
class ProductCommandService {
    
    // Dependencias que Spring inyecta automáticamente
    private final ProductRepository products;
    private final ApplicationEventPublisher eventPublisher;
    
    /**
     * Crea un nuevo producto.
     * 
     * ¿Qué pasa paso a paso?
     * 1. Crear objeto Product con datos recibidos
     * 2. Guardar en base de datos  
     * 3. Publicar evento ProductCreated
     * 4. Devolver el ID del producto creado
     * 
     * ¿Qué pasa si algo falla?
     * - @Transactional deshace todo automáticamente
     * - La base de datos queda como estaba antes
     * - El evento NO se publica si falló el guardado
     */
    ProductIdentifier createProduct(String name, String description, BigDecimal price, 
                                   Integer stock, String category) {
        
        // Paso 1: Crear el objeto Product
        var product = new Product();
        product.setName(name);
        product.setDescription(description);
        product.setPrice(price);
        product.setStock(stock);
        product.setCategory(category);
        
        // Paso 2: Guardar en base de datos
        var saved = products.save(product);
        
        // Paso 3: Publicar evento (solo si el guardado fue exitoso)
        eventPublisher.publishEvent(
            new ProductCreated(saved.getId(), saved.getName(), saved.getDescription(), 
                             saved.getPrice(), saved.getStock(), saved.getCategory())
        );
        
        // Paso 4: Devolver el ID
        return saved.getId();
    }
    
    /**
     * Actualiza un producto existente.
     * 
     * ¿Por qué lanzar excepción si no existe?
     * - Es mejor fallar rápido que continuar con datos incorrectos
     * - El controller puede manejar la excepción y devolver 404
     * - El usuario sabe exactamente qué pasó
     */
    void updateProduct(ProductIdentifier productId, String name, String description, 
                      BigDecimal price, Integer stock, String category) {
        
        // Buscar el producto o fallar si no existe
        var product = products.findById(productId)
            .orElseThrow(() -> new IllegalArgumentException("Product not found with id: " + productId));
        
        // Actualizar los datos
        product.setName(name);
        product.setDescription(description);
        product.setPrice(price);
        product.setStock(stock);
        product.setCategory(category);
        
        // Guardar cambios
        products.save(product);
        
        // Notificar que el producto cambió
        eventPublisher.publishEvent(
            new ProductUpdated(product.getId(), product.getName(), product.getDescription(),
                             product.getPrice(), product.getStock(), product.getCategory())
        );
    }
    
    /**
     * Agrega un review a un producto.
     * 
     * ¿Por qué validar aquí y no en Product?
     * - El servicio es responsable de validaciones de negocio
     * - Product se enfoca en mantener consistencia interna
     * - Separación clara de responsabilidades
     * 
     * ¿Por qué usar map() y flatMap()?
     * - Es programación funcional: más expresivo y menos propenso a errores
     * - Evita if/else anidados
     * - El código lee como una secuencia de pasos
     */
    Review addReview(ProductIdentifier productId, Integer vote, String comment) {
        // Validación de negocio a nivel de servicio
        if (vote < 0 || vote > 5) {
            throw new IllegalArgumentException("Vote must be between 0 and 5");
        }
        
        // Crear el review
        var review = new Review();
        review.setVote(vote);
        review.setComment(comment);
        
        // Buscar producto, agregar review, guardar
        var product = products.findById(productId)
            .map(it -> it.add(review))      // Agregar review al producto
            .map(products::save)            // Guardar el producto actualizado
            .orElseThrow(() -> new IllegalArgumentException("Product not found with id: " + productId));
        
        // Notificar que se agregó un review
        eventPublisher.publishEvent(
            new ProductReviewed(product.getId(), review.getId(), review.getVote(), review.getComment())
        );
        
        return review;
    }
}
```

### ¿Qué pasa cuando publicas un evento?

1. **Inmediatamente**: Spring busca clases con `@ApplicationModuleListener`
2. **Asíncronamente**: Ejecuta esos métodos en background
3. **Con persistencia**: Si falla, Spring Modulith puede reintentar después
4. **Sin bloquear**: Tu método continúa normalmente

## 7. ProductCommandController - La Puerta de Entrada HTTP

### ¿Qué hace un Controller?

**Definición simple**: Recibe peticiones HTTP y devuelve respuestas HTTP
**Responsabilidad**: Ser la "cara" de tu aplicación hacia el mundo exterior
**Regla**: Delgado - solo convierte HTTP a llamadas de servicio

### Conceptos importantes:

#### ¿Qué son los DTOs (Data Transfer Objects)?
**Definición simple**: Objetos que transportan datos entre capas
**¿Por qué como records internos?**: Son simples, inmutables y no necesitan archivo separado

#### ¿Qué códigos HTTP usar?
- **201 Created**: Cuando creas algo nuevo
- **200 OK**: Cuando actualizas algo existente  
- **204 No Content**: Cuando actualizas pero no devuelves datos
- **404 Not Found**: Cuando no encuentras lo que buscan

### Implementación paso a paso

```java
// src/main/java/com/example/store/products/command/ProductCommandController.java
package com.example.store.products.command;

import com.example.store.products.command.Product.ProductIdentifier;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.net.URI;

/**
 * REST Controller para operaciones de comando de productos.
 * 
 * ¿Por qué package-private?
 * - Solo necesita ser accesible desde el framework web (Spring)
 * - Otros módulos no deberían llamar directamente a controllers
 * - La comunicación entre módulos debe ser vía eventos
 */
@RestController                 // Spring maneja las peticiones HTTP
@RequestMapping("/api/products") // Todas las URLs empiezan con /api/products
@RequiredArgsConstructor        // Inyección de dependencias por constructor
class ProductCommandController {
    
    private final ProductCommandService commandService;
    
    /**
     * POST /api/products
     * Crea un nuevo producto.
     * 
     * ¿Por qué devolver el ID?
     * - El cliente necesita saber el ID del producto creado
     * - Puede usarlo para hacer más operaciones (agregar reviews, etc.)
     * 
     * ¿Por qué ResponseEntity en lugar de solo ProductIdentifier?
     * - Permite controlar el código de estado HTTP (201 Created)
     * - Permite agregar headers como Location
     * - Más control sobre la respuesta HTTP
     */
    @PostMapping
    ResponseEntity<ProductIdentifier> createProduct(@RequestBody CreateProductRequest request) {
        // Delegar al servicio
        var id = commandService.createProduct(
            request.name(), request.description(), request.price(), 
            request.stock(), request.category()
        );
        
        // Devolver 201 Created con Location header
        return ResponseEntity
            .created(URI.create("/api/products/" + id.id()))  // Header Location
            .body(id);                                        // Cuerpo de respuesta
    }
    
    /**
     * PUT /api/products/{id}
     * Actualiza un producto existente.
     * 
     * ¿Por qué PUT y no PATCH?
     * - PUT reemplaza el objeto completo
     * - PATCH actualiza solo campos específicos
     * - Usamos PUT porque enviamos todos los campos
     * 
     * ¿Por qué ResponseEntity<Void>?
     * - No necesitamos devolver datos
     * - Solo confirmamos que se actualizó
     * - 204 No Content es el código apropiado
     */
    @PutMapping("/{id}")
    ResponseEntity<Void> updateProduct(@PathVariable ProductIdentifier id,
                                     @RequestBody UpdateProductRequest request) {
        commandService.updateProduct(
            id, request.name(), request.description(), request.price(),
            request.stock(), request.category()
        );
        
        return ResponseEntity.noContent().build(); // 204 No Content
    }
    
    /**
     * POST /api/products/{id}/reviews
     * Agrega un review a un producto.
     * 
     * ¿Por qué POST y no PUT?
     * - Estamos CREANDO un nuevo review
     * - POST es para crear recursos
     * - PUT es para actualizar recursos existentes
     */
    @PostMapping("/{id}/reviews")
    ResponseEntity<ReviewIdentifier> addReview(@PathVariable ProductIdentifier productIdentifier,
                                             @RequestBody AddReviewRequest request) {
        var review = commandService.addReview(productIdentifier, request.vote(), request.comment());
        var reviewId = review.getId();
        
        return ResponseEntity
            .created(URI.create("/api/products/" + productIdentifier.id() + "/reviews/" + reviewId.id()))
            .body(reviewId);
    }
    
    // ===============================================
    // DTOs como records internos
    // ===============================================
    
    /**
     * DTO para crear productos.
     * 
     * ¿Por qué un record?
     * - Inmutable: no se puede cambiar después de crear
     * - Automático: equals(), hashCode(), toString() gratis
     * - Conciso: menos código que una clase normal
     * 
     * ¿Por qué interno?
     * - Solo este controller lo necesita
     * - No contamina el namespace global
     * - Cambios no afectan otras clases
     */
    record CreateProductRequest(
        String name, 
        String description, 
        BigDecimal price, 
        Integer stock,
        String category
    ) {}
    
    /**
     * DTO para actualizar productos.
     * 
     * ¿Por qué separado de CreateProductRequest?
     * - Podrían tener validaciones diferentes
     * - Podrían evolucionar independientemente
     * - Más claro qué operación estás haciendo
     */
    record UpdateProductRequest(
        String name, 
        String description, 
        BigDecimal price, 
        Integer stock,
        String category
    ) {}
    
    /**
     * DTO para agregar reviews.
     * 
     * ¿Por qué no incluir el productId?
     * - Ya viene en la URL: /api/products/{id}/reviews
     * - No repetir información
     * - URL es la fuente de verdad para el ID
     */
    record AddReviewRequest(Integer vote, String comment) {}
}
```

### ¿Cómo maneja Spring las peticiones?

1. **Petición HTTP llega** → `POST /api/products`
2. **Spring busca** → Método con `@PostMapping` 
3. **Convierte JSON** → A `CreateProductRequest`
4. **Llama método** → `createProduct(request)`
5. **Convierte respuesta** → De `ResponseEntity` a HTTP

## 8. Lado Query - Modelos Optimizados para Lectura

### ¿Qué es el lado Query en CQRS?

**Definición simple**: La parte que se especializa en leer datos rápidamente
**Diferencia con Command**: Optimizada para consultas, no para mantener consistencia
**Características**: Datos desnormalizados, sin validaciones complejas, solo lectura

### ProductView - El Modelo de Lectura

#### ¿Por qué ProductView y no usar Product directamente?

**Problema con usar Product para queries**:
```java
// ❌ INEFICIENTE: Para mostrar productos con rating promedio
public List<ProductSummary> getProductsWithRating() {
    List<Product> products = productRepository.findAll();
    return products.stream().map(product -> {
        // Para cada producto, calcular rating promedio
        double average = product.getReviews().stream()
            .mapToInt(Review::getVote)
            .average()
            .orElse(0.0);
        return new ProductSummary(product.getName(), average);
    }).collect(toList());
    // Esto es LENTO si tienes muchos productos
}
```

**Solución con ProductView**:
```java
// ✅ RÁPIDO: Rating ya está calculado y guardado
public List<ProductView> getProductsByRating() {
    return repository.findAllOrderByRatingDesc(); // Una sola query SQL
}
```

### Implementación de ProductView

```java
// src/main/java/com/example/store/products/query/ProductView.java
package com.example.store.products.query;

import com.example.store.products.command.Product.ProductIdentifier;
import com.example.store.products.command.ProductEvents.ProductReviewed;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.Setter;
import org.jmolecules.architecture.cqrs.QueryModel;
import org.jmolecules.ddd.types.AggregateRoot;

import java.math.BigDecimal;

/**
 * Modelo de lectura optimizado para consultas.
 * 
 * ¿Por qué implementar AggregateRoot aquí también?
 * - jMolecules genera automáticamente las anotaciones JPA
 * - Nos da @Entity, @Table, @Id sin escribirlas manualmente
 * - Es agreggate desde perspectiva de queries, no de comandos
 * 
 * ¿Qué es @QueryModel?
 * - Marca esta clase como modelo de lectura en CQRS
 * - Documenta la intención del diseño
 * - Herramientas pueden usar esta información
 */
@Getter                    // Getters para todos los campos
@Setter                    // Setters normales para campos básicos
@QueryModel               // Marca como modelo de query CQRS
class ProductView implements AggregateRoot<ProductView, ProductIdentifier> {
    
    // Campos básicos replicados del lado comando
    private ProductIdentifier id;
    private String name, description, category;
    private BigDecimal price;
    private Integer stock;
    
    // Campos desnormalizados para consultas eficientes
    // @Setter(AccessLevel.NONE) = solo se pueden cambiar con métodos específicos
    private @Setter(AccessLevel.NONE) Double averageRating = 0.0;
    private @Setter(AccessLevel.NONE) Integer reviewCount = 0;
    
    /**
     * Procesa un evento ProductReviewed para actualizar estadísticas.
     * 
     * ¿Cómo funciona el cálculo incremental?
     * - Promedio actual = suma total / cantidad reviews
     * - Suma total = promedio actual * cantidad reviews
     * - Nueva suma = suma total + nuevo voto
     * - Nuevo promedio = nueva suma / (cantidad + 1)
     * 
     * ¿Por qué incremental en lugar de recalcular todo?
     * - Mucho más rápido (O(1) vs O(n))
     * - No necesita cargar todos los reviews
     * - Escala bien con millones de reviews
     */
    public ProductView on(ProductReviewed event) {
        // Calcular nueva suma total
        double currentTotal = averageRating * reviewCount;
        
        // Actualizar contadores
        this.reviewCount = reviewCount + 1;
        
        // Calcular nuevo promedio
        this.averageRating = (currentTotal + event.vote()) / reviewCount;
        
        return this;  // Para poder hacer: repository.save(view.on(event))
    }
}
```

### ProductEventHandler - Sincronización Asíncrona

#### ¿Qué es @ApplicationModuleListener?

**Definición simple**: Escucha eventos de otros módulos y reacciona asíncronamente

**¿Cómo funciona?**
1. CommandService publica `ProductCreated`
2. Spring Modulith encuentra este método
3. Lo ejecuta **en otra transacción**
4. Si falla, puede reintentar después

**¿Por qué asíncrono?**
- El comando no se bloquea esperando las actualizaciones de query
- Si falla el query, no afecta al comando
- Mejor performance del sistema

### Implementación paso a paso

```java
// src/main/java/com/example/store/products/query/ProductEventHandler.java
package com.example.store.products.query;

import com.example.store.products.command.ProductEvents.ProductCreated;
import com.example.store.products.command.ProductEvents.ProductReviewed;
import com.example.store.products.command.ProductEvents.ProductUpdated;
import lombok.RequiredArgsConstructor;
import org.jmolecules.event.annotation.DomainEventHandler;
import org.springframework.modulith.events.ApplicationModuleListener;
import org.springframework.stereotype.Component;

/**
 * Maneja eventos de dominio para mantener sincronizada la vista de productos.
 * 
 * ¿Por qué @Component y no @Service?
 * - @Service implica lógica de negocio
 * - Esto es más bien un "adaptador" entre eventos y vistas
 * - @Component es más genérico y apropiado
 * 
 * ¿Qué es @DomainEventHandler?
 * - Documenta que esta clase maneja eventos de dominio
 * - Mejora la legibilidad del código
 * - Herramientas pueden usar esta información
 */
@Component                // Spring maneja esta clase
@RequiredArgsConstructor  // Constructor con dependencias finales
class ProductEventHandler {
    
    private final ProductViewRepository viewRepository;
    
    /**
     * Crea una nueva vista cuando se crea un producto.
     * 
     * ¿Qué hace @ApplicationModuleListener?
     * - Escucha eventos ProductCreated de cualquier módulo
     * - Ejecuta este método asíncronamente
     * - En su propia transacción (independiente del comando)
     * - Con retry automático si falla
     * 
     * ¿Por qué @DomainEventHandler?
     * - Documenta la intención del método
     * - Mejora la comprensión del código
     * - Consistente con jMolecules CQRS
     */
    @DomainEventHandler
    @ApplicationModuleListener
    void on(ProductCreated event) {
        // Crear nueva vista con datos del evento
        var view = new ProductView();
        view.setId(event.id());
        view.setName(event.name());
        view.setDescription(event.description());
        view.setPrice(event.price());
        view.setStock(event.stock());
        view.setCategory(event.category());
        
        // Guardar en la base de datos de queries
        viewRepository.save(view);
    }
    
    /**
     * Actualiza vista cuando se actualiza un producto.
     * 
     * ¿Por qué ifPresent() en lugar de orElseThrow()?
     * - Eventos pueden llegar fuera de orden
     * - El producto podría no existir aún en la vista
     * - Es mejor ser resiliente que fallar
     */
    @DomainEventHandler
    @ApplicationModuleListener
    void on(ProductUpdated event) {
        viewRepository.findById(event.id()).ifPresent(view -> {
            // Actualizar todos los campos básicos
            view.setName(event.name());
            view.setDescription(event.description());
            view.setPrice(event.price());
            view.setStock(event.stock());
            view.setCategory(event.category());
            
            // Guardar cambios
            viewRepository.save(view);
        });
    }
    
    /**
     * Actualiza estadísticas cuando se agrega review.
     * 
     * ¿Por qué usar map() y ifPresent()?
     * - Programación funcional: más expresiva
     * - Evita if/else anidados
     * - Manejo automático de casos nulos
     */
    @ApplicationModuleListener
    void on(ProductReviewed event) {
        viewRepository.findById(event.productId())
            .map(it -> it.on(event))        // Actualizar estadísticas
            .ifPresent(viewRepository::save); // Guardar si existe
    }
}
```

### ¿Qué pasa si falla un evento?

1. **Spring Modulith guarda el evento** en la tabla `event_publication`
2. **Al reiniciar la aplicación** republica eventos pendientes
3. **Puedes configurar retry** con backoff exponencial
4. **Eventos archivados** van a `event_publication_archive` para auditoría

## 9. Repository de Query - Consultas Optimizadas

### ¿Por qué métodos de query específicos?

**Ejemplo de query genérica vs específica**:

```java
// ❌ INEFICIENTE: Query genérica
public List<ProductView> getProductsByRating() {
    return repository.findAll()  // Trae TODOS los productos
        .stream()
        .filter(p -> p.getReviewCount() > 0)  // Filtra en memoria
        .sorted((a, b) -> b.getAverageRating().compareTo(a.getAverageRating()))  // Ordena en memoria
        .collect(toList());
}

// ✅ EFICIENTE: Query específica  
@Query("SELECT pv FROM ProductView pv WHERE pv.reviewCount > 0 ORDER BY pv.averageRating DESC")
List<ProductView> findAllOrderByRatingDesc();
```

### Implementación del Repository de Query

```java
// src/main/java/com/example/store/products/query/ProductViewRepository.java
package com.example.store.products.query;

import com.example.store.products.command.Product.ProductIdentifier;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.ListCrudRepository;

import java.math.BigDecimal;
import java.util.List;

/**
 * Repository para consultar vistas de productos.
 * 
 * ¿Por qué ListCrudRepository y no CrudRepository?
 * - findAll() devuelve List en lugar de Iterable
 * - Más conveniente para la mayoría de casos de uso
 * - Evita conversiones manuales
 */
interface ProductViewRepository extends ListCrudRepository<ProductView, ProductIdentifier> {
    
    /**
     * Encuentra productos por categoría.
     * 
     * ¿Cómo sabe Spring qué SQL generar?
     * - Lee el nombre del método: findByCategory
     * - "findBy" + "Category" = WHERE category = ?
     * - Genera automáticamente: SELECT * FROM product_view WHERE category = ?
     */
    List<ProductView> findByCategory(String categoryName);
    
    /**
     * Encuentra productos dentro de rango de precios.
     * 
     * ¿Por qué @Query manual en lugar de método generado?
     * - Spring no puede generar automáticamente BETWEEN
     * - Más control sobre la query exacta
     * - Mejor performance para queries complejas
     */
    @Query("SELECT pv FROM ProductView pv WHERE pv.price BETWEEN :minPrice AND :maxPrice")
    List<ProductView> findByPriceRange(BigDecimal minPrice, BigDecimal maxPrice);
    
    /**
     * Encuentra productos ordenados por rating.
     * 
     * ¿Por qué filtrar reviewCount > 0?
     * - Productos sin reviews tienen rating 0.0
     * - No tiene sentido mostrarlos en "mejores calificados"
     * - Mejor experiencia de usuario
     * 
     * ¿Por qué ORDER BY averageRating DESC?
     * - DESC = descendente (mayor a menor)
     * - Los mejor calificados aparecen primero
     * - Es lo que esperan los usuarios
     */
    @Query("SELECT pv FROM ProductView pv WHERE pv.reviewCount > 0 ORDER BY pv.averageRating DESC")
    List<ProductView> findAllOrderByRatingDesc();
}
```

## 10. Controller de Query - APIs de Lectura

### Implementación del Controller de Query

```java
// src/main/java/com/example/store/products/query/ProductViewController.java
package com.example.store.products.query;

import com.example.store.products.command.Product.ProductIdentifier;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.util.List;

/**
 * REST Controller para operaciones de consulta de productos.
 * 
 * ¿Por qué separar de ProductCommandController?
 * - Diferentes responsabilidades (lectura vs escritura)
 * - Pueden evolucionar independientemente
 * - Más fácil aplicar caché solo a queries
 * - Mejor para equipos especializados
 */
@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
class ProductViewController {
    
    private final ProductViewRepository repository;
    
    /**
     * GET /api/products
     * Obtiene todos los productos.
     * 
     * ¿Por qué List en lugar de ResponseEntity<List>?
     * - Spring convierte automáticamente a JSON
     * - 200 OK es el código por defecto
     * - Menos código para casos simples
     */
    @GetMapping
    List<ProductView> getAllProducts() {
        return repository.findAll();
    }
    
    /**
     * GET /api/products/{id}
     * Obtiene un producto específico.
     * 
     * ¿Por qué ResponseEntity.of(Optional)?
     * - Si existe: 200 OK con el producto
     * - Si no existe: 404 Not Found automáticamente
     * - Una línea en lugar de if/else
     */
    @GetMapping("/{id}")
    ResponseEntity<ProductView> getProductById(@PathVariable ProductIdentifier id) {
        return ResponseEntity.of(repository.findById(id));
    }
    
    /**
     * GET /api/products/by-category?category=Electronics
     * Obtiene productos por categoría.
     * 
     * ¿Por qué @RequestParam en lugar de @PathVariable?
     * - Más flexible: /api/products/by-category?category=Electronics
     * - Permite múltiples parámetros: ?category=Electronics&minPrice=100
     * - Mejor para filtros opcionales
     */
    @GetMapping("/by-category")
    List<ProductView> getProductsByCategory(@RequestParam String category) {
        return repository.findByCategory(category);
    }
    
    /**
     * GET /api/products/by-price?min=100&max=500
     * Obtiene productos en rango de precios.
     */
    @GetMapping("/by-price")
    List<ProductView> getProductsByPriceRange(@RequestParam BigDecimal min, 
                                            @RequestParam BigDecimal max) {
        return repository.findByPriceRange(min, max);
    }
    
    /**
     * GET /api/products/by-rating
     * Obtiene productos ordenados por rating.
     * 
     * ¿Por qué una URL específica?
     * - Es una consulta común y importante
     * - Merece su propia URL para caché
     * - Fácil de recordar y usar
     */
    @GetMapping("/by-rating")
    List<ProductView> getProductsByRating() {
        return repository.findAllOrderByRatingDesc();
    }
}
```

## 11. Testing Independiente de Módulos

### ¿Por qué Testing Independiente?

**Problema con @SpringBootTest tradicional**:
```java
@SpringBootTest  // Carga TODA la aplicación
class ProductTest {
    // Este test es LENTO porque:
    // - Carga todos los módulos (products, orders, inventory, etc.)
    // - Inicializa todas las dependencias
    // - Si falla orders, este test también puede fallar
}
```

**Solución con @ApplicationModuleTest**:
```java
@ApplicationModuleTest  // Carga SOLO el módulo products
class ProductTest {
    // Este test es RÁPIDO porque:
    // - Solo carga el módulo bajo test
    // - Dependencias mínimas
    // - Aislado de problemas en otros módulos
}
```

### Conceptos de Testing que Vamos a Usar

#### ¿Qué es @ApplicationModuleTest?
- Carga **solo el módulo especificado**
- Usa **TestContainers** para base de datos real
- Permite verificar **eventos publicados**
- **Tests más rápidos** que @SpringBootTest completo

#### ¿Qué es TestContainers?
**Definición simple**: Levanta una base de datos PostgreSQL real para cada test
**¿Por qué no H2 en memoria?**: PostgreSQL en test = PostgreSQL en producción (más confiable)

#### ¿Qué es PublishedEvents?
- Spring Modulith **captura automáticamente** los eventos publicados durante el test
- Puedes **verificar** qué eventos se publicaron
- Puedes **verificar** los datos del evento

#### ¿Qué es Scenario?
- Para tests **asíncronos** con eventos
- Publica un evento y **espera** a que se procese
- **Verifica** el estado resultante

### Test del Command Service

```java
// src/test/java/com/example/store/products/command/ProductCommandServiceTest.java
package com.example.store.products.command;

import static org.assertj.core.api.Assertions.*;

import lombok.RequiredArgsConstructor;
import org.junit.jupiter.api.Test;
import org.springframework.modulith.test.ApplicationModuleTest;
import org.springframework.modulith.test.PublishedEvents;

import java.math.BigDecimal;
import java.util.UUID;

/**
 * Test de integración para el módulo de comando de products.
 * 
 * ¿Qué hace @ApplicationModuleTest?
 * 1. Carga SOLO el módulo 'products'
 * 2. Levanta PostgreSQL real con TestContainers
 * 3. Permite verificar eventos publicados
 * 4. Tests más rápidos que @SpringBootTest
 * 
 * ¿Por qué @RequiredArgsConstructor en un test?
 * - Spring inyecta automáticamente las dependencias
 * - Más limpio que @Autowired en cada campo
 * - Consistente con el resto del código
 */
@ApplicationModuleTest
@RequiredArgsConstructor
public class ProductCommandServiceTest {
    
    // Spring inyecta estas dependencias automáticamente
    private final ProductCommandService productCommandService;
    private final ProductRepository productRepository;
    
    /**
     * Test: Crear producto debería guardarlo en base de datos.
     * 
     * ¿Qué verifica este test?
     * 1. El servicio acepta los parámetros correctamente
     * 2. Devuelve un ID válido
     * 3. Guarda el producto en la base de datos
     * 4. Los datos guardados son correctos
     */
    @Test
    void testAddProduct() {
        // Arrange: Preparar datos de entrada
        String name = "Test Product";
        String description = "Test Product Description";
        BigDecimal price = BigDecimal.ONE;
        Integer stock = 100;
        String category = "Test Category";
        
        // Act: Ejecutar la operación
        var productId = productCommandService.createProduct(name, description, price, stock, category);
        
        // Assert: Verificar resultados
        assertThat(productId.id()).isNotNull();
        
        // Verificar que se guardó en la base de datos
        assertThat(productRepository.findById(productId)).hasValueSatisfying(product -> {
            assertThat(product.getName()).isEqualTo(name);
            assertThat(product.getDescription()).isEqualTo(description);
            assertThat(product.getPrice()).isEqualTo(price);
            assertThat(product.getStock()).isEqualTo(stock);
            assertThat(product.getCategory()).isEqualTo(category);
            assertThat(product.getProductReviews()).isEmpty();
        });
    }
    
    /**
     * Test: Actualizar producto debería modificar datos existentes.
     * 
     * ¿Por qué usar UUID hardcodeado?
     * - Los datos de ejemplo en V1__create_initial_schema.sql tienen IDs específicos
     * - Permite testing con datos predecibles
     * - En producción, los IDs se generan automáticamente
     */
    @Test
    void testUpdateProduct() {
        // Arrange: Usar ID de datos de ejemplo
        var id = new Product.ProductIdentifier(UUID.fromString("f47ac10b-58cc-4372-a567-0e02b2c3d479"));
        
        // Act: Actualizar el producto
        productCommandService.updateProduct(
            id, "Laptop Pro 2", "High performance laptop for true PROs",
            new BigDecimal("1299.99"), 25, "Electronics"
        );
        
        // Assert: Verificar que se actualizó
        assertThat(productRepository.findById(id)).hasValueSatisfying(product -> {
            assertThat(product.getName()).isEqualTo("Laptop Pro 2");
            assertThat(product.getDescription()).isEqualTo("High performance laptop for true PROs");
            assertThat(product.getPrice()).isEqualTo(new BigDecimal("1299.99"));
            assertThat(product.getStock()).isEqualTo(25);
        });
    }
    
    /**
     * Test: Agregar review debería agregarlo a la lista del producto.
     * 
     * ¿Cómo verifica que el review se agregó correctamente?
     * 1. Busca el producto en la base de datos
     * 2. Filtra la lista de reviews para encontrar el específico
     * 3. Verifica los datos del review encontrado
     */
    @Test
    void testAddReview() {
        // Arrange: Usar producto existente
        var id = new Product.ProductIdentifier(UUID.fromString("f47ac10b-58cc-4372-a567-0e02b2c3d479"));
        
        // Act: Agregar review
        var review = productCommandService.addReview(id, 5, "Great product!");
        
        // Assert: Verificar que se agregó al producto
        assertThat(productRepository.findById(id)).hasValueSatisfying(product -> {
            var assignedReview = product.getProductReviews().stream()
                .filter(review::equals)  // Buscar este review específico
                .findFirst();
            
            assertThat(assignedReview).hasValueSatisfying(foundReview -> {
                assertThat(foundReview.getVote()).isEqualTo(5);
                assertThat(foundReview.getComment()).isEqualTo("Great product!");
            });
        });
    }
    
    /**
     * Test: Verificar que se publican eventos correctamente.
     * 
     * ¿Qué es PublishedEvents?
     * - Spring Modulith captura automáticamente eventos durante el test
     * - Permite verificar qué eventos se publicaron
     * - Puedes examinar los datos del evento
     */
    @Test
    void testEventsArePublished(PublishedEvents events) {
        // Arrange
        String name = "Event Test Product";
        String description = "Test Description";
        BigDecimal price = new BigDecimal("99.99");
        Integer stock = 50;
        String category = "Test";
        
        // Act: Crear producto
        var productId = productCommandService.createProduct(name, description, price, stock, category);
        
        // Assert: Verificar que se publicó el evento ProductCreated
        var productCreatedEvents = events.ofType(ProductEvents.ProductCreated.class);
        
        assertThat(productCreatedEvents).hasSize(1);
        
        productCreatedEvents.forEach(event -> {
            assertThat(event.id()).isEqualTo(productId);
            assertThat(event.name()).isEqualTo(name);
            assertThat(event.description()).isEqualTo(description);
            assertThat(event.price()).isEqualTo(price);
            assertThat(event.stock()).isEqualTo(stock);
            assertThat(event.category()).isEqualTo(category);
        });
    }
}
```

### Test del Event Handler

```java
// src/test/java/com/example/store/products/query/EventHandlerIntegrationTest.java
package com.example.store.products.query;

import static org.assertj.core.api.Assertions.*;

import com.example.store.products.command.Product.ProductIdentifier;
import com.example.store.products.command.ProductEvents.ProductCreated;
import com.example.store.products.command.ProductEvents.ProductReviewed;
import com.example.store.products.command.ReviewIdentifier;
import lombok.RequiredArgsConstructor;
import org.junit.jupiter.api.Test;
import org.springframework.modulith.test.ApplicationModuleTest;
import org.springframework.modulith.test.Scenario;

import java.math.BigDecimal;
import java.util.UUID;

/**
 * Test específico para verificar el manejo de eventos.
 * 
 * ¿Por qué un test separado para eventos?
 * - Los eventos son asíncronos
 * - Necesitan Scenario para coordinar timing
 * - Verifican integración entre command y query sides
 */
@ApplicationModuleTest
@RequiredArgsConstructor
class EventHandlerIntegrationTest {
    
    private final ProductViewRepository repository;
    
    /**
     * Test: Procesar eventos en secuencia.
     * 
     * ¿Qué es Scenario?
     * - Herramienta para tests asíncronos
     * - Publica evento y espera a que se procese
     * - Verifica estado resultante
     * 
     * ¿Por qué andWaitForStateChange()?
     * - Los eventos se procesan asíncronamente
     * - Necesitamos esperar a que termine el procesamiento
     * - Evita race conditions en tests
     */
    @Test
    void whenReviewEventIsReceived_thenUpdateProductView(Scenario scenario) {
        var id = new ProductIdentifier(UUID.randomUUID());
        
        // Paso 1: Crear vista de producto
        scenario.publish(new ProductCreated(id, "test", "test", BigDecimal.TEN, 100, "test"))
            .andWaitForStateChange(() -> repository.findById(id));
        
        // Paso 2: Agregar review y verificar actualización
        scenario.publish(new ProductReviewed(id, new ReviewIdentifier(UUID.randomUUID()), 5, "test"))
            .andWaitForStateChange(() -> repository.findById(id))
            .andVerify(view -> assertThat(view).hasValueSatisfying(productView -> {
                assertThat(productView.getReviewCount()).isEqualTo(1);
                assertThat(productView.getAverageRating()).isEqualTo(5.0);
            }));
    }
    
    /**
     * Test: Verificar cálculo de promedio con múltiples reviews.
     */
    @Test
    void testAverageRatingCalculation(Scenario scenario) {
        var id = new ProductIdentifier(UUID.randomUUID());
        
        // Crear producto
        scenario.publish(new ProductCreated(id, "test", "test", BigDecimal.TEN, 100, "test"))
            .andWaitForStateChange(() -> repository.findById(id));
        
        // Agregar primer review (5 estrellas)
        scenario.publish(new ProductReviewed(id, new ReviewIdentifier(UUID.randomUUID()), 5, "excellent"))
            .andWaitForStateChange(() -> repository.findById(id));
        
        // Agregar segundo review (3 estrellas)
        scenario.publish(new ProductReviewed(id, new ReviewIdentifier(UUID.randomUUID()), 3, "ok"))
            .andWaitForStateChange(() -> repository.findById(id))
            .andVerify(view -> assertThat(view).hasValueSatisfying(productView -> {
                assertThat(productView.getReviewCount()).isEqualTo(2);
                assertThat(productView.getAverageRating()).isEqualTo(4.0); // (5 + 3) / 2 = 4.0
            }));
    }
}
```

### Test de Estructura del Módulo

```java
// src/test/java/com/example/store/products/ProductsModuleIntegrationTests.java
package com.example.store.products;

import org.junit.jupiter.api.Test;
import org.springframework.modulith.test.ApplicationModuleTest;

/**
 * Test simple para verificar que el módulo arranca correctamente.
 * 
 * ¿Para qué sirve un test tan simple?
 * - Verifica que todas las dependencias se resuelven
 * - Confirma que la configuración es correcta
 * - Detecta problemas de wiring temprano
 * - Es rápido y da confianza básica
 */
@ApplicationModuleTest
class ProductsModuleIntegrationTests {
    
    /**
     * Si este test pasa, significa que:
     * - El módulo se puede cargar
     * - Todas las dependencias están disponibles
     * - La configuración de Spring está correcta
     * - Las entities de JPA están bien definidas
     */
    @Test
    void bootstrapsModule() {
        // Si llegamos aquí, el módulo arrancó exitosamente
    }
}
```

### ¿Cómo funciona TestContainers automáticamente?

Cuando ejecutas `@ApplicationModuleTest`, Spring Modulith realiza la siguiente secuencia automáticamente:

#### 1. Detección Automática de Base de Datos
```yaml
# Spring lee tu application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/store_db  # Ve que usas PostgreSQL
    driver-class-name: org.postgresql.Driver         # Confirma el driver
```

#### 2. Levantamiento de Contenedor
- **Detecta**: "Ah, esta aplicación usa PostgreSQL"
- **Descarga**: Imagen `postgres:15-alpine` si no existe
- **Levanta**: Contenedor Docker con PostgreSQL
- **Configura**: Puerto aleatorio para evitar conflictos

#### 3. Configuración Dinámica
```java
// Spring cambia automáticamente la configuración:
// De: jdbc:postgresql://localhost:5432/store_db
// A:  jdbc:postgresql://localhost:49153/test (puerto dinámico)
```

#### 4. Inicialización de Schema
- **Ejecuta**: Migraciones de Flyway automáticamente
- **Carga**: Scripts desde `src/main/resources/db/migration/`
- **Inserta**: Datos de ejemplo para testing

#### 5. Ejecución del Test
```java
@Test
void myTest() {
    // En este punto:
    // - PostgreSQL está corriendo en un contenedor
    // - Schema está creado con Flyway
    // - Datos de ejemplo están cargados
    // - La aplicación está conectada a la BD de test
}
```

#### 6. Limpieza Automática
- **Al finalizar**: Destruye el contenedor automáticamente
- **Sin rastros**: No quedan procesos ni datos
- **Aislamiento**: Cada test tiene su propia BD limpia

### Ventajas de Este Enfoque

#### ✅ **Testing con BD Real**
```java
// ❌ Con H2 en memoria
@Test 
void testWithH2() {
    // Usa H2, no PostgreSQL
    // Dialectos SQL diferentes
    // Comportamiento puede diferir
}

// ✅ Con TestContainers
@Test
void testWithPostgreSQL() {
    // Usa PostgreSQL real
    // Mismo motor que producción
    // Resultados confiables
}
```

#### ✅ **Aislamiento Perfecto**
```java
@Test
void test1() {
    // Contenedor 1: puerto 49152
    // BD limpia y vacía
}

@Test  
void test2() {
    // Contenedor 2: puerto 49153  
    // BD limpia y vacía
    // No interferencia con test1
}
```

#### ✅ **Sin Configuración Manual**
- No necesitas instalar PostgreSQL localmente
- No necesitas configurar puertos
- No necesitas limpiar datos entre tests
- Todo es automático y transparente

### Comandos de Testing Útiles

```bash
# Ejecutar solo tests del módulo products
mvn test -Dtest="com.example.store.products.**"

# Ejecutar solo tests de command
mvn test -Dtest="*CommandServiceTest"

# Ejecutar con logs de SQL para debugging
mvn test -Dtest=ProductCommandServiceTest -Dspring.jpa.show-sql=true

# Ver qué contenedores están corriendo durante tests
docker ps  # (ejecutar en otra terminal mientras corre el test)
```

### Tips para Testing Efectivo

#### 1. **Usar datos de ejemplo consistentes**
```java
// ✅ Reutilizar IDs de V1__create_initial_schema.sql
UUID LAPTOP_ID = UUID.fromString("f47ac10b-58cc-4372-a567-0e02b2c3d479");

// ❌ Generar IDs aleatorios cada vez
UUID randomId = UUID.randomUUID(); // Impredecible
```

#### 2. **Verificar tanto estado como eventos**
```java
@Test
void testCompleteFlow(PublishedEvents events) {
    // Act
    var id = service.createProduct(...);
    
    // Assert: Estado en BD
    assertThat(repository.findById(id)).isPresent();
    
    // Assert: Evento publicado
    assertThat(events.ofType(ProductCreated.class)).hasSize(1);
}
```

#### 3. **Usar AssertJ para mejor legibilidad**
```java
// ✅ Expresivo y claro
assertThat(product).satisfies(p -> {
    assertThat(p.getName()).isEqualTo("Laptop Pro");
    assertThat(p.getPrice()).isGreaterThan(BigDecimal.ZERO);
});

// ❌ Verboso y difícil de leer
assertEquals("Laptop Pro", product.getName());
assertTrue(product.getPrice().compareTo(BigDecimal.ZERO) > 0);
```

## 12. Próximos Pasos y Mejoras

### Funcionalidades Adicionales que Puedes Implementar

#### 1. **Módulo de Inventario**
```java
// src/main/java/com/example/store/inventory/InventoryService.java
@ApplicationModuleListener
void on(ProductCreated event) {
    // Crear stock inicial cuando se crea un producto
    inventoryRepository.save(new StockEntry(event.id(), event.stock()));
}
```

#### 2. **Módulo de Órdenes**
```java
// Escuchar cuando se hace una orden para reducir stock
@ApplicationModuleListener  
void on(OrderCreated event) {
    event.items().forEach(item -> 
        inventoryService.reserveStock(item.productId(), item.quantity())
    );
}
```

#### 3. **Módulo de Notificaciones**
```java
// Enviar email cuando un producto recibe review malo
@ApplicationModuleListener
void on(ProductReviewed event) {
    if (event.vote() <= 2) {
        emailService.notifyLowRating(event);
    }
}
```

### Optimizaciones de Performance

#### 1. **Cacheo de Consultas Frecuentes**
```java
@GetMapping("/by-rating")
@Cacheable("products-by-rating")
List<ProductView> getProductsByRating() {
    return repository.findAllOrderByRatingDesc();
}
```

#### 2. **Proyecciones para APIs Públicas**
```java
// Solo devolver campos necesarios
public interface ProductSummary {
    String getName();
    BigDecimal getPrice();
    Double getAverageRating();
}

@Query("SELECT p.name as name, p.price as price, p.averageRating as averageRating FROM ProductView p")
List<ProductSummary> findProductSummaries();
```

#### 3. **Paginación para Listas Grandes**
```java
@GetMapping
Page<ProductView> getAllProducts(Pageable pageable) {
    return repository.findAll(pageable);
}
```

### Monitoreo y Observabilidad

#### 1. **Métricas de Eventos**
```java
@Component
class EventMetrics {
    private final MeterRegistry meterRegistry;
    
    @ApplicationModuleListener
    void on(ProductCreated event) {
        meterRegistry.counter("products.created", "category", event.category()).increment();
    }
}
```

#### 2. **Health Checks Personalizados**
```java
@Component
class ProductsHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        long count = productRepository.count();
        return count > 0 ? Health.up() : Health.down();
    }
}
```

### Migración a Microservicios (Cuando Sea Necesario)

#### 1. **Identificar Bounded Contexts**
- Cuando un módulo crece demasiado
- Cuando necesita escalar independientemente
- Cuando tiene equipos separados

#### 2. **Extraer Módulo a Servicio**
```java
// 1. Crear nuevo proyecto Spring Boot
// 2. Copiar el módulo completo
// 3. Cambiar eventos por HTTP/messaging
// 4. Migrar datos gradualmente
```

### Conclusión

Has aprendido a implementar CQRS con Spring Modulith desde cero, incluyendo:

- **Separación clara** entre comandos y queries
- **Comunicación asíncrona** vía eventos
- **Testing independiente** de módulos
- **Arquitectura modular** sin complejidad de microservicios

Esta base te permite **evolucionar gradualmente** hacia microservicios cuando realmente los necesites, no por moda tecnológica.

**Recuerda**: La mejor arquitectura es la que resuelve tus problemas reales con la menor complejidad posible.
