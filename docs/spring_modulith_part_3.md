# Guía CQRS Spring Modulith - Parte 3: Observabilidad y Deployment

## Continuando desde la Parte 2

En la Parte 2 implementamos completamente el patrón CQRS con Spring Modulith. Ahora vamos a agregar observabilidad, automatización y deployment para tener un sistema completo listo para producción.

## Tabla de Contenidos - Parte 3
1. [Observabilidad con Zipkin](#observabilidad-con-zipkin)
2. [Automatización con Taskfile](#automatización-con-taskfile)
3. [Eventos Externos con Kafka](#eventos-externos-con-kafka)
4. [Testing de Integración Avanzado](#testing-de-integración-avanzado)
5. [Deployment con Docker](#deployment-con-docker)
6. [Demo Final](#demo-final)
7. [Conclusión del Workshop](#conclusión-del-workshop)

## Observabilidad con Zipkin

### ¿Qué es Observabilidad?

**Definición simple**: La capacidad de entender qué está pasando dentro de tu aplicación mientras funciona.

**Analogía**: Es como tener "rayos X" de tu aplicación para ver cómo fluyen las peticiones y datos.

### ¿Por qué Necesitamos Observabilidad en Módulos?

**Problema sin observabilidad**:
```java
// Usuario reporta: "Agregué un review pero no se actualiza el promedio"
// ¿Dónde está el problema?
// - ¿Se guardó el review?
// - ¿Se publicó el evento?
// - ¿Se procesó el evento?
// - ¿Se actualizó la vista?
// NO LO SABEMOS
```

**Con observabilidad**:
```
Traza completa:
1. POST /products/123/reviews → 200ms
2. ProductCommandService.addReview → 50ms
3. ProductRepository.save → 30ms
4. EventPublisher.publishEvent → 5ms
5. ProductEventHandler.on → 80ms ← AQUÍ FALLÓ
6. ProductViewRepository.save → NO EJECUTADO
```

### ¿Qué es Zipkin?

**Definición**: Zipkin es una herramienta que rastrea peticiones a través de sistemas distribuidos.

**¿Para qué sirve?**
- Ver el tiempo que toma cada operación
- Identificar cuellos de botella
- Debuggear problemas de performance
- Entender el flujo de datos entre módulos

**¿Por qué Zipkin y no logs?**

**Logs tradicionales**:
```
2024-01-15 10:30:01 INFO ProductService - Creating product
2024-01-15 10:30:01 INFO EventPublisher - Publishing event
2024-01-15 10:30:02 INFO ProductHandler - Processing event
```
**Problema**: ¿Qué evento pertenece a qué producto? ¿Cuánto tiempo tomó en total?

**Zipkin (Distributed Tracing)**:
```
Trace ID: abc123
├─ Span 1: ProductService.createProduct [100ms]
├─ Span 2: ProductRepository.save [30ms]
└─ Span 3: ProductEventHandler.on [70ms]
```
**Ventaja**: Ves el flujo completo con tiempos exactos y relaciones.

### ¿Cómo Funciona la Trazabilidad?

**Conceptos clave**:

#### 1. Trace (Traza)
**Definición**: El viaje completo de una petición a través de tu sistema.

**Ejemplo**: "Crear un producto con review"
```
Trace: "POST /products + POST /reviews"
├─ Crear producto
├─ Publicar evento ProductCreated  
├─ Actualizar vista de producto
├─ Agregar review
├─ Publicar evento ProductReviewed
└─ Actualizar estadísticas
```

#### 2. Span
**Definición**: Una operación individual dentro de una traza.

**Ejemplo**: "Guardar en base de datos"
- Inicio: 10:30:01.100
- Fin: 10:30:01.130
- Duración: 30ms
- Operación: ProductRepository.save

#### 3. Trace ID y Span ID
**Definición**: Identificadores únicos para correlacionar operaciones.

**Analogía**: Como el número de seguimiento de un paquete que te permite ver todos los pasos del envío.

### Configuración de Trazabilidad

#### Paso 1: Agregar Dependencias de Observabilidad

```xml
<!-- pom.xml - Agregar estas dependencias para observabilidad -->
<dependencies>
    <!-- Las dependencias existentes... -->
    
    <!-- 
    ¿Qué es? Puente entre Spring y Brave (biblioteca de tracing)
    ¿Para qué? Permite que Spring Boot genere automáticamente trazas
    ¿Sin esto? No habría trazabilidad entre métodos
    -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-brave</artifactId>
    </dependency>
    
    <!-- 
    ¿Qué es? Cliente que envía trazas a Zipkin
    ¿Para qué? Transporta las trazas desde tu app hasta Zipkin UI
    ¿Sin esto? Las trazas se quedan en memoria y no las ves
    -->
    <dependency>
        <groupId>io.zipkin.reporter2</groupId>
        <artifactId>zipkin-reporter-brave</artifactId>
    </dependency>
    
    <!-- 
    ¿Qué es? Endpoints de monitoreo de Spring Boot
    ¿Para qué? Expone /actuator/health, /actuator/modulith, etc.
    ¿Sin esto? No puedes ver el estado de tu aplicación externamente
    -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

#### Paso 2: Configuración Explicada

```yaml
# src/main/resources/application.yml - Agregar configuración de observabilidad
management:
  endpoints:
    web:
      exposure:
        # ¿Qué hace? Expone endpoints útiles vía HTTP
        # ¿Por qué estos? health=estado, modulith=información de módulos
        include: health,info,metrics,modulith
  endpoint:
    health:
      # ¿Qué hace? Muestra detalles de salud (BD, Kafka, etc.)
      # ¿Por qué? Para debugging rápido sin logs
      show-details: always
    modulith:
      # ¿Qué hace? Endpoint específico de Spring Modulith
      # ¿Para qué? Ver qué módulos tienes y sus dependencias
      enabled: true
  
  # Configuración de trazas
  tracing:
    sampling:
      # ¿Qué es? Porcentaje de peticiones que se rastrean
      # ¿Por qué 1.0? En desarrollo queremos ver todo
      # ¿En producción? 0.1 (10%) para no saturar
      probability: 1.0
  zipkin:
    tracing:
      # ¿Qué hace? URL donde Zipkin recibe las trazas
      # ¿Por qué esta URL? Es el endpoint estándar de Zipkin
      endpoint: http://localhost:9411/api/v2/spans

# Logging para ver IDs de traza en consola
logging:
  pattern:
    # ¿Qué hace? Agrega traceId y spanId a cada log
    # ¿Para qué? Correlacionar logs con trazas en Zipkin
    level: "%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]"
```

### ¿Qué Información Nos Da el Endpoint de Módulos?

```bash
# Ver información de módulos
curl http://localhost:8080/actuator/modulith
```

**Respuesta explicada**:
```json
{
  "modules": [
    {
      "name": "products",                    // Nombre del módulo
      "basePackage": "com.example.store.products",  // Paquete raíz
      "dependencies": ["common"],            // De qué módulos depende
      "exposedTypes": [                      // APIs públicas
        "ProductService", 
        "Product"
      ]
    }
  ],
  "violations": []                           // Reglas arquitectónicas violadas
}
```

**¿Para qué sirve esto?**
- **Documentación automática**: Siempre actualizada
- **Debugging de dependencias**: Ver quién depende de quién
- **Validación de arquitectura**: Detectar violaciones
- **Onboarding de nuevos desarrolladores**: Entender la estructura

## Automatización con Taskfile

### ¿Qué es Taskfile?

**Definición**: Taskfile es una herramienta de automatización que usa YAML en lugar de Makefiles o scripts bash.

**Analogía**: Es como tener un "control remoto" para tu proyecto donde cada botón ejecuta una secuencia de comandos.

### ¿Por Qué Taskfile en Lugar de Scripts Bash?

#### Problemas con Scripts Bash

**1. Multiplataforma**:
```bash
# archivo.sh - Solo funciona en Linux/Mac
#!/bin/bash
./mvnw clean test

# archivo.bat - Solo funciona en Windows  
@echo off
mvnw.cmd clean test
```

**2. Dependencias manuales**:
```bash
# build.sh
./test.sh          # ¿Qué pasa si test.sh falla?
./compile.sh       # ¿Se ejecuta igual?
./package.sh       # ¿En qué orden van?
```

**3. Manejo de errores complejo**:
```bash
# Complejo de leer y mantener
set -e
if ! ./mvnw test; then
    echo "Tests failed"
    exit 1
fi
if ! docker build .; then
    echo "Build failed"  
    exit 1
fi
```

#### Ventajas de Taskfile

**1. Multiplataforma automático**:
```yaml
# Funciona igual en Windows, Linux, Mac
vars:
  MVNW: '{{if eq .GOOS "windows"}}mvnw.cmd{{else}}./mvnw{{end}}'
```

**2. Dependencias declarativas**:
```yaml
build:
  deps: [test]        # Ejecuta 'test' antes de 'build'
  cmds:
    - "{{.MVNW}} package"
```

**3. Sintaxis clara**:
```yaml
# Fácil de leer y entender
test:
  desc: "Ejecuta todos los tests"
  cmds:
    - "{{.MVNW}} clean verify"
```

### Implementación del Taskfile

Crea `Taskfile.yml` en la raíz del proyecto:

```yaml
# Taskfile.yml
version: '3'

# Variables globales - Reutilizables en todo el archivo
vars:
  GOOS: "{{default OS .GOOS}}"                    # Sistema operativo
  MVNW: '{{if eq .GOOS "windows"}}mvnw.cmd{{else}}./mvnw{{end}}'  # Maven wrapper correcto
  DC_DIR: "deployment"                            # Directorio de Docker Compose
  APP_NAME: "store-cqrs"                         # Nombre de la aplicación

tasks:
  
  # =====================================
  # TAREAS DE DESARROLLO
  # =====================================
  
  default:
    # ¿Qué hace? Se ejecuta cuando solo escribes 'task'
    # ¿Por qué? Acción más común = ejecutar tests
    desc: "Ejecuta tests y verifica módulos"
    cmds:
      - task: test

  format:
    # ¿Qué hace? Formatea el código con Spotless
    # ¿Por qué? Mantiene estilo consistente sin discusiones
    desc: "Formatea el código"
    cmds:
      - "{{.MVNW}} spotless:apply"
  
  test:
    # ¿Qué hace? Ejecuta todos los tests
    # ¿Por qué deps? Siempre formatea antes de testear
    desc: "Ejecuta todos los tests"
    deps: [format]
    cmds:
      - "{{.MVNW}} clean verify"
  
  test:modulith:
    # ¿Qué hace? Solo verifica que la estructura modular sea correcta
    # ¿Cuándo usar? Para debugging rápido de arquitectura
    desc: "Verifica estructura modular"
    cmds:
      - "{{.MVNW}} test -Dtest=ModularityTest"
      - echo "✅ Estructura modular verificada"

  # =====================================
  # DOCUMENTACIÓN
  # =====================================
  
  docs:
    # ¿Qué hace? Genera diagramas de la estructura modular
    # ¿Por qué? Documentación siempre actualizada automáticamente
    desc: "Genera documentación de módulos"
    cmds:
      - "{{.MVNW}} test -Dtest=ModularityTest"
      - echo "📚 Documentación generada en target/spring-modulith-docs/"

  # =====================================
  # CONSTRUCCIÓN
  # =====================================
  
  build:
    # ¿Qué hace? Compila y empaqueta la aplicación
    # ¿Por qué deps? Asegura que tests pasan antes de compilar
    desc: "Construye la aplicación"
    deps: [test]
    cmds:
      - "{{.MVNW}} clean package -DskipTests"
  
  build:image:
    # ¿Qué hace? Crea imagen Docker de la aplicación
    # ¿Por qué? Para despliegue containerizado
    desc: "Construye imagen Docker"
    deps: [build]
    cmds:
      - "{{.MVNW}} spring-boot:build-image -DskipTests"
      - echo "🐳 Imagen construida: {{.APP_NAME}}:latest"

  # =====================================
  # INFRAESTRUCTURA
  # =====================================
  
  infra:start:
    # ¿Qué hace? Inicia solo los servicios base (BD, Zipkin, Kafka)
    # ¿Por qué separado? Para desarrollo local sin la app
    desc: "Inicia servicios base (PostgreSQL, Zipkin, Kafka)"
    cmds:
      - docker compose -f "{{.DC_DIR}}/docker-compose.yml" up -d postgres zipkin kafka
      - echo "🚀 Esperando servicios..."
      - task: infra:wait
      - echo "✅ Infraestructura lista"
  
  infra:wait:
    # ¿Qué hace? Espera a que todos los servicios estén listos
    # ¿Por qué? Evita errores de "connection refused"
    desc: "Espera a que los servicios estén listos"
    cmds:
      - |
        echo "⏳ Esperando PostgreSQL..."
        timeout 60 sh -c 'until docker compose -f {{.DC_DIR}}/docker-compose.yml exec -T postgres pg_isready -U store_user; do sleep 1; done'
        echo "⏳ Esperando Zipkin..."
        timeout 30 sh -c 'until curl -f http://localhost:9411/health; do sleep 1; done'
        echo "⏳ Esperando Kafka..."
        timeout 30 sh -c 'until docker compose -f {{.DC_DIR}}/docker-compose.yml exec -T kafka kafka-topics.sh --bootstrap-server localhost:9092 --list; do sleep 1; done'
    silent: true

  infra:stop:
    desc: "Detiene todos los servicios"
    cmds:
      - docker compose -f "{{.DC_DIR}}/docker-compose.yml" down

  infra:clean:
    desc: "Detiene servicios y limpia volúmenes"
    cmds:
      - docker compose -f "{{.DC_DIR}}/docker-compose.yml" down -v
      - docker system prune -f

  # =====================================
  # DESARROLLO LOCAL
  # =====================================
  
  dev:
    # ¿Qué hace? Prepara todo para desarrollo local
    # ¿Qué incluye? Solo infraestructura, la app se ejecuta con IDE
    desc: "Inicia entorno de desarrollo"
    cmds:
      - task: infra:start
      - echo "🔧 Entorno listo. Ejecuta: {{.MVNW}} spring-boot:run"
      - echo "📊 Zipkin: http://localhost:9411"
      - echo "🐘 PostgreSQL: localhost:5432"
      - echo "📡 Kafka: localhost:9092"

  # =====================================
  # DEMO COMPLETO
  # =====================================
  
  demo:
    desc: "Demo completo con todos los servicios"
    cmds:
      - task: build:image
      - docker compose -f "{{.DC_DIR}}/docker-compose.yml" up -d
      - task: infra:wait
      - echo "🎉 Demo listo!"
      - echo "🌐 Aplicación: http://localhost:8080"
      - echo "❤️ Health Check: http://localhost:8080/actuator/health"
      - echo "🗂️ Módulos: http://localhost:8080/actuator/modulith"
      - echo "📊 Zipkin: http://localhost:9411"

  demo:stop:
    desc: "Detiene el demo"
    cmds:
      - docker compose -f "{{.DC_DIR}}/docker-compose.yml" down

  demo:clean:
    desc: "Limpia completamente el demo"
    cmds:
      - docker compose -f "{{.DC_DIR}}/docker-compose.yml" down -v
      - docker image rm {{.APP_NAME}}:latest || true
```

### ¿Cómo se Usa en la Práctica?

```bash
# Comandos más comunes en desarrollo
task                    # = task default = tests
task dev               # Prepara entorno, luego usar IDE
task test:modulith     # Solo verificar arquitectura
task docs              # Generar documentación
task build             # Compilar aplicación
task infra:start       # Solo servicios base
task demo              # Demo completo
```

## Eventos Externos con Kafka

### ¿Qué es Kafka?

**Definición**: Apache Kafka es una plataforma de streaming de eventos distribuida.

**Analogía**: Es como un "buzón de correo masivo" donde:
- Los **productores** envían mensajes (eventos)
- Los **tópicos** organizan los mensajes por tema
- Los **consumidores** leen mensajes que les interesan

### ¿Por Qué Necesitamos Eventos Externos?

#### Problema: Comunicación con Sistemas Externos

**Escenario**: Tu aplicación de productos debe notificar a otros sistemas cuando algo importante pasa.

**Sistemas que pueden interesarse**:
- **Sistema de inventario**: Actualizar stock cuando se vende
- **Sistema de recomendaciones**: Ajustar algoritmos con nuevos reviews
- **Sistema de analytics**: Registrar métricas de productos
- **Sistema de notificaciones**: Enviar emails de nuevos productos

#### Opciones de Comunicación

**❌ Opción 1: Llamadas HTTP Directas**
```java
// Problema: Alto acoplamiento
productService.createProduct(...);
httpClient.post("http://inventory-service/update");      // ¿Y si está caído?
httpClient.post("http://analytics-service/track");       // ¿Y si es lento?
httpClient.post("http://recommendations-service/sync");  // ¿Y si cambia la URL?
```

**❌ Opción 2: Base de Datos Compartida**
```java
// Problema: Acoplamiento de datos
productRepository.save(product);
sharedDatabase.insert("inventory_updates", ...);  // Todos acceden a la misma BD
sharedDatabase.insert("analytics_events", ...);   // Violates microservices principles
```

**✅ Opción 3: Eventos con Kafka**
```java
// Solución: Bajo acoplamiento
productService.createProduct(...);
eventPublisher.publishEvent(new ProductCreated(...));  // Solo publicar
// Otros sistemas se suscriben si les interesa
// No conoces quién consume, no es tu problema
```

### ¿Cómo Funciona Kafka?

#### Conceptos Fundamentales

**1. Tópico (Topic)**
**Definición**: Un canal de comunicación para un tipo específico de evento.

**Ejemplo**:
```
Tópico: "products.created"
├─ Evento 1: {id: "123", name: "iPhone", price: 999}
├─ Evento 2: {id: "124", name: "MacBook", price: 1999}
└─ Evento 3: {id: "125", name: "iPad", price: 599}
```

**2. Productor (Producer)**
**Definición**: Aplicación que envía eventos a un tópico.

**En nuestro caso**: La aplicación Spring Modulith

**3. Consumidor (Consumer)**
**Definición**: Aplicación que lee eventos de un tópico.

**Ejemplos**:
- Servicio de inventario lee "products.created"
- Servicio de analytics lee "products.reviewed"
- Servicio de notificaciones lee ambos

#### Ventajas de Kafka vs Otras Opciones

| Aspecto | HTTP Directo | Base de Datos | Kafka |
|---------|-------------|---------------|-------|
| **Acoplamiento** | Alto | Alto | Bajo |
| **Disponibilidad** | Si un servicio cae, todo falla | BD puede saturarse | Resiliente |
| **Escalabilidad** | Limitada | Limitada | Muy alta |
| **Auditoría** | Difícil | Manual | Automática |
| **Reprocessing** | No | Difícil | Fácil |

### Configuración de Kafka

#### Paso 1: Agregar Dependencias de Kafka

```xml
<!-- pom.xml - Agregar dependencias de Kafka -->
<dependency>
    <!-- ¿Qué es? Cliente oficial de Kafka para Spring -->
    <!-- ¿Para qué? Enviar y recibir mensajes de Kafka -->
    <!-- ¿Sin esto? No hay comunicación con Kafka -->
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>

<dependency>
    <!-- ¿Qué es? Integración entre Spring Modulith y Kafka -->
    <!-- ¿Para qué? Publicar eventos internos automáticamente a Kafka -->
    <!-- ¿Sin esto? Tendrías que publicar manualmente a Kafka -->
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-events-kafka</artifactId>
</dependency>
```

#### Paso 2: Configuración Explicada

```yaml
# application.yml - Agregar configuración de Kafka
spring:
  kafka:
    # ¿Qué es? Dirección del cluster de Kafka
    # ¿Por qué localhost:9092? Puerto estándar de Kafka
    bootstrap-servers: localhost:9092
    
    producer:
      # ¿Qué hace? Serializa la clave del mensaje
      # ¿Por qué String? Claves simples como "product-123"
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      
      # ¿Qué hace? Convierte objetos Java a JSON
      # ¿Por qué JSON? Formato estándar, legible, interoperable
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    
    consumer:
      # ¿Qué es? Identificador del grupo de consumidores
      # ¿Para qué? Kafka garantiza que solo uno del grupo procese cada mensaje
      group-id: store-cqrs
      
      # ¿Qué hace? Lee claves como String
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      
      # ¿Qué hace? Convierte JSON de vuelta a objetos Java
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      
      properties:
        # ¿Qué es? Paquetes Java confiables para deserialización
        # ¿Por qué? Seguridad: evita crear objetos de paquetes maliciosos
        spring.json.trusted.packages: "com.example.store"

  modulith:
    events:
      externalization:
        # ¿Qué hace? Habilita publicación automática a sistemas externos
        # ¿Cómo? Eventos marcados con @Externalized van a Kafka
        enabled: true
```

#### Paso 3: Configurar Qué Eventos Van a Kafka

```java
// src/main/java/com/example/store/config/EventsConfig.java
package com.example.store.config;

import com.example.store.products.command.ProductEvents.ProductCreated;
import com.example.store.products.command.ProductEvents.ProductReviewed;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.modulith.events.config.EventExternalizationConfiguration;

/**
 * Configuración para decidir qué eventos se publican externamente.
 */
@Configuration
public class EventsConfig {
    
    @Bean
    EventExternalizationConfiguration eventExternalizationConfiguration() {
        return EventExternalizationConfiguration.externalizing()
            // ProductCreated va al tópico "products.created"
            .route(ProductCreated.class).to("products.created")
            
            // ProductReviewed va al tópico "products.reviewed"  
            .route(ProductReviewed.class).to("products.reviewed")
            
            // ProductUpdated NO se incluye = solo interno
            .build();
    }
}
```

**¿Por qué algunos eventos van a Kafka y otros no?**

**A Kafka** (eventos que interesan a otros sistemas):
- `ProductCreated`: Inventario necesita saber para crear stock
- `ProductReviewed`: Analytics necesita para métricas

**Solo internos** (eventos de implementación):
- `ProductUpdated`: Solo le importa a nuestra aplicación

### ¿Qué Pasa Cuando Publicas un Evento?

**Flujo completo**:

1. **Comando ejecutado**:
   ```java
   productService.createProduct("iPhone", ...);
   ```

2. **Evento publicado internamente**:
   ```java
   eventPublisher.publishEvent(new ProductCreated(...));
   ```

3. **Spring Modulith actúa**:
   - Busca listeners internos → Ejecuta `ProductEventHandler.on()`
   - Ve configuración → Envía también a Kafka

4. **En Kafka**:
   ```json
   Tópico: "products.created"
   Mensaje: {
     "id": "123",
     "name": "iPhone 15",
     "price": 999.99,
     "category": "Electronics"
   }
   ```

5. **Sistemas externos**:
   - Servicio de inventario consume y crea stock inicial
   - Servicio de analytics registra nueva categoria
   - Servicio de recomendaciones actualiza algoritmos

## Testing de Integración Avanzado

### Testing con Kafka

Para testear la integración con Kafka, necesitamos tests que verifiquen que los eventos se publican correctamente.

```java
// src/test/java/com/example/store/events/KafkaIntegrationTest.java
package com.example.store.events;

import com.example.store.products.command.Product.ProductIdentifier;
import com.example.store.products.command.ProductCommandService;
import com.example.store.products.command.ProductEvents.ProductCreated;
import lombok.RequiredArgsConstructor;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.kafka.core.DefaultKafkaConsumerFactory;
import org.springframework.kafka.listener.ContainerProperties;
import org.springframework.kafka.listener.KafkaMessageListenerContainer;
import org.springframework.kafka.listener.MessageListener;
import org.springframework.kafka.support.serializer.JsonDeserializer;
import org.springframework.kafka.test.EmbeddedKafkaBroker;
import org.springframework.kafka.test.context.EmbeddedKafka;
import org.springframework.kafka.test.utils.KafkaTestUtils;
import org.springframework.test.annotation.DirtiesContext;

import java.math.BigDecimal;
import java.time.Duration;
import java.util.Map;
import java.util.concurrent.BlockingQueue;
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Test de integración para verificar que eventos se publican a Kafka.
 * 
 * ¿Qué hace @EmbeddedKafka?
 * - Levanta un broker de Kafka real para testing
 * - No necesita Docker durante tests
 * - Aislado por test
 */
@SpringBootTest
@EmbeddedKafka(topics = {"products.created", "products.reviewed"})
@DirtiesContext  // Limpia contexto después de cada test
@RequiredArgsConstructor
class KafkaIntegrationTest {
    
    private final ProductCommandService productCommandService;
    private final EmbeddedKafkaBroker embeddedKafka;
    
    @Test
    void shouldPublishProductCreatedEventToKafka() throws InterruptedException {
        // Arrange: Configurar consumidor de Kafka para test
        Map<String, Object> consumerProps = KafkaTestUtils.consumerProps("test-group", "true", embeddedKafka);
        consumerProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        consumerProps.put(JsonDeserializer.TRUSTED_PACKAGES, "com.example.store");
        
        var consumerFactory = new DefaultKafkaConsumerFactory<String, ProductCreated>(consumerProps);
        var container = new KafkaMessageListenerContainer<>(consumerFactory, 
            new ContainerProperties("products.created"));
        
        BlockingQueue<ConsumerRecord<String, ProductCreated>> records = new LinkedBlockingQueue<>();
        container.setupMessageListener((MessageListener<String, ProductCreated>) records::add);
        container.start();
        
        // Act: Crear producto
        ProductIdentifier productId = productCommandService.createProduct(
            "Test Product", "Description", new BigDecimal("99.99"), 10, "Electronics"
        );
        
        // Assert: Verificar que evento llegó a Kafka
        ConsumerRecord<String, ProductCreated> record = records.poll(10, TimeUnit.SECONDS);
        assertThat(record).isNotNull();
        assertThat(record.value().id()).isEqualTo(productId);
        assertThat(record.value().name()).isEqualTo("Test Product");
        
        // Cleanup
        container.stop();
    }
}
```

### Testing de Observabilidad

```java
// src/test/java/com/example/store/observability/TracingTest.java
package com.example.store.observability;

import com.example.store.products.command.ProductCommandService;
import io.micrometer.tracing.TraceContext;
import io.micrometer.tracing.Tracer;
import lombok.RequiredArgsConstructor;
import org.junit.jupiter.api.Test;
import org.springframework.modulith.test.ApplicationModuleTest;

import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Test para verificar que la trazabilidad funciona correctamente.
 */
@ApplicationModuleTest
@RequiredArgsConstructor
class TracingTest {
    
    private final ProductCommandService productCommandService;
    private final Tracer tracer;
    
    @Test
    void shouldCreateTraceWhenCreatingProduct() {
        // Arrange: Verificar que no hay traza activa
        assertThat(tracer.currentTraceContext().context()).isNull();
        
        // Act: Crear producto dentro de una traza
        var span = tracer.nextSpan().name("test-create-product").start();
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            TraceContext context = tracer.currentTraceContext().context();
            assertThat(context).isNotNull();
            
            productCommandService.createProduct(
                "Traced Product", "Description", new BigDecimal("49.99"), 5, "Test"
            );
            
            // Assert: Verificar que la traza sigue activa
            assertThat(tracer.currentTraceContext().context()).isEqualTo(context);
            
        } finally {
            span.end();
        }
    }
}
```

## Deployment con Docker

### ¿Qué es Docker?

**Definición**: Docker es una plataforma que permite empaquetar aplicaciones y sus dependencias en contenedores.

**Analogía**: Es como un "contenedor de envío" que garantiza que tu aplicación funcione igual en cualquier lugar.

### ¿Por qué Docker para Desarrollo?

#### Problemas Sin Docker

**"Funciona en Mi Máquina"**:
```
Desarrollador A: Java 17, PostgreSQL 14, Ubuntu
Desarrollador B: Java 11, PostgreSQL 13, Windows  
Servidor QA: Java 18, PostgreSQL 15, CentOS
Producción: Java 17, PostgreSQL 16, AWS Linux

¿Resultado? Comportamientos diferentes en cada ambiente
```

**Instalación Manual**:
```bash
# Cada desarrollador debe instalar:
- PostgreSQL (¿qué versión?)
- Kafka (¿cómo configurarlo?)  
- Zipkin (¿dónde descargarlo?)
- Java (¿OpenJDK vs Oracle?)
```

#### Ventajas con Docker

**Ambiente Consistente**:
```yaml
# Todos usan exactamente:
postgres:15-alpine    # Misma versión de PostgreSQL
confluentinc/cp-kafka # Misma distribución de Kafka  
openzipkin/zipkin     # Misma versión de Zipkin
```

**Instalación Simple**:
```bash
# Solo necesitas:
task dev              # Todo se instala automáticamente
```

### Docker Compose Explicado

Crea `deployment/docker-compose.yml`:

```yaml
# deployment/docker-compose.yml
version: '3.8'

services:
  # =====================================
  # BASE DE DATOS
  # =====================================
  postgres:
    # ¿Qué es? Imagen oficial de PostgreSQL optimizada (Alpine = pequeña)
    image: postgres:15-alpine
    
    # ¿Qué es? Nombre del contenedor (para referenciarlo fácilmente)
    container_name: store-postgres
    
    # ¿Qué son? Variables que PostgreSQL lee al iniciar
    environment:
      POSTGRES_DB: store_db          # Nombre de la base de datos
      POSTGRES_USER: store_user      # Usuario que creará
      POSTGRES_PASSWORD: store_password  # Contraseña del usuario
    
    # ¿Qué hace? Mapea puerto del contenedor al host
    # host:contenedor -> localhost:5432 apunta al puerto 5432 del contenedor
    ports:
      - "5432:5432"
    
    # ¿Qué es? Almacenamiento persistente
    # Sin esto: Datos se pierden al borrar el contenedor
    volumes:
      - postgres_data:/var/lib/postgresql/data
    
    # ¿Qué hace? Verifica si el servicio está listo
    # ¿Por qué? Otros servicios esperan hasta que PostgreSQL responda
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U store_user -d store_db"]
      interval: 10s      # Cada 10 segundos
      timeout: 5s        # Máximo 5 segundos de espera
      retries: 5         # Máximo 5 intentos
    
    # ¿Qué es? Red virtual para que contenedores se comuniquen
    networks:
      - store-network

  # =====================================
  # TRAZABILIDAD
  # =====================================
  zipkin:
    # ¿Qué es? Imagen oficial de Zipkin para trazabilidad
    image: openzipkin/zipkin:latest
    container_name: store-zipkin
    
    ports:
      # Puerto estándar de Zipkin
      - "9411:9411"
    
    environment:
      # ¿Qué hace? Usa memoria en lugar de base de datos
      # ¿Por qué? Más simple para desarrollo (datos no persisten)
      - STORAGE_TYPE=mem
    
    healthcheck:
      # ¿Cómo verifica? Hace petición HTTP al endpoint de salud
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:9411/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - store-network

  # =====================================
  # KAFKA
  # =====================================
  zookeeper:
    # ¿Qué es Zookeeper? Servicio que Kafka necesita para coordinación
    # ¿Por qué lo necesitamos? Kafka depende de él (por ahora)
    image: confluentinc/cp-zookeeper:latest
    container_name: store-zookeeper
    environment:
      # Puerto estándar donde Kafka se conecta a Zookeeper
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"
    networks:
      - store-network

  kafka:
    # ¿Qué es? Imagen de Kafka de Confluent (empresa que mantiene Kafka)
    image: confluentinc/cp-kafka:latest
    container_name: store-kafka
    
    environment:
      # ¿Dónde está Zookeeper? (Kafka necesita esta info)
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      
      # ¿Cómo anunciar la dirección? Clientes se conectan a localhost:9092
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      
      # ¿Cuántas réplicas? 1 para desarrollo (normalmente 3 en producción)
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      
      # ¿Crear tópicos automáticamente? Sí, para facilitar desarrollo
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
    
    ports:
      - "9092:9092"    # Puerto estándar de Kafka
    
    # ¿Orden de inicio? Kafka necesita que Zookeeper esté listo primero
    depends_on:
      - zookeeper
    
    healthcheck:
      # ¿Cómo verificar? Listar tópicos (si funciona, Kafka está listo)
      test: ["CMD", "kafka-topics", "--bootstrap-server", "localhost:9092", "--list"]
      interval: 30s
      timeout: 10s
      retries: 5
    networks:
      - store-network

  # =====================================
  # APLICACIÓN PRINCIPAL
  # =====================================
  app:
    # ¿De dónde viene? Se construye con task build:image
    image: store-cqrs:latest
    container_name: store-app
    
    # ¿Orden de inicio? App necesita que todos los servicios estén listos
    depends_on:
      postgres:
        condition: service_healthy    # Espera hasta que PostgreSQL responda
      zipkin:
        condition: service_healthy    # Espera hasta que Zipkin responda  
      kafka:
        condition: service_healthy    # Espera hasta que Kafka responda
    
    # ¿Configuración? Variables que Spring Boot lee al iniciar
    environment:
      # ¿Qué profile? 'docker' tiene configuración específica para contenedores
      SPRING_PROFILES_ACTIVE: docker
      
      # ¿Base de datos? 'postgres' = nombre del servicio, no 'localhost'
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/store_db
      SPRING_DATASOURCE_USERNAME: store_user
      SPRING_DATASOURCE_PASSWORD: store_password
      
      # ¿Zipkin? 'zipkin' = nombre del servicio en la red de Docker
      MANAGEMENT_ZIPKIN_TRACING_ENDPOINT: http://zipkin:9411/api/v2/spans
      
      # ¿Kafka? 'kafka' = nombre del servicio  
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    
    ports:
      - "8080:8080"    # Aplicación accesible en localhost:8080
    
    healthcheck:
      # ¿Está lista? Verificar endpoint de salud de Spring Boot
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s    # Esperar 60s antes del primer check (app tarda en iniciar)
    networks:
      - store-network

# =====================================
# ALMACENAMIENTO PERSISTENTE
# =====================================
volumes:
  postgres_data:
    # ¿Qué es? Volumen administrado por Docker
    # ¿Para qué? Los datos de PostgreSQL sobreviven al borrar contenedores
    driver: local

# =====================================
# REDES
# =====================================
networks:
  store-network:
    # ¿Qué tipo? Bridge = red local donde contenedores se ven entre sí
    # ¿Por qué? 'app' puede conectarse a 'postgres' por nombre
    driver: bridge
```

### Profile Específico para Docker

Crea `src/main/resources/application-docker.yml`:

```yaml
# application-docker.yml
# ¿Para qué? Configuración específica cuando app corre en contenedor

spring:
  datasource:
    # ¿Por qué 'postgres'? En Docker, servicios se conectan por nombre
    url: jdbc:postgresql://postgres:5432/store_db
    username: store_user
    password: store_password
  
  kafka:
    # ¿Por qué 'kafka'? Nombre del servicio en docker-compose.yml
    bootstrap-servers: kafka:9092
  
management:
  zipkin:
    tracing:
      # ¿Por qué 'zipkin'? Nombre del servicio en la red de Docker
      endpoint: http://zipkin:9411/api/v2/spans

logging:
  level:
    # ¿Por qué INFO? En contenedores queremos menos logs
    com.example.store: INFO
```

## Demo Final

### ¿Qué Incluye el Demo?

**El demo muestra el flujo completo**:
1. **Crear producto** → Ver evento interno + externo
2. **Agregar review** → Ver actualización de vista + Kafka
3. **Verificar trazabilidad** → Zipkin muestra el flujo
4. **Validar módulos** → Endpoint de arquitectura

### Comandos del Demo

```bash
# 1. Demo completo automatizado
task demo

# Lo que hace internamente:
# - task build:image (construye app)
# - docker compose up (inicia todo)
# - Espera servicios listos
# - Muestra URLs importantes
```

### APIs para Probar Manualmente

```bash
# 1. Crear producto
curl -X POST http://localhost:8080/api/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "iPhone 15 Pro",
    "description": "Latest iPhone model",
    "price": 999.99,
    "stock": 50,
    "category": "Electronics"
  }'

# Respuesta esperada:
# {"id": "550e8400-e29b-41d4-a716-446655440000"}

# 2. Ver todos los productos
curl http://localhost:8080/api/products

# 3. Agregar review (reemplaza {id} con el ID real)
curl -X POST http://localhost:8080/api/products/{id}/reviews \
  -H "Content-Type: application/json" \
  -d '{
    "vote": 5,
    "comment": "Excelente producto!"
  }'

# 4. Ver productos ordenados por rating
curl http://localhost:8080/api/products/by-rating
```

### ¿Qué Verificar en el Demo?

#### 1. Aplicación Funcionando
```bash
# Health check
curl http://localhost:8080/actuator/health

# Debe devolver:
{
  "status": "UP",
  "components": {
    "db": {"status": "UP"},
    "kafka": {"status": "UP"}
  }
}
```

#### 2. Información de Módulos
```bash
# Ver estructura modular
curl http://localhost:8080/actuator/modulith

# Debe mostrar:
{
  "modules": [
    {"name": "products", "dependencies": ["common"]},
    {"name": "common", "dependencies": []}
  ],
  "violations": []  # ← ¡Importante! Sin violaciones
}
```

#### 3. Trazabilidad en Zipkin
- **URL**: http://localhost:9411
- **Qué buscar**: Trazas que muestran:
  ```
  POST /api/products
  ├─ ProductCommandService.createProduct
  ├─ ProductRepository.save
  ├─ ApplicationEventPublisher.publishEvent
  └─ ProductEventHandler.on (async)
  ```

#### 4. Eventos en Kafka
```bash
# Ver tópicos creados
docker exec store-kafka kafka-topics --bootstrap-server localhost:9092 --list

# Debe mostrar:
# products.created
# products.reviewed

# Consumir eventos (para ver qué se envió)
docker exec store-kafka kafka-console-consumer \
  --bootstrap-server localhost:9092 \
  --topic products.created \
  --from-beginning

# Debe mostrar eventos JSON:
# {"id":"123","name":"iPhone 15 Pro",...}
```

### URLs del Demo

Después de `task demo`, estos endpoints están disponibles:

- **🌐 Aplicación**: http://localhost:8080
- **❤️ Health Check**: http://localhost:8080/actuator/health  
- **🗂️ Información de Módulos**: http://localhost:8080/actuator/modulith
- **📊 Trazas Zipkin**: http://localhost:9411

### Limpieza del Demo

```bash
# Detener demo (mantiene datos)
task demo:stop

# Limpiar completamente (borra datos)
task demo:clean
```

## Conclusión del Workshop

### ¿Qué Hemos Logrado?

**✅ Modularidad sin Complejidad**
- Módulos independientes dentro de un monolito
- Reglas arquitectónicas automáticas
- Testing independiente por módulo

**✅ CQRS Funcional**  
- Separación clara comandos vs queries
- Modelos optimizados para cada lado
- Sincronización automática vía eventos

**✅ Observabilidad Completa**
- Trazabilidad como en microservicios
- Información de módulos en tiempo real
- Debugging simplificado

**✅ Integración Externa**
- Eventos a Kafka para otros sistemas
- Bajo acoplamiento entre aplicaciones
- Escalabilidad futura

**✅ Automatización Total**
- Un comando inicia todo el entorno
- Docker garantiza consistencia
- Deployment simplificado

### ¿Cuándo Usar Spring Modulith?

**👍 Ideal para:**
- Equipos de 2-10 desarrolladores
- Aplicaciones con dominios claros
- Necesidad de desarrollo rápido
- Infraestructura limitada
- Monolitos existentes que mejorar

**👎 No ideal para:**
- Equipos muy grandes (>20 personas)
- Dominios completamente independientes
- Requisitos de escalado muy específicos
- Tecnologías muy diferentes por área

### Próximos Pasos

1. **Practicar**: Implementa más módulos (`orders`, `inventory`)
2. **Profundizar**: Event Sourcing, Sagas, CQRS avanzado
3. **Optimizar**: Cache, performance, índices BD
4. **Migrar**: Cuando llegue el momento, extraer módulos a microservicios

### Evolución Gradual

**Fase 1: Monolito Modular** (donde estamos ahora)
- Módulos claros
- Eventos internos
- Testing independiente

**Fase 2: Híbrido** (cuando sea necesario)
- Algunos módulos extraídos
- Comunicación vía Kafka
- Bases de datos separadas

**Fase 3: Microservicios** (solo si es necesario)
- Servicios completamente independientes
- Infraestructura compleja
- Equipos especializados

### Comandos de Referencia Rápida

```bash
# Desarrollo diario
task                    # Tests completos
task dev               # Entorno de desarrollo
task test:modulith     # Solo verificar arquitectura

# Demo y deployment  
task demo              # Demo completo
task build:image       # Construir imagen Docker
task infra:start       # Solo servicios base

# Limpieza
task demo:clean        # Limpiar demo
task infra:clean       # Limpiar infraestructura
```

**Recuerda**: No hay arquitectura perfecta, solo arquitectura adecuada para tu contexto actual.

Spring Modulith te da la flexibilidad de empezar simple y evolucionar según tus necesidades reales, no según las modas tecnológicas.

¡Feliz codificación! 🎉