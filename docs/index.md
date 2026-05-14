# Workshop: Monolito Modular Real con Spring Modulith

## Bienvenido al Taller

**Duración**: ~2.5 horas  
**Nivel**: Intermedio — Avanzado  
**Prerequisitos**: Spring Boot, JPA/Hibernate, Docker básico

Este workshop no es un tutorial de "hola mundo con módulos".
Es el camino que recorre un proyecto real: empezamos de una aplicación **acoplada con package-by-layer**, la rompemos intencionalmente, entendemos cada error de arquitectura que Spring Modulith nos reporta, y construimos paso a paso un sistema que podría estar en producción la semana que viene.

---

## El Problema que Resolvemos

Esto es lo que pasa en la mayoría de proyectos Spring Boot pasados 6 meses:

```java
// OrderService.java — después de "solo añadir una cosa rápida" 15 veces
@Service
public class OrderService {

    @Autowired private OrderRepository orderRepo;
    @Autowired private ProductRepository productRepo;   // ¿por qué está aquí?
    @Autowired private UserRepository userRepo;         // esto tampoco debería
    @Autowired private EmailService emailService;       // ni esto
    @Autowired private InventoryRepository invRepo;     // 😬
    @Autowired private NotificationService notifService;// ya fue
}
```

Nadie lo planificó. Ocurrió bajo presión de entrega, con buenas intenciones, un ticket a la vez.

### ¿Microservicios entonces?

Quizás no todavía. Los microservicios resuelven el acoplamiento pero añaden una complejidad enorme:

- Latencia de red entre servicios
- Transacciones distribuidas (Saga, 2PC)
- Observabilidad distribuida obligatoria
- Múltiples repositorios, pipelines CI/CD, bases de datos
- El debugging de un bug que cruza 5 servicios a las 2am

**Spring Modulith** es el punto medio: módulos con fronteras claras y verificadas automáticamente, todo en un solo JAR, sin la complejidad de los sistemas distribuidos.

---

## Lo que Construiremos

Una **Bookstore** — tienda de libros técnicos — con tres módulos independientes:

```
bookstore-modulith/
├── catalog/     ← Gestión del catálogo (con CQRS)
├── orders/      ← Ciclo de vida de órdenes
└── inventory/   ← Control de stock
```

Al final del workshop tendrás:

=== "Arquitectura"
    - Módulos con fronteras verificadas automáticamente
    - CQRS aplicado al módulo `catalog` (read model / write model separados)
    - Comunicación por eventos entre módulos (sin acoplamiento directo)
    - Schemas separados por módulo en PostgreSQL

=== "Eventos"
    - Event Publication Registry (Outbox Pattern sin infraestructura extra)
    - Eventos externalizados a RabbitMQ con `@Externalized`
    - Reintentos automáticos ante fallos transitorios

=== "Tests"
    - Verificación de arquitectura con `ModularityTest`
    - Tests de módulos en aislamiento con `@ApplicationModuleTest`
    - Verificación de eventos publicados con `AssertablePublishedEvents`
    - Tests de flujos event-driven con `Scenario`

=== "Observabilidad"
    - Documentación C4 Model generada desde el código
    - Trazabilidad entre módulos con Zipkin
    - Métricas con Actuator + Prometheus

---

## Stack Técnico

| Tecnología | Versión | Para qué |
|---|---|---|
| Java | 25 (LTS) | Lenguaje — Records, Sealed interfaces, Text Blocks |
| Spring Boot | 4.0.6 | Framework base |
| Spring Modulith | 2.0.6 | Modularidad, eventos, tests |
| PostgreSQL | 17 | Persistencia con schemas por módulo |
| Flyway | — | Migraciones versionadas |
| RabbitMQ | 3.13 | Externalización de eventos |
| Testcontainers | — | Tests de integración reales |
| Zipkin | 3 | Distributed tracing |

---

## Prerequisitos Técnicos

```bash
# Verificar entorno
java -version     # Debe mostrar openjdk 25+
mvn -version      # Maven 3.9+ (o usar ./mvnw del proyecto)
docker --version  # Docker Desktop corriendo
```

### Clonar el proyecto base

```bash
git clone https://github.com/geovannymcode/bookstore-modulith.git
cd bookstore-modulith
```

### Levantar la infraestructura

```bash
docker compose up -d
# Levanta: Postgres:5432, RabbitMQ:5672/15672, Zipkin:9411
```

### Verificar que todo compila

```bash
./mvnw compile
# Debe completar sin errores
```

---

## Estructura del Workshop

| Parte | Título | Duración |
|---|---|---|
| [Parte 0](parte_0.md) | El antes: la app acoplada | 15 min |
| [Parte 1](parte_1.md) | Migración a módulos + Spring Modulith | 30 min |
| [Parte 2](parte_2.md) | Boundaries: violations, circulares, explícitas | 25 min |
| [Parte 3](parte_3.md) | CQRS en el módulo Catalog | 30 min |
| [Parte 4](parte_4.md) | Eventos, Outbox y externalización | 25 min |
| [Parte 5](parte_5.md) | Testing en aislamiento | 20 min |
| [Parte 6](parte_6.md) | C4 Docs, Observabilidad y Docker | 15 min |

---

## Para Quién Es Este Workshop

**Ideal para:**

- Desarrolladores Java con experiencia en Spring Boot que sienten que su código se está convirtiendo en un "gran bola de barro"
- Arquitectos evaluando Spring Modulith como alternativa a microservicios prematuros
- Tech leads que quieren introducir reglas arquitectónicas verificables en el equipo
- Equipos con un monolito existente que quieren modularizarlo gradualmente

**No recomendado si:**

- Estás aprendiendo Spring Boot desde cero
- Tu aplicación ya es una suite de microservicios funcionando bien
- El negocio tiene un solo dominio muy simple sin perspectivas de crecer

---

!!! tip "Consejo antes de empezar"
    No copies y pegues el código directamente. Escríbelo a mano o por lo menos léelo antes de ejecutarlo. Los errores que cometas en el camino son parte del aprendizaje — Spring Modulith tiene mensajes de error muy descriptivos y cada uno enseña algo.

**¡Comenzamos en la [Parte 0](parte_0.md)!**
