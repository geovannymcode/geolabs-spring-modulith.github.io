# Guía Completa: Implementando CQRS con Spring Modulith desde Cero

## Tabla de Contenidos
1. [¿Por qué Spring Modulith? El Problema del Monolito](#por-qué-spring-modulith-el-problema-del-monolito)
2. [Entendiendo los Monolitos Modulares](#entendiendo-los-monolitos-modulares)
3. [Creando el Proyecto desde Cero](#creando-el-proyecto-desde-cero)
4. [Configuración de Spring Modulith](#configuración-de-spring-modulith)
5. [¿Por qué CQRS en una Tienda Online?](#por-qué-cqrs-en-una-tienda-online)
6. [Implementando el Primer Módulo](#implementando-el-primer-módulo)
7. [Verificación de la Estructura Modular](#verificación-de-la-estructura-modular)
8. [Primer Test: Verificando que Todo Funciona](#primer-test-verificando-que-todo-funciona)

## ¿Por qué Spring Modulith? El Problema del Monolito

### El Ciclo de Vida Típico de un Proyecto

Empezaste un proyecto nuevo. Al principio todo está limpio y organizado:

```
📁 mi-proyecto/
├── 📁 controllers/
├── 📁 services/
├── 📁 repositories/
└── 📁 models/
```

**Después de 6 meses:**
```java
// ProductController.java
@RestController
public class ProductController {
    
    @Autowired
    private ProductService productService;
    
    @Autowired 
    private UserRepository userRepository; // ¿Por qué está aquí?
    
    @Autowired
    private OrderService orderService; // Esto no debería estar
    
    @Autowired
    private EmailService emailService; // Tampoco esto
}
```

**¿Qué pasó?** Bajo presión de fechas de entrega, los desarrolladores empezaron a tomar atajos:

- "Solo necesito este dato, accedo directo al repositorio"
- "Es solo una línea, lo pongo aquí temporalmente"
- "Después refactorizamos esto"

### Los Tres Enfoques y Sus Problemas

#### 1. Monolito Tradicional

**Problemas:**

- 🔴 **Big Ball of Mud**: Todo conectado con todo
- 🔴 **Cambios riesgosos**: Modificar una parte rompe 10 lugares
- 🔴 **Difícil de entender**: Nuevos desarrolladores se pierden
- 🔴 **Testing complejo**: Necesitas cargar toda la aplicación

#### 2. Microservicios

**Problemas:**

- 🔴 **Complejidad distribuida**: Network latency, timeouts, circuit breakers
- 🔴 **Monitoring complejo**: Necesitas rastrear llamadas entre servicios
- 🔴 **Costos de infraestructura**: Múltiples bases de datos, servicios
- 🔴 **Testing difícil**: Necesitas levantar múltiples servicios

#### 3. Spring Modulith (La Solución Moderna)

**Beneficios:**

- ✅ **Modularidad sin distribución**: Módulos claros en un solo JAR
- ✅ **Reglas arquitectónicas automáticas**: El framework previene violaciones
- ✅ **Testing independiente**: Cada módulo se puede testear por separado
- ✅ **Evolución gradual**: Fácil migración a microservicios cuando sea necesario
- ✅ **Observabilidad**: Trazabilidad entre módulos como en microservicios

## Entendiendo los Monolitos Modulares

### ¿Qué es un Módulo en Spring Modulith?

Spring Modulith considera que cada **paquete directo** bajo tu clase principal es un **módulo independiente**.

```yaml
📁 com.geovannycode.store/         <- Paquete raíz
├── 📄 StoreApplication.java       <- Clase principal
├── 📁 products/                   <- MÓDULO: Products
├── 📁 orders/                     <- MÓDULO: Orders  
├── 📁 inventory/                  <- MÓDULO: Inventory
├── 📁 notifications/              <- MÓDULO: Notifications
└── 📁 common/                     <- MÓDULO: Shared
```

### Reglas de Acceso Entre Módulos

Spring Modulith implementa un conjunto de reglas para garantizar una correcta modularidad y encapsulación en aplicaciones monolíticas. Estas reglas son verificadas automáticamente en tiempo de compilación y en los tests.

#### Regla 1: Solo las clases públicas en la raíz del módulo son accesibles

Esta regla es fundamental para entender cómo Spring Modulith implementa la encapsulación a nivel de módulo:

1. **¿Qué significa "raíz del módulo"?**
      - La raíz del módulo es el paquete principal del módulo (por ejemplo, `com.geovannycode.store.products`)
      - No incluye los sub-paquetes (como `com.geovannycode.store.products.internal`)

2. **Visibilidad automática:**
      - Solo las clases **públicas** que están **directamente** en el paquete raíz son visibles para otros módulos
      - Las clases en sub-paquetes están automáticamente "protegidas", incluso si tienen el modificador `public`
      - Esto funciona como un "firewall" automático que evita dependencias incorrectas

3. **Beneficios prácticos:**
      - Fuerza a los desarrolladores a pensar en qué clases deben ser parte de la API pública
      - Evita el "acoplamiento accidental" donde otros módulos dependen de detalles de implementación
      - Permite refactorizar internamente el módulo sin romper otros módulos

### Ejemplo:

```java
📁 products/
├── 📄 ProductService.java         <- public class (en raíz) - ACCESIBLE
├── 📄 Product.java                <- public class (en raíz) - ACCESIBLE
└── 📁 internal/
    ├── 📄 ProductRepository.java  <- NO accesible desde otros módulos
    └── 📄 ProductValidator.java   <- NO accesible desde otros módulos
```

Spring Modulith verifica automáticamente que ningún otro módulo intente importar `ProductRepository` o `ProductValidator`, y fallaría los tests si lo hicieran.

#### Regla 2: Named Interfaces para exponer sub-paquetes

Cuando necesitas hacer una excepción a la Regla 1, puedes usar `@NamedInterface`:

1. **¿Cuándo usar Named Interfaces?**
      - Cuando necesitas exponer clases específicas de sub-paquetes a otros módulos
      - Casos comunes: eventos de dominio, DTOs, interfaces de servicio

2. **Cómo funciona:**
      - Creas un archivo `package-info.java` en el sub-paquete
      - Anotas el paquete con `@NamedInterface("nombre")`
      - Esto "expone" todas las clases públicas de ese sub-paquete a otros módulos

3. **Beneficio:**
      - Control explícito: debes declarar intencionalmente qué quieres exponer
      - Documentación automática: el nombre de la interfaz muestra el propósito de ese sub-paquete

### Ejemplo:

```java
// Si necesitas exponer clases de sub-paquetes:
📁 products/
├── 📁 events/                    <- Sub-paquete que contiene eventos
│   ├── 📄 package-info.java      <- Declaración de Named Interface
│   └── 📄 ProductCreated.java    <- Ahora ACCESIBLE por otros módulos

// package-info.java
@NamedInterface("events")
package com.geovannycode.store.products.events;

import org.springframework.modulith.NamedInterface;
```

Ahora, otros módulos pueden importar y usar `ProductCreated.java` porque está en un sub-paquete marcado como `@NamedInterface`.

## En la práctica

Estas reglas ayudan a mantener la modularidad porque:

1. Por defecto, todo está "escondido" excepto lo que pones explícitamente en la raíz
2. Cuando necesitas exponer algo de un sub-paquete, lo haces conscientemente
3. Spring Modulith verifica automáticamente estas reglas en los tests

Esto facilita saber exactamente qué está expuesto a otros módulos y qué no, lo que hace la aplicación más mantenible a largo plazo.

## Creando el Proyecto desde Cero

### Paso 1: Spring Initializr (Configuración Base)

Visita [https://start.spring.io](https://start.spring.io) y configura:

**Configuración del Proyecto:**

- **Project**: Maven Project
- **Language**: Java  
- **Spring Boot**: 3.5.5 (o la más reciente)
- **Group**: `com.geovannycode`
- **Artifact**: `store-cqrs`
- **Name**: `store-cqrs`
- **Package name**: `com.geovannycode.store`
- **Packaging**: Jar
- **Java**: 21

**Dependencias a seleccionar:**

- **Spring Web** - Para crear APIs REST
- **Spring Data JPA** - Para acceso a base de datos
- **PostgreSQL Driver** - Base de datos que usaremos
- **Spring Boot DevTools** - Herramientas de desarrollo
- **Lombok** - Reduce código repetitivo
- **✅ Spring Modulith** - ¡Disponible en Developer Tools!
- **Testcontainers** - Levantar Postgres (u otros) en Docker para pruebas de integración.

### Paso 2: Descargar y Abrir el Proyecto

1. Haz clic en "Generate" para descargar el ZIP
2. Extrae el archivo
3. Abre el proyecto en tu IDE favorito
4. Espera a que Maven descargue las dependencias

### Paso 3: Verificar que el Proyecto Arranca

```bash
# Desde la raíz del proyecto
./mvnw spring-boot:run

# Deberías ver en consola:
# Started StoreApplication in X.XXX seconds
```

Si ves errores de base de datos, no te preocupes. Los corregiremos en los siguientes pasos.

## Configuración de Spring Modulith

### Paso 1: Verificar Dependencias de Spring Modulith

Spring Initializr ya incluye las dependencias básicas de Spring Modulith. Verifica que tu `pom.xml` tenga:

```xml
<!-- Spring Modulith ya incluido desde Spring Initializr -->
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-core</artifactId>
</dependency>
```

Si necesitas funcionalidades adicionales, agrega estas dependencias:

```xml
<!-- Para eventos persistentes en base de datos (opcional) -->
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-jpa</artifactId>
</dependency>

<!-- Para testing avanzado de módulos (opcional) -->
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- Para usar PostgreSQL real en tests (opcional) -->
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
```

### Paso 2: Habilitar Spring Modulith en la Aplicación

Modifica tu clase principal `StoreApplication.java`:

```java
// src/main/java/com/geovannycode/store/StoreApplication.java
package com.geovannycode.store;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.modulith.Modulithic;

/**
 * Clase principal de la aplicación con Spring Modulith habilitado.
 * 
 * ¿Qué hace @Modulithic?
 * 1. Le dice a Spring que esta app usará módulos
 * 2. Cada paquete directo bajo 'com.geovannycode.store' será un módulo
 * 3. Habilita verificación automática de reglas arquitectónicas
 * 4. Permite comunicación entre módulos vía eventos
 */
@SpringBootApplication
@Modulithic  // 👈 Esta anotación habilita Spring Modulith
public class StoreApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(StoreApplication.class, args);
    }
}
```

### Paso 3: Configurar Base de Datos

Reemplaza el contenido de `application.properties` por `application.yml`:

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: store-cqrs
  
  # Configuración de PostgreSQL
  datasource:
    url: jdbc:postgresql://localhost:5432/store_db
    username: store_user
    password: store_password
    driver-class-name: org.postgresql.Driver
  
  # Configuración de JPA/Hibernate
  jpa:
    hibernate:
      # validate = usar Flyway para esquema, no Hibernate
      ddl-auto: validate
    show-sql: true        # Ver SQL en desarrollo
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
  
  # Configuración de Flyway para migraciones
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true

# Configuración específica de Spring Modulith
modulith:
  events:
    # ¿Qué hace? Guarda eventos procesados para auditoría
    # Alternativa: delete (elimina eventos procesados para ahorrar espacio)
    completion-mode: archive
    
    # ¿Para qué? Si la app se reinicia, re-procesa eventos pendientes
    # Garantiza que ningún evento se pierda
    republish-outstanding-events-on-restart: true

# Logging para desarrollo - ver qué hace Spring Modulith
logging:
  level:
    com.geovannycode.store: DEBUG
    org.springframework.modulith: DEBUG
```

### Paso 4: Docker Compose para PostgreSQL

Crea `docker-compose.yml` en la raíz del proyecto:

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Base de datos PostgreSQL
  postgres:
    image: postgres:15-alpine
    container_name: store-postgres
    environment:
      POSTGRES_DB: store_db
      POSTGRES_USER: store_user
      POSTGRES_PASSWORD: store_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U store_user -d store_db"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
    driver: local
```

### Paso 5: Iniciar PostgreSQL

```bash
# Iniciar PostgreSQL
docker-compose up -d

# Verificar que esté corriendo
docker-compose ps
# Deberías ver: store-postgres   Up (healthy)
```

## ¿Por qué CQRS en una Tienda Online?

### El Problema: Desbalance de Operaciones en E-commerce

En una tienda online típica, existe un desbalance significativo entre operaciones de lectura y escritura:

- **95% de operaciones**: Son lecturas (consultar productos, ver detalles, leer reseñas)
- **5% de operaciones**: Son escrituras (comprar, agregar reseñas, actualizar inventario)

Esta proporción crea un desafío técnico importante: **estamos optimizando nuestro sistema para ambos tipos de operaciones cuando tienen necesidades completamente diferentes**.

## El Enfoque Tradicional: Un Modelo Único

Sin CQRS, usamos el mismo modelo para todas las operaciones:

**Modelo único para todo**:
```java
// ❌ Un solo modelo Product para escritura Y lectura
@Entity
public class Product {
    private Long id;
    private String name;
    private BigDecimal price;
    private Integer stock;
    private List<Review> reviews;  // 1000+ reseñas por producto
    private List<Category> categories;
    private List<Image> images;
    private List<Variant> variants;
    // ... 20+ campos más
}
```

### Problemas Concretos:

1. **Sobrecarga de Datos**: Para mostrar un simple listado de productos, cargas TODA la información incluidas las relaciones (como 1000+ reseñas)
   ```java
   List<Product> products = productRepository.findAll(); // ¡Carga excesiva!
   ```
2. **Cálculos Repetitivos**: Para mostrar el rating promedio, realizas el cálculo cada vez que alguien ve el producto
   ```java
   double rating = product.getReviews().stream()
       .mapToInt(Review::getRating)
       .average()
       .orElse(0.0);
   ```
3. **Queries Complejas**: Necesitas JOINs constantes incluso para operaciones simples

4. **Bloqueos de Base de Datos**: Las escrituras pueden bloquear las lecturas, afectando la experiencia del 95% de los usuarios

## **La Solución CQRS: Modelos Especializados**

CQRS divide tu sistema en dos partes con modelos diferentes:

### Lado Command (Escritura) - 5% de operaciones

Optimizado para **consistencia e integridad**:

```java
@Entity
public class Product {
    private ProductId id;
    private String name;
    private BigDecimal price;
    private Integer stock;
    private List<Review> reviews;
    
    // Métodos de negocio
    public void addReview(Review review) {
        validateReview(review);
        this.reviews.add(review);
        // Publicar evento para actualizar vista
        publishEvent(new ProductReviewed(...));
    }
}
```

#### Lado Query (Lectura) - 95% de operaciones

Optimizado para **velocidad y simplicidad**:

```java
@Entity
public class ProductView {
    private ProductId id;
    private String name;
    private BigDecimal price;
    private Integer stock;
    
    // ¡Datos desnormalizados para rapidez!
    private Double averageRating;    // Ya calculado
    private Integer reviewCount;     // Ya calculado
    private String categoryName;     // Sin JOIN
    private String mainImageUrl;     // Sin JOIN
}
```

### Beneficios Específicos para E-commerce

#### 1. **Performance de Búsqueda Dramáticamente Mejorada**

Con CQRS:
```java
// ✅ Query súper rápida - datos ya listos
@Query("SELECT p FROM ProductView p WHERE p.categoryName = :category ORDER BY p.averageRating DESC")
List<ProductView> findByCategoryOrderByRating(String category);

Sin CQRS:
// ❌ Consulta lenta con múltiples JOINs y cálculos
@Query("SELECT p FROM Product p JOIN p.categories c JOIN p.reviews r WHERE c.name = :category GROUP BY p ORDER BY AVG(r.rating) DESC")
```

#### 2. **Escalabilidad Horizontal de Lecturas**

Puedes tener múltiples réplicas de base de datos dedicadas solo para lecturas:

```java
@Configuration
public class DatabaseConfig {
    @Bean("readOnlyDataSource")
    public DataSource readOnlyDataSource() {
        // Conexión a réplicas de solo lectura
    }
}
```

Esto significa que puedes escalar la capacidad de lectura independientemente de la capacidad de escritura.

#### 3. **Mejor Experiencia de Usuario con Actualizaciones Asíncronas**

Cuando un usuario agrega una reseña:

1. La reseña se guarda inmediatamente (Command)
2. El usuario recibe confirmación sin esperar
3. Los datos de lectura se actualizan asíncronamente (Query)

```java
@ApplicationModuleListener
void on(ProductReviewed event) {
    // Actualizar rating promedio en background
    updateProductViewRating(event.productId(), event.rating());
}
```

### CQRS vs. Caché: ¿Por qué no usar simplemente caché?

**Caché:**

```java
// ❌ Caché: Datos pueden estar desactualizados
@Cacheable("products")
public List<Product> getProducts() {
    return productRepository.findAll(); // Datos de hace 1 hora
}
```

**CQRS:**

```java
// ✅ CQRS: Datos específicamente optimizados para lectura
public List<ProductView> getProducts() {
    return productViewRepository.findAll(); // Modelo específicamente diseñado
}
```

**CQRS es superior porque**:

1. **Modelos Específicos**: Diseñados exactamente para cada caso de uso
2. **Actualización Controlada**: Sabes exactamente cuándo y cómo se actualizan los datos
3. **Sin Problemas de Invalidación**: No hay la complejidad de gestionar cuándo invalidar cachés
4. **Queries Inherentemente Simples**: Sin necesidad de JOINs complejos o transformaciones

## Implementando el Primer Módulo

### Estructura de Módulos para E-commerce

Vamos a crear la estructura básica que necesita una tienda online:

```
📁 src/main/java/com/geovannycode/store/
├── 📄 StoreApplication.java       <- Clase principal
├── 📁 products/                   <- Módulo de productos (empezamos aquí)
│   ├── 📁 command/               <- Operaciones de escritura
│   ├── 📁 query/                 <- Operaciones de lectura
│   └── 📁 events/                <- Eventos entre módulos
├── 📁 common/                     <- Utilidades compartidas
└── 📁 config/                     <- Configuración global
```

### Paso 1: Crear Estructura de Directorios

Crea estas carpetas en `src/main/java/com/geovannycode/store/`:

```bash
# Desde tu IDE o terminal
mkdir -p src/main/java/com/geovannycode/store/products/command
mkdir -p src/main/java/com/geovannycode/store/products/query  
mkdir -p src/main/java/com/geovannycode/store/products/events
mkdir -p src/main/java/com/geovannycode/store/common
mkdir -p src/main/java/com/geovannycode/store/config
```

### Paso 2: Crear Migración de Base de Datos

Crea el directorio y archivo de migración:

```bash
mkdir -p src/main/resources/db/migration
```

Crea `src/main/resources/db/migration/V1__create_initial_schema.sql`:

```sql
-- V1__create_initial_schema.sql
-- Esquema inicial para el módulo products con CQRS

-- =====================================================
-- LADO COMMAND: Modelos para escritura (normalizados)
-- =====================================================

-- Tabla principal de productos
CREATE TABLE products (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    category VARCHAR(100),
    price DECIMAL(10,2) NOT NULL CHECK (price >= 0),
    stock INTEGER NOT NULL CHECK (stock >= 0),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Tabla de reviews (relación 1:N con products)
CREATE TABLE reviews (
    id UUID PRIMARY KEY,
    product_id UUID NOT NULL REFERENCES products(id),
    vote INTEGER NOT NULL CHECK (vote >= 1 AND vote <= 5),
    comment TEXT,
    author VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Índices para el lado command
CREATE INDEX idx_reviews_product_id ON reviews(product_id);
CREATE INDEX idx_products_category ON products(category);

-- =====================================================
-- LADO QUERY: Modelos para lectura (desnormalizados)
-- =====================================================

-- Vista de productos optimizada para consultas
CREATE TABLE product_views (
    id UUID PRIMARY KEY,
    name VARCHAR(255),
    description TEXT,
    category VARCHAR(100),
    price DECIMAL(10,2),
    stock INTEGER,
    
    -- Campos desnormalizados para performance
    average_rating DECIMAL(3,2) DEFAULT 0.0,
    review_count INTEGER DEFAULT 0,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Índices optimizados para consultas comunes
CREATE INDEX idx_product_views_category ON product_views(category);
CREATE INDEX idx_product_views_rating ON product_views(average_rating DESC);
CREATE INDEX idx_product_views_price ON product_views(price);

-- =====================================================
-- DATOS DE EJEMPLO PARA TESTING
-- =====================================================

-- Insertar productos de ejemplo
INSERT INTO products (id, name, description, category, price, stock) VALUES
    ('f47ac10b-58cc-4372-a567-0e02b2c3d479', 'iPhone 15 Pro', 'Latest iPhone with titanium design', 'Electronics', 999.99, 50),
    ('6ba7b810-9dad-11d1-80b4-00c04fd430c8', 'MacBook Air M3', 'Powerful laptop with M3 chip', 'Electronics', 1199.99, 25),
    ('6ba7b811-9dad-11d1-80b4-00c04fd430c8', 'AirPods Pro', 'Wireless earbuds with noise cancellation', 'Electronics', 249.99, 100);

-- Insertar reviews de ejemplo
INSERT INTO reviews (id, product_id, vote, comment, author) VALUES
    ('r1', 'f47ac10b-58cc-4372-a567-0e02b2c3d479', 5, 'Amazing phone, love the titanium!', 'John Doe'),
    ('r2', 'f47ac10b-58cc-4372-a567-0e02b2c3d479', 4, 'Great camera quality', 'Jane Smith'),
    ('r3', '6ba7b810-9dad-11d1-80b4-00c04fd430c8', 5, 'Best laptop I have ever used', 'Tech Reviewer');

-- Copiar datos a la vista de lectura
INSERT INTO product_views (id, name, description, category, price, stock, average_rating, review_count)
SELECT 
    p.id, 
    p.name, 
    p.description, 
    p.category, 
    p.price, 
    p.stock,
    COALESCE(r.avg_rating, 0.0) as average_rating,
    COALESCE(r.review_count, 0) as review_count
FROM products p
LEFT JOIN (
    SELECT 
        product_id,
        ROUND(AVG(vote::numeric), 2) as avg_rating,
        COUNT(*) as review_count
    FROM reviews 
    GROUP BY product_id
) r ON p.id = r.product_id;
```

### Paso 3: Crear Clase de Utilidades Comunes

Crea `src/main/java/com/geovannycode/store/common/PagedResult.java`:

```java
// src/main/java/com/geovannycode/store/common/PagedResult.java
package com.geovannycode.store.common;

import java.util.List;

/**
 * Resultado paginado genérico que puede usar cualquier módulo.
 * 
 * ¿Por qué en common? Porque múltiples módulos necesitarán paginación
 * ¿Por qué no en cada módulo? Para evitar duplicación de código
 */
public record PagedResult<T>(
    List<T> content,
    int page,
    int size,
    long totalElements,
    int totalPages,
    boolean hasNext,
    boolean hasPrevious
) {
    /**
     * Crear resultado paginado con cálculos automáticos.
     */
    public static <T> PagedResult<T> of(List<T> content, int page, int size, long totalElements) {
        int totalPages = (int) Math.ceil((double) totalElements / size);
        boolean hasNext = page < totalPages - 1;
        boolean hasPrevious = page > 0;
        
        return new PagedResult<>(content, page, size, totalElements, totalPages, hasNext, hasPrevious);
    }
}
```

## Verificación de la Estructura Modular

### ¿Por qué Verificar Antes de Implementar?

Antes de escribir código de negocio, necesitamos asegurar que Spring Modulith reconoce correctamente nuestra estructura de módulos.

### Crear Test de Verificación

Crea `src/test/java/com/geovannycode/store/ModularityTest.java`:

```java
// src/test/java/com/geovannycode/store/ModularityTest.java
package com.geovannycode.store;

import org.junit.jupiter.api.Test;
import org.springframework.modulith.core.ApplicationModules;
import org.springframework.modulith.docs.Documenter;

/**
 * Test CRÍTICO que verifica la estructura modular.
 * 
 * ¿Para qué sirve?
 * 1. Valida que Spring Modulith reconoce nuestros módulos
 * 2. Detecta violaciones de encapsulamiento
 * 3. Previene dependencias circulares
 * 4. Genera documentación automática
 * 
 * Este test debe ejecutarse ANTES de implementar funcionalidad.
 */
class ModularityTest {
    
    // Spring Modulith analiza la aplicación y encuentra módulos
    ApplicationModules modules = ApplicationModules.of(StoreCqrsApplication.class);
    
    /**
     * Test principal - verifica todas las reglas modulares.
     * 
     * ¿Qué verifica?
     * - Cada paquete bajo com.geovannycode.store es un módulo
     * - No hay violaciones de acceso entre módulos
     * - No hay dependencias circulares
     * - Las APIs públicas están bien definidas
     */
    @Test
    void shouldHaveValidModularStructure() {
        // Si hay violaciones, este método lanza excepción con detalles
        modules.verify();
    }
    
    /**
     * Genera documentación automática de la arquitectura.
     * 
     * ¿Qué genera?
     * - Diagramas PlantUML de los módulos
     * - Documentación AsciiDoc
     * - Module Canvas (diagramas C4)
     * 
     * Los archivos se generan en: target/spring-modulith-docs/
     */
    @Test
    void shouldGenerateDocumentation() throws Exception {
        new Documenter(modules)
            .writeDocumentation()           // AsciiDoc
            .writeModuleCanvases()          // Diagramas C4
            .writeIndividualModulesAsPlantUml(); // PlantUML
    }
    
    /**
     * Test informativo - muestra qué módulos encontró Spring Modulith.
     */
    @Test
    void showModuleStructure() {
        System.out.println("\n🏗️ Estructura de módulos detectada:");
        modules.forEach(module -> {
            System.out.println("📦 Módulo: " + module.getName());
            System.out.println("   📁 Paquete: " + module.getBasePackage());
            System.out.println("   🔗 Dependencias: " + module.getDependencies().size());
            System.out.println();
        });
    }
}
```

## Primer Test: Verificando que Todo Funciona

### Paso 1: Ejecutar Test de Modularidad

```bash
# Ejecutar solo el test de modularidad
./mvnw test -Dtest=ModularityTest

# Si todo está bien, verás:
# ✅ Tests run: 3, Failures: 0, Errors: 0
```

**Salida esperada:**
```
🏗️ Estructura de módulos detectada:
📦 Módulo: common
   📁 Paquete: com.geovannycode.store.common
   🔗 Dependencias: 0

📦 Módulo: config  
   📁 Paquete: com.geovannycode.store.config
   🔗 Dependencias: 0

📦 Módulo: products
   📁 Paquete: com.geovannycode.store.products
   🔗 Dependencias: 1
```

### Paso 2: Verificar Documentación Generada

Después de ejecutar los tests, verifica que se generó documentación:

```bash
# Verificar archivos generados
ls -la target/spring-modulith-docs/

# Deberías ver archivos como:
# - modules.adoc
# - module-products.adoc
# - module-common.adoc
# - modules.puml
```

### Paso 3: Ejecutar la Aplicación

```bash
# Iniciar la aplicación
./mvnw spring-boot:run

# Deberías ver en consola:
# Started StoreApplication in X.XXX seconds (JVM running for Y.YYY)
```

**¿Qué pasa si hay errores?**

**Error común 1: PostgreSQL no conecta**
```bash
# Solución: Verificar Docker
docker-compose ps
# Si no está running: docker-compose up -d
```

**Error común 2: Flyway falla**
```bash
# Solución: Limpiar y reiniciar BD
docker-compose down -v
docker-compose up -d
./mvnw spring-boot:run
```

### Paso 4: Verificar Endpoints Básicos

Abre otra terminal y verifica que la aplicación responde:

```bash
# Verificar que la app está viva
curl http://localhost:8080/actuator/health

# Respuesta esperada:
# {"status":"UP"}
```

### ¿Qué Hemos Logrado Hasta Ahora?

✅ **Proyecto base funcional** con Spring Boot + Spring Modulith  
✅ **Estructura modular** reconocida automáticamente  
✅ **Base de datos PostgreSQL** configurada y corriendo  
✅ **Esquema de BD** creado con datos de ejemplo  
✅ **Tests de arquitectura** que verifican reglas modulares  
✅ **Documentación automática** generada  

### Próximos Pasos

En la **Parte 2** implementaremos:

- Lado Command con entidades y servicios
- Lado Query con modelos optimizados  
- Eventos entre módulos
- Controllers REST funcionales
- Testing independiente de módulos

¡Tu proyecto ya está listo para implementar CQRS con Spring Modulith!