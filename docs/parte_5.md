# Parte 5 — Testing en Aislamiento

**Duración**: 20 minutos  
**Objetivo**: Testear cada módulo de forma independiente usando las herramientas de test de Spring Modulith

---

## El Problema con `@SpringBootTest`

Antes de ver la solución, entendamos por qué el testing tradicional no escala en un monolito modular.

```java
// La forma tradicional
@SpringBootTest
@Import(TestcontainersConfiguration.class)
class OrderServiceTests {
    // Spring carga:
    // - Todos los beans de catalog (ProductCommandService, CatalogApi, ProductView...)
    // - Todos los beans de orders (OrderService, OrderRepository...)
    // - Todos los beans de inventory (InventoryService, OrderEventsInventoryHandler...)
    // - Flyway ejecuta las 6 migraciones
    // - Testcontainers levanta Postgres + RabbitMQ
    // Solo para testear que OrderService crea una orden correctamente.
}
```

Con 4 módulos y 15 tests, el tiempo de setup se multiplica. Con 10 módulos y 100 tests, los builds se vuelven lentos sin razón — cada test carga cosas que no necesita.

### La Solución: `@ApplicationModuleTest`

Spring Modulith incluye una anotación que carga **solo el módulo que estás testeando** y sus dependencias declaradas:

```java
@ApplicationModuleTest  // en lugar de @SpringBootTest
class OrderServiceTests {
    // Spring carga SOLO los beans del módulo orders
    // catalog no se carga (pero podemos mockear CatalogApi)
    // inventory no se carga
    // Solo las migraciones de orders.* se aplican
}
```

---

## Los Tres Modos de Bootstrap

`@ApplicationModuleTest` tiene tres modos que controlan qué se carga:

```java
// STANDALONE (default): solo el módulo bajo test
@ApplicationModuleTest
// equivale a:
@ApplicationModuleTest(bootstrapMode = BootstrapMode.STANDALONE)

// DIRECT_DEPENDENCIES: el módulo + sus dependencias directas
@ApplicationModuleTest(bootstrapMode = BootstrapMode.DIRECT_DEPENDENCIES)

// ALL_DEPENDENCIES: el módulo + todas sus dependencias transitivas
@ApplicationModuleTest(bootstrapMode = BootstrapMode.ALL_DEPENDENCIES)
```

Para la mayoría de los casos, `STANDALONE` con mocks para las dependencias es la mejor opción — máximo aislamiento, máxima velocidad.

---

## Test 1: Módulo `catalog` en Aislamiento

El módulo `catalog` no depende de ningún otro módulo, así que `STANDALONE` carga todo lo que necesita sin mocks.

