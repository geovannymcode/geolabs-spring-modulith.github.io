# Parte 5 — Testing en Aislamiento

**Duración**: 25 minutos  
**Objetivo**: Testear cada módulo de forma independiente usando las herramientas de test de Spring Modulith

---

## ¿Por qué no usar @SpringBootTest para todo?

Con `@SpringBootTest` Spring carga el contexto completo: todos los módulos, todas las migraciones Flyway, Testcontainers levanta Postgres y RabbitMQ. Para un test que verifica que `ProductCommandService` crea un producto correctamente, eso es excesivo.

Con 4 módulos y 15 tests el costo es soportable. Con 10 módulos y 100 tests el build empieza a doler. `@ApplicationModuleTest` carga **solo el módulo que estás testeando** — los demás no existen para ese test.

---

## Los tres modos de bootstrap

```java
// STANDALONE (default): solo el módulo bajo test
@ApplicationModuleTest

// DIRECT_DEPENDENCIES: el módulo + sus dependencias directas
@ApplicationModuleTest(bootstrapMode = BootstrapMode.DIRECT_DEPENDENCIES)

// ALL_DEPENDENCIES: el módulo + todas sus dependencias transitivas
@ApplicationModuleTest(bootstrapMode = BootstrapMode.ALL_DEPENDENCIES)
```

En esta parte usamos `STANDALONE` para todo — máximo aislamiento.

---

## Prerequisitos para los tres tests

Antes de escribir el primer test, hay dos cosas que configurar una sola vez.

### 1. application.properties de test

El Event Publication Registry necesita su tabla para funcionar. Sin esta propiedad, cualquier test que ejecute código que publica eventos fallará con `relation "event_publication" does not exist`.

Agrega en `src/test/resources/application.properties`:

```properties
# Crea la tabla event_publication en el contexto de test
spring.modulith.events.jdbc.schema-initialization.enabled=true

# Logs limpios en tests
management.tracing.enabled=false
logging.level.org.testcontainers=WARN
logging.level.com.github.dockerjava=WARN
```

### 2. catalog-test-data.sql

El módulo `catalog` tiene CQRS con dos tablas: `catalog.products` (write model) y `catalog.product_views` (read model). Los tests de catalog necesitan datos en **ambas tablas** porque:

- Las consultas (GET) leen de `catalog.product_views`
- La validación de duplicados (POST con código existente) lee de `catalog.products`

Si solo insertas en una, la mitad de los tests fallan.

Crea `src/test/resources/catalog-test-data.sql`:

```sql
-- Limpia datos previos para evitar duplicate key entre tests.
-- Testcontainers reutiliza el contenedor entre tests del mismo contexto.
DELETE FROM catalog.product_views WHERE code IN ('P001','P002','P003','P004','P005');
DELETE FROM catalog.products WHERE code IN ('P001','P002','P003','P004','P005');

-- Write model: para que la validación de duplicados funcione
INSERT INTO catalog.products (code, name, description, image_url, price, category, created_at)
VALUES
    ('P001', 'Clean Code', 'Un manual para crear software ágil.', null, 45.99, 'Ingeniería de Software', now()),
    ('P002', 'The Pragmatic Programmer', 'De novato a maestro.', null, 49.99, 'Ingeniería de Software', now()),
    ('P003', 'Designing Data-Intensive Applications', 'Sistemas de datos a escala.', null, 59.99, 'Sistemas Distribuidos', now()),
    ('P004', 'Domain-Driven Design', 'Tackling complexity.', null, 55.99, 'Arquitectura', now()),
    ('P005', 'Microservices Patterns', 'Con ejemplos en Java.', null, 52.99, 'Arquitectura', now());

-- Read model: para que las consultas GET devuelvan datos
INSERT INTO catalog.product_views
    (code, name, description, image_url, price, category, average_rating, review_count)
VALUES
    ('P001', 'Clean Code', 'Un manual para crear software ágil.', null, 45.99, 'Ingeniería de Software', 4.8, 235),
    ('P002', 'The Pragmatic Programmer', 'De novato a maestro.', null, 49.99, 'Ingeniería de Software', 4.7, 189),
    ('P003', 'Designing Data-Intensive Applications', 'Sistemas de datos a escala.', null, 59.99, 'Sistemas Distribuidos', 4.9, 312),
    ('P004', 'Domain-Driven Design', 'Tackling complexity.', null, 55.99, 'Arquitectura', 4.6, 145),
    ('P005', 'Microservices Patterns', 'Con ejemplos en Java.', null, 52.99, 'Arquitectura', 4.7, 178);
```

