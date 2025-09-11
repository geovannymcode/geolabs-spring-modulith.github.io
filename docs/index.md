# Spring Modulith en la Práctica
## Arquitecturas Modulares y CQRS

---

## Agenda

1. Introducción a la arquitectura modular
2. Problemas del monolito tradicional
3. Spring Modulith como solución
4. Conceptos clave de Spring Modulith
5. Implementando CQRS con Spring Modulith
6. Caso práctico: Transformando un e-commerce
7. Testing de aplicaciones modulares
8. Comunicación basada en eventos
9. Mejores prácticas y antipatrones
10. Conclusiones y recursos

---

## 1. Introducción a la Arquitectura Modular

[IMAGEN 1: Diagrama de código acoplado en monolitos tradicionales]

* **¿Qué es una arquitectura modular?**
  * Diseño que divide la aplicación en módulos independientes
  * Cada módulo tiene responsabilidades bien definidas
  * Interfaces claras entre módulos

* **Beneficios:**
  * Mejor organización del código
  * Mayor mantenibilidad
  * Desarrollo paralelo
  * Testing independiente
  * Escalabilidad selectiva

---

## 2. Problemas del Monolito Tradicional

[IMAGEN 1: Diagrama de código acoplado en monolitos tradicionales]

* **Organización por capas (package-by-layer):**
  * Controllers, Services, Repositories, Entities...
  * No expresa el propósito de negocio de la aplicación

* **Acoplamiento elevado:**
  * Clases públicas accesibles desde cualquier lugar
  * Dependencias implícitas difíciles de rastrear

* **Desafíos:**
  * Código espagueti
  * Difícil añadir o modificar funcionalidades
  * Testing complejo
  * Dificultad para escalar partes específicas

---

## 3. Spring Modulith como Solución

[IMAGEN 10: Diagrama de BookStore Modulith con módulos]

* **Spring Modulith:**
  * Extensión de Spring Boot para arquitecturas modulares
  * Permite crear "monolitos modulares"
  * Facilita la evolución hacia microservicios (si es necesario)

* **Características principales:**
  * Verificación de fronteras entre módulos
  * Comunicación basada en eventos
  * Soporte para testing modular
  * Documentación automática (Modelo C4)

---

## 4. Conceptos Clave de Spring Modulith

[IMAGEN 11: Diagrama mostrando módulos con APIs y componentes internos]

* **Módulos de aplicación:**
  * Paquetes de nivel superior bajo el paquete de aplicación
  * Cada módulo tiene su API pública y componentes internos

* **Tipos de módulos:**
  * **Simple:** API pública limitada
  * **OPEN:** Todo el contenido es público
  * **Advanced:** Control detallado de la exposición

* **NamedInterfaces:**
  * Exponer tipos o paquetes adicionales
  * Control granular de la API pública

---

## 5. CQRS: Command Query Responsibility Segregation

[IMAGEN 11: Diagrama mostrando módulos con APIs y componentes internos]

* **Fundamentos de CQRS:**
  * Separación de operaciones de lectura y escritura
  * Commands: modifican estado (escriben)
  * Queries: recuperan información (leen)

* **Beneficios con Spring Modulith:**
  * Modelos optimizados para cada caso de uso
  * Escalabilidad independiente
  * Rendimiento mejorado para lecturas
  * Menor contención de recursos

---

## 6. Dos Enfoques para CQRS con Spring Modulith

* **Enfoque 1: CQRS como estructura de módulos**
  ```
  gae.piaz.modulith.cqrs/  
  ├── command/  (CLOSED module)
  │   ├── api/  
  │   ├── domain/ 
  │   └── events/
  ├── query/    (CLOSED module)
  │   ├── api/
  │   ├── domain/  
  └── shared/    (OPEN module)
  ```

* **Enfoque 2: CQRS dentro del módulo (recomendado)**
  ```
  com.example.products/  (MODULE)
  ├── command/  
  │   ├── api/  
  │   └── internal/ 
  ├── query/    
  │   ├── api/
  │   └── internal/  
  └── shared/    
  ```

---

## 7. Implementando CQRS con Spring Modulith

* **Domain Events como puente:**

```java
// Command Side
@Service
public class ProductCommandService {
    private final ProductRepository repository;
    private final ApplicationEventPublisher publisher;
    
    @Transactional
    public Long createProduct(ProductRequest request) {
        Product product = new Product(
            request.name(), request.description(), request.price());
        Product saved = repository.save(product);
        
        // Publicar evento
        publisher.publishEvent(new ProductCreatedEvent(
            saved.getId(), saved.getName(), saved.getDescription(), 
            saved.getPrice(), saved.getStock(), saved.getCategory()));
        
        return saved.getId();
    }
}
```

---

## 8. Manejo de Eventos CQRS

```java
// Query Side
@Service
public class ProductEventHandler {
    private final ProductViewRepository viewRepository;
    
    @ApplicationModuleListener
    public void on(ProductCreatedEvent event) {
        ProductView view = new ProductView();
        view.setId(event.id());
        view.setName(event.name());
        view.setDescription(event.description());
        view.setPrice(event.price());
        view.setStock(event.stock());
        view.setCategory(event.category());
        
        viewRepository.save(view);
    }
    
    @ApplicationModuleListener
    public void on(ProductReviewEvent event) {
        viewRepository.findById(event.productId()).ifPresent(view -> {
            // Calcular nueva calificación promedio
            double currentTotal = view.getAverageRating() * view.getReviewCount();
            int newCount = view.getReviewCount() + 1;
            double newAverage = (currentTotal + event.vote()) / newCount;
            
            // Actualizar vista desnormalizada
            view.setReviewCount(newCount);
            view.setAverageRating(newAverage);
            
            viewRepository.save(view);
        });
    }
}
```

---

## 9. Modelos Optimizados para Cada Propósito

**Command Side (Optimizado para Escritura):**
```java
@Entity  
@Table(name = "product")  
public class Product {  
    @Id @GeneratedValue
    private Long id;  
    private String name;  
    private String description;  
    private BigDecimal price;  
    private Integer stock;  
    private String category;  
  
    @OneToMany(mappedBy = "product")
    private List<Review> reviews = new ArrayList<>();  
}
```

**Query Side (Optimizado para Lectura):**
```java
@Entity
@Table(name = "product_views")  
public class ProductView {  
    @Id  
    private Long id;  
    private String name;  
    private String description;  
    private BigDecimal price;  
    private Integer stock;  
    private String category;  
      
    // Datos desnormalizados
    private Double averageRating = 0.0;  
    private Integer reviewCount = 0;  
}
```

---

## 10. Caso Práctico: E-commerce Modular

[IMAGEN 10: Diagrama de BookStore Modulith con módulos]

* **Aplicación demo: BookStore**
  * Refactorización de monolito a arquitectura modular

* **Módulos:**
  * **Common:** Componentes compartidos
  * **Catalog:** Gestión de productos (CQRS para catálogo)
  * **Orders:** Gestión de pedidos
  * **Inventory:** Control de stock

* **Comunicación entre módulos:**
  * API pública explícita
  * Eventos de dominio

---

## 11. Refactorizando hacia Módulos

### Paso 1: Reorganización del Código

[IMAGEN 4: Estructura de directorios del proyecto en IDE]

* **Package-by-feature en lugar de package-by-layer**
  * Estructura:
    ```
    bookstore
      |- config
      |- common
      |- catalog
      |   - domain
      |   - web
      |- orders
      |   - domain
      |   - web
      |- inventory
    ```

* **Visibilidad controlada:** Reducir uso de `public`

---

## 12. Añadiendo Spring Modulith

* **Dependencias Maven:**
  ```xml
  <dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-core</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
  ```

* **Verificación de la estructura modular:**
  ```java
  @Test
  void verifiesModularStructure() {
      ApplicationModules.of(BookStoreApplication.class).verify();
  }
  ```

[IMAGEN 5: Captura de pantalla mostrando módulos en la estructura de IntelliJ IDEA]

---

## 13. Definición de Módulos

[IMAGEN 6: Captura de violaciones de módulos en IDE]

* **Configurando tipo de módulo:**

```java
@ApplicationModule(type = ApplicationModule.Type.OPEN)
package com.sivalabs.bookstore.common;

import org.springframework.modulith.ApplicationModule;
```

```java
@ApplicationModule(allowedDependencies = {"catalog", "common"})
package com.sivalabs.bookstore.orders;

import org.springframework.modulith.ApplicationModule;
```

* **Definiendo NamedInterfaces:**

```java
@NamedInterface("order-models")
package com.sivalabs.bookstore.orders.domain.models;

import org.springframework.modulith.NamedInterface;
```

---

## 14. Implementando CQRS en el módulo Catálogo

* **Separación de modelos:**

```java
// Modelo Command
@Entity
@Table(name = "products", schema = "catalog")
public class ProductEntity {
    @Id
    private String code;
    private String name;
    private String description;
    private BigDecimal price;
    private String imageUrl;
    // Campos optimizados para escritura
}

// Modelo Query
@Entity
@Table(name = "product_views", schema = "catalog_query")
public class ProductView {
    @Id
    private String code;
    private String name;
    private String description;
    private BigDecimal price;
    private String imageUrl;
    // Campos desnormalizados optimizados para lectura
    private Double averageRating;
    private Integer reviewCount;
    private String categoryName;
    private String tags;
}
```

---

## 15. Definiendo APIs Públicas de Módulos

[IMAGEN 8: Captura mostrando violación de dependencia de módulo]

* **API explícita para Catálogo:**
  ```java
  @Service
  public class CatalogApi {
      private final ProductService productService;
      
      public Optional<Product> getByCode(String code) {
          return productService.getByCode(code);
      }
  }
  ```

* **Evitar acceso directo a componentes internos:**
  * OrderService debe usar CatalogApi, no ProductService

[IMAGEN 9: Configuración de dependencias permitidas]

---

## 16. Comunicación Basada en Eventos

* **Publicación de eventos:**
  ```java
  @Service
  class OrderService {
      private final ApplicationEventPublisher publisher;
  
      @Transactional
      void createOrder(CreateOrderRequest request) {
          // Lógica de creación
          OrderEntity order = // ...
          
          // Publicar evento
          publisher.publishEvent(new OrderCreatedEvent(
              order.getOrderId(),
              order.getProductCode(),
              order.getQuantity(),
              order.getCustomer()
          ));
      }
  }
  ```

---

## 17. Manejo de Eventos

* **Escucha de eventos con Spring Modulith:**
  ```java
  @Component
  class OrderCreatedEventHandler {
      private final InventoryService inventoryService;
      
      @ApplicationModuleListener
      void handle(OrderCreatedEvent event) {
          log.info("Procesando pedido: {}", event.orderNumber());
          
          // Actualizar inventario
          inventoryService.updateStock(
              event.productCode(), 
              -event.quantity()
          );
      }
  }
  ```

* **@ApplicationModuleListener combina:**
  * `@Async`
  * `@Transactional(propagation = Propagation.REQUIRES_NEW)`
  * `@TransactionalEventListener`

---

## 18. Persistencia de Eventos

* **Event Publication Registry:**
  ```xml
  <dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-jdbc</artifactId>
  </dependency>
  ```

  ```properties
  spring.modulith.events.jdbc.schema-initialization.enabled=true
  spring.modulith.events.completion-mode=update
  spring.modulith.events.republish-outstanding-events-on-restart=true
  ```

* **Beneficios:**
  * Resiliencia ante fallos
  * Auditoría de eventos
  * Garantía de entrega

---

## 19. Externalización de Eventos

* **Integración con sistemas de mensajería:**
  ```xml
  <dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-events-amqp</artifactId>
  </dependency>
  ```

  ```java
  @Externalized("BookStoreExchange::orders.new")
  public record OrderCreatedEvent(
      String orderNumber,
      String productCode,
      int quantity,
      Customer customer
  ) {}
  ```

---

## 20. Testing de Módulos Aislados

* **@ApplicationModuleTest:**
  ```java
  @ApplicationModuleTest(webEnvironment = RANDOM_PORT)
  @Import(TestcontainersConfiguration.class)
  @AutoConfigureMockMvc
  class OrderRestControllerTests {
      @MockitoBean
      CatalogApi catalogApi;
      
      @BeforeEach
      void setUp() {
          Product product = new Product("P100", "The Hunger Games", "", 
                  null, new BigDecimal("34.0"));
          given(catalogApi.getByCode("P100")).willReturn(Optional.of(product));
      }
      
      // Pruebas independientes del módulo Orders
  }
  ```

---

## 21. Verificación de Eventos

* **Testing de publicación de eventos:**
  ```java
  @Test
  void shouldCreateOrderSuccessfully(AssertablePublishedEvents events) {
      MvcTestResult testResult = mockMvcTester.post().uri("/api/orders")
              .contentType(MediaType.APPLICATION_JSON)
              .content("""
                      {
                          "productCode": "P100",
                          "quantity": 2,
                          "customer": {
                              "name": "Siva",
                              "email": "siva123@gmail.com",
                              "phone": "9987654"
                          }
                      }
                      """)
              .exchange();
      assertThat(testResult).hasStatus(HttpStatus.CREATED);

      // Verificar publicación del evento
      assertThat(events)
              .contains(OrderCreatedEvent.class)
              .matching(e -> e.customer().email(), "siva123@gmail.com")
              .matching(OrderCreatedEvent::productCode, "P100");
  }
  ```

---

## 22. Testing de Manejadores de Eventos

* **Pruebas de integración para eventos:**

```java
@ApplicationModuleTest
@Import(TestcontainersConfiguration.class)
class InventoryIntegrationTests {
    @Autowired
    private InventoryService inventoryService;
    
    @Test
    void handleOrderCreatedEvent(Scenario scenario) {
        var productCode = "P114";
        var customer = new Customer("Siva", "siva@gmail.com", "9987654");
        var event = new OrderCreatedEvent(UUID.randomUUID().toString(), 
                productCode, 2, customer);

        scenario.publish(event)
                .andWaitForStateChange(() -> 
                    inventoryService.getStockLevel(productCode) == 598)
                .andVerify(result -> assertThat(result).isTrue());
    }
}
```

---

## 23. Documentación Automática

* **Generación de documentación C4:**
  ```java
  @Test
  void verifiesModularStructure() {
      modules.verify();
      new Documenter(modules).writeDocumentation();
  }
  ```

* **Resultado:**
  * Diagramas automáticos de la estructura modular
  * Documentación de dependencias entre módulos
  * Generados en `target/spring-modulith-docs`

---

## 24. Mejores Prácticas

[IMAGEN 11: Diagrama mostrando módulos con APIs y componentes internos]

* **Diseño de módulos:**
  * Cohesión alta, acoplamiento bajo
  * Diseñar interfaces públicas cuidadosamente
  * Evitar dependencias circulares

* **Comunicación entre módulos:**
  * Preferir eventos para comunicación asíncrona
  * API explícita para operaciones síncronas
  * Evitar accesos directos a componentes internos

* **CQRS como implementación interna:**
  * Modelar módulos por capacidades de negocio, no por concerns técnicos
  * CQRS como detalle de implementación dentro del módulo
  * Separar modelos de lectura/escritura donde aporte valor real

---

## 25. Antipatrones a Evitar

* **Módulos demasiado acoplados**
* **Dependencias circulares**
* **Módulos con responsabilidades mixtas**
* **Exposición excesiva de detalles internos**
* **Estructurar módulos por aspectos técnicos en lugar de por dominio**
* **Uso excesivo de clases públicas**
* **CQRS innecesariamente complejo para casos simples**

---

## 26. Conclusiones

[IMAGEN 10: Diagrama de BookStore Modulith con módulos]

* **Arquitectura modular con Spring Boot:**
  * Mantiene ventajas del monolito
  * Resuelve problemas de acoplamiento
  * Facilita evolución y mantenimiento

* **CQRS:**
  * Rendimiento optimizado para cada caso de uso
  * Separación clara de responsabilidades

* **Spring Modulith:**
  * Herramienta perfecta para monolitos modulares
  * Verificación automática de reglas arquitectónicas
  * Comunicación robusta basada en eventos

---

## 27. Recursos

* GitHub: [https://github.com/sivaprasadreddy/spring-modulith-workshop](https://github.com/sivaprasadreddy/spring-modulith-workshop)
* CQRS con Spring Modulith: [https://github.com/odrotbohm/cqrs-spring-modulith](https://github.com/odrotbohm/cqrs-spring-modulith)
* Blog: [https://spring.io/blog/2022/10/21/introducing-spring-modulith](https://spring.io/blog/2022/10/21/introducing-spring-modulith)

---

## ¡Gracias!

**Preguntas y Respuestas**