```java
// src/test/java/com/geovannycode/bookstore/catalog/web/ProductRestControllerTests.java
package com.geovannycode.bookstore.catalog.web;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.geovannycode.bookstore.TestcontainersConfiguration;
import com.geovannycode.bookstore.catalog.command.CreateProductCommand;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.context.annotation.Import;
import org.springframework.http.MediaType;
import org.springframework.modulith.test.ApplicationModuleTest;
import org.springframework.test.web.servlet.MockMvc;

import java.math.BigDecimal;

import static org.hamcrest.Matchers.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

/**
 * Test del módulo catalog en aislamiento total.
 *
 * Con @ApplicationModuleTest (STANDALONE), Spring carga:
 *   ✅ catalog.command.*  (ProductEntity, ProductCommandService, ProductRepository)
 *   ✅ catalog.query.*    (ProductView, ProductQueryService, ProductViewRepository)
 *   ✅ catalog.internal.* (CatalogEventHandler)
 *   ✅ catalog.web.*      (ProductRestController, CatalogExceptionHandler)
 *   ✅ common.*           (PagedResult — es OPEN, se incluye)
 *
 *   ❌ orders.*     (no se carga)
 *   ❌ inventory.*  (no se carga)
 *
 * Verifica en los logs que el ApplicationContext tiene muchos menos beans
 * que con @SpringBootTest.
 */
@ApplicationModuleTest
@Import(TestcontainersConfiguration.class)
@AutoConfigureMockMvc
class ProductRestControllerTests {

    @Autowired
    MockMvc mockMvc;

    @Autowired
    ObjectMapper objectMapper;

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
    void shouldReturn404WithProblemDetailForNonExistentProduct() throws Exception {
        mockMvc.perform(get("/api/catalog/products/NONEXISTENT"))
                .andExpect(status().isNotFound())
                // Verifica RFC 9457 Problem Details
                .andExpect(jsonPath("$.title", is("Producto no encontrado")))
                .andExpect(jsonPath("$.status", is(404)));
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
        var command = new CreateProductCommand(
                "P001", "Duplicado", "Ya existe P001",
                null, new BigDecimal("10.00"), "Testing"
        );

        mockMvc.perform(post("/api/catalog/products")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(objectMapper.writeValueAsString(command)))
                .andExpect(status().isConflict())
                .andExpect(jsonPath("$.title", is("Producto ya existe")));
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

### Ejecutar solo el módulo catalog

```bash
./mvnw test -Dtest="ProductRestControllerTests"
# o con Taskfile:
task test:catalog
```

Observa en los logs cuántos beans se cargan. Serán mucho menos que con `@SpringBootTest`.

---

## Test 2: Módulo `orders` en Aislamiento con Mocks y Eventos

El módulo `orders` depende de `catalog` via `CatalogApi`. Como usamos `STANDALONE`, `catalog` no se carga — necesitamos mockear `CatalogApi`.

Además, usaremos `AssertablePublishedEvents` para verificar que el módulo publica exactamente los eventos que debe.

```java
// src/test/java/.../orders/web/OrderRestControllerTests.java
package com.geovannycode.bookstore.orders.web;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.geovannycode.bookstore.TestcontainersConfiguration;
import com.geovannycode.bookstore.catalog.CatalogApi;
import com.geovannycode.bookstore.catalog.Product;
import com.geovannycode.bookstore.orders.OrderCreatedEvent;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.context.annotation.Import;
import org.springframework.http.MediaType;
import org.springframework.modulith.test.ApplicationModuleTest;
import org.springframework.modulith.test.AssertablePublishedEvents;
import org.springframework.test.context.bean.override.mockito.MockitoBean;
import org.springframework.test.web.servlet.MockMvc;

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
 * Puntos clave:
 *
 * 1. @MockitoBean CatalogApi:
 *    CatalogApi pertenece al módulo 'catalog' que no se carga en STANDALONE.
 *    Mockito provee un bean vacío que podemos configurar por test.
 *    Esto es exactamente lo que queremos: orders se testea sin necesitar
 *    la base de datos de catalog ni sus migraciones.
 *
 * 2. AssertablePublishedEvents:
 *    Se inyecta como parámetro del test (no como @Autowired).
 *    Captura todos los eventos publicados durante la ejecución del test.
 *    Permite verificar que orders cumplió su contrato de publicación.
 */
@ApplicationModuleTest
@Import(TestcontainersConfiguration.class)
@AutoConfigureMockMvc
class OrderRestControllerTests {

    @Autowired
    MockMvc mockMvc;

    @Autowired
    ObjectMapper objectMapper;

    /**
     * @MockitoBean reemplaza CatalogApi con un mock de Mockito.
     * Spring Modulith entiende que catalog no está disponible en este contexto
     * y acepta el bean mock como sustituto.
     */
    @MockitoBean
    CatalogApi catalogApi;

    @BeforeEach
    void setUp() {
        var product = new Product(
                "P001", "Clean Code", "Un manual de calidad",
                null, new BigDecimal("45.99"), "Ingeniería de Software",
                4.8, 235
        );
        given(catalogApi.getByCode("P001")).willReturn(Optional.of(product));
        given(catalogApi.getByCode("INEXISTENTE")).willReturn(Optional.empty());
    }

    /**
     * AssertablePublishedEvents se inyecta como parámetro del test.
     * Spring Modulith lo provee automáticamente — no necesitas nada extra.
     *
     * El parámetro captura los eventos publicados DURANTE la ejecución
     * de este método de test específico.
     */
    @Test
    void shouldCreateOrderSuccessfully(AssertablePublishedEvents events) throws Exception {
        var request = """
                {
                  "productCode": "P001",
                  "quantity": 2,
                  "customerName": "Geovanny Mendoza",
                  "customerEmail": "geo@barranquillajug.com",
                  "customerPhone": "+57 300 1234567",
                  "deliveryAddress": "Calle 72 #45-10, Barranquilla"
                }
                """;

        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(request))
                .andDo(print())
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.orderNumber", startsWith("ORD-")));

        // ── Verificación de eventos ───────────────────────────────────────
        //
        // AssertablePublishedEvents.ofType() devuelve solo los eventos del tipo
        // especificado. Podemos iterar sobre ellos con assertj.
        //
        // Esto valida que orders cumplió su CONTRATO DE PUBLICACIÓN:
        // cuando se crea una orden, debe publicar OrderCreatedEvent
        // con los datos correctos.
        //
        // Si alguien elimina el eventPublisher.publishEvent() de OrderService,
        // este test falla — esa es exactamente la regresión que queremos detectar.
        var orderEvents = events.ofType(OrderCreatedEvent.class);

