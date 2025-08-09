# Workshop: CQRS con Spring Modulith desde Cero

## Bienvenido al Taller Práctico

**Duración**: 1.5 horas  
**Nivel**: Intermedio  
**Prerequisitos**: Conocimientos básicos de Spring Boot y Java

## ¿Qué es Spring Modulith?

Spring Modulith es una solución arquitectónica que te permite construir **monolitos modulares** - aplicaciones que combinan la simplicidad del monolito con la organización clara de los microservicios.

### El Problema que Resuelve

¿Te ha pasado esto?

```java
// Al inicio: todo limpio y organizado
@RestController 
public class ProductController {
    private final ProductService productService;
}

// Después de 6 meses: bajo presión de entrega
@RestController
public class ProductController {
    @Autowired private ProductService productService;
    @Autowired private UserRepository userRepository;    // ¿Por qué está aquí?
    @Autowired private OrderService orderService;       // Esto no debería estar
    @Autowired private EmailService emailService;       // Tampoco esto
}
```

**¿Qué pasó?** Bajo presión, los desarrolladores toman atajos y el código se vuelve un "gran bola de barro".

### La Solución: Monolitos Modulares

Spring Modulith te da **reglas arquitectónicas automáticas** que previenen este deterioro:

- **Módulos independientes** con límites claros
- **Testing automático** de la arquitectura 
- **Comunicación controlada** entre módulos
- **Documentación automática** siempre actualizada

## ¿Qué Construiremos?

Durante el taller implementaremos una **tienda online** con arquitectura CQRS y módulos independientes:

```
📦 store-cqrs/
├── 🛍️ products/          # Catálogo de productos
│   ├── command/          # Operaciones de escritura
│   ├── query/            # Operaciones de lectura  
│   └── events/           # Comunicación entre módulos
├── 🔧 common/            # Utilidades compartidas
└── 📋 config/            # Configuración global
```

### Funcionalidades Implementadas

**Gestión de Productos**:
- Crear y actualizar productos
- Agregar reviews y calificaciones
- Consultas optimizadas por categoría y rating

**Arquitectura CQRS**:
- **Lado Command**: Modelos para escritura (consistencia)
- **Lado Query**: Modelos para lectura (performance)
- **Sincronización automática** via eventos

**Observabilidad**:
- Trazabilidad entre módulos con Zipkin
- Métricas y health checks
- Eventos externos con Kafka

## Estructura del Workshop

### Parte 1: Fundamentos (30 min)
- ¿Por qué Spring Modulith?
- Configuración del proyecto
- Primer módulo funcional
- Verificación de reglas arquitectónicas

### Parte 2: CQRS en Acción (45 min)  
- Implementación lado Command
- Implementación lado Query
- Eventos entre módulos
- Testing independiente

### Parte 3: Producción (15 min)
- Observabilidad con Zipkin
- Automatización con Taskfile
- Deployment con Docker
- Demo final completo

## Pre-requisitos Técnicos

### Software Necesario

- **Java 21+** 
- **Maven 3.8+**
- **Docker Desktop**
- **IDE** (IntelliJ IDEA, VS Code, Eclipse)
- **Git**

### Verificación del Entorno
```bash
# Verificar instalaciones
java -version    # Debe mostrar Java 17+
mvn -version     # Debe mostrar Maven 3.8+
docker --version # Docker funcionando
```

### Opcional: Taskfile
```bash
# Instalar Taskfile para automatización
# macOS: brew install go-task/tap/go-task
# Windows: choco install go-task
# Linux: sh -c "$(curl -ssL https://taskfile.dev/install.sh)"
```

## ¿Qué Aprenderás?

Al finalizar el workshop podrás:

### 🏗️ **Arquitectura**

- Diseñar monolitos modulares
- Implementar CQRS correctamente
- Definir límites de módulos claros
- Prevenir el "big ball of mud"

### 🔧 **Herramientas**

- Spring Modulith para modularidad
- Zipkin para trazabilidad
- Kafka para eventos externos
- Docker para deployment

### 🧪 **Buenas Prácticas**

- Testing independiente de módulos
- Automatización con Taskfile
- Documentación automática
- Observabilidad integrada

### 🚀 **Productividad**

- Setup de proyecto en minutos
- Pipeline de desarrollo completo
- Demo funcional desde el primer día

## Enfoque del Taller

### Aprendizaje Práctico

- **Menos teoría, más código**
- Cada concepto se implementa inmediatamente
- Proyecto funcional al final

### Explicaciones Claras

- **¿Qué es?** - Definiciones simples
- **¿Por qué?** - Problemas que resuelve
- **¿Cómo?** - Implementación paso a paso
- **¿Cuándo?** - Contexto de uso apropiado

### Herramientas Modernas

- Taskfile en lugar de scripts bash
- TestContainers para testing real
- Docker Compose para entorno completo

## Resultados Esperados

### Al Final del Workshop Tendrás:

**✅ Aplicación Funcional**
```bash
task demo  # Inicia todo el entorno automáticamente
```

**✅ APIs Funcionando**

- Crear productos: `POST /api/products`
- Ver catálogo: `GET /api/products`
- Agregar reviews: `POST /api/products/{id}/reviews`
- Productos por rating: `GET /api/products/by-rating`

**✅ Observabilidad Completa**

- Aplicación: http://localhost:8080
- Trazas: http://localhost:9411
- Health checks: http://localhost:8080/actuator/health
- Info de módulos: http://localhost:8080/actuator/modulith

**✅ Arquitectura Validada**

```bash
task test:modulith  # Verifica reglas arquitectónicas
# ✅ Sin violaciones de encapsulamiento
# ✅ Sin dependencias circulares  
# ✅ APIs públicas bien definidas
```

## Comparación: Antes vs Después

### Desarrollo Tradicional

```java
// ❌ Código acoplado
@Service
class ProductService {
    @Autowired private OrderRepository orderRepo;     // ¿Por qué?
    @Autowired private UserService userService;       // Acoplamiento
    @Autowired private EmailService emailService;     // Responsabilidades mezcladas
}
```

### Con Spring Modulith

```java
// ✅ Módulos independientes
@Service
class ProductService {
    // Solo dependencias del dominio products
    private final ProductRepository productRepository;
    private final ApplicationEventPublisher eventPublisher;
    
    // Comunicación vía eventos, no dependencias directas
    eventPublisher.publishEvent(new ProductCreated(...));
}
```

## Para Quien es Este Workshop

### 👍 Ideal para:

- **Desarrolladores Java** con experiencia en Spring Boot
- **Arquitectos de software** evaluando alternativas a microservicios
- **Tech leads** buscando mejorar estructura de monolitos existentes
- **Equipos** que quieren modularidad sin complejidad distribuida

### 👎 No recomendado para:

- Principiantes en Spring Boot
- Proyectos que ya son microservicios exitosos
- Equipos con infrastructure muy limitada
- Aplicaciones con un solo dominio muy simple

## Recursos del Workshop

### Código Fuente

- Repository con todo el código
- Commits por cada paso del workshop
- Branches para cada parte

### Documentación

- Guías detalladas paso a paso
- Diagramas de arquitectura generados automáticamente
- Comandos de referencia rápida

### Herramientas de Práctica

```bash
# Comandos principales que usaremos
task                    # Ejecutar tests
task dev               # Entorno de desarrollo  
task demo              # Demo completo
task docs              # Generar documentación
task test:modulith     # Verificar arquitectura
```

## Próximos Pasos Después del Workshop

### Inmediatos

1. **Experimentar** con más módulos (`orders`, `inventory`)
2. **Aplicar** los conceptos en proyectos reales
3. **Compartir** conocimientos con el equipo

### A Mediano Plazo

1. **Migrar** monolitos existentes gradualmente
2. **Implementar** observabilidad en proyectos actuales
3. **Evaluar** cuándo extraer módulos como microservicios

### Recursos Adicionales

- [Documentación oficial de Spring Modulith](https://docs.spring.io/spring-modulith/reference/index.html)
- [Ejemplos y workshops](https://github.com/spring-projects/spring-modulith)
- [Comunidad y discusiones](https://github.com/spring-projects/spring-modulith/discussions)

---

## ¡Comencemos!

¿Listo para construir monolitos que no se conviertan en pesadillas?

**[👉 Ir a Parte 1: Configuración y Primer Módulo](./parte1.md)**

### Estructura del Workshop

1. **[Parte 1: Fundamentos](./parte1.md)** - Setup y primer módulo
2. **[Parte 2: CQRS Completo](./parte2.md)** - Command, Query y Events  
3. **[Parte 3: Observabilidad y Deploy](./parte3.md)** - Zipkin, Kafka y Docker

### Información del Instructor

Este workshop está diseñado para ser autoconducido, pero si tienes preguntas:

- Revisa la documentación paso a paso en cada parte
- Usa los comandos de verificación incluidos
- El código final está disponible como referencia

**¡Que disfrutes construyendo arquitecturas limpias y mantenibles!**
