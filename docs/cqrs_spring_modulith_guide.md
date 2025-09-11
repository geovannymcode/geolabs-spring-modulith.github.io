# Guía Completa: Implementando CQRS con Spring Modulith desde Cero

## Tabla de Contenidos
1. [¿Por qué Spring Modulith? El Problema de los Monolitos](#por-qué-spring-modulith-el-problema-de-los-monolitos)
2. [Entendiendo los Módulos Monolíticos](#entendiendo-los-módulos-monolíticos)
3. [Configuración del Entorno](#configuración-del-entorno)
4. [Creando la Estructura Base del Proyecto](#creando-la-estructura-base-del-proyecto)
5. [Configuración de Dependencias Maven](#configuración-de-dependencias-maven)
6. [Implementando el Primer Módulo](#implementando-el-primer-módulo)
7. [Verificación de la Estructura Modular](#verificación-de-la-estructura-modular)
8. [Implementación de CQRS](#implementación-de-cqrs)
9. [Comunicación Entre Módulos con Eventos](#comunicación-entre-módulos-con-eventos)
10. [Testing Independiente de Módulos](#testing-independiente-de-módulos)
11. [Documentación Automática](#documentación-automática)
12. [Observabilidad y Trazabilidad](#observabilidad-y-trazabilidad)
13. [Despliegue con Docker](#despliegue-con-docker)

## ¿Por qué Spring Modulith? El Problema de los Monolitos

### El Ciclo de Vida Típico de un Proyecto

Imagina que empiezas un proyecto nuevo. Al principio todo está limpio y organizado:

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

#### 3. Spring Modulith (La Solución Intermedia)

**Beneficios:**

- ✅ **Modularidad sin distribución**: Módulos claros en un solo JAR
- ✅ **Reglas arquitectónicas automáticas**: El framework previene violaciones
- ✅ **Testing independiente**: Cada módulo se puede testear por separado
- ✅ **Evolución gradual**: Fácil migración a microservicios cuando sea necesario
- ✅ **Observabilidad**: Trazabilidad entre módulos como en microservicios

## Entendiendo los Módulos Monolíticos

### ¿Qué es un Módulo en Spring Modulith?

Spring Modulith considera que cada **paquete directo** bajo tu clase principal es un **módulo independiente**.

```
📁 com.ejemplo.tienda/          <- Paquete raíz
├── 📄 TiendaApplication.java   <- Clase principal
├── 📁 productos/               <- MÓDULO: Productos
├── 📁 pedidos/                 <- MÓDULO: Pedidos  
├── 📁 inventario/              <- MÓDULO: Inventario
├── 📁 notificaciones/          <- MÓDULO: Notificaciones
└── 📁 comun/                   <- MÓDULO: Compartido
```

### Reglas de Acceso Entre Módulos

#### Regla 1: Solo las clases públicas en la raíz del módulo son accesibles

```java
// ✅ CORRECTO: Accesible desde otros módulos
📁 productos/
├── 📄 ProductoService.java     <- public class (en raíz)
├── 📄 Producto.java           <- public class (en raíz)
└── 📁 internal/
    ├── 📄 ProductoRepository.java  <- NO accesible
    └── 📄 ProductoValidator.java   <- NO accesible
```

#### Regla 2: Named Interfaces para exponer sub-paquetes

```java
// Si necesitas exponer clases de sub-paquetes:
📁 productos/
├── 📁 events/
│   ├── 📄 package-info.java
│   └── 📄 ProductoCreado.java
└── 📁 api/
    ├── 📄 package-info.java
    └── 📄 ProductoDTO.java

// package-info.java
@NamedInterface("eventos")
package com.ejemplo.tienda.productos.events;

import org.springframework.modulith.NamedInterface;
```

#### Regla 3: Módulos Abiertos para casos especiales

```java
// Para módulos compartidos (como utilidades)
📁 comun/
└── 📄 package-info.java

// package-info.java
@ApplicationModule(type = ApplicationModule.Type.OPEN)
package com.ejemplo.tienda.comun;

import org.springframework.modulith.ApplicationModule;
```

## Configuración del Entorno

### Prerequisitos y Herramientas

1. **Java 21 o superior** - Spring Modulith requiere versiones modernas
2. **Maven 3.8+** - Para gestión de dependencias
3. **IDE moderno** - IntelliJ IDEA, Eclipse, VS Code
4. **Docker Desktop** - Para PostgreSQL y servicios auxiliares
5. **Postman/cURL** - Para probar APIs

### Verificación del Entorno

```bash
# Verificar Java
java -version
# Debería mostrar: openjdk version "17.0.x" o superior

# Verificar Maven
mvn -version
# Debería mostrar: Apache Maven 3.8.x o superior

# Verificar Docker
docker --version
docker-compose --version
```

## Creando la Estructura Base del Proyecto

### Paso 1: Crear Proyecto con Spring Initializr

Visita [https://start.spring.io](https://start.spring.io) y configura:

- **Project**: Maven
- **Language**: Java  
- **Spring Boot**: 3.2.5 (o la más reciente)
- **Group**: `com.ejemplo`
- **Artifact**: `tienda-cqrs`
- **Name**: `tienda-cqrs`
- **Package name**: `com.ejemplo.tienda`
- **Packaging**: Jar
- **Java**: 17

**Dependencias iniciales a agregar:**

- Spring Web
- Spring Data JPA
- PostgreSQL Driver
- Spring Boot DevTools
- Lombok

### Paso 2: Estructura de Directorios Recomendada

```
tienda-cqrs/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/ejemplo/tienda/
│   │   │       ├── TiendaApplication.java
│   │   │       ├── productos/          # Módulo Productos
│   │   │       │   ├── command/        # Operaciones de escritura
│   │   │       │   ├── query/          # Operaciones de lectura
│   │   │       │   └── events/         # Eventos del módulo
│   │   │       ├── pedidos/            # Módulo Pedidos
│   │   │       ├── inventario/         # Módulo Inventario
│   │   │       ├── notificaciones/     # Módulo Notificaciones
│   │   │       └── comun/              # Código compartido
│   │   └── resources/
│   │       ├── application.yml
│   │       └── db/migration/           # Scripts Flyway
│   └── test/
├── docker-compose.yml
└── pom.xml
```

## Configuración de Dependencias Maven

### Archivo pom.xml Explicado Línea por Línea

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <!-- Heredamos configuración de Spring Boot -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.5</version>
        <relativePath/>
    </parent>
    
    <!-- Información del proyecto -->
    <groupId>com.ejemplo</groupId>
    <artifactId>tienda-cqrs</artifactId>
    <version>1.0.0</version>
    <name>Tienda CQRS con Spring Modulith</name>
    
    <!-- Versiones de librerías -->
    <properties>
        <java.version>17</java.version>
        <!-- 
        Spring Modulith versión estable que incluye:
        - Verificación de reglas arquitectónicas
        - Testing independiente de módulos  
        - Generación automática de documentación
        -->
        <spring-modulith.version>1.1.4</spring-modulith.version>
        <!-- 
        jMolecules para modelado DDD más claro:
        - Annotations para Aggregates, Entities, Value Objects
        - CQRS interfaces (Command, Query, EventHandler)
        - Generación automática de metadatos JPA
        -->
        <jmolecules.version>1.9.0</jmolecules.version>
    </properties>
    
    <dependencies>
        <!-- === DEPENDENCIAS CORE DE SPRING BOOT === -->
        
        <!-- 
        Spring Web: Para crear APIs REST
        Incluye: Tomcat embebido, Jackson para JSON, Spring MVC
        -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <!-- 
        Spring Data JPA: Para acceso a base de datos
        Incluye: Hibernate, Spring Data repositories, transacciones
        -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        
        <!-- 
        Validation: Para validar requests HTTP
        Incluye: Bean Validation (JSR-303), Hibernate Validator
        -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        
        <!-- === DEPENDENCIAS DE SPRING MODULITH === -->
        
        <!-- 
        Core de Spring Modulith: Funcionalidad básica
        - Verificación de reglas de módulos
        - Definición de módulos con @Modulithic
        - Named interfaces para exposición controlada
        -->
        <dependency>
            <groupId>org.springframework.modulith</groupId>
            <artifactId>spring-modulith-starter-core</artifactId>
            <version>${spring-modulith.version}</version>
        </dependency>
        
        <!-- 
        API de Eventos: Para comunicación entre módulos
        - ApplicationModuleListener para manejo asíncrono
        - Event publication y persistencia
        - Retry automático de eventos fallidos
        -->
        <dependency>
            <groupId>org.springframework.modulith</groupId>
            <artifactId>spring-modulith-events-api</artifactId>
            <version>${spring-modulith.version}</version>
        </dependency>
        
        <!-- 
        Persistencia de Eventos: Almacena eventos en BD
        - Tabla automática para eventos
        - Republishing de eventos al reiniciar
        - Audit trail de todos los eventos de dominio
        -->
        <dependency>
            <groupId>org.springframework.modulith</groupId>
            <artifactId>spring-modulith-starter-jpa</artifactId>
            <version>${spring-modulith.version}</version>
        </dependency>
        
        <!-- 
        Actuator para Modulith: Endpoints de monitoreo
        - /actuator/modulith: Vista de módulos y dependencias
        - Métricas de eventos procesados
        - Estado de la arquitectura modular
        -->
        <dependency>
            <groupId>org.springframework.modulith</groupId>
            <artifactId>spring-modulith-actuator</artifactId>
            <version>${spring-modulith.version}</version>
        </dependency>
        
        <!-- === DEPENDENCIAS DE jMOLECULES === -->
        
        <!-- 
        DDD Annotations: Para modelado de dominio
        - @AggregateRoot, @Entity, @ValueObject
        - @Repository, @Service annotations semánticas
        - Mejora la expresividad del código
        -->
        <dependency>
            <groupId>org.jmolecules</groupId>
            <artifactId>jmolecules-ddd</artifactId>
            <version>${jmolecules.version}</version>
        </dependency>
        
        <!-- 
        CQRS Architecture: Interfaces para CQRS
        - @CommandHandler, @QueryHandler
        - @EventHandler para eventos de dominio
        - Command, Query interfaces
        -->
        <dependency>
            <groupId>org.jmolecules</groupId>
            <artifactId>jmolecules-cqrs-architecture</artifactId>
            <version>${jmolecules.version}</version>
        </dependency>
        
        <!-- 
        Integración con Spring: Conecta jMolecules con Spring
        - Generación automática de metadatos JPA
        - Mapeo de annotations DDD a Spring annotations
        -->
        <dependency>
            <groupId>org.jmolecules.integrations</groupId>
            <artifactId>jmolecules-spring</artifactId>
            <version>${jmolecules.version}</version>
        </dependency>
        
        <!-- === BASE DE DATOS === -->
        
        <!-- 
        PostgreSQL Driver: Conexión a PostgreSQL
        ¿Por qué PostgreSQL y no H2?
        - Mejor para producción
        - Soporte completo de tipos de datos
        - Transacciones ACID confiables
        -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        
        <!-- 
        Flyway: Migraciones de base de datos
        - Versionado de esquemas
        - Migrations reproducibles
        - Control de cambios en BD
        -->
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>
        
        <!-- === HERRAMIENTAS DE DESARROLLO === -->
        
        <!-- 
        Lombok: Reduce boilerplate code
        - @Data, @Getter, @Setter
        - @AllArgsConstructor, @NoArgsConstructor
        - @Slf4j para logging
        -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <!-- === TESTING === -->
        
        <!-- 
        Spring Boot Test: Testing completo
        - @SpringBootTest, @WebMvcTest
        - TestContainers integration
        - MockMvc para testing de APIs
        -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        
        <!-- 
        Modulith Test: Testing específico de módulos
        - @ApplicationModuleTest para test independientes
        - PublishedEvents para verificar eventos
        - Carga solo el módulo bajo test
        -->
        <dependency>
            <groupId>org.springframework.modulith</groupId>
            <artifactId>spring-modulith-starter-test</artifactId>
            <version>${spring-modulith.version}</version>
            <scope>test</scope>
        </dependency>
        
        <!-- 
        TestContainers PostgreSQL: BD real en tests
        - PostgreSQL real para integration tests
        - Aislamiento total entre tests
        - Más confiable que mocks de BD
        -->
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>postgresql</artifactId>
            <scope>test</scope>
        </dependency>
        
        <!-- 
        TestContainers JUnit: Integración con JUnit 5
        - @Container annotations
        - Lifecycle management automático
        -->
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    
    <build>
        <plugins>
            <!-- Plugin estándar de Spring Boot -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
            
            <!-- 
            ByteBuddy Plugin: Para jMolecules
            Genera metadatos JPA automáticamente basado en annotations DDD
            Transforma @AggregateRoot en @Entity, etc.
            -->
            <plugin>
                <groupId>net.bytebuddy</groupId>
                <artifactId>byte-buddy-maven-plugin</artifactId>
                <version>1.14.12</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>transform</goal>
                        </goals>
                    </execution>
                </executions>
                <dependencies>
                    <dependency>
                        <groupId>org.jmolecules.integrations</groupId>
                        <artifactId>jmolecules-bytebuddy</artifactId>
                        <version>${jmolecules.version}</version>
                    </dependency>
                </dependencies>
            </plugin>
        </plugins>
    </build>
</project>
```

### ¿Por qué Estas Dependencias Específicas?

#### Spring Modulith vs Alternativas

| Característica | Spring Modulith | ArchUnit Solo | Microservicios |
|----------------|-----------------|---------------|----------------|
| **Verificación arquitectónica** | ✅ Automática | ✅ Manual | ❌ Complejo |
| **Testing independiente** | ✅ Built-in | ❌ No | ✅ Nativo |
| **Complejidad infraestructura** | ✅ Mínima | ✅ Mínima | ❌ Alta |
| **Documentación automática** | ✅ Si | ❌ No | ❌ Manual |
| **Event sourcing** | ✅ Incluido | ❌ No | ✅ Con herramientas |
| **Observabilidad** | ✅ Built-in | ❌ No | ✅ Con APM tools |

#### jMolecules vs Plain Spring

```java
// Sin jMolecules (tradicional)
@Entity
@Table(name = "productos")
public class Producto {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    // ... muchas annotations JPA
}

// Con jMolecules (expresivo)
@AggregateRoot  // Se convierte automáticamente en @Entity
public class Producto {
    private ProductoId id;  // Se mapea automáticamente como @Id
    // Código más limpio y expresivo
}
```

## Implementando el Primer Módulo

### Paso 1: Clase Principal con Modulith

```java
// src/main/java/com/ejemplo/tienda/TiendaApplication.java
package com.ejemplo.tienda;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.modulith.Modulithic;

/**
 * Clase principal de la aplicación.
 * 
 * @Modulithic habilita Spring Modulith y establece que:
 * - Cada paquete directo bajo 'com.ejemplo.tienda' es un módulo
 * - Se aplican reglas de encapsulación automáticamente
 * - Se habilita event publishing entre módulos
 */
@SpringBootApplication
@Modulithic
public class TiendaApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(TiendaApplication.class, args);
    }
}
```

### Paso 2: Configuración de Base de Datos

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: tienda-cqrs
  
  # Configuración de PostgreSQL
  datasource:
    url: jdbc:postgresql://localhost:5432/tienda_db
    username: tienda_user
    password: tienda_password
    driver-class-name: org.postgresql.Driver
  
  # Configuración de JPA/Hibernate
  jpa:
    # IMPORTANTE: validate en producción, create-drop solo en desarrollo
    hibernate:
      ddl-auto: validate  # Flyway maneja el esquema
    show-sql: true        # Solo en desarrollo
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
        # Optimizaciones para producción
        jdbc:
          batch_size: 20
        order_inserts: true
        order_updates: true
        jdbc:
          batch_versioned_data: true
  
  # Configuración de Flyway para migraciones
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    # Validar que las migrations no hayan cambiado
    validate-on-migrate: true

# Configuración específica de Spring Modulith
modulith:
  events:
    # archive: Guarda eventos procesados para auditoría
    # delete: Elimina eventos procesados (ahorra espacio)
    completion-mode: archive
    
    # Republica eventos no procesados al reiniciar
    # Crucial para garantizar consistencia eventual
    republish-outstanding-events-on-restart: true
    
    # Timeout para procesamiento de eventos (opcional)
    # republish-incomplete-events-after: PT1M

# Configuración de Actuator para monitoreo
management:
  endpoints:
    web:
      exposure:
        include: health,info,modulith,metrics
  endpoint:
    health:
      show-details: always
    modulith:
      enabled: true

# Logging para desarrollo
logging:
  level:
    com.ejemplo.tienda: DEBUG
    org.springframework.modulith: DEBUG
    # Solo en desarrollo - muestra SQL queries
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

### Paso 3: Docker Compose para PostgreSQL

```yaml
# docker-compose.yml
version: '3.8'

services:
  # Base de datos principal
  postgres:
    image: postgres:15-alpine
    container_name: tienda-postgres
    environment:
      POSTGRES_DB: tienda_db
      POSTGRES_USER: tienda_user
      POSTGRES_PASSWORD: tienda_password
      # Configuraciones de rendimiento
      POSTGRES_SHARED_PRELOAD_LIBRARIES: pg_stat_statements
    ports:
      - "5432:5432"
    volumes:
      # Persistencia de datos
      - postgres_data:/var/lib/postgresql/data
      # Scripts de inicialización si los necesitas
      - ./init-scripts:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U tienda_user -d tienda_db"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - tienda-network

  # Herramienta de administración de BD (opcional)
  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: tienda-pgadmin
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@tienda.com
      PGADMIN_DEFAULT_PASSWORD: admin123
    ports:
      - "5050:80"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - tienda-network

volumes:
  postgres_data:
    driver: local

networks:
  tienda-network:
    driver: bridge
```

### Paso 4: Primer Migration de Flyway

```sql
-- src/main/resources/db/migration/V1__crear_esquema_inicial.sql

-- =====================================================
-- MIGRATION V1: Esquema inicial para módulo productos
-- =====================================================

-- Tabla para productos (lado comando)
CREATE TABLE IF NOT EXISTS productos (
    id BIGSERIAL PRIMARY KEY,
    nombre VARCHAR(255) NOT NULL,
    descripcion TEXT,
    precio DECIMAL(10,2) NOT NULL CHECK (precio > 0),
    stock INTEGER NOT NULL CHECK (stock >= 0),
    categoria VARCHAR(100) NOT NULL,
    activo BOOLEAN NOT NULL DEFAULT true,
    creado_en TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    actualizado_en TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Tabla para reviews de productos
CREATE TABLE IF NOT EXISTS producto_reviews (
    id BIGSERIAL PRIMARY KEY,
    producto_id BIGINT NOT NULL REFERENCES productos(id) ON DELETE CASCADE,
    calificacion INTEGER NOT NULL CHECK (calificacion >= 1 AND calificacion <= 5),
    comentario TEXT,
    autor_email VARCHAR(255),
    creado_en TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Vista materializada para consultas (lado query)
CREATE TABLE IF NOT EXISTS productos_vista (
    id BIGINT PRIMARY KEY,
    nombre VARCHAR(255) NOT NULL,
    descripcion TEXT,
    precio DECIMAL(10,2) NOT NULL,
    stock INTEGER NOT NULL,
    categoria VARCHAR(100) NOT NULL,
    activo BOOLEAN NOT NULL DEFAULT true,
    
    -- Datos desnormalizados para consultas eficientes
    calificacion_promedio DECIMAL(3,2) DEFAULT 0.0,
    total_reviews INTEGER DEFAULT 0,
    
    -- Metadatos
    creado_en TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    actualizado_en TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- =====================================================
-- ÍNDICES PARA OPTIMIZACIÓN DE CONSULTAS
-- =====================================================

-- Índices para productos
CREATE INDEX idx_productos_categoria ON productos(categoria);
CREATE INDEX idx_productos_activo ON productos(activo);
CREATE INDEX idx_productos_precio ON productos(precio);
CREATE INDEX idx_productos_actualizado_en ON productos(actualizado_en);

-- Índices para reviews
CREATE INDEX idx_reviews_producto_id ON producto_reviews(producto_id);
CREATE INDEX idx_reviews_calificacion ON producto_reviews(calificacion);

-- Índices para vista de productos (optimizado para lecturas)
CREATE INDEX idx_productos_vista_categoria ON productos_vista(categoria);
CREATE INDEX idx_productos_vista_calificacion ON productos_vista(calificacion_promedio DESC);
CREATE INDEX idx_productos_vista_precio ON productos_vista(precio);
CREATE INDEX idx_productos_vista_activo ON productos_vista(activo);

-- Índice compuesto para consultas complejas
CREATE INDEX idx_productos_vista_categoria_activo_precio 
ON productos_vista(categoria, activo, precio);

-- =====================================================
-- FUNCIÓN PARA ACTUALIZAR TIMESTAMP AUTOMÁTICAMENTE
-- =====================================================

CREATE OR REPLACE FUNCTION actualizar_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.actualizado_en = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Trigger para productos
CREATE TRIGGER trigger_productos_actualizado_en
    BEFORE UPDATE ON productos
    FOR EACH ROW
    EXECUTE FUNCTION actualizar_timestamp();

-- Trigger para productos_vista
CREATE TRIGGER trigger_productos_vista_actualizado_en
    BEFORE UPDATE ON productos_vista
    FOR EACH ROW
    EXECUTE FUNCTION actualizar_timestamp();

-- =====================================================
-- DATOS DE EJEMPLO PARA DESARROLLO
-- =====================================================

-- Insertar categorías base
INSERT INTO productos (nombre, descripcion, precio, stock, categoria) VALUES
('Laptop Gaming', 'Laptop para gaming de alta gama', 1299.99, 10, 'Electronics'),
('Mouse Inalámbrico', 'Mouse ergonómico inalámbrico', 29.99, 50, 'Electronics'),
('Camiseta Básica', 'Camiseta de algodón 100%', 19.99, 100, 'Clothing'),
('Smartphone Pro', 'Smartphone con cámara profesional', 899.99, 25, 'Electronics'),
('Jeans Clásicos', 'Jeans de mezclilla clásicos', 59.99, 75, 'Clothing');

-- Copiar datos iniciales a la vista
INSERT INTO productos_vista (id, nombre, descripcion, precio, stock, categoria, activo, creado_en)
SELECT id, nombre, descripcion, precio, stock, categoria, activo, creado_en
FROM productos;

-- Reviews de ejemplo
INSERT INTO producto_reviews (producto_id, calificacion, comentario, autor_email) VALUES
(1, 5, 'Excelente laptop, muy rápida', 'usuario1@email.com'),
(1, 4, 'Buena relación calidad-precio', 'usuario2@email.com'),
(2, 5, 'Mouse muy cómodo', 'usuario3@email.com'),
(3, 3, 'Calidad promedio', 'usuario4@email.com'),
(4, 5, 'Mejor smartphone que he tenido', 'usuario5@email.com');

-- Actualizar estadísticas en la vista
UPDATE productos_vista 
SET 
    calificacion_promedio = (
        SELECT ROUND(AVG(calificacion::numeric), 2)
        FROM producto_reviews 
        WHERE producto_id = productos_vista.id
    ),
    total_reviews = (
        SELECT COUNT(*)
        FROM producto_reviews 
        WHERE producto_id = productos_vista.id
    )
WHERE id IN (SELECT DISTINCT producto_id FROM producto_reviews);
```

## Verificación de la Estructura Modular

### ¿Por qué Necesitamos Verificar la Estructura?

Antes de implementar funcionalidad, necesitamos asegurar que nuestra estructura modular es correcta. Spring Modulith proporciona herramientas para esto.

### Test de Verificación Modular

```java
// src/test/java/com/ejemplo/tienda/ModularityTest.java
package com.ejemplo.tienda;

import org.junit.jupiter.api.Test;
import org.springframework.modulith.core.ApplicationModules;
import org.springframework.modulith.docs.Documenter;

/**
 * Test que verifica que la estructura modular sea correcta.
 * 
 * Este test es CRÍTICO porque:
 * 1. Valida que no hay violaciones de encapsulación
 * 2. Detecta dependencias circulares entre módulos
 * 3. Verifica que las reglas de acceso se cumplan
 * 4. Genera documentación automáticamente
 */
class ModularityTest {
    
    /**
     * ApplicationModules representa la estructura modular de la aplicación.
     * Escanea desde TiendaApplication.class y considera cada paquete directo
     * como un módulo independiente.
     */
    ApplicationModules modules = ApplicationModules.of(TiendaApplication.class);
    
    /**
     * Test principal que verifica todas las reglas modulares.
     * 
     * ¿Qué verifica?
     * - No hay acceso a clases internas de otros módulos
     * - No hay dependencias circulares
     * - Los named interfaces están correctamente expuestos
     * - Los módulos abiertos permiten acceso completo
     */
    @Test
    void shouldHaveValidModularStructure() {
        // Este método lanza excepción si hay violaciones
        modules.verify();
    }
    
    /**
     * Genera documentación automática de la estructura modular.
     * 
     * Crea en target/spring-modulith-docs/:
     * - Diagramas C4 de cada módulo
     * - Documentación AsciiDoc
     * - Diagramas PlantUML de dependencias
     */
    @Test
    void shouldGenerateDocumentation() throws Exception {
        new Documenter(modules)
            .writeDocumentation()           // AsciiDoc
            .writeModuleCanvases()          // Diagramas C4
            .writeIndividualModulesAsPlantUml(); // PlantUML
    }
    
    /**
     * Test que muestra información detallada de los módulos.
     * Útil para debugging de la estructura.
     */
    @Test
    void shouldPrintModuleInformation() {
        System.out.println("=== INFORMACIÓN DE MÓDULOS ===");
        
        modules.forEach(module -> {
            System.out.printf("Módulo: %s%n", module.getName());
            System.out.printf("  Package: %s%n", module.getBasePackage());
            System.out.printf("  Named Interfaces: %s%n", module.getNamedInterfaces());
            System.out.printf("  Dependencias: %s%n", 
                module.getDependencies(modules).stream()
                    .map(dep -> dep.getName())
                    .toList());
            System.out.println();
        });
    }
}
```

### Ejecutar Verificación

```bash
# Ejecutar solo el test de modularidad
mvn test -Dtest=ModularityTest

# Si todo está bien, debería ver:
# [INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0
```

### Ejemplo de Violación y Cómo Solucionarla

```java
// ❌ PROBLEMA: Acceso directo a internals de otro módulo
package com.ejemplo.tienda.pedidos;

import com.ejemplo.tienda.productos.internal.ProductoRepository; // VIOLACIÓN!

@Service
public class PedidoService {
    
    private final ProductoRepository productoRepository; // NO permitido
    
    // Error: Accediendo a clase interna de módulo productos
}
```

**Error en el test:**
```
Module 'pedidos' depends on non-exposed type 'ProductoRepository' within module 'productos'
```

**✅ SOLUCIÓN: Usar API pública del módulo**

```java
// Correcto: Usar API pública
package com.ejemplo.tienda.pedidos;

import com.ejemplo.tienda.productos.ProductoService; // ✅ API pública

@Service
public class PedidoService {
    
    private final ProductoService productoService; // ✅ Correcto
    
    public void validarProducto(Long productoId) {
        // Usar método público del servicio
        var producto = productoService.buscarPorId(productoId);
        // ...
    }
}
```

## Implementación de CQRS

### ¿Qué es CQRS y Por Qué Usarlo?

**Command Query Responsibility Segregation (CQRS)** separa las operaciones de escritura (Commands) de las de lectura (Queries).

#### Beneficios en Nuestro Contexto:

1. **Modelos Optimizados**: 
   - Commands: Modelo normalizado para integridad
   - Queries: Modelo desnormalizado para performance

2. **Escalabilidad Independiente**:
   - Diferentes estrategias de caché para lecturas
   - Réplicas de BD para queries

3. **Claridad en el Código**:
   - Separación clara de responsabilidades
   - Testing más sencillo

### Estructura del Módulo Productos

```
📁 productos/
├── 📁 command/                 # Operaciones de escritura
│   ├── 📄 Producto.java        # Aggregate Root
│   ├── 📄 ProductoRepository.java
│   ├── 📄 ProductoCommandService.java
│   └── 📄 ProductoCommandController.java
├── 📁 query/                   # Operaciones de lectura  
│   ├── 📄 ProductoVista.java   # Read Model
│   ├── 📄 ProductoVistaRepository.java
│   ├── 📄 ProductoQueryService.java
│   └── 📄 ProductoQueryController.java
├── 📁 events/                  # Eventos de dominio
│   ├── 📄 package-info.java
│   ├── 📄 ProductoCreado.java
│   ├── 📄 ProductoActualizado.java
│   └── 📄 ProductoReviewAgregado.java
└── 📁 shared/                  # DTOs y Value Objects
    ├── 📄 ProductoId.java
    ├── 📄 CreateProductoRequest.java
    └── 📄 ProductoResponse.java
```

### Implementación del Command Side

#### 1. Value Objects y Identificadores

```java
// src/main/java/com/ejemplo/tienda/productos/shared/ProductoId.java
package com.ejemplo.tienda.productos.shared;

import org.jmolecules.ddd.types.Identifier;

/**
 * Value Object que representa el identificador único de un producto.
 * 
 * ¿Por qué usar un Value Object en lugar de Long?
 * 1. Type Safety: No puedes confundir ProductoId con PedidoId
 * 2. Expresividad: El código es más claro sobre qué representa
 * 3. Validación: Podemos agregar validaciones específicas
 * 4. Evolución: Fácil cambiar la implementación interna
 */
public record ProductoId(Long value) implements Identifier {
    
    public ProductoId {
        if (value != null && value <= 0) {
            throw new IllegalArgumentException("ProductoId debe ser positivo");
        }
    }
    
    public static ProductoId of(Long value) {
        return new ProductoId(value);
    }
    
    public static ProductoId generate() {
        return new ProductoId(null); // Se generará en la BD
    }
    
    @Override
    public String toString() {
        return value != null ? value.toString() : "NEW";
    }
}
```

```java
// src/main/java/com/ejemplo/tienda/productos/shared/CreateProductoRequest.java
package com.ejemplo.tienda.productos.shared;

import jakarta.validation.constraints.*;
import java.math.BigDecimal;

/**
 * DTO para crear nuevos productos.
 * 
 * ¿Por qué un DTO separado?
 * 1. Validación específica para creación
 * 2. No expone detalles internos del modelo
 * 3. Puede evolucionar independientemente
 * 4. Facilita versionado de API
 */
public record CreateProductoRequest(
    
    @NotBlank(message = "El nombre es obligatorio")
    @Size(min = 2, max = 255, message = "El nombre debe tener entre 2 y 255 caracteres")
    String nombre,
    
    @Size(max = 1000, message = "La descripción no puede exceder 1000 caracteres")
    String descripcion,
    
    @NotNull(message = "El precio es obligatorio")
    @DecimalMin(value = "0.01", message = "El precio debe ser mayor a 0")
    @Digits(integer = 8, fraction = 2, message = "Formato de precio inválido")
    BigDecimal precio,
    
    @NotNull(message = "El stock es obligatorio")
    @Min(value = 0, message = "El stock no puede ser negativo")
    @Max(value = 999999, message = "El stock no puede exceder 999,999 unidades")
    Integer stock,
    
    @NotBlank(message = "La categoría es obligatoria")
    @Pattern(regexp = "^[A-Za-z0-9\\s-_]+$", message = "Categoría contiene caracteres inválidos")
    String categoria
) {
    
    /**
     * Factory method para crear desde datos básicos.
     * Útil en tests y otras situaciones.
     */
    public static CreateProductoRequest of(String nombre, BigDecimal precio, Integer stock, String categoria) {
        return new CreateProductoRequest(nombre, null, precio, stock, categoria);
    }
}
```

#### 2. Aggregate Root

```java
// src/main/java/com/ejemplo/tienda/productos/command/Producto.java
package com.ejemplo.tienda.productos.command;

import com.ejemplo.tienda.productos.shared.ProductoId;
import com.ejemplo.tienda.productos.events.*;
import org.jmolecules.ddd.types.AggregateRoot;
import org.springframework.data.domain.AbstractAggregateRoot;
import lombok.Getter;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;
import java.time.LocalDateTime;

/**
 * Aggregate Root del dominio Producto.
 * 
 * Responsabilidades:
 * 1. Mantener consistencia de datos del producto
 * 2. Aplicar reglas de negocio
 * 3. Publicar eventos de dominio
 * 4. Controlar acceso a entidades relacionadas
 * 
 * ¿Por qué extender AbstractAggregateRoot?
 * - Permite publicar eventos de dominio automáticamente
 * - Integración transparente con Spring Data
 * - Manejo del ciclo de vida de eventos
 */
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED) // Para JPA
public class Producto extends AbstractAggregateRoot<Producto> 
    implements AggregateRoot<Producto, ProductoId> {
    
    private ProductoId id;
    private String nombre;
    private String descripcion;
    private BigDecimal precio;
    private Integer stock;
    private String categoria;
    private Boolean activo;
    private LocalDateTime creadoEn;
    private LocalDateTime actualizadoEn;
    
    /**
     * Constructor para crear nuevo producto.
     * Aplica reglas de negocio y publica evento.
     */
    public Producto(String nombre, String descripcion, BigDecimal precio, 
                   Integer stock, String categoria) {
        
        // Validaciones de negocio
        validarNombre(nombre);
        validarPrecio(precio);
        validarStock(stock);
        validarCategoria(categoria);
        
        // Asignar valores
        this.id = ProductoId.generate();
        this.nombre = nombre.trim();
        this.descripcion = descripcion != null ? descripcion.trim() : null;
        this.precio = precio;
        this.stock = stock;
        this.categoria = categoria.trim();
        this.activo = true;
        this.creadoEn = LocalDateTime.now();
        this.actualizadoEn = LocalDateTime.now();
        
        // Publicar evento de dominio
        registerEvent(new ProductoCreado(
            this.id,
            this.nombre,
            this.descripcion,
            this.precio,
            this.stock,
            this.categoria
        ));
    }
    
    /**
     * Actualiza la información del producto.
     * Aplica reglas de negocio y publica evento si hay cambios.
     */
    public void actualizar(String nombre, String descripcion, BigDecimal precio, 
                          Integer stock, String categoria) {
        
        // Validaciones
        validarProductoActivo();
        validarNombre(nombre);
        validarPrecio(precio);
        validarStock(stock);
        validarCategoria(categoria);
        
        // Verificar si hay cambios reales
        boolean huboActualizacion = false;
        
        if (!this.nombre.equals(nombre.trim())) {
            this.nombre = nombre.trim();
            huboActualizacion = true;
        }
        
        String nuevaDescripcion = descripcion != null ? descripcion.trim() : null;
        if (!Objects.equals(this.descripcion, nuevaDescripcion)) {
            this.descripcion = nuevaDescripcion;
            huboActualizacion = true;
        }
        
        if (this.precio.compareTo(precio) != 0) {
            this.precio = precio;
            huboActualizacion = true;
        }
        
        if (!this.stock.equals(stock)) {
            this.stock = stock;
            huboActualizacion = true;
        }
        
        if (!this.categoria.equals(categoria.trim())) {
            this.categoria = categoria.trim();
            huboActualizacion = true;
        }
        
        if (huboActualizacion) {
            this.actualizadoEn = LocalDateTime.now();
            
            registerEvent(new ProductoActualizado(
                this.id,
                this.nombre,
                this.descripcion,
                this.precio,
                this.stock,
                this.categoria
            ));
        }
    }
    
    /**
     * Agrega una review al producto y publica evento.
     */
    public void agregarReview(Integer calificacion, String comentario, String autorEmail) {
        validarProductoActivo();
        validarCalificacion(calificacion);
        
        registerEvent(new ProductoReviewAgregado(
            this.id,
            calificacion,
            comentario,
            autorEmail
        ));
    }
    
    /**
     * Desactiva el producto (soft delete).
     */
    public void desactivar() {
        if (!this.activo) {
            throw new IllegalStateException("El producto ya está desactivado");
        }
        
        this.activo = false;
        this.actualizadoEn = LocalDateTime.now();
        
        registerEvent(new ProductoDesactivado(this.id));
    }
    
    // =====================================================
    // MÉTODOS DE VALIDACIÓN PRIVADOS
    // =====================================================
    
    private void validarProductoActivo() {
        if (!this.activo) {
            throw new IllegalStateException("No se puede modificar un producto desactivado");
        }
    }
    
    private void validarNombre(String nombre) {
        if (nombre == null || nombre.trim().isEmpty()) {
            throw new IllegalArgumentException("El nombre del producto es obligatorio");
        }
        if (nombre.trim().length() > 255) {
            throw new IllegalArgumentException("El nombre no puede exceder 255 caracteres");
        }
    }
    
    private void validarPrecio(BigDecimal precio) {
        if (precio == null) {
            throw new IllegalArgumentException("El precio es obligatorio");
        }
        if (precio.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("El precio debe ser mayor a 0");
        }
        if (precio.compareTo(new BigDecimal("999999.99")) > 0) {
            throw new IllegalArgumentException("El precio no puede exceder 999,999.99");
        }
    }
    
    private void validarStock(Integer stock) {
        if (stock == null) {
            throw new IllegalArgumentException("El stock es obligatorio");
        }
        if (stock < 0) {
            throw new IllegalArgumentException("El stock no puede ser negativo");
        }
        if (stock > 999999) {
            throw new IllegalArgumentException("El stock no puede exceder 999,999 unidades");
        }
    }
    
    private void validarCategoria(String categoria) {
        if (categoria == null || categoria.trim().isEmpty()) {
            throw new IllegalArgumentException("La categoría es obligatoria");
        }
        if (categoria.trim().length() > 100) {
            throw new IllegalArgumentException("La categoría no puede exceder 100 caracteres");
        }
    }
    
    private void validarCalificacion(Integer calificacion) {
        if (calificacion == null || calificacion < 1 || calificacion > 5) {
            throw new IllegalArgumentException("La calificación debe estar entre 1 y 5");
        }
    }
}
```

### Continuación de CQRS - Command Side

#### 3. Repository Pattern

```java
// src/main/java/com/ejemplo/tienda/productos/command/ProductoRepository.java
package com.ejemplo.tienda.productos.command;

import com.ejemplo.tienda.productos.shared.ProductoId;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;
import java.util.Optional;

/**
 * Repository para operaciones de escritura en productos.
 * 
 * ¿Por qué interface package-private?
 * - Solo ProductoCommandService debería usar este repository
 * - Otros módulos deben usar ProductoService (API pública)
 * - Encapsulación de la capa de persistencia
 * 
 * Nota: Spring Modulith verificará que este repository
 * no sea accedido desde otros módulos.
 */
interface ProductoRepository extends JpaRepository<Producto, ProductoId> {
    
    /**
     * Busca productos activos por categoría.
     * Útil para validaciones en el command side.
     */
    @Query("SELECT p FROM Producto p WHERE p.categoria = :categoria AND p.activo = true")
    List<Producto> findActivosPorCategoria(@Param("categoria") String categoria);
    
    /**
     * Verifica si existe un producto activo con el nombre dado.
     * Útil para validar nombres únicos.
     */
    @Query("SELECT COUNT(p) > 0 FROM Producto p WHERE p.nombre = :nombre AND p.activo = true")
    boolean existeProductoActivoConNombre(@Param("nombre") String nombre);
    
    /**
     * Busca producto activo por ID.
     * Más específico que findById para operaciones de comando.
     */
    @Query("SELECT p FROM Producto p WHERE p.id = :id AND p.activo = true")
    Optional<Producto> findActivoPorId(@Param("id") ProductoId id);
}
```

#### 4. Service Layer (Command)

```java
// src/main/java/com/ejemplo/tienda/productos/command/ProductoCommandService.java
package com.ejemplo.tienda.productos.command;

import com.ejemplo.tienda.productos.shared.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * Servicio para operaciones de escritura en productos.
 * 
 * Responsabilidades:
 * 1. Coordinar operaciones de comando
 * 2. Aplicar validaciones de negocio transversales
 * 3. Manejar transacciones
 * 4. Logging de operaciones críticas
 * 
 * ¿Por qué package-private?
 * - Solo el controller del mismo módulo debería acceder
 * - La API pública será ProductoService (en el root del módulo)
 */
@Slf4j
@Service
@RequiredArgsConstructor
@Transactional // Todas las operaciones son transaccionales
class ProductoCommandService {
    
    private final ProductoRepository repository;
    
    /**
     * Crea un nuevo producto.
     * 
     * @param request Datos del producto a crear
     * @return ID del producto creado
     * @throws IllegalArgumentException Si los datos son inválidos
     * @throws IllegalStateException Si ya existe un producto con el mismo nombre
     */
    public ProductoId crearProducto(CreateProductoRequest request) {
        log.info("Creando producto: {}", request.nombre());
        
        // Validación de negocio: nombre único
        if (repository.existeProductoActivoConNombre(request.nombre())) {
            throw new IllegalStateException(
                "Ya existe un producto activo con el nombre: " + request.nombre()
            );
        }
        
        // Crear aggregate y aplicar reglas de dominio
        Producto producto = new Producto(
            request.nombre(),
            request.descripcion(),
            request.precio(),
            request.stock(),
            request.categoria()
        );
        
        // Persistir (esto también publicará los eventos de dominio)
        Producto guardado = repository.save(producto);
        
        log.info("Producto creado exitosamente con ID: {}", guardado.getId());
        return guardado.getId();
    }
    
    /**
     * Actualiza un producto existente.
     * 
     * @param productoId ID del producto a actualizar
     * @param request Nuevos datos del producto
     * @throws ProductoNoEncontradoException Si el producto no existe
     * @throws IllegalStateException Si el producto está desactivado
     */
    public void actualizarProducto(ProductoId productoId, CreateProductoRequest request) {
        log.info("Actualizando producto: {}", productoId);
        
        Producto producto = repository.findActivoPorId(productoId)
            .orElseThrow(() -> new ProductoNoEncontradoException(productoId));
        
        // Validar nombre único (excluyendo el producto actual)
        if (!producto.getNombre().equals(request.nombre()) &&
            repository.existeProductoActivoConNombre(request.nombre())) {
            throw new IllegalStateException(
                "Ya existe otro producto activo con el nombre: " + request.nombre()
            );
        }
        
        // Aplicar cambios (eventos se publican automáticamente)
        producto.actualizar(
            request.nombre(),
            request.descripcion(),
            request.precio(),
            request.stock(),
            request.categoria()
        );
        
        repository.save(producto);
        log.info("Producto actualizado exitosamente: {}", productoId);
    }
    
    /**
     * Agrega una review a un producto.
     * 
     * @param productoId ID del producto
     * @param calificacion Calificación de 1 a 5
     * @param comentario Comentario opcional
     * @param autorEmail Email del autor de la review
     */
    public void agregarReview(ProductoId productoId, Integer calificacion, 
                             String comentario, String autorEmail) {
        log.info("Agregando review al producto: {} por {}", productoId, autorEmail);
        
        Producto producto = repository.findActivoPorId(productoId)
            .orElseThrow(() -> new ProductoNoEncontradoException(productoId));
        
        producto.agregarReview(calificacion, comentario, autorEmail);
        repository.save(producto);
        
        log.info("Review agregada exitosamente al producto: {}", productoId);
    }
    
    /**
     * Desactiva un producto (soft delete).
     * 
     * @param productoId ID del producto a desactivar
     */
    public void desactivarProducto(ProductoId productoId) {
        log.info("Desactivando producto: {}", productoId);
        
        Producto producto = repository.findActivoPorId(productoId)
            .orElseThrow(() -> new ProductoNoEncontradoException(productoId));
        
        producto.desactivar();
        repository.save(producto);
        
        log.info("Producto desactivado exitosamente: {}", productoId);
    }
}
```

#### 5. Exception Handling

```java
// src/main/java/com/ejemplo/tienda/productos/command/ProductoNoEncontradoException.java
package com.ejemplo.tienda.productos.command;

import com.ejemplo.tienda.productos.shared.ProductoId;

/**
 * Excepción específica de dominio para productos no encontrados.
 * 
 * ¿Por qué una excepción específica?
 * 1. Más expresiva que RuntimeException genérica
 * 2. Permite manejo específico en exception handlers
 * 3. Facilita testing y debugging
 * 4. Mejor logging y monitoreo
 */
public class ProductoNoEncontradoException extends RuntimeException {
    
    private final ProductoId productoId;
    
    public ProductoNoEncontradoException(ProductoId productoId) {
        super("Producto no encontrado con ID: " + productoId);
        this.productoId = productoId;
    }
    
    public ProductoId getProductoId() {
        return productoId;
    }
}
```

## Comunicación Entre Módulos con Eventos

### ¿Por Qué Eventos en Lugar de Llamadas Directas?

#### Problema con Llamadas Directas:
```java
// ❌ ACOPLAMIENTO FUERTE
@Service
public class ProductoCommandService {
    
    private final InventarioService inventarioService;     // Dependencia directa
    private final NotificacionService notificacionService; // Otra dependencia
    private final AnalyticsService analyticsService;       // Y otra más...
    
    public void crearProducto(CreateProductoRequest request) {
        // Lógica principal
        Producto producto = new Producto(...);
        repository.save(producto);
        
        // Efectos secundarios - TODOS SÍNCRONOS
        inventarioService.inicializarStock(producto.getId(), request.stock());
        notificacionService.notificarNuevoProducto(producto);
        analyticsService.registrarCreacionProducto(producto);
        
        // ¿Qué pasa si uno falla? ¿Todo se rollback?
        // ¿Y si agregamos más efectos secundarios?
    }
}
```

#### Solución con Eventos:
```java
// ✅ BAJO ACOPLAMIENTO
@Service
public class ProductoCommandService {
    
    // Solo dependemos del repository
    private final ProductoRepository repository;
    
    public void crearProducto(CreateProductoRequest request) {
        // Lógica principal
        Producto producto = new Producto(...); // Publica evento automáticamente
        repository.save(producto); // ¡Evento se publica aquí!
        
        // Los efectos secundarios se manejan asíncronamente
        // por los event handlers correspondientes
    }
}
```

### Definición de Eventos de Dominio

```java
// src/main/java/com/ejemplo/tienda/productos/events/package-info.java
/**
 * Eventos de dominio del módulo productos.
 * 
 * Estos eventos representan hechos importantes que ocurrieron
 * en el dominio y que pueden interesar a otros módulos.
 * 
 * @NamedInterface permite que otros módulos accedan a estos eventos
 * para suscribirse como listeners.
 */
@NamedInterface("eventos")
package com.ejemplo.tienda.productos.events;

import org.springframework.modulith.NamedInterface;
```

```java
// src/main/java/com/ejemplo/tienda/productos/events/ProductoCreado.java
package com.ejemplo.tienda.productos.events;

import com.ejemplo.tienda.productos.shared.ProductoId;

import java.math.BigDecimal;
import java.time.LocalDateTime;

/**
 * Evento que indica que un producto fue creado exitosamente.
 * 
 * ¿Por qué un record?
 * 1. Inmutable por defecto
 * 2. Equals/hashCode automáticos
 * 3. Serialización sencilla
 * 4. Menos boilerplate
 * 
 * ¿Qué información incluir?
 * - Solo lo que los consumers necesitan saber
 * - No detalles internos de implementación
 * - Suficiente info para tomar decisiones de negocio
 */
public record ProductoCreado(
    ProductoId productoId,
    String nombre,
    String descripcion,
    BigDecimal precio,
    Integer stock,
    String categoria,
    LocalDateTime ocurridoEn
) {
    
    /**
     * Constructor factory que asigna timestamp automáticamente.
     */
    public ProductoCreado(ProductoId productoId, String nombre, String descripcion,
                         BigDecimal precio, Integer stock, String categoria) {
        this(productoId, nombre, descripcion, precio, stock, categoria, LocalDateTime.now());
    }
    
    /**
     * Información básica para logging.
     */
    @Override
    public String toString() {
        return String.format("ProductoCreado{id=%s, nombre='%s', categoria='%s'}", 
                           productoId, nombre, categoria);
    }
}
```

```java
// src/main/java/com/ejemplo/tienda/productos/events/ProductoActualizado.java
package com.ejemplo.tienda.productos.events;

import com.ejemplo.tienda.productos.shared.ProductoId;

import java.math.BigDecimal;
import java.time.LocalDateTime;

/**
 * Evento que indica que un producto fue actualizado.
 */
public record ProductoActualizado(
    ProductoId productoId,
    String nombre,
    String descripcion,
    BigDecimal precio,
    Integer stock,
    String categoria,
    LocalDateTime ocurridoEn
) {
    
    public ProductoActualizado(ProductoId productoId, String nombre, String descripcion,
                              BigDecimal precio, Integer stock, String categoria) {
        this(productoId, nombre, descripcion, precio, stock, categoria, LocalDateTime.now());
    }
}
```

```java
// src/main/java/com/ejemplo/tienda/productos/events/ProductoReviewAgregado.java
package com.ejemplo.tienda.productos.events;

import com.ejemplo.tienda.productos.shared.ProductoId;

import java.time.LocalDateTime;

/**
 * Evento que indica que se agregó una review a un producto.
 */
public record ProductoReviewAgregado(
    ProductoId productoId,
    Integer calificacion,
    String comentario,
    String autorEmail,
    LocalDateTime ocurridoEn
) {
    
    public ProductoReviewAgregado(ProductoId productoId, Integer calificacion, 
                                 String comentario, String autorEmail) {
        this(productoId, calificacion, comentario, autorEmail, LocalDateTime.now());
    }
}
```

### Query Side - Manejo de Eventos

#### 1. Read Model Optimizado

```java
// src/main/java/com/ejemplo/tienda/productos/query/ProductoVista.java
package com.ejemplo.tienda.productos.query;

import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;
import lombok.extern.slf4j.Slf4j;
import org.jmolecules.cqrs.QueryModel;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDateTime;

/**
 * Read Model optimizado para consultas de productos.
 * 
 * Características:
 * 1. Datos desnormalizados para consultas eficientes
 * 2. Campos calculados (promedio rating, total reviews)
 * 3. Optimizado para patrones de lectura específicos
 * 4. Sin lógica de negocio compleja
 * 
 * @QueryModel indica que es un modelo de lectura (jMolecules)
 */
@Slf4j
@Getter
@Setter
@NoArgsConstructor
@QueryModel
public class ProductoVista {
    
    // Campos básicos (replicados del command side)
    private Long id;
    private String nombre;
    private String descripcion;
    private BigDecimal precio;
    private Integer stock;
    private String categoria;
    private Boolean activo;
    
    // Campos desnormalizados para consultas eficientes
    private BigDecimal calificacionPromedio = BigDecimal.ZERO;
    private Integer totalReviews = 0;
    
    // Metadatos útiles para consultas
    private LocalDateTime creadoEn;
    private LocalDateTime actualizadoEn;
    
    /**
     * Constructor para crear vista inicial desde evento ProductoCreado.
     */
    public ProductoVista(Long id, String nombre, String descripcion, 
                        BigDecimal precio, Integer stock, String categoria) {
        this.id = id;
        this.nombre = nombre;
        this.descripcion = descripcion;
        this.precio = precio;
        this.stock = stock;
        this.categoria = categoria;
        this.activo = true;
        this.calificacionPromedio = BigDecimal.ZERO;
        this.totalReviews = 0;
        this.creadoEn = LocalDateTime.now();
        this.actualizadoEn = LocalDateTime.now();
    }
    
    /**
     * Actualiza información básica del producto.
     */
    public void actualizar(String nombre, String descripcion, BigDecimal precio, 
                          Integer stock, String categoria) {
        this.nombre = nombre;
        this.descripcion = descripcion;
        this.precio = precio;
        this.stock = stock;
        this.categoria = categoria;
        this.actualizadoEn = LocalDateTime.now();
        
        log.debug("Vista de producto actualizada: {}", this.id);
    }
    
    /**
     * Agrega una nueva review y recalcula el promedio.
     * 
     * Algoritmo incremental para eficiencia:
     * nuevo_promedio = (promedio_actual * total_reviews + nueva_calificacion) / (total_reviews + 1)
     */
    public void agregarReview(Integer calificacion) {
        if (calificacion < 1 || calificacion > 5) {
            log.warn("Calificación inválida recibida: {} para producto {}", calificacion, this.id);
            return;
        }
        
        // Cálculo incremental del promedio
        BigDecimal sumaActual = calificacionPromedio.multiply(BigDecimal.valueOf(totalReviews));
        BigDecimal nuevaSuma = sumaActual.add(BigDecimal.valueOf(calificacion));
        int nuevoTotal = totalReviews + 1;
        
        this.calificacionPromedio = nuevaSuma.divide(
            BigDecimal.valueOf(nuevoTotal), 
            2, 
            RoundingMode.HALF_UP
        );
        this.totalReviews = nuevoTotal;
        this.actualizadoEn = LocalDateTime.now();
        
        log.debug("Review agregada a producto {}: nueva calificación promedio = {}", 
                 this.id, this.calificacionPromedio);
    }
    
    /**
     * Desactiva el producto en la vista.
     */
    public void desactivar() {
        this.activo = false;
        this.actualizadoEn = LocalDateTime.now();
        log.debug("Producto desactivado en vista: {}", this.id);
    }
    
    /**
     * Indica si el producto tiene reviews.
     */
    public boolean tieneReviews() {
        return totalReviews > 0;
    }
    
    /**
     * Calificación formateada para mostrar.
     */
    public String getCalificacionFormateada() {
        if (totalReviews == 0) {
            return "Sin calificaciones";
        }
        return String.format("%.1f (%d reviews)", calificacionPromedio.doubleValue(), totalReviews);
    }
}
```

#### 2. Event Handler para Sincronización

```java
// src/main/java/com/ejemplo/tienda/productos/query/ProductoEventHandler.java
package com.ejemplo.tienda.productos.query;

import com.ejemplo.tienda.productos.events.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.jmolecules.cqrs.EventHandler;
import org.springframework.modulith.ApplicationModuleListener;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * Event Handler que mantiene sincronizada la vista de productos
 * con los eventos del command side.
 * 
 * ¿Por qué @ApplicationModuleListener?
 * 1. Procesamiento asíncrono (no bloquea el command)
 * 2. Transaccional independiente
 * 3. Retry automático en caso de falla
 * 4. Persistencia de eventos para reprocessing
 * 
 * @EventHandler marca esta clase como manejador de eventos (jMolecules)
 */
@Slf4j
@Service
@RequiredArgsConstructor
@EventHandler
public class ProductoEventHandler {
    
    private final ProductoVistaRepository repository;
    
    /**
     * Maneja la creación de nuevos productos.
     * 
     * @ApplicationModuleListener hace que este método sea:
     * - Asíncrono: No bloquea la transacción del command
     * - Transaccional: Tiene su propia transacción
     * - Resiliente: Se reintenta automáticamente si falla
     */
    @ApplicationModuleListener
    @Transactional
    public void handle(ProductoCreado evento) {
        log.info("Procesando evento ProductoCreado: {}", evento);
        
        try {
            // Crear nueva vista del producto
            ProductoVista vista = new ProductoVista(
                evento.productoId().value(),
                evento.nombre(),
                evento.descripcion(),
                evento.precio(),
                evento.stock(),
                evento.categoria()
            );
            
            repository.save(vista);
            log.info("Vista de producto creada exitosamente: {}", evento.productoId());
            
        } catch (Exception e) {
            log.error("Error procesando ProductoCreado para {}: {}", 
                     evento.productoId(), e.getMessage(), e);
            // Spring Modulith reintentará automáticamente
            throw e;
        }
    }
    
    /**
     * Maneja actualizaciones de productos.
     */
    @ApplicationModuleListener
    @Transactional
    public void handle(ProductoActualizado evento) {
        log.info("Procesando evento ProductoActualizado: {}", evento);
        
        try {
            repository.findById(evento.productoId().value())
                .ifPresentOrElse(
                    vista -> {
                        vista.actualizar(
                            evento.nombre(),
                            evento.descripcion(),
                            evento.precio(),
                            evento.stock(),
                            evento.categoria()
                        );
                        repository.save(vista);
                        log.info("Vista de producto actualizada: {}", evento.productoId());
                    },
                    () -> {
                        log.warn("No se encontró vista para producto actualizado: {}", 
                                evento.productoId());
                        // En producción, podrías crear la vista o enviar alerta
                    }
                );
                
        } catch (Exception e) {
            log.error("Error procesando ProductoActualizado para {}: {}", 
                     evento.productoId(), e.getMessage(), e);
            throw e;
        }
    }
    
    /**
     * Maneja nuevas reviews de productos.
     */
    @ApplicationModuleListener
    @Transactional
    public void handle(ProductoReviewAgregado evento) {
        log.info("Procesando evento ProductoReviewAgregado: {}", evento);
        
        try {
            repository.findById(evento.productoId().value())
                .ifPresentOrElse(
                    vista -> {
                        vista.agregarReview(evento.calificacion());
                        repository.save(vista);
                        log.info("Review agregada a vista de producto: {} (nueva cal: {})", 
                                evento.productoId(), vista.getCalificacionPromedio());
                    },
                    () -> {
                        log.warn("No se encontró vista para review de producto: {}", 
                                evento.productoId());
                    }
                );
                
        } catch (Exception e) {
            log.error("Error procesando ProductoReviewAgregado para {}: {}", 
                     evento.productoId(), e.getMessage(), e);
            throw e;
        }
    }
    
    /**
     * Maneja desactivación de productos.
     */
    @ApplicationModuleListener
    @Transactional
    public void handle(ProductoDesactivado evento) {
        log.info("Procesando evento ProductoDesactivado: {}", evento);
        
        try {
            repository.findById(evento.productoId().value())
                .ifPresentOrElse(
                    vista -> {
                        vista.desactivar();
                        repository.save(vista);
                        log.info("Producto desactivado en vista: {}", evento.productoId());
                    },
                    () -> {
                        log.warn("No se encontró vista para producto desactivado: {}", 
                                evento.productoId());
                    }
                );
                
        } catch (Exception e) {
            log.error("Error procesando ProductoDesactivado para {}: {}", 
                     evento.productoId(), e.getMessage(), e);
            throw e;
        }
    }
}
```

## Testing Independiente de Módulos

### ¿Por Qué Testing Independiente?

En microservicios, cada servicio se testea independientemente. Con Spring Modulith, podemos lograr lo mismo **dentro del monolito**.

#### Beneficios:
- **Tests más rápidos**: Solo carga el módulo bajo test
- **Mejor aislamiento**: Fallas en un módulo no afectan tests de otros
- **Testing de integración real**: Con base de datos real usando TestContainers
- **Feedback más claro**: Sabes exactamente qué módulo falló

### Test del Módulo Productos (Command Side)

```java
// src/test/java/com/ejemplo/tienda/productos/ProductosCommandModuleTest.java
package com.ejemplo.tienda.productos;

import com.ejemplo.tienda.productos.command.*;
import com.ejemplo.tienda.productos.events.*;
import com.ejemplo.tienda.productos.shared.*;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.modulith.test.ApplicationModuleTest;
import org.springframework.modulith.test.PublishedEvents;
import org.springframework.test.context.TestPropertySource;
import org.springframework.transaction.annotation.Transactional;
import org.testcontainers.junit.jupiter.Testcontainers;

import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.*;

/**
 * Test de integración para el módulo de productos (command side).
 * 
 * @ApplicationModuleTest hace que:
 * 1. Solo se cargue el módulo 'productos'
 * 2. Se use una BD real con TestContainers
 * 3. Se puedan verificar eventos publicados
 * 4. Tests sean más rápidos que @SpringBootTest completo
 */
@ApplicationModuleTest
@Testcontainers
@TestPropertySource(properties = {
    "spring.jpa.hibernate.ddl-auto=create-drop",
    "logging.level.org.springframework.modulith=DEBUG"
})
class ProductosCommandModuleTest {
    
    @Autowired
    private ProductoCommandService commandService;
    
    @Autowired
    private ProductoRepository repository;
    
    @Test
    @Transactional
    void shouldCreateProductAndPublishEvent(PublishedEvents events) {
        // Given
        var request = new CreateProductoRequest(
            "Laptop Gaming",
            "Laptop de alta gama para gaming",
            new BigDecimal("1299.99"),
            10,
            "Electronics"
        );
        
        // When
        ProductoId productoId = commandService.crearProducto(request);
        
        // Then
        assertThat(productoId).isNotNull();
        assertThat(productoId.value()).isNotNull();
        
        // Verificar que el producto se guardó
        var producto = repository.findById(productoId);
        assertThat(producto).isPresent();
        assertThat(producto.get().getNombre()).isEqualTo(request.nombre());
        assertThat(producto.get().getPrecio()).isEqualTo(request.precio());
        assertThat(producto.get().getActivo()).isTrue();
        
        // Verificar que se publicó el evento correcto
        var productosCreados = events.ofType(ProductoCreado.class);
        assertThat(productosCreados).hasSize(1);
        
        ProductoCreado evento = productosCreados.iterator().next();
        assertThat(evento.productoId()).isEqualTo(productoId);
        assertThat(evento.nombre()).isEqualTo(request.nombre());
        assertThat(evento.precio()).isEqualTo(request.precio());
        assertThat(evento.categoria()).isEqualTo(request.categoria());
    }
    
    @Test
    @Transactional
    void shouldUpdateProductAndPublishEvent(PublishedEvents events) {
        // Given - crear producto inicial
        var createRequest = new CreateProductoRequest(
            "Producto Original",
            "Descripción original",
            new BigDecimal("100.00"),
            20,
            "TestCategory"
        );
        ProductoId productoId = commandService.crearProducto(createRequest);
        
        // Clear events from creation
        events.ofType(ProductoCreado.class);
        
        // When - actualizar producto
        var updateRequest = new CreateProductoRequest(
            "Producto Actualizado",
            "Descripción actualizada",
            new BigDecimal("150.00"),
            15,
            "TestCategory"
        );
        commandService.actualizarProducto(productoId, updateRequest);
        
        // Then
        var producto = repository.findById(productoId).orElseThrow();
        assertThat(producto.getNombre()).isEqualTo("Producto Actualizado");
        assertThat(producto.getPrecio()).isEqualTo(new BigDecimal("150.00"));
        assertThat(producto.getStock()).isEqualTo(15);
        
        // Verificar evento de actualización
        var productosActualizados = events.ofType(ProductoActualizado.class);
        assertThat(productosActualizados).hasSize(1);
    }
    
    @Test
    @Transactional
    void shouldAddReviewAndPublishEvent(PublishedEvents events) {
        // Given
        var request = new CreateProductoRequest(
            "Producto para Review",
            "Descripción test",
            new BigDecimal("50.00"),
            5,
            "TestCategory"
        );
        ProductoId productoId = commandService.crearProducto(request);
        
        // When
        commandService.agregarReview(productoId, 5, "Excelente producto!", "test@email.com");
        
        // Then
        var reviewEvents = events.ofType(ProductoReviewAgregado.class);
        assertThat(reviewEvents).hasSize(1);
        
        ProductoReviewAgregado evento = reviewEvents.iterator().next();
        assertThat(evento.productoId()).isEqualTo(productoId);
        assertThat(evento.calificacion()).isEqualTo(5);
        assertThat(evento.comentario()).isEqualTo("Excelente producto!");
        assertThat(evento.autorEmail()).isEqualTo("test@email.com");
    }
    
    @Test
    void shouldThrowExceptionWhenProductNotFound() {
        // Given
        ProductoId inexistenteId = ProductoId.of(99999L);
        
        // When & Then
        assertThatThrownBy(() -> 
            commandService.actualizarProducto(inexistenteId, 
                new CreateProductoRequest("Test", "Test", new BigDecimal("10.00"), 1, "Test"))
        )
        .isInstanceOf(ProductoNoEncontradoException.class)
        .hasMessageContaining("99999");
    }
    
    @Test
    void shouldThrowExceptionWhenDuplicateName() {
        // Given
        var request = new CreateProductoRequest(
            "Producto Único",
            "Descripción",
            new BigDecimal("100.00"),
            10,
            "TestCategory"
        );
        commandService.crearProducto(request);
        
        // When & Then
        assertThatThrownBy(() -> commandService.crearProducto(request))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("Ya existe un producto activo con el nombre");
    }
}
```

### Test del Event Handler (Query Side)

```java
// src/test/java/com/ejemplo/tienda/productos/ProductosQueryEventHandlerTest.java
package com.ejemplo.tienda.productos;

import com.ejemplo.tienda.productos.events.*;
import com.ejemplo.tienda.productos.query.*;
import com.ejemplo.tienda.productos.shared.ProductoId;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.modulith.test.ApplicationModuleTest;
import org.springframework.test.context.TestPropertySource;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.*;
import static org.awaitility.Awaitility.*;

/**
 * Test específico para verificar que los eventos se manejan correctamente
 * en el query side del módulo productos.
 */
@ApplicationModuleTest
@TestPropertySource(properties = {
    "spring.jpa.hibernate.ddl-auto=create-drop"
})
class ProductosQueryEventHandlerTest {
    
    @Autowired
    private ProductoEventHandler eventHandler;
    
    @Autowired
    private ProductoVistaRepository vistaRepository;
    
    @Test
    @Transactional
    void shouldCreateProductViewWhenProductCreated() {
        // Given
        var evento = new ProductoCreado(
            ProductoId.of(1L),
            "Test Product",
            "Test Description",
            new BigDecimal("29.99"),
            100,
            "Electronics"
        );
        
        // When
        eventHandler.handle(evento);
        
        // Then
        var vista = vistaRepository.findById(1L);
        assertThat(vista).isPresent();
        assertThat(vista.get().getNombre()).isEqualTo("Test Product");
        assertThat(vista.get().getPrecio()).isEqualTo(new BigDecimal("29.99"));
        assertThat(vista.get().getCalificacionPromedio()).isEqualTo(BigDecimal.ZERO);
        assertThat(vista.get().getTotalReviews()).isEqualTo(0);
    }
    
    @Test
    @Transactional
    void shouldUpdateProductViewWhenProductUpdated() {
        // Given - crear vista inicial
        var vista = new ProductoVista(
            1L, "Original", "Desc", new BigDecimal("10.00"), 5, "Cat"
        );
        vistaRepository.save(vista);
        
        // When - procesar evento de actualización
        var evento = new ProductoActualizado(
            ProductoId.of(1L),
            "Updated Product",
            "Updated Description", 
            new BigDecimal("15.00"),
            10,
            "UpdatedCategory"
        );
        eventHandler.handle(evento);
        
        // Then
        var vistaActualizada = vistaRepository.findById(1L).orElseThrow();
        assertThat(vistaActualizada.getNombre()).isEqualTo("Updated Product");
        assertThat(vistaActualizada.getPrecio()).isEqualTo(new BigDecimal("15.00"));
        assertThat(vistaActualizada.getStock()).isEqualTo(10);
    }
    
    @Test
    @Transactional
    void shouldUpdateRatingWhenReviewAdded() {
        // Given - crear vista con algunas reviews
        var vista = new ProductoVista(
            1L, "Product", "Desc", new BigDecimal("50.00"), 10, "Electronics"
        );
        vista.agregarReview(4); // Primera review: 4 estrellas
        vista.agregarReview(5); // Segunda review: 5 estrellas
        // Promedio actual: 4.5
        vistaRepository.save(vista);
        
        // When - agregar nueva review de 2 estrellas
        var evento = new ProductoReviewAgregado(
            ProductoId.of(1L),
            2,
            "No me gustó",
            "usuario@test.com"
        );
        eventHandler.handle(evento);
        
        // Then - promedio debe ser (4 + 5 + 2) / 3 = 3.67
        var vistaActualizada = vistaRepository.findById(1L).orElseThrow();
        assertThat(vistaActualizada.getTotalReviews()).isEqualTo(3);
        assertThat(vistaActualizada.getCalificacionPromedio())
            .isEqualTo(new BigDecimal("3.67"));
    }
}
```

### Test End-to-End del Módulo

```java
// src/test/java/com/ejemplo/tienda/productos/ProductosEndToEndTest.java
package com.ejemplo.tienda.productos;

import com.ejemplo.tienda.productos.shared.CreateProductoRequest;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.http.MediaType;
import org.springframework.test.context.TestPropertySource;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;
import static org.awaitility.Awaitility.*;
import static java.time.Duration.ofSeconds;

/**
 * Test end-to-end que verifica el flujo completo:
 * API → Command → Event → Query → API
 */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
@AutoConfigureMockMvc
@TestPropertySource(properties = {
    "spring.jpa.hibernate.ddl-auto=create-drop"
})
class ProductosEndToEndTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @Test
    @Transactional
    void shouldCreateProductAndQueryItAsynchronously() throws Exception {
        // 1. Crear producto via API de comandos
        var createRequest = new CreateProductoRequest(
            "Test Product E2E",
            "Test Description E2E", 
            new BigDecimal("29.99"),
            100,
            "Electronics"
        );
        
        String location = mockMvc.perform(post("/api/productos")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(createRequest)))
            .andExpect(status().isCreated())
            .andExpect(header().exists("Location"))
            .andReturn()
            .getResponse()
            .getHeader("Location");
        
        // 2. Extraer ID del producto
        String productoId = location.substring(location.lastIndexOf("/") + 1);
        
        // 3. Verificar que el producto está disponible en queries
        // Nota: Como el evento se procesa asíncronamente, usamos Awaitility
        await().atMost(ofSeconds(5))
            .pollInterval(ofSeconds(1))
            .untilAsserted(() -> {
                mockMvc.perform(get("/api/productos/" + productoId))
                    .andExpect(status().isOk())
                    .andExpect(jsonPath("$.nombre").value("Test Product E2E"))
                    .andExpect(jsonPath("$.precio").value(29.99))
                    .andExpect(jsonPath("$.calificacionPromedio").value(0.0))
                    .andExpect(jsonPath("$.totalReviews").value(0));
            });
        
        // 4. Agregar review via API de comandos
        var reviewRequest = new AddReviewRequest(5, "Excelente producto!");
        
        mockMvc.perform(post("/api/productos/" + productoId + "/reviews")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(reviewRequest)))
            .andExpect(status().isCreated());
        
        // 5. Verificar que la review se procesó asíncronamente en el query side
        await().atMost(ofSeconds(5))
            .pollInterval(ofSeconds(1))
            .untilAsserted(() -> {
                mockMvc.perform(get("/api/productos/" + productoId))
                    .andExpected(status().isOk())
                    .andExpect(jsonPath("$.calificacionPromedio").value(5.0))
                    .andExpect(jsonPath("$.totalReviews").value(1));
            });
        
        // 6. Verificar consultas optimizadas
        mockMvc.perform(get("/api/productos/by-rating"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$[0].nombre").value("Test Product E2E"))
            .andExpect(jsonPath("$[0].calificacionPromedio").value(5.0));
    }
}
```

## APIs REST

### ¿Por Qué Separar APIs de Command y Query?

Aunque usamos el mismo protocolo HTTP, separamos las responsabilidades:

- **Command API**: Operaciones que modifican estado
- **Query API**: Operaciones de solo lectura

#### Beneficios:
1. **Escalabilidad diferenciada**: Puedes escalar lecturas y escrituras independientemente
2. **Caché específico**: Solo las queries pueden usar caché agresivo
3. **Seguridad granular**: Permisos diferentes para leer vs escribir
4. **Monitoreo específico**: Métricas separadas para cada tipo de operación

### Command API Implementation

#### 1. DTOs para Requests

```java
// src/main/java/com/ejemplo/tienda/productos/shared/AddReviewRequest.java
package com.ejemplo.tienda.productos.shared;

import jakarta.validation.constraints.*;

/**
 * DTO para agregar reviews a productos.
 */
public record AddReviewRequest(
    
    @NotNull(message = "La calificación es obligatoria")
    @Min(value = 1, message = "La calificación mínima es 1")
    @Max(value = 5, message = "La calificación máxima es 5")
    Integer calificacion,
    
    @Size(max = 500, message = "El comentario no puede exceder 500 caracteres")
    String comentario,
    
    @Email(message = "El email del autor debe ser válido")
    @NotBlank(message = "El email del autor es obligatorio")
    String autorEmail
) {
    
    public static AddReviewRequest of(Integer calificacion, String comentario) {
        return new AddReviewRequest(calificacion, comentario, "anonimo@test.com");
    }
}
```

#### 2. Controller de Comandos

```java
// src/main/java/com/ejemplo/tienda/productos/command/ProductoCommandController.java
package com.ejemplo.tienda.productos.command;

import com.ejemplo.tienda.productos.shared.*;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.net.URI;

/**
 * Controller para operaciones de escritura en productos.
 * 
 * Principios seguidos:
 * 1. Solo operaciones que modifican estado
 * 2. Validación exhaustiva de inputs
 * 3. Respuestas mínimas (solo confirmación)
 * 4. URLs RESTful claras
 * 5. Status codes HTTP apropiados
 */
@Slf4j
@RestController
@RequestMapping("/api/productos")
@RequiredArgsConstructor
class ProductoCommandController {
    
    private final ProductoCommandService commandService;
    
    /**
     * Crea un nuevo producto.
     * 
     * @param request Datos del producto a crear
     * @return 201 Created con Location header del recurso creado
     */
    @PostMapping
    public ResponseEntity<Void> crearProducto(@Valid @RequestBody CreateProductoRequest request) {
        log.info("Solicitud de creación de producto: {}", request.nombre());
        
        try {
            ProductoId productoId = commandService.crearProducto(request);
            
            URI location = URI.create("/api/productos/" + productoId.value());
            
            log.info("Producto creado exitosamente: {}", productoId);
            return ResponseEntity.created(location).build();
            
        } catch (IllegalStateException e) {
            log.warn("Error de negocio creando producto '{}': {}", request.nombre(), e.getMessage());
            throw e; // Se maneja en @ControllerAdvice
        } catch (Exception e) {
            log.error("Error inesperado creando producto '{}': {}", request.nombre(), e.getMessage(), e);
            throw e;
        }
    }
    
    /**
     * Actualiza un producto existente.
     * 
     * @param id ID del producto a actualizar
     * @param request Nuevos datos del producto
     * @return 200 OK si la actualización fue exitosa
     */
    @PutMapping("/{id}")
    public ResponseEntity<Void> actualizarProducto(
            @PathVariable Long id,
            @Valid @RequestBody CreateProductoRequest request) {
        
        log.info("Solicitud de actualización de producto: {}", id);
        
        try {
            ProductoId productoId = ProductoId.of(id);
            commandService.actualizarProducto(productoId, request);
            
            log.info("Producto actualizado exitosamente: {}", id);
            return ResponseEntity.ok().build();
            
        } catch (ProductoNoEncontradoException e) {
            log.warn("Producto no encontrado para actualizar: {}", id);
            throw e;
        } catch (IllegalStateException e) {
            log.warn("Error de negocio actualizando producto {}: {}", id, e.getMessage());
            throw e;
        }
    }
    
    /**
     * Agrega una review a un producto.
     * 
     * @param id ID del producto
     * @param request Datos de la review
     * @return 201 Created con Location de la review creada
     */
    @PostMapping("/{id}/reviews")
    public ResponseEntity<Void> agregarReview(
            @PathVariable Long id,
            @Valid @RequestBody AddReviewRequest request) {
        
        log.info("Solicitud de review para producto: {} por {}", id, request.autorEmail());
        
        try {
            ProductoId productoId = ProductoId.of(id);
            commandService.agregarReview(
                productoId, 
                request.calificacion(), 
                request.comentario(),
                request.autorEmail()
            );
            
            // En un sistema real, tendríamos un ID de review para el Location
            URI location = URI.create("/api/productos/" + id + "/reviews");
            
            log.info("Review agregada exitosamente al producto: {}", id);
            return ResponseEntity.created(location).build();
            
        } catch (ProductoNoEncontradoException e) {
            log.warn("Producto no encontrado para review: {}", id);
            throw e;
        }
    }
    
    /**
     * Desactiva un producto (soft delete).
     * 
     * @param id ID del producto a desactivar
     * @return 204 No Content
     */
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> desactivarProducto(@PathVariable Long id) {
        log.info("Solicitud de desactivación de producto: {}", id);
        
        try {
            ProductoId productoId = ProductoId.of(id);
            commandService.desactivarProducto(productoId);
            
            log.info("Producto desactivado exitosamente: {}", id);
            return ResponseEntity.noContent().build();
            
        } catch (ProductoNoEncontradoException e) {
            log.warn("Producto no encontrado para desactivar: {}", id);
            throw e;
        }
    }
}
```

### Query API Implementation

#### 1. DTOs para Responses

```java
// src/main/java/com/ejemplo/tienda/productos/shared/ProductoResponse.java
package com.ejemplo.tienda.productos.shared;

import com.ejemplo.tienda.productos.query.ProductoVista;
import com.fasterxml.jackson.annotation.JsonFormat;

import java.math.BigDecimal;
import java.time.LocalDateTime;

/**
 * DTO optimizado para respuestas de consultas de productos.
 * 
 * ¿Por qué un DTO específico?
 * 1. Control total sobre la API externa
 * 2. Versionado independiente del modelo interno
 * 3. Campos calculados y formateados
 * 4. Optimización de serialización JSON
 */
public record ProductoResponse(
    Long id,
    String nombre,
    String descripcion,
    BigDecimal precio,
    Integer stock,
    String categoria,
    Boolean activo,
    
    // Campos específicos del query side
    BigDecimal calificacionPromedio,
    Integer totalReviews,
    String calificacionFormateada,
    Boolean tieneReviews,
    
    // Metadatos útiles para el frontend
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss")
    LocalDateTime creadoEn,
    
    @JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ss")
    LocalDateTime actualizadoEn
) {
    
    /**
     * Factory method para convertir desde ProductoVista.
     */
    public static ProductoResponse from(ProductoVista vista) {
        return new ProductoResponse(
            vista.getId(),
            vista.getNombre(),
            vista.getDescripcion(),
            vista.getPrecio(),
            vista.getStock(),
            vista.getCategoria(),
            vista.getActivo(),
            vista.getCalificacionPromedio(),
            vista.getTotalReviews(),
            vista.getCalificacionFormateada(),
            vista.tieneReviews(),
            vista.getCreadoEn(),
            vista.getActualizadoEn()
        );
    }
}
```

#### 2. Repository de Query con Consultas Optimizadas

```java
// src/main/java/com/ejemplo/tienda/productos/query/ProductoVistaRepository.java
package com.ejemplo.tienda.productos.query;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.math.BigDecimal;
import java.util.List;

/**
 * Repository optimizado para consultas de productos.
 * 
 * Características:
 * 1. Consultas específicas para patrones de lectura
 * 2. Uso de índices optimizados
 * 3. Soporte de paginación
 * 4. Queries que aprovechan datos desnormalizados
 */
interface ProductoVistaRepository extends JpaRepository<ProductoVista, Long> {
    
    /**
     * Encuentra productos activos con paginación.
     * Consulta más común en el sistema.
     */
    @Query("SELECT pv FROM ProductoVista pv WHERE pv.activo = true ORDER BY pv.actualizadoEn DESC")
    Page<ProductoVista> findActivosRecientes(Pageable pageable);
    
    /**
     * Productos ordenados por calificación promedio (mejores primero).
     * Aprovecha el campo desnormalizado para eficiencia.
     */
    @Query("SELECT pv FROM ProductoVista pv WHERE pv.activo = true AND pv.totalReviews > 0 " +
           "ORDER BY pv.calificacionPromedio DESC, pv.totalReviews DESC")
    List<ProductoVista> findActivosOrdernadosPorCalificacion();
    
    /**
     * Productos por categoría con paginación.
     * Índice en (categoria, activo) para eficiencia.
     */
    @Query("SELECT pv FROM ProductoVista pv WHERE pv.categoria = :categoria AND pv.activo = true " +
           "ORDER BY pv.calificacionPromedio DESC")
    Page<ProductoVista> findActivosPorCategoria(@Param("categoria") String categoria, Pageable pageable);
    
    /**
     * Productos en rango de precio específico.
     * Útil para filtros de e-commerce.
     */
    @Query("SELECT pv FROM ProductoVista pv WHERE pv.activo = true " +
           "AND pv.precio BETWEEN :precioMin AND :precioMax " +
           "ORDER BY pv.precio ASC")
    List<ProductoVista> findActivosEnRangoPrecio(
        @Param("precioMin") BigDecimal precioMin,
        @Param("precioMax") BigDecimal precioMax
    );
    
    /**
     * Productos populares (con muchas reviews).
     * Para secciones tipo "trending" en el frontend.
     */
    @Query("SELECT pv FROM ProductoVista pv WHERE pv.activo = true AND pv.totalReviews >= :minReviews " +
           "ORDER BY pv.totalReviews DESC, pv.calificacionPromedio DESC")
    List<ProductoVista> findPopulares(@Param("minReviews") Integer minReviews, Pageable pageable);
    
    /**
     * Búsqueda de texto en nombre y descripción.
     * PostgreSQL full-text search para mejor performance.
     */
    @Query("SELECT pv FROM ProductoVista pv WHERE pv.activo = true " +
           "AND (LOWER(pv.nombre) LIKE LOWER(CONCAT('%', :termino, '%')) " +
           "OR LOWER(pv.descripcion) LIKE LOWER(CONCAT('%', :termino, '%'))) " +
           "ORDER BY pv.calificacionPromedio DESC")
    Page<ProductoVista> buscarPorTexto(@Param("termino") String termino, Pageable pageable);
    
    /**
     * Estadísticas por categoría.
     * Útil para dashboards y analytics.
     */
    @Query("SELECT pv.categoria, COUNT(pv), AVG(pv.precio), AVG(pv.calificacionPromedio) " +
           "FROM ProductoVista pv WHERE pv.activo = true " +
           "GROUP BY pv.categoria ORDER BY COUNT(pv) DESC")
    List<Object[]> obtenerEstadisticasPorCategoria();
}
```

#### 3. Query Service

```java
// src/main/java/com/ejemplo/tienda/productos/query/ProductoQueryService.java
package com.ejemplo.tienda.productos.query;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

/**
 * Servicio para operaciones de consulta de productos.
 * 
 * Características:
 * 1. Solo operaciones de lectura
 * 2. Uso agresivo de caché
 * 3. Optimizaciones específicas para queries
 * 4. Transacciones de solo lectura
 */
@Slf4j
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true) // Optimización para lecturas
class ProductoQueryService {
    
    private final ProductoVistaRepository repository;
    
    /**
     * Busca producto por ID.
     * Cacheable ya que los productos no cambian frecuentemente.
     */
    @Cacheable(value = "productos", key = "#id")
    public Optional<ProductoVista> buscarPorId(Long id) {
        log.debug("Buscando producto por ID: {}", id);
        return repository.findById(id);
    }
    
    /**
     * Lista productos activos con paginación.
     * Cache por página para mejorar performance.
     */
    @Cacheable(value = "productos-activos", key = "#pageable.pageNumber + '-' + #pageable.pageSize")
    public Page<ProductoVista> listarActivos(Pageable pageable) {
        log.debug("Listando productos activos - página: {}, tamaño: {}", 
                 pageable.getPageNumber(), pageable.getPageSize());
        return repository.findActivosRecientes(pageable);
    }
    
    /**
     * Lista productos ordenados por calificación.
     * Cache más agresivo ya que cambia menos frecuentemente.
     */
    @Cacheable(value = "productos-por-rating", unless = "#result.isEmpty()")
    public List<ProductoVista> listarPorCalificacion() {
        log.debug("Listando productos ordenados por calificación");
        return repository.findActivosOrdernadosPorCalificacion();
    }
    
    /**
     * Lista productos por categoría.
     */
    @Cacheable(value = "productos-categoria", key = "#categoria + '-' + #pageable.pageNumber")
    public Page<ProductoVista> listarPorCategoria(String categoria, Pageable pageable) {
        log.debug("Listando productos por categoría: {}", categoria);
        return repository.findActivosPorCategoria(categoria, pageable);
    }
    
    /**
     * Lista productos en rango de precio.
     * No cacheamos rangos dinámicos para evitar memory leak.
     */
    public List<ProductoVista> listarEnRangoPrecio(BigDecimal precioMin, BigDecimal precioMax) {
        log.debug("Listando productos en rango de precio: {} - {}", precioMin, precioMax);
        
        if (precioMin.compareTo(precioMax) > 0) {
            throw new IllegalArgumentException("El precio mínimo no puede ser mayor al máximo");
        }
        
        return repository.findActivosEnRangoPrecio(precioMin, precioMax);
    }
    
    /**
     * Lista productos populares.
     */
    @Cacheable(value = "productos-populares", key = "#minReviews + '-' + #limite")
    public List<ProductoVista> listarPopulares(Integer minReviews, Integer limite) {
        log.debug("Listando productos populares (min reviews: {}, límite: {})", minReviews, limite);
        
        Pageable pageable = PageRequest.of(0, limite);
        return repository.findPopulares(minReviews, pageable);
    }
    
    /**
     * Búsqueda de texto.
     * No cacheamos búsquedas para evitar memory leak.
     */
    public Page<ProductoVista> buscar(String termino, Pageable pageable) {
        log.debug("Buscando productos con término: '{}'", termino);
        
        if (termino == null || termino.trim().isEmpty()) {
            return listarActivos(pageable);
        }
        
        return repository.buscarPorTexto(termino.trim(), pageable);
    }
    
    /**
     * Obtiene estadísticas por categoría.
     * Cache de larga duración ya que son agregados.
     */
    @Cacheable(value = "estadisticas-categoria")
    public List<Object[]> obtenerEstadisticasPorCategoria() {
        log.debug("Obteniendo estadísticas por categoría");
        return repository.obtenerEstadisticasPorCategoria();
    }
}
```

#### 4. Query Controller

```java
// src/main/java/com/ejemplo/tienda/productos/query/ProductoQueryController.java
package com.ejemplo.tienda.productos.query;

import com.ejemplo.tienda.productos.shared.ProductoResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.util.List;

/**
 * Controller para operaciones de consulta de productos.
 * 
 * Principios:
 * 1. Solo operaciones de lectura
 * 2. Respuestas ricas en datos
 * 3. Cache headers apropiados
 * 4. Paginación por defecto
 * 5. Filtros flexibles
 */
@Slf4j
@RestController
@RequestMapping("/api/productos")
@RequiredArgsConstructor
class ProductoQueryController {
    
    private final ProductoQueryService queryService;
    
    /**
     * Obtiene un producto por ID.
     */
    @GetMapping("/{id}")
    public ResponseEntity<ProductoResponse> obtenerProducto(@PathVariable Long id) {
        log.debug("Consulta de producto: {}", id);
        
        return queryService.buscarPorId(id)
            .map(ProductoResponse::from)
            .map(response -> ResponseEntity.ok()
                .cacheControl(org.springframework.http.CacheControl.maxAge(java.time.Duration.ofMinutes(5)))
                .body(response))
            .orElse(ResponseEntity.notFound().build());
    }
    
    /**
     * Lista productos con paginación.
     */
    @GetMapping
    public Page<ProductoResponse> listarProductos(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        
        log.debug("Consulta de productos - página: {}, tamaño: {}", page, size);
        
        Pageable pageable = PageRequest.of(page, size);
        return queryService.listarActivos(pageable)
            .map(ProductoResponse::from);
    }
    
    /**
     * Lista productos ordenados por calificación.
     */
    @GetMapping("/by-rating")
    public List<ProductoResponse> listarPorCalificacion() {
        log.debug("Consulta de productos por calificación");
        
        return queryService.listarPorCalificacion()
            .stream()
            .map(ProductoResponse::from)
            .toList();
    }
    
    /**
     * Lista productos por categoría.
     */
    @GetMapping("/categoria/{categoria}")
    public Page<ProductoResponse> listarPorCategoria(
            @PathVariable String categoria,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        
        log.debug("Consulta de productos por categoría: {}", categoria);
        
        Pageable pageable = PageRequest.of(page, size);
        return queryService.listarPorCategoria(categoria, pageable)
            .map(ProductoResponse::from);
    }
    
    /**
     * Lista productos en rango de precio.
     */
    @GetMapping("/precio-rango")
    public List<ProductoResponse> listarEnRangoPrecio(
            @RequestParam BigDecimal precioMin,
            @RequestParam BigDecimal precioMax) {
        
        log.debug("Consulta de productos en rango de precio: {} - {}", precioMin, precioMax);
        
        return queryService.listarEnRangoPrecio(precioMin, precioMax)
            .stream()
            .map(ProductoResponse::from)
            .toList();
    }
    
    /**
     * Lista productos populares.
     */
    @GetMapping("/populares")
    public List<ProductoResponse> listarPopulares(
            @RequestParam(defaultValue = "5") Integer minReviews,
            @RequestParam(defaultValue = "10") Integer limite) {
        
        log.debug("Consulta de productos populares");
        
        return queryService.listarPopulares(minReviews, limite)
            .stream()
            .map(ProductoResponse::from)
            .toList();
    }
    
    /**
     * Búsqueda de productos por texto.
     */
    @GetMapping("/buscar")
    public Page<ProductoResponse> buscarProductos(
            @RequestParam String q,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        
        log.debug("Búsqueda de productos: '{}'", q);
        
        Pageable pageable = PageRequest.of(page, size);
        return queryService.buscar(q, pageable)
            .map(ProductoResponse::from);
    }
}
```

### API Pública del Módulo

Para exponer una API unificada desde el módulo, creamos un servicio público:

```java
// src/main/java/com/ejemplo/tienda/productos/ProductoService.java
package com.ejemplo.tienda.productos;

import com.ejemplo.tienda.productos.command.ProductoCommandService;
import com.ejemplo.tienda.productos.query.ProductoQueryService;
import com.ejemplo.tienda.productos.shared.*;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;

import java.util.Optional;

/**
 * API pública del módulo productos.
 * 
 * Esta clase está en el root del módulo, por lo que es accesible
 * desde otros módulos. Actúa como fachada que unifica
 * operaciones de command y query.
 * 
 * ¿Por qué una fachada?
 * 1. API simple para otros módulos
 * 2. Encapsula la separación CQRS interna
 * 3. Permite evolución independiente
 * 4. Facilita testing de integration
 */
@Service
@RequiredArgsConstructor
public class ProductoService {
    
    private final ProductoCommandService commandService;
    private final ProductoQueryService queryService;
    
    // ===============================================
    // OPERACIONES DE COMANDO (DELEGADAS)
    // ===============================================
    
    public ProductoId crearProducto(CreateProductoRequest request) {
        return commandService.crearProducto(request);
    }
    
    public void actualizarProducto(ProductoId productoId, CreateProductoRequest request) {
        commandService.actualizarProducto(productoId, request);
    }
    
    public void agregarReview(ProductoId productoId, Integer calificacion, String comentario, String autorEmail) {
        commandService.agregarReview(productoId, calificacion, comentario, autorEmail);
    }
    
    // ===============================================
    // OPERACIONES DE QUERY (DELEGADAS)
    // ===============================================
    
    public Optional<ProductoVista> buscarPorId(Long id) {
        return queryService.buscarPorId(id);
    }
    
    public Page<ProductoVista> listarActivos(Pageable pageable) {
        return queryService.listarActivos(pageable);
    }
    
    public Page<ProductoVista> listarPorCategoria(String categoria, Pageable pageable) {
        return queryService.listarPorCategoria(categoria, pageable);
    }
    
    // ===============================================
    // OPERACIONES DE CONVENIENCIA
    // ===============================================
    
    /**
     * Verifica si un producto existe y está activo.
     * Útil para validaciones desde otros módulos.
     */
    public boolean existeProductoActivo(Long id) {
        return queryService.buscarPorId(id)
            .map(producto -> producto.getActivo())
            .orElse(false);
    }
    
    /**
     * Obtiene información básica de un producto.
     * Útil para otros módulos que necesitan datos básicos.
     */
    public Optional<ProductoInfo> obtenerInfoBasica(Long id) {
        return queryService.buscarPorId(id)
            .map(vista -> new ProductoInfo(
                vista.getId(),
                vista.getNombre(),
                vista.getPrecio(),
                vista.getStock(),
                vista.getActivo()
            ));
    }
}
```

```java
// src/main/java/com/ejemplo/tienda/productos/ProductoInfo.java
package com.ejemplo.tienda.productos;

import java.math.BigDecimal;

/**
 * DTO ligero con información básica de producto.
 * Para uso por otros módulos.
 */
public record ProductoInfo(
    Long id,
    String nombre,
    BigDecimal precio,
    Integer stock,
    Boolean activo
) {}
```

## Documentación Automática

### Configuración de Spring Modulith Docs

```java
// src/test/java/com/ejemplo/tienda/DocumentationTest.java
package com.ejemplo.tienda;

import org.junit.jupiter.api.Test;
import org.springframework.modulith.core.ApplicationModules;
import org.springframework.modulith.docs.Documenter;

/**
 * Genera documentación automática de la arquitectura modular.
 * 
 * Los diagramas se generan en: target/spring-modulith-docs/
 */
class DocumentationTest {
    
    ApplicationModules modules = ApplicationModules.of(TiendaApplication.class);
    
    /**
     * Genera documentación completa del sistema.
     * 
     * Genera:
     * - component-diagram.puml: Diagrama de componentes general
     * - module-{nombre}.adoc: Documentación AsciiDoc por módulo
     * - module-{nombre}.puml: Diagramas PlantUML específicos
     */
    @Test
    void generateDocumentation() throws Exception {
        new Documenter(modules)
            .writeDocumentation()
            .writeIndividualModulesAsPlantUml()
            .writeModuleCanvases(); // Diagramas C4
    }
    
    /**
     * Genera solo diagramas PlantUML.
     * Útil para CI/CD pipelines.
     */
    @Test 
    void generatePlantUMLDiagrams() throws Exception {
        new Documenter(modules)
            .writeIndividualModulesAsPlantUml();
    }
}
```

### Integración con CI/CD para Documentación

```yaml
# .github/workflows/documentation.yml
name: Generate Architecture Documentation

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  documentation:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up JDK 17
      uses: actions/setup-java@v3
      with:
        java-version: '17'
        distribution: 'temurin'
    
    - name: Generate Documentation
      run: mvn test -Dtest=DocumentationTest
    
    - name: Setup PlantUML
      run: |
        sudo apt-get update
        sudo apt-get install -y plantuml
    
    - name: Generate PNG diagrams
      run: |
        find target/spring-modulith-docs -name "*.puml" -exec plantuml {} \;
    
    - name: Upload Documentation Artifacts
      uses: actions/upload-artifact@v3
      with:
        name: architecture-docs
        path: target/spring-modulith-docs/
    
    - name: Deploy to GitHub Pages (if main branch)
      if: github.ref == 'refs/heads/main'
      uses: peaceiris/actions-gh-pages@v3
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}
        publish_dir: target/spring-modulith-docs/
```

## Observabilidad y Trazabilidad

### Configuración de Tracing Distribuido

```yaml
# application.yml - Configuración adicional para observabilidad
management:
  tracing:
    enabled: true
    sampling:
      probability: 1.0 # 100% en desarrollo, reducir en producción
  
  zipkin:
    tracing:
      endpoint: http://localhost:9411/api/v2/spans

  metrics:
    export:
      prometheus:
        enabled: true
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5,0.9,0.95,0.99

# Logging estructurado
logging:
  pattern:
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

### Docker Compose con Observabilidad

```yaml
# docker-compose.yml - Versión extendida con observabilidad
version: '3.8'

services:
  # Base de datos principal
  postgres:
    image: postgres:15-alpine
    container_name: tienda-postgres
    environment:
      POSTGRES_DB: tienda_db
      POSTGRES_USER: tienda_user
      POSTGRES_PASSWORD: tienda_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - tienda-network

  # Aplicación principal
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: tienda-app
    depends_on:
      - postgres
      - zipkin
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/tienda_db
      SPRING_DATASOURCE_USERNAME: tienda_user
      SPRING_DATASOURCE_PASSWORD: tienda_password
      MANAGEMENT_ZIPKIN_TRACING_ENDPOINT: http://zipkin:9411/api/v2/spans
    ports:
      - "8080:8080"
    networks:
      - tienda-network

  # Zipkin para distributed tracing
  zipkin:
    image: openzipkin/zipkin:latest
    container_name: tienda-zipkin
    ports:
      - "9411:9411"
    networks:
      - tienda-network

  # Prometheus para métricas
  prometheus:
    image: prom/prometheus:latest
    container_name: tienda-prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    networks:
      - tienda-network

  # Grafana para dashboards
  grafana:
    image: grafana/grafana:latest
    container_name: tienda-grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources
    networks:
      - tienda-network

volumes:
  postgres_data:
  prometheus_data:
  grafana_data:

networks:
  tienda-network:
    driver: bridge
```

### Configuración de Prometheus

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'tienda-app'
    static_configs:
      - targets: ['app:8080']
    metrics_path: '/actuator/prometheus'
    scrape_interval: 5s
```

### Métricas Personalizadas

```java
// src/main/java/com/ejemplo/tienda/productos/command/ProductoMetrics.java
package com.ejemplo.tienda.productos.command;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

/**
 * Métricas personalizadas para el módulo productos.
 * 
 * Métricas importantes para monitorear:
 * 1. Throughput de creación de productos
 * 2. Latencia de operaciones
 * 3. Errores por tipo
 * 4. Distribución por categoría
 */
@Component
@RequiredArgsConstructor
public class ProductoMetrics {
    
    private final MeterRegistry meterRegistry;
    
    // Contadores
    private final Counter productosCreados = Counter.builder("productos.creados")
        .description("Total de productos creados")
        .register(meterRegistry);
        
    private final Counter productosActualizados = Counter.builder("productos.actualizados")
        .description("Total de productos actualizados")
        .register(meterRegistry);
        
    private final Counter reviewsAgregadas = Counter.builder("productos.reviews.agregadas")
        .description("Total de reviews agregadas")
        .register(meterRegistry);
    
    // Timers para latencia
    private final Timer tiempoCreacion = Timer.builder("productos.creacion.tiempo")
        .description("Tiempo de creación de productos")
        .register(meterRegistry);
        
    private final Timer tiempoActualizacion = Timer.builder("productos.actualizacion.tiempo")
        .description("Tiempo de actualización de productos")
        .register(meterRegistry);
    
    // Métodos para registrar métricas
    public void registrarProductoCreado(String categoria) {
        productosCreados.increment();
        Counter.builder("productos.creados.por.categoria")
            .tag("categoria", categoria)
            .register(meterRegistry)
            .increment();
    }
    
    public void registrarProductoActualizado() {
        productosActualizados.increment();
    }
    
    public void registrarReviewAgregada(Integer calificacion) {
        reviewsAgregadas.increment();
        Counter.builder("productos.reviews.por.calificacion")
            .tag("calificacion", String.valueOf(calificacion))
            .register(meterRegistry)
            .increment();
    }
    
    public Timer.Sample iniciarMedicionCreacion() {
        return Timer.start(meterRegistry);
    }
    
    public void finalizarMedicionCreacion(Timer.Sample sample) {
        sample.stop(tiempoCreacion);
    }
    
    public Timer.Sample iniciarMedicionActualizacion() {
        return Timer.start(meterRegistry);
    }
    
    public void finalizarMedicionActualizacion(Timer.Sample sample) {
        sample.stop(tiempoActualizacion);
    }
}
```

## Despliegue con Docker

### Dockerfile Optimizado

```dockerfile
# Dockerfile
# Multi-stage build para optimizar tamaño de imagen

# Stage 1: Build
FROM openjdk:17-jdk-slim AS builder

WORKDIR /app

# Copiar archivos de configuración de Maven
COPY pom.xml .
COPY .mvn .mvn
COPY mvnw .

# Descargar dependencias (cacheadas si no cambia pom.xml)
RUN ./mvnw dependency:go-offline -B

# Copiar código fuente
COPY src src

# Build de la aplicación
RUN ./mvnw clean package -DskipTests

# Stage 2: Runtime
FROM openjdk:17-jre-slim

WORKDIR /app

# Crear usuario no-root para seguridad
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Instalar herramientas útiles para debugging
RUN apt-get update && apt-get install -y \
    curl \
    netcat-traditional \
    && rm -rf /var/lib/apt/lists/*

# Copiar JAR desde build stage
COPY --from=builder /app/target/*.jar app.jar

# Cambiar ownership al usuario no-root
RUN chown appuser:appuser app.jar

USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

# Puerto de la aplicación
EXPOSE 8080

# JVM optimizations para contenedores
ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XX:+UseG1GC -XX:+UnlockExperimentalVMOptions -XX:+UseJVMCICompiler"

# Comando de inicio
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

### Script de Despliegue

```bash
#!/bin/bash
# deploy.sh - Script completo de despliegue

set -e  # Exit on any error

echo "🚀 Iniciando despliegue de Tienda CQRS..."

# Configuración
APP_NAME="tienda-cqrs"
DOCKER_IMAGE="$APP_NAME:latest"
DOCKER_COMPOSE_FILE="docker-compose.yml"

# Función para logging
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Función para verificar prerequisitos
check_prerequisites() {
    log "Verificando prerequisitos..."
    
    if ! command -v docker &> /dev/null; then
        echo "❌ Docker no está instalado"
        exit 1
    fi
    
    if ! command -v docker-compose &> /dev/null; then
        echo "❌ Docker Compose no está instalado"
        exit 1
    fi
    
    if ! command -v mvn &> /dev/null; then
        echo "❌ Maven no está instalado"
        exit 1
    fi
    
    log "✅ Todos los prerequisitos están instalados"
}

# Función para ejecutar tests
run_tests() {
    log "Ejecutando tests..."
    
    mvn clean test
    
    if [ $? -ne 0 ]; then
        echo "❌ Tests fallaron"
        exit 1
    fi
    
    log "✅ Todos los tests pasaron"
}

# Función para verificar estructura modular
verify_modular_structure() {
    log "Verificando estructura modular..."
    
    mvn test -Dtest=ModularityTest
    
    if [ $? -ne 0 ]; then
        echo "❌ Verificación de estructura modular falló"
        exit 1
    fi
    
    log "✅ Estructura modular verificada"
}

# Función para build de la aplicación
build_application() {
    log "Compilando aplicación..."
    
    mvn clean package -DskipTests
    
    if [ $? -ne 0 ]; then
        echo "❌ Build falló"
        exit 1
    fi
    
    log "✅ Aplicación compilada exitosamente"
}

# Función para build de imagen Docker
build_docker_image() {
    log "Construyendo imagen Docker: $DOCKER_IMAGE"
    
    docker build -t $DOCKER_IMAGE .
    
    if [ $? -ne 0 ]; then
        echo "❌ Build de imagen Docker falló"
        exit 1
    fi
    
    log "✅ Imagen Docker construida exitosamente"
}

# Función para iniciar servicios
start_services() {
    log "Iniciando servicios con Docker Compose..."
    
    # Detener servicios existentes si están corriendo
    docker-compose -f $DOCKER_COMPOSE_FILE down
    
    # Iniciar servicios
    docker-compose -f $DOCKER_COMPOSE_FILE up -d
    
    if [ $? -ne 0 ]; then
        echo "❌ Falló al iniciar servicios"
        exit 1
    fi
    
    log "✅ Servicios iniciados exitosamente"
}

# Función para verificar salud de servicios
check_service_health() {
    log "Verificando salud de servicios..."
    
    # Esperar que PostgreSQL esté listo
    log "Esperando PostgreSQL..."
    timeout 60 sh -c 'until docker-compose exec -T postgres pg_isready -U tienda_user; do sleep 1; done'
    
    # Esperar que la aplicación esté lista
    log "Esperando aplicación..."
    timeout 120 sh -c 'until curl -f http://localhost:8080/actuator/health; do sleep 2; done'
    
    # Verificar endpoints clave
    log "Verificando endpoints..."
    
    # Health check
    if ! curl -f http://localhost:8080/actuator/health > /dev/null 2>&1; then
        echo "❌ Health check falló"
        docker-compose logs app
        exit 1
    fi
    
    # Verificar API de productos
    if ! curl -f http://localhost:8080/api/productos > /dev/null 2>&1; then
        echo "❌ API de productos no responde"
        docker-compose logs app
        exit 1
    fi
    
    log "✅ Todos los servicios están saludables"
}

# Función para mostrar información de servicios
show_service_info() {
    log "📋 Información de servicios desplegados:"
    echo ""
    echo "🌐 Aplicación:     http://localhost:8080"
    echo "📊 Actuator:      http://localhost:8080/actuator"
    echo "🔍 Zipkin:        http://localhost:9411"
    echo "📈 Prometheus:    http://localhost:9090"
    echo "📊 Grafana:       http://localhost:3000 (admin/admin123)"
    echo "🗄️  PgAdmin:       http://localhost:5050 (admin@tienda.com/admin123)"
    echo ""
    echo "📚 Documentación API:"
    echo "   Productos:     http://localhost:8080/api/productos"
    echo "   Actuator:      http://localhost:8080/actuator/modulith"
    echo ""
    echo "🐳 Para ver logs: docker-compose logs -f [service]"
    echo "🛑 Para detener:  docker-compose down"
}

# Función principal
main() {
    case "${1:-all}" in
        "check")
            check_prerequisites
            ;;
        "test")
            run_tests
            ;;
        "verify")
            verify_modular_structure
            ;;
        "build")
            build_application
            build_docker_image
            ;;
        "deploy")
            start_services
            check_service_health
            show_service_info
            ;;
        "all")
            check_prerequisites
            run_tests
            verify_modular_structure
            build_application
            build_docker_image
            start_services
            check_service_health
            show_service_info
            ;;
        *)
            echo "Uso: $0 {check|test|verify|build|deploy|all}"
            echo ""
            echo "Comandos:"
            echo "  check   - Verificar prerequisitos"
            echo "  test    - Ejecutar tests"
            echo "  verify  - Verificar estructura modular"
            echo "  build   - Compilar aplicación e imagen Docker"
            echo "  deploy  - Desplegar servicios"
            echo "  all     - Ejecutar todo el pipeline (default)"
            exit 1
            ;;
    esac
}

# Ejecutar función principal
main "$@"
```

### Uso del Script de Despliegue

```bash
# Hacer el script ejecutable
chmod +x deploy.sh

# Despliegue completo
./deploy.sh

# O ejecutar pasos específicos
./deploy.sh check    # Solo verificar prerequisitos
./deploy.sh test     # Solo ejecutar tests
./deploy.sh build    # Solo compilar
./deploy.sh deploy   # Solo desplegar
```

## Consideraciones de Producción

### 1. Configuración por Ambientes

```yaml
# application-prod.yml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:tienda_db}
    username: ${DB_USERNAME:tienda_user}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000

  jpa:
    hibernate:
      ddl-auto: validate # NUNCA create-drop en producción
    show-sql: false      # No mostrar SQL en producción
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true

modulith:
  events:
    completion-mode: archive
    republish-outstanding-events-on-restart: true
    # Timeout más conservador en producción
    republish-incomplete-events-after: PT5M

# Cache configuration
spring:
  cache:
    type: redis
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}

# Observabilidad en producción
management:
  tracing:
    sampling:
      probability: 0.1  # Solo 10% de traces en producción
  metrics:
    export:
      prometheus:
        enabled: true
        step: 30s

# Logging estructurado
logging:
  level:
    com.ejemplo.tienda: INFO
    org.springframework.modulith: WARN
  config: classpath:logback-spring.xml
```

### 2. Monitoring y Alertas

```yaml
# monitoring/alerts.yml - Alertas de Prometheus
groups:
  - name: tienda-cqrs-alerts
    rules:
      - alert: ApplicationDown
        expr: up{job="tienda-app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Aplicación Tienda CQRS caída"
          description: "La aplicación no responde por más de 1 minuto"

      - alert: HighErrorRate
        expr: rate(http_server_requests_total{status=~"5.."}[5m]) > 0.1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Alta tasa de errores 5xx"
          description: "Más del 10% de requests devuelven error 5xx"

      - alert: EventProcessingLag
        expr: spring_modulith_events_outstanding > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Lag en procesamiento de eventos"
          description: "Hay más de 100 eventos pendientes de procesar"

      - alert: DatabaseConnectionsHigh
        expr: hikaricp_connections_active / hikaricp_connections_max > 0.8
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "Alto uso de conexiones de BD"
          description: "Usando más del 80% del pool de conexiones"
```

### 3. Backup y Disaster Recovery

```bash
#!/bin/bash
# backup.sh - Script de backup automatizado

BACKUP_DIR="/var/backups/tienda-cqrs"
DB_NAME="tienda_db"
DB_USER="tienda_user"
RETENTION_DAYS=30

# Crear directorio de backup
mkdir -p $BACKUP_DIR

# Backup de base de datos
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_FILE="$BACKUP_DIR/tienda_db_$TIMESTAMP.sql"

pg_dump -h localhost -U $DB_USER -d $DB_NAME > $BACKUP_FILE

# Comprimir backup
gzip $BACKUP_FILE

# Limpiar backups antiguos
find $BACKUP_DIR -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup completado: ${BACKUP_FILE}.gz"
```

### 4. Migración Gradual a Microservicios

Si eventualmente decides migrar a microservicios, Spring Modulith facilita el proceso:

```java
// Ejemplo: Extraer módulo productos como microservicio

// 1. Crear nuevo proyecto Spring Boot
// 2. Copiar el módulo completo
// 3. Convertir eventos internos a mensajes externos (Kafka/RabbitMQ)
// 4. Actualizar el módulo original para usar HTTP calls

// En el monolito original:
@Component
public class ProductoServiceClient {
    
    private final WebClient webClient;
    
    public Optional<ProductoInfo> obtenerInfoBasica(Long id) {
        return webClient.get()
            .uri("/api/productos/{id}", id)
            .retrieve()
            .bodyToMono(ProductoInfo.class)
            .blockOptional();
    }
}
```

## Conclusión y Próximos Pasos

Esta guía te ha mostrado cómo implementar CQRS con Spring Modulith, creando una arquitectura que combina lo mejor de monolitos y microservicios:

### ✅ Lo que Logramos:
1. **Modularidad sin distribución**: Módulos independientes en un solo JAR
2. **CQRS bien implementado**: Separación clara entre escrituras y lecturas
3. **Event-driven architecture**: Comunicación asíncrona entre módulos
4. **Testing independiente**: Cada módulo se puede testear por separado
5. **Documentación automática**: Siempre actualizada con el código
6. **Observabilidad completa**: Trazabilidad como en microservicios
7. **Deployment sencillo**: Un solo artefacto, fácil de desplegar

### 🚀 Próximos Pasos Recomendados:

1. **Agregar más módulos**: `pedidos`, `inventario`, `notificaciones`
2. **Implementar seguridad**: JWT, OAuth2, autorización por módulo
3. **Optimizar performance**: Cache distribuido, read replicas
4. **Event sourcing**: Para auditabilía completa de cambios
5. **Sagas**: Para transacciones distribuidas entre módulos
6. **API Gateway**: Para exponer APIs públicas de manera unificada

### 🎯 Cuándo Migrar a Microservicios:

- Cuando un módulo necesite escalar independientemente
- Cuando equipos diferentes necesiten deployar independientemente
- Cuando la complejidad del negocio lo justifique
- Cuando tengas la infraestructura para manejar la complejidad distribuida

Spring Modulith te da la flexibilidad de empezar simple y evolucionar según tus necesidades reales, no según las modas tecnológicas.

¡Happy coding! 🎉