        assertThat(orderEvents).isNotEmpty();
        assertThat(orderEvents)
                .anySatisfy(event -> {
                    assertThat(event.productCode()).isEqualTo("P001");
                    assertThat(event.quantity()).isEqualTo(2);
                    assertThat(event.customer().email()).isEqualTo("geo@barranquillajug.com");
                    assertThat(event.orderNumber()).startsWith("ORD-");
                });
    }

    @Test
    void shouldReturn400WhenProductNotFound() throws Exception {
        var request = """
                {
                  "productCode": "INEXISTENTE",
                  "quantity": 1,
                  "customerName": "Test User",
                  "customerEmail": "test@test.com",
                  "customerPhone": "+57 300 0000000",
                  "deliveryAddress": "Test Address"
                }
                """;

        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(request))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.title", is("Orden inválida")));
    }

    @Test
    void shouldReturn404ForNonExistentOrder() throws Exception {
        mockMvc.perform(get("/api/orders/ORD-NOTFOUND"))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.title", is("Orden no encontrada")));
    }

    @Test
    void shouldReturn400ForInvalidRequest() throws Exception {
        var invalidRequest = """
                {
                  "productCode": "",
                  "quantity": 0,
                  "customerName": "",
                  "customerEmail": "not-an-email",
                  "customerPhone": "",
                  "deliveryAddress": ""
                }
                """;

        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(invalidRequest))
                .andExpect(status().isBadRequest());
    }

    @Test
    void shouldNotPublishEventWhenOrderFails(AssertablePublishedEvents events) throws Exception {
        // Cuando el producto no existe, no debe publicarse ningún evento
        var request = """
                {
                  "productCode": "INEXISTENTE",
                  "quantity": 1,
                  "customerName": "Test",
                  "customerEmail": "test@test.com",
                  "customerPhone": "+57 300 0000000",
                  "deliveryAddress": "Test"
                }
                """;

        mockMvc.perform(post("/api/orders")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(request))
                .andExpect(status().isBadRequest());

        // Si la orden falla, NO debe haberse publicado el evento
        assertThat(events.ofType(OrderCreatedEvent.class)).isEmpty();
    }
}
```

!!! info "¿Por qué verificar que el evento NO se publica?"
    El test `shouldNotPublishEventWhenOrderFails` verifica que la transaccionalidad funciona correctamente. Si la orden no se guarda, el evento tampoco debe publicarse. Gracias al Event Publication Registry, esto está garantizado — pero tener el test lo hace explícito y documentado.

### Ejecutar solo el módulo orders

```bash
./mvnw test -Dtest="OrderRestControllerTests"
# o:
task test:orders
```

---

## Test 3: Módulo `inventory` con `Scenario`

El módulo `inventory` es **puramente reactivo** — no expone endpoints HTTP, solo escucha eventos. Testearlo con `MockMvc` no tiene sentido. Aquí es donde `Scenario` brilla.

`Scenario` es la herramienta de Spring Modulith para testear **flujos event-driven** sin necesitar el módulo publicador:

```java
scenario.publish(event)
        .andWaitForStateChange(() -> alguna_condición)
        .andVerify(resultado -> ...);
```

```java
// src/test/java/.../inventory/InventoryIntegrationTests.java
package com.geovannycode.bookstore.inventory;

import com.geovannycode.bookstore.TestcontainersConfiguration;
import com.geovannycode.bookstore.orders.OrderCreatedEvent;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Import;
import org.springframework.modulith.test.ApplicationModuleTest;
import org.springframework.modulith.test.Scenario;

import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Test del módulo inventory en aislamiento usando Scenario.
 *
 * Scenario permite testear event handlers sin levantar el módulo que
 * publica el evento. El módulo 'orders' no se carga en este test —
 * Scenario publica el evento directamente como si orders lo hubiera enviado.
 *
 * El patrón:
 *   1. scenario.publish(event) → simula que el evento fue publicado
 *   2. .andWaitForStateChange() → espera a que el handler asíncrono procese
 *   3. .andVerify() → verifica el estado final del sistema
 *
 * Esto testa exactamente lo que queremos: "dado este evento, ¿inventory
 * hace lo correcto?" — independientemente de cómo se generó el evento.
 */