---

## Patrón común: MockMvc y FlywayTestConfig

En Spring Boot 4.x con `@ApplicationModuleTest` hay dos cosas que no funcionan como en `@SpringBootTest`:

**MockMvc no está disponible como bean** — `@AutoConfigureMockMvc` no existe en Spring Boot 4.x y aunque existiera, `@ApplicationModuleTest` no lo activa. La solución es construirlo manualmente con `WebApplicationContext`.

**FlywayConfig no puede importarse** — `FlywayConfig` está en el módulo `config`. Spring Modulith rechaza importar beans de módulos ajenos en un contexto aislado. La solución es una clase `@TestConfiguration` interna que redefine Flyway solo para ese test.

Estos dos patrones se repiten en los tres tests. Es un poco de boilerplate, pero es el precio del verdadero aislamiento.

---

## Test 1: Módulo catalog en aislamiento

```java
// src/test/java/com/geovannycode/bookstore/catalog/web/ProductRestControllerTests.java
package com.geovannycode.bookstore.catalog.web;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.geovannycode.bookstore.TestcontainersConfiguration;
import com.geovannycode.bookstore.catalog.command.CreateProductCommand;
import org.flywaydb.core.Flyway;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;
import org.springframework.http.MediaType;
import org.springframework.modulith.test.ApplicationModuleTest;
import org.springframework.test.context.jdbc.Sql;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

import javax.sql.DataSource;
import java.math.BigDecimal;

import static org.hamcrest.Matchers.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

/**
 * Test del módulo catalog en aislamiento total.
 *
 * Spring carga únicamente:
 *   catalog.command.*, catalog.query.*, catalog.internal.*, catalog.web.*, common.*
 *
 * orders e inventory no existen para este test.
 *
 * FlywayTestConfig redefine Flyway localmente porque FlywayConfig está en
 * el módulo 'config' y Spring Modulith no permite importarlo en un contexto
 * aislado de otro módulo.
 *
 * @Sql carga datos en ambas tablas CQRS antes de cada test porque:
 * - catalog.products: para que la validación de duplicados funcione
 * - catalog.product_views: para que las consultas GET devuelvan datos
 */
@ApplicationModuleTest
@Import(TestcontainersConfiguration.class)
@Sql("/catalog-test-data.sql")
class ProductRestControllerTests {

    @TestConfiguration
    static class FlywayTestConfig {
        @Bean(initMethod = "migrate")
        Flyway flyway(DataSource dataSource) {
            return Flyway.configure()
                    .dataSource(dataSource)
                    .locations("classpath:db/migration")
                    .load();
        }
    }

    @Autowired
    WebApplicationContext context;

    MockMvc mockMvc;

    // ObjectMapper no está disponible como bean en el contexto aislado.
    // Se instancia directamente — es suficiente para serializar los commands.
    final ObjectMapper objectMapper = new ObjectMapper();

    @BeforeEach
    void setUp() {
        // MockMvc no se autoconfigura en @ApplicationModuleTest con Spring Boot 4.x.
        // Se construye manualmente con el WebApplicationContext del módulo.
        mockMvc = MockMvcBuilders.webAppContextSetup(context).build();
    }

    @Test
    void shouldReturnProductsPagedResult() throws Exception {
        mockMvc.perform(get("/api/catalog/products")
                        .param("page", "1")
                        .param("size", "5"))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data", hasSize(greaterThan(0))))
                .andExpect(jsonPath("$.pageNumber", is(1)))
                .andExpect(jsonPath("$.isFirst", is(true)));
    }

    @Test
    void shouldReturnProductByCode() throws Exception {
        mockMvc.perform(get("/api/catalog/products/P001"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code", is("P001")))
                .andExpect(jsonPath("$.name", is("Clean Code")))
                .andExpect(jsonPath("$.averageRating", greaterThan(0.0)));
    }

    @Test
    void shouldReturn404ForNonExistentProduct() throws Exception {
        // Solo verificamos el status — en el contexto aislado los message converters
        // de Spring Boot no están completamente configurados y el body de
        // Problem Details puede no serializarse correctamente.
        mockMvc.perform(get("/api/catalog/products/NONEXISTENT"))
                .andExpect(status().isNotFound());
    }

    @Test
    void shouldReturnProductsByCategory() throws Exception {
        mockMvc.perform(get("/api/catalog/products/category/Arquitectura"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$", hasSize(greaterThan(0))))
                .andExpect(jsonPath("$[0].category", is("Arquitectura")));
    }

    @Test
    void shouldReturnTopRatedProducts() throws Exception {
        mockMvc.perform(get("/api/catalog/products/top-rated")
                        .param("minRating", "4.7"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$[0].averageRating", greaterThanOrEqualTo(4.7)));
    }

    @Test
    void shouldCreateProductSuccessfully() throws Exception {
        var command = new CreateProductCommand(
                "P999", "Test Book", "Un libro de prueba",
                null, new BigDecimal("29.99"), "Testing"
        );

        mockMvc.perform(post("/api/catalog/products")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(command)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.code", is("P999")))
                .andExpect(jsonPath("$.name", is("Test Book")));
    }

    @Test
    void shouldReturn409WhenCreatingDuplicateProduct() throws Exception {
        // P001 existe en catalog.products (cargado por @Sql).
        // ProductCommandService.existsByCode() lo encuentra y lanza
        // ProductAlreadyExistsException → CatalogExceptionHandler → 409
        var command = new CreateProductCommand(
                "P001", "Duplicado", "Ya existe P001",
                null, new BigDecimal("10.00"), "Testing"
        );

        mockMvc.perform(post("/api/catalog/products")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(command)))
                .andExpect(status().isConflict());
    }

    @Test
    void shouldReturn400WhenCreatingProductWithInvalidData() throws Exception {
        var invalidJson = """
                {
                  "code": "",
                  "name": "Sin código",
                  "price": -5.00,
                  "category": "Test"
                }
                """;

        mockMvc.perform(post("/api/catalog/products")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(invalidJson))
                .andExpect(status().isBadRequest());
    }
}
```

### Ejecutar

```bash
mvn test -Dtest="ProductRestControllerTests"
```

Observa en los logs cuántos beans se crean comparado con `@SpringBootTest`. Verás catalog, common — y nada más.

---

## Test 2: Módulo orders en aislamiento con Mocks y Eventos

`orders` depende de `catalog` via `CatalogApi`. En `STANDALONE`, `catalog` no se carga — por eso mockeamos `CatalogApi` con `@MockitoBean`.

`AssertablePublishedEvents` se inyecta como parámetro del método de test. Captura todos los eventos publicados durante esa ejecución y nos permite verificar que `orders` cumplió su contrato de publicación: cuando se crea una orden, debe publicar `OrderCreatedEvent` con los datos correctos.

!!! info "¿Por qué verificar el evento y no solo el status HTTP?"
    El status 201 confirma que la orden se guardó. Pero el contrato del módulo incluye también la publicación del evento — si alguien borra el `eventPublisher.publishEvent()` de `OrderService`, el status sigue siendo 201 y ningún test lo detecta. `AssertablePublishedEvents` cierra ese hueco.

```java
// src/test/java/com/geovannycode/bookstore/orders/web/OrderRestControllerTests.java
package com.geovannycode.bookstore.orders.web;

import com.geovannycode.bookstore.TestcontainersConfiguration;
import com.geovannycode.bookstore.catalog.CatalogApi;
import com.geovannycode.bookstore.catalog.Product;
import com.geovannycode.bookstore.orders.OrderCreatedEvent;
import org.flywaydb.core.Flyway;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;
import org.springframework.http.MediaType;
import org.springframework.modulith.test.ApplicationModuleTest;
import org.springframework.modulith.test.AssertablePublishedEvents;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.setup.MockMvcBuilders;
import org.springframework.web.context.WebApplicationContext;

import javax.sql.DataSource;
import java.math.BigDecimal;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.hamcrest.Matchers.startsWith;
import static org.mockito.BDDMockito.given;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

/**
 * Test del módulo orders en aislamiento.
 *
 * @MockitoBean CatalogApi: catalog no se carga en STANDALONE.
 * Mockito provee el bean para que OrderService pueda inyectarlo.
 * En @BeforeEach configuramos qué devuelve para cada código de producto.
 *
 * AssertablePublishedEvents: se inyecta como parámetro del test.
 * Captura los eventos publicados durante la ejecución del método.
 */
@ApplicationModuleTest
@Import(TestcontainersConfiguration.class)
class OrderRestControllerTests {

    @TestConfiguration
    static class FlywayTestConfig {
        @Bean(initMethod = "migrate")
        Flyway flyway(DataSource dataSource) {
            return Flyway.configure()
                    .dataSource(dataSource)
                    .locations("classpath:db/migration")
                    .load();
        }
    }

    @Autowired
    WebApplicationContext context;

    MockMvc mockMvc;

    @MockitoBean
    CatalogApi catalogApi;

    @BeforeEach
    void setUp() {
        mockMvc = MockMvcBuilders.webAppContextSetup(context).build();

        var product = new Product(
                "P001", "Clean Code", "Un manual de calidad",
                null, new BigDecimal("45.99"), "Ingeniería de Software",
                4.8, 235
        );
        given(catalogApi.getByCode("P001")).willReturn(Optional.of(product));
        given(catalogApi.getByCode("INEXISTENTE")).willReturn(Optional.empty());
    }

    @Test
    void shouldCreateOrderSuccessfully(AssertablePublishedEvents events) throws Exception {
        var request = """
                {
                  "customerName": "Geovanny Mendoza",
                  "customerEmail": "geo@barranquillajug.com",
                  "customerPhone": "+57 300 1234567",
                  "deliveryAddress": "Calle 72 #45-10, Barranquilla",
                  "items": [
                    { "productCode": "P001", "quantity": 2 }
                  ]
                }
                """;

        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(request))
                .andDo(print())
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.orderNumber", startsWith("ORD-")));

        // Verificamos que orders publicó el evento con los datos correctos.
        // Si se elimina el eventPublisher.publishEvent() de OrderService,
        // este test falla aunque el status HTTP siga siendo 201.
        var orderEvents = events.ofType(OrderCreatedEvent.class);

        assertThat(orderEvents).isNotEmpty();
        assertThat(orderEvents)
                .anySatisfy(event -> {
                    assertThat(event.orderNumber()).startsWith("ORD-");
                    assertThat(event.customer().email()).isEqualTo("geo@barranquillajug.com");
                    assertThat(event.items()).hasSize(1);
                    assertThat(event.items().get(0).productCode()).isEqualTo("P001");
                    assertThat(event.items().get(0).quantity()).isEqualTo(2);
                });
    }

    @Test
    void shouldReturn400WhenProductNotFound() throws Exception {
        // El producto no existe en el catálogo → InvalidOrderException → 400.
        // Nota: InvalidOrderException debe lanzarse en OrderService (no OrderNotFoundException)
        // cuando el producto no se encuentra durante la creación.
        // InvalidOrderException → 400 (entrada inválida)
        // OrderNotFoundException → 404 (orden no encontrada)
        var request = """
                {
                  "customerName": "Test User",
                  "customerEmail": "test@test.com",
                  "customerPhone": "+57 300 0000000",
                  "deliveryAddress": "Test Address",
                  "items": [
                    { "productCode": "INEXISTENTE", "quantity": 1 }
                  ]
                }
                """;

        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(request))
                .andExpect(status().isBadRequest());
    }

    @Test
    void shouldReturn404ForNonExistentOrder() throws Exception {
        mockMvc.perform(get("/api/orders/ORD-NOTFOUND"))
                .andExpect(status().isNotFound());
    }

    @Test
    void shouldReturn400ForInvalidRequest() throws Exception {
        // Lista de ítems vacía y campos de cliente vacíos → validación de Bean Validation → 400
        var invalidRequest = """
                {
                  "customerName": "",
                  "customerEmail": "not-an-email",
                  "customerPhone": "",
                  "deliveryAddress": "",
                  "items": []
                }
                """;

        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(invalidRequest))
                .andExpect(status().isBadRequest());
    }

    @Test
    void shouldNotPublishEventWhenOrderFails(AssertablePublishedEvents events) throws Exception {
        // Si la orden falla, el evento no debe publicarse.
        // Esto verifica que el evento y la transacción son atómicos:
        // si no se guarda la orden, no hay evento.
        var request = """
                {
                  "customerName": "Test",
                  "customerEmail": "test@test.com",
                  "customerPhone": "+57 300 0000000",
                  "deliveryAddress": "Test",
                  "items": [
                    { "productCode": "INEXISTENTE", "quantity": 1 }
                  ]
                }
                """;

        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(request))
                .andExpect(status().isBadRequest());

        assertThat(events.ofType(OrderCreatedEvent.class)).isEmpty();
    }
}
```

### Nota importante sobre InvalidOrderException vs OrderNotFoundException

`OrderService` debe lanzar `InvalidOrderException` (no `OrderNotFoundException`) cuando el producto no existe durante la creación. El error semántico es diferente:

- `InvalidOrderException` → el cliente envió datos incorrectos → HTTP 400
- `OrderNotFoundException` → se buscó una orden por número y no existe → HTTP 404

Si ves el test `shouldReturn400WhenProductNotFound` fallando con 404 en lugar de 400, verifica que en `OrderService.create()` el `.orElseThrow()` lanza `InvalidOrderException`.

### Ejecutar

```bash
mvn test -Dtest="OrderRestControllerTests"
```

---

## Test 3: Módulo inventory con Scenario

`inventory` es puramente reactivo — no tiene endpoints HTTP, solo escucha eventos. Testearlo con MockMvc no aplica. `Scenario` publica el evento directamente, como si `orders` lo hubiera enviado, sin necesitar que el módulo `orders` esté en el contexto.

El patrón de Scenario es:
```
scenario.publish(evento)
        .andWaitForStateChange(() -> consulta el nuevo estado)
        .andVerify(nuevoEstado -> verifica que es correcto)
```

Internamente usa polling: consulta el estado periódicamente hasta que cambia o hasta que expira el timeout. Eso es necesario porque `@ApplicationModuleListener` es asíncrono — el handler puede ejecutarse unos milisegundos después de publicar el evento.

```java
// src/test/java/com/geovannycode/bookstore/inventory/InventoryIntegrationTests.java
package com.geovannycode.bookstore.inventory;

import com.geovannycode.bookstore.TestcontainersConfiguration;
import com.geovannycode.bookstore.orders.OrderCreatedEvent;
import org.flywaydb.core.Flyway;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;
import org.springframework.modulith.test.ApplicationModuleTest;
import org.springframework.modulith.test.Scenario;

import javax.sql.DataSource;
import java.math.BigDecimal;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Test del módulo inventory en aislamiento usando Scenario.
 *
 * Scenario publica el evento directamente — el módulo orders no se carga.
 * Testa exactamente la responsabilidad de inventory:
 * "dado este OrderCreatedEvent, ¿se descuenta el stock de cada ítem?"
 *
 * Con mock del eventPublisher solo verificarías que publishEvent() fue llamado.
 * Con Scenario verificas que el handler ejecutó Y que el estado en BD cambió.
 * Esa diferencia importa.
 */
@ApplicationModuleTest
@Import(TestcontainersConfiguration.class)
class InventoryIntegrationTests {

    @TestConfiguration
    static class FlywayTestConfig {
        @Bean(initMethod = "migrate")
        Flyway flyway(DataSource dataSource) {
            return Flyway.configure()
                    .dataSource(dataSource)
                    .locations("classpath:db/migration")
                    .load();
        }
    }

    @Autowired
    InventoryService inventoryService;

    @Test
    void shouldDecreaseStockWhenOrderCreatedEventReceived(Scenario scenario) {
        int stockBefore = inventoryService.getStockLevel("P001");
        assertThat(stockBefore).isGreaterThan(0);

        // Construimos el evento con el formato multi-ítem de la Parte 4.
        // Un solo ítem — tres unidades de P001.
        var event = new OrderCreatedEvent(
                "ORD-TEST-001",
                List.of(
                    new OrderCreatedEvent.Item("P001", "Clean Code", new BigDecimal("45.99"), 3)
                ),
                new OrderCreatedEvent.Customer(
                        "Test User", "test@test.com",
                        "+57 300 0000000", "Test Address"
                )
        );

        // Publicamos el evento directamente.
        // Scenario espera a que el stock de P001 cambie (polling).
        // andVerify recibe el nuevo valor del stock.
        scenario.publish(event)
                .andWaitForStateChange(
                        () -> inventoryService.getStockLevel("P001")
                )
                .andVerify(newStockLevel ->
                        assertThat(newStockLevel).isEqualTo(stockBefore - 3)
                );
    }

    @Test
    void shouldHandleMultipleItemsInOrder(Scenario scenario) {
        // Orden con dos productos distintos — verifica que el handler
        // itera correctamente sobre todos los ítems del evento.
        int stockP002Before = inventoryService.getStockLevel("P002");
        int stockP003Before = inventoryService.getStockLevel("P003");

        var event = new OrderCreatedEvent(
                "ORD-TEST-002",
                List.of(
                    new OrderCreatedEvent.Item("P002", "The Pragmatic Programmer", new BigDecimal("49.99"), 5),
                    new OrderCreatedEvent.Item("P003", "Designing Data-Intensive Applications", new BigDecimal("59.99"), 2)
                ),
                new OrderCreatedEvent.Customer(
                        "Bulk Buyer", "bulk@test.com",
                        "+57 311 0000000", "Warehouse"
                )
        );

        // Esperamos a que P002 cambie (primer ítem procesado).
        // Luego verificamos ambos productos en andVerify.
        scenario.publish(event)
                .andWaitForStateChange(
                        () -> inventoryService.getStockLevel("P002")
                )
                .andVerify(newStockP002 -> {
                    assertThat(newStockP002).isEqualTo(stockP002Before - 5);
                    assertThat(inventoryService.getStockLevel("P003"))
                            .isEqualTo(stockP003Before - 2);
                });
    }

    @Test
    void shouldGetCurrentStockLevel() {
        // Test directo sin eventos — verifica que Flyway cargó stock para P003.
        // No usamos un número exacto porque otros tests del mismo contenedor
        // podrían haber modificado el stock antes de que este test corra.
        int stock = inventoryService.getStockLevel("P003");
        assertThat(stock).isGreaterThan(0);
    }
}
```

### Ejecutar

```bash
mvn test -Dtest="InventoryIntegrationTests"
```

---

## Smoke test y ModularityTest

```java
// src/test/java/com/geovannycode/bookstore/BookstoreApplicationTests.java
@SpringBootTest
@Import(TestcontainersConfiguration.class)
class BookstoreApplicationTests {

    @Test
    void contextLoads() {
        // Verifica que el contexto completo carga: Flyway, JPA, RabbitMQ, Spring Modulith.
        // Si este test pasa, el entorno está bien configurado.
    }
}
```

```java
// src/test/java/com/geovannycode/bookstore/ModularityTest.java
class ModularityTest {

    static final ApplicationModules modules =
            ApplicationModules.of(BookstoreApplication.class);

    @Test
    void verifiesModularStructure() {
        modules.verify();
    }

    @Test
    void printsModuleStructure() {
        modules.forEach(System.out::println);
    }
}
```

---

## Lo que aprendimos sobre @ApplicationModuleTest en Spring Boot 4.x

Spring Boot 4.x cambió algunas cosas que afectan los tests de módulos. Vale documentarlas porque no están en la mayoría de tutoriales:

| Problema | Causa | Solución |
|---|---|---|
| `MockMvc` no disponible como bean | `@AutoConfigureMockMvc` no existe en Boot 4.x | `MockMvcBuilders.webAppContextSetup(context).build()` en `@BeforeEach` |
| `ObjectMapper` no disponible como bean | El contexto aislado no autoconfigura Jackson | `new ObjectMapper()` directamente |
| `FlywayConfig` no importable | Pertenece al módulo `config`, Spring Modulith lo rechaza | `@TestConfiguration` static inner class |
| `event_publication` no existe | La tabla no se crea automáticamente en el contexto aislado | `spring.modulith.events.jdbc.schema-initialization.enabled=true` en `src/test/resources/application.properties` |
| Duplicate key en `@Sql` | Testcontainers reutiliza el contenedor entre tests | `DELETE` antes del `INSERT` en el script SQL |
| Problem Details sin body | Message converters no completamente configurados en aislamiento | Solo verificar el status HTTP, no el body del error |