@ApplicationModuleTest
@Import(TestcontainersConfiguration.class)
class InventoryIntegrationTests {

    @Autowired
    InventoryService inventoryService;

    @Test
    void shouldDecreaseStockWhenOrderCreatedEventReceived(Scenario scenario) {
        // Stock inicial del producto P001 (cargado por V6 de Flyway)
        int stockBefore = inventoryService.getStockLevel("P001");
        assertThat(stockBefore).isGreaterThan(0);

        var event = new OrderCreatedEvent(
                "ORD-TEST-001",
                "P001",
                "Clean Code",
                new BigDecimal("45.99"),
                3,  // quantity
                new OrderCreatedEvent.Customer(
                        "Test User", "test@test.com",
                        "+57 300 0000000", "Test Address"
                )
        );

        // Publicamos el evento como si 'orders' lo hubiera enviado.
        // Scenario se encarga de que el handler lo reciba.
        scenario.publish(event)
                // andWaitForStateChange espera hasta que la condición cambie.
                // Internamente usa polling — verifica periódicamente.
                // El resultado del Supplier es el nuevo estado.
                .andWaitForStateChange(
                        () -> inventoryService.getStockLevel("P001")
                )
                // andVerify recibe el resultado de andWaitForStateChange
                .andVerify(newStockLevel ->
                        assertThat(newStockLevel)
                                .isEqualTo(stockBefore - 3)
                );
    }

    @Test
    void shouldHandleMultipleUnitsInOrder(Scenario scenario) {
        int stockBefore = inventoryService.getStockLevel("P002");

        var event = new OrderCreatedEvent(
                "ORD-TEST-002",
                "P002",
                "The Pragmatic Programmer",
                new BigDecimal("49.99"),
                5,
                new OrderCreatedEvent.Customer(
                        "Bulk Buyer", "bulk@test.com",
                        "+57 311 0000000", "Warehouse"
                )
        );

        scenario.publish(event)
                .andWaitForStateChange(
                        () -> inventoryService.getStockLevel("P002")
                )
                .andVerify(newStock ->
                        assertThat(newStock).isEqualTo(stockBefore - 5)
                );
    }

    @Test
    void shouldGetCurrentStockLevel() {
        // Test directo (sin eventos) — verifica que V6 cargó los datos
        int stock = inventoryService.getStockLevel("P003");
        assertThat(stock).isEqualTo(299);
    }
}
```

### ¿Por qué `Scenario` y no un mock del eventPublisher?

Con un mock, verificarías que `eventPublisher.publishEvent()` fue llamado con ciertos parámetros — pero no verificarías que el **handler realmente procesó el evento y actualizó el estado**. `Scenario` va un nivel más abajo: publica el evento de verdad y espera el cambio de estado observable en la base de datos. Es un test de integración real del comportamiento del módulo.

```bash
./mvnw test -Dtest="InventoryIntegrationTests"
# o:
task test:inventory
```

---

## El Smoke Test de Arquitectura

```java
// src/test/java/com/geovannycode/bookstore/BookstoreApplicationTests.java
package com.geovannycode.bookstore;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.context.annotation.Import;

/**
 * Smoke test: verifica que el contexto completo carga sin errores.
 *
 * Si este test pasa, significa que:
 *   ✅ Flyway ejecutó todas las migraciones V1-V6 sin errores
 *   ✅ JPA validó el mapeo de entidades contra el schema real
 *   ✅ Spring Modulith inicializó el Event Publication Registry
 *   ✅ La conexión a Postgres y RabbitMQ está establecida
 *   ✅ Todos los beans se inyectan correctamente
 *
 * Lo ejecutamos como parte del build (./mvnw verify) para detectar
 * problemas de configuración antes de desplegar.
 */
@SpringBootTest
@Import(TestcontainersConfiguration.class)
class BookstoreApplicationTests {

    @Test
    void contextLoads() {
        // Si el contexto falla al cargar, el test falla con el error exacto
    }
}
```

---

## La Infraestructura de Tests

Los tests anteriores dependen de dos clases compartidas:

### `TestcontainersConfiguration`

```java
// src/test/java/com/geovannycode/bookstore/TestcontainersConfiguration.java
@TestConfiguration(proxyBeanMethods = false)
public class TestcontainersConfiguration {