---

## Comparación: @SpringBootTest vs @ApplicationModuleTest

| Aspecto | `@SpringBootTest` | `@ApplicationModuleTest` |
|---|---|---|
| Beans cargados | Todos los módulos | Solo el módulo bajo test |
| Tiempo de startup | ~15s | ~4s |
| Flyway | Todas las migraciones | Solo las necesarias |
| Dependencias externas | Postgres + RabbitMQ | Solo las del módulo |
| Módulos externos | Reales | Mockeados |
| Uso ideal | Smoke test, integración total | Cada módulo por separado |

---

## Resumen de herramientas

| Herramienta | Para qué |
|---|---|
| `@ApplicationModuleTest` | Carga solo el módulo bajo test |
| `@MockitoBean` | Reemplaza dependencias de otros módulos |
| `AssertablePublishedEvents` | Verifica eventos publicados en el test |
| `Scenario` | Testea flujos event-driven sin el módulo publicador |
| `ModularityTest` | Verifica la arquitectura completa |

---

## Checklist de la Parte 5

- [ ] `src/test/resources/application.properties` con `spring.modulith.events.jdbc.schema-initialization.enabled=true`
- [ ] `src/test/resources/catalog-test-data.sql` con DELETE + INSERT en ambas tablas CQRS
- [ ] `ProductRestControllerTests` pasa — 8 tests en verde ✅
- [ ] `OrderRestControllerTests` pasa — 5 tests en verde ✅
- [ ] `InventoryIntegrationTests` pasa — 3 tests en verde ✅
- [ ] `BookstoreApplicationTests` (smoke test) pasa ✅
- [ ] `ModularityTest` pasa ✅
- [ ] `mvn test` completa sin errores ✅

---

**Anterior**: [Parte 4 — Eventos y Outbox](parte_4.md) &nbsp;&nbsp;&nbsp;&nbsp; **Siguiente**: [Parte 6 — C4 Docs, Observabilidad y Docker](parte_6.md)