    /**
     * @ServiceConnection en Spring Boot 3.1+ configura automáticamente
     * el datasource a partir del contenedor. Sin @DynamicPropertySource manual.
     */
    @Bean
    @ServiceConnection
    PostgreSQLContainer<?> postgresContainer() {
        return new PostgreSQLContainer<>(DockerImageName.parse("postgres:16-alpine"))
                .withDatabaseName("bookstore")
                .withUsername("bookstore")
                .withPassword("bookstore");
    }

    @Bean
    @ServiceConnection
    RabbitMQContainer rabbitMQContainer() {
        return new RabbitMQContainer(DockerImageName.parse("rabbitmq:3.13-alpine"));
    }
}
```

### `TestBookstoreApplication` — Dev sin Docker

```java
// src/test/java/com/geovannycode/bookstore/TestBookstoreApplication.java
public class TestBookstoreApplication {

    /**
     * Arranca la aplicación usando Testcontainers en lugar de Docker Compose.
     * Útil para desarrollo cuando no quieres gestionar contenedores manualmente.
     *
     * Ejecutar:  ./mvnw spring-boot:test-run
     * o en IntelliJ: click derecho → Run 'TestBookstoreApplication'
     */
    public static void main(String[] args) {
        SpringApplication
                .from(BookstoreApplication::main)
                .with(TestcontainersConfiguration.class)
                .run(args);
    }
}
```

---

## Ejecutar Todos los Tests

```bash
# Todos los tests
./mvnw test

# Solo los tests de módulos (sin el smoke test)
./mvnw test -Dtest="ProductRestControllerTests,OrderRestControllerTests,InventoryIntegrationTests,ModularityTest"

# Con Taskfile
task test
```

---

## Comparación: `@SpringBootTest` vs `@ApplicationModuleTest`

| Aspecto | `@SpringBootTest` | `@ApplicationModuleTest` (STANDALONE) |
|---|---|---|
| Beans cargados | Todos los módulos | Solo el módulo bajo test |
| Tiempo de startup | ~10-15s | ~3-5s |
| Migraciones Flyway | Todas (V1-V6) | Solo las relevantes |
| Dependencias externas | Todas (Postgres, RabbitMQ) | Solo las del módulo |
| Dependencias a otros módulos | Reales | Mockeadas |
| Feedback de cambios | Lento | Rápido |
| Para verificar todo el sistema | ✅ Ideal | ❌ No aplica |
| Para verificar un módulo | ❌ Excesivo | ✅ Ideal |

La estrategia recomendada:

```
Pirámide de tests en Spring Modulith:

                    ┌──────────────────┐
                    │   @SpringBootTest │  1-2 smoke tests
                    │  (contexto total) │
                    └──────────────────┘
               ┌──────────────────────────────┐
               │    @ApplicationModuleTest     │  5-10 tests por módulo
               │  (módulo en aislamiento)      │
               └──────────────────────────────┘
          ┌──────────────────────────────────────────┐
          │           Unit Tests                     │  Muchos, rápidos
          │   (lógica de dominio sin Spring)         │
          └──────────────────────────────────────────┘
```

---

## Resumen de las Herramientas de Test

| Herramienta | Para qué | Ejemplo |
|---|---|---|
| `@ApplicationModuleTest` | Carga un módulo en aislamiento | `@ApplicationModuleTest` en la clase |
| `@MockitoBean` | Reemplaza dependencias de otros módulos | `@MockitoBean CatalogApi catalogApi` |
| `AssertablePublishedEvents` | Verifica eventos publicados en el test | `events.ofType(OrderCreatedEvent.class)` |
| `Scenario` | Testea flujos event-driven | `scenario.publish(event).andWaitForStateChange(...)` |
| `ModularityTest` | Verifica arquitectura completa | `modules.verify()` |

---

## Checklist de la Parte 5

- [ ] `TestcontainersConfiguration` creado y compartido por todos los tests
- [ ] `ProductRestControllerTests` con `@ApplicationModuleTest` pasa ✅
- [ ] `OrderRestControllerTests` con `@MockitoBean CatalogApi` pasa ✅
- [ ] `OrderRestControllerTests` verifica eventos con `AssertablePublishedEvents` ✅
- [ ] `InventoryIntegrationTests` con `Scenario` pasa ✅
- [ ] `BookstoreApplicationTests` (smoke test) pasa ✅
- [ ] `ModularityTest` pasa ✅
- [ ] `./mvnw test` completa sin errores ✅

---

**Anterior**: [Parte 4 — Eventos y Outbox](parte_4.md) &nbsp;&nbsp;&nbsp;&nbsp; **Siguiente**: [Parte 6 — C4 Docs, Observabilidad y Docker](parte_6.md)
