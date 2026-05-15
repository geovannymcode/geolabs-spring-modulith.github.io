# Parte 6 — C4 Docs, Observabilidad y Docker

**Duración**: 20 minutos  
**Objetivo**: Generar documentación de arquitectura desde el código, activar el endpoint de módulos en Actuator, configurar Zipkin correctamente y centralizar los comandos del proyecto en Taskfile

---

## ¿Qué hacemos aquí?

El sistema funciona, los tests pasan y la arquitectura está verificada. En esta parte lo terminamos de redondear:

1. Documentación C4 Model generada automáticamente desde el código
2. Actuator con el endpoint `/actuator/modulith` — la arquitectura como dato en runtime
3. Zipkin para trazar el flujo completo entre módulos
4. Taskfile para centralizar todos los comandos del proyecto

---

## Paso 1: Dependencias necesarias

Antes de empezar hay que agregar tres dependencias al `pom.xml` que habilitan las capacidades de esta parte.

### spring-modulith-actuator

Esta dependencia es la que activa el endpoint `/actuator/modulith`. Sin ella, el endpoint simplemente no existe aunque tengas Actuator en el classpath. Spring Modulith y Spring Boot Actuator son proyectos independientes — hay que conectarlos explícitamente.

```xml
<!-- pom.xml -->

<!-- Habilita /actuator/modulith — expone la estructura de módulos en runtime -->
<dependency>
    <groupId>org.springframework.modulith</groupId>
    <artifactId>spring-modulith-actuator</artifactId>
    <scope>runtime</scope>
</dependency>
```

### Tracing para Zipkin

Estas dos dependencias trabajan juntas. `micrometer-tracing-bridge-brave` es el puente entre Micrometer (el sistema de métricas de Spring Boot) y Brave (la librería de tracing de Zipkin). `zipkin-reporter-brave` es el que envía las trazas al servidor de Zipkin.

```xml
<!-- Puente entre Micrometer Tracing y Brave (librería de tracing de Zipkin) -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>

<!-- Envía las trazas al servidor Zipkin -->
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

El `pom.xml` con las tres dependencias nuevas completas:

```xml
<dependencies>
    <!-- ... dependencias existentes ... -->

    <!-- Spring Modulith Actuator -->
    <dependency>
        <groupId>org.springframework.modulith</groupId>
        <artifactId>spring-modulith-actuator</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- Tracing con Zipkin -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-brave</artifactId>
    </dependency>
    <dependency>
        <groupId>io.zipkin.reporter2</groupId>
        <artifactId>zipkin-reporter-brave</artifactId>
    </dependency>
</dependencies>
```

---

## Paso 2: Configurar application.properties

Con las dependencias en el classpath, hay que decirle a la aplicación cómo comportarse.

```properties
# ── Actuator ──────────────────────────────────────────────────────────
# Expone los endpoints necesarios para esta parte.
# "modulith" requiere spring-modulith-actuator en el classpath.
management.endpoints.web.exposure.include=health,info,metrics,modulith
management.endpoint.health.show-details=always
management.info.env.enabled=true

# Información que aparece en /actuator/info
info.app.name=Bookstore Modulith
info.app.version=1.0.0
info.app.description=Workshop BarranquillaJUG — Spring Modulith

# ── Zipkin ────────────────────────────────────────────────────────────
# 1.0 samplea el 100% de las peticiones — solo en desarrollo.
# En producción usar 0.1 (10%) para no saturar el servidor.
management.tracing.sampling.probability=1.0
management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans
```

---

## Paso 3: Agregar Zipkin al Docker Compose

Zipkin es el servidor que recibe las trazas de la aplicación y las almacena para consultarlas en su UI. Sin este contenedor corriendo, la app sigue funcionando pero las trazas se pierden — Spring Boot las envía y Zipkin no está ahí para recibirlas.

Actualiza `compose.yml` para incluir el contenedor de Zipkin:

```yaml
# compose.yml
services:
  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_DB: bookstore
      POSTGRES_USER: bookstore
      POSTGRES_PASSWORD: bookstore
    ports:
      - "5432:5432"
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U bookstore" ]
      interval: 10s
      timeout: 5s
      retries: 5

  rabbitmq:
    image: rabbitmq:3.13-management-alpine
    environment:
      RABBITMQ_DEFAULT_USER: guest
      RABBITMQ_DEFAULT_PASS: guest
    ports:
      - "5672:5672"
      - "15672:15672"
    healthcheck:
      test: rabbitmq-diagnostics check_port_connectivity
      interval: 10s
      timeout: 5s
      retries: 5

  zipkin:
    image: openzipkin/zipkin:3
    ports:
      - "9411:9411"   # UI y API: http://localhost:9411
```

### ¿Por qué Zipkin no tiene healthcheck?

Zipkin no necesita uno en este contexto. La app envía trazas de forma "fire and forget" — si Zipkin no está disponible, simplemente pierde esas trazas pero la aplicación sigue funcionando. No hay dependencia de startup entre la app y Zipkin.

---

## Paso 4: Por qué Zipkin no muestra nada y cómo resolverlo

Si abriste Zipkin en `http://localhost:9411` y ejecutaste la consulta sin resultados, hay una razón concreta.

La app envía trazas a `http://localhost:9411/api/v2/spans`. Pero cuando la app corre dentro del mismo proceso que gestiona `spring-boot-docker-compose`, el `localhost` de la app puede no ser el mismo `localhost` donde está Zipkin. Más frecuentemente, el problema es que la app arrancó antes de que Zipkin estuviera listo.

**Solución 1 — Arrancar en el orden correcto:**

```bash
# 1. Levanta Docker Compose primero (incluye Zipkin)
docker compose up -d

# 2. Espera a que los servicios estén listos
sleep 5

# 3. Arranca la app
mvn spring-boot:run
```

**Solución 2 — Deshabilitar spring-boot-docker-compose y manejarlo tú:**

```properties
# application.properties
spring.docker.compose.enabled=false
```

Luego siempre:
```bash
docker compose up -d
mvn spring-boot:run
```

**Verificar que las trazas se están enviando:**

En los logs de la app, busca estas líneas — confirman que el reporter de Zipkin está activo:

```
INFO  o.s.b.actuate.autoconfigure.tracing : Tracing is enabled
```

Si no aparece, revisa que `micrometer-tracing-bridge-brave` está en el `pom.xml`.

**Hacer una petición y buscar en Zipkin:**

```bash
# 1. Crea una orden
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerName": "Geo",
    "customerEmail": "geo@test.com",
    "customerPhone": "+57 300 0000000",
    "deliveryAddress": "Barranquilla",
    "items": [{"productCode": "P001", "quantity": 1}]
  }'

# 2. Abre Zipkin
open http://localhost:9411

# 3. En la barra de búsqueda selecciona el servicio "bookstore"
# 4. Haz clic en "Ejecutar Consulta"
```

!!! tip "Si el servicio no aparece en el dropdown"
    Zipkin solo muestra servicios que han enviado al menos una traza. Si acabas de arrancar la app y Zipkin, espera a que la primera petición llegue. A veces tarda 5-10 segundos en procesar.

---

## Paso 5: Verificar /actuator/modulith

Con `spring-modulith-actuator` en el classpath y configurado en Actuator, este endpoint devuelve la estructura de módulos en tiempo de ejecución:

```bash
curl http://localhost:8080/actuator/modulith | python3 -m json.tool
```

Respuesta esperada:

```json
{
  "modules": [
    {
      "name": "catalog",
      "basePackage": "com.geovannycode.bookstore.catalog",
      "type": "CLOSED",
      "dependencies": []
    },
    {
      "name": "orders",
      "basePackage": "com.geovannycode.bookstore.orders",
      "type": "CLOSED",
      "allowedDependencies": ["catalog", "common"],
      "dependencies": ["catalog", "common"]
    },
    {
      "name": "inventory",
      "basePackage": "com.geovannycode.bookstore.inventory",
      "type": "CLOSED",
      "dependencies": []
    },
    {
      "name": "common",
      "basePackage": "com.geovannycode.bookstore.common",
      "type": "OPEN",
      "dependencies": []
    }
  ]
}
```

Si el endpoint devuelve 404, el problema es que falta `spring-modulith-actuator` en el `pom.xml`. Verifica el Paso 1.

---

## Paso 6: Documentación C4 con ModularityTest

Spring Modulith puede generar diagramas PlantUML de la arquitectura directamente desde el código. Los diagramas siempre están actualizados porque se generan del análisis del bytecode — no los dibuja nadie a mano.

Actualiza `ModularityTest.java` con el método de generación:

```java
package com.geovannycode.bookstore;

import org.junit.jupiter.api.Test;
import org.springframework.modulith.core.ApplicationModules;
import org.springframework.modulith.docs.Documenter;

class ModularityTest {

    static final ApplicationModules modules =
            ApplicationModules.of(BookstoreApplication.class);

    @Test
    void verifiesModularStructure() {
        modules.verify();
    }

    /**
     * Genera documentación C4 en target/spring-modulith-docs/
     *
     * Archivos generados:
     *   - components.puml: diagrama global con todos los módulos
     *   - catalog.puml, orders.puml, inventory.puml: canvas por módulo
     *   - aggregating-document.adoc: todo el documento en AsciiDoc
     */
    @Test
    void writesDocumentationSnippets() {
        new Documenter(modules)
                .writeModulesAsPlantUml()
                .writeIndividualModulesAsPlantUml()
                .writeAggregatingDocument();
    }

    @Test
    void printsModuleStructure() {
        modules.forEach(System.out::println);
    }
}
```

Genera la documentación:

```bash
mvn test -Dtest=ModularityTest
```

Verifica los archivos generados:

```bash
ls target/spring-modulith-docs/
# components.puml
# catalog.puml
# orders.puml
# inventory.puml
# common.puml
# aggregating-document.adoc
```

### Visualizar los diagramas en IntelliJ

Tienes el plugin `plantuml4idea` instalado (imagen de referencia: plugin de Vojtěch Krása, 4.8M descargas). Para renderizar:

1. Abre cualquier archivo `.puml` en `target/spring-modulith-docs/`
2. El plugin renderiza el diagrama automáticamente en el panel de preview a la derecha
3. Si no aparece, usa el atajo `Alt+D` o el ícono de PlantUML en la barra de herramientas

Alternativamente, copia el contenido de cualquier `.puml` y pégalo en [plantuml.com](https://plantuml.com) para renderizarlo online.

---

## Paso 7: El Taskfile

Taskfile centraliza todos los comandos del ciclo de desarrollo en un solo lugar. Nadie tiene que memorizar flags de Maven ni rutas de Docker.

Crea `Taskfile.yml` en la raíz del proyecto:

```yaml
# Taskfile.yml
version: '3'

tasks:
  default:
    desc: "Ejecuta todos los tests"
    cmds:
      - mvn verify

  dev:
    desc: "Levanta Docker y arranca la aplicación"
    cmds:
      - docker compose up -d
      - sleep 3
      - mvn spring-boot:run

  infra:up:
    desc: "Levanta Postgres, RabbitMQ y Zipkin"
    cmds:
      - docker compose up -d

  infra:down:
    desc: "Baja la infraestructura"
    cmds:
      - docker compose down

  infra:reset:
    desc: "Baja la infraestructura y borra los volúmenes (reset completo)"
    cmds:
      - docker compose down -v

  infra:logs:
    desc: "Muestra logs de todos los contenedores"
    cmds:
      - docker compose logs -f

  test:
    desc: "Ejecuta todos los tests"
    cmds:
      - mvn test

  test:modulith:
    desc: "Verifica la arquitectura modular (ModularityTest)"
    cmds:
      - mvn test -Dtest=ModularityTest

  test:catalog:
    desc: "Tests del módulo catalog en aislamiento"
    cmds:
      - mvn test -Dtest=ProductRestControllerTests

  test:orders:
    desc: "Tests del módulo orders en aislamiento"
    cmds:
      - mvn test -Dtest=OrderRestControllerTests

  test:inventory:
    desc: "Tests del módulo inventory en aislamiento"
    cmds:
      - mvn test -Dtest=InventoryIntegrationTests

  docs:
    desc: "Genera documentación C4 en target/spring-modulith-docs/"
    cmds:
      - mvn test -Dtest=ModularityTest
      - echo "Documentación generada en target/spring-modulith-docs/"

  build:
    desc: "Compila y empaqueta el JAR (sin tests)"
    cmds:
      - mvn package -DskipTests

  clean:
    desc: "Limpia el directorio target"
    cmds:
      - mvn clean

  format:
    desc: "Formatea el código con Spotless"
    cmds:
      - mvn spotless:apply

  format:check:
    desc: "Verifica el formato del código sin modificarlo"
    cmds:
      - mvn spotless:check

  demo:
    desc: "Levanta todo el entorno para demostración"
    deps: [ infra:up ]
    cmds:
      - sleep 5
      - mvn spring-boot:run
```

### Instalar Taskfile

```bash
# macOS
brew install go-task

# Linux
sh -c "$(curl --location https://taskfile.dev/install.sh)" -- -d -b ~/.local/bin

# Windows (Scoop)
scoop install task
```

Verifica la instalación:

```bash
task --version
# Task version: 3.x.x
```

### Configurar Spotless en pom.xml

`format` y `format:check` del Taskfile requieren el plugin Spotless. Agrega esto en `pom.xml`:

```xml
<build>
    <plugins>
        <!-- ... spring-boot-maven-plugin ... -->

        <plugin>
            <groupId>com.diffplug.spotless</groupId>
            <artifactId>spotless-maven-plugin</artifactId>
            <version>2.44.0</version>
            <configuration>
                <java>
                    <!-- Google Java Format -->
                    <googleJavaFormat>
                        <version>1.22.0</version>
                        <style>AOSP</style>
                    </googleJavaFormat>
                    <!-- Elimina imports no usados -->
                    <removeUnusedImports/>
                    <!-- Ordena los imports -->
                    <importOrder/>
                </java>
            </configuration>
        </plugin>
    </plugins>
</build>
```

Con esto puedes formatear todo el código con:

```bash
task format
```

Y verificar el formato antes de hacer commit con:

```bash
task format:check
```

---

## Flujo de trabajo con Taskfile

```bash
# Primera vez — setup completo
task infra:up
task test              # Verifica que todo compila y los tests pasan

# Ciclo de desarrollo diario
task test:modulith     # Verifica arquitectura (rápido, sin BD)
task test:catalog      # Solo el módulo en el que trabajas
task format            # Formatea el código antes de commit

# Antes de hacer commit
task format:check      # Falla si el código no está formateado
task test              # Todos los tests
task docs              # Documentación C4 actualizada

# Demo o presentación
task infra:up
task demo              # Levanta la app con todos los servicios listos

# Reset completo (cuando algo está raro)
task infra:reset
task infra:up
```

---

## Tabla de URLs del entorno

| Servicio | URL | Credenciales |
|---|---|---|
| API REST | http://localhost:8080 | — |
| Actuator Health | http://localhost:8080/actuator/health | — |
| Actuator Módulos | http://localhost:8080/actuator/modulith | — |
| Zipkin | http://localhost:9411 | — |
| RabbitMQ Admin | http://localhost:15672 | guest / guest |
| PostgreSQL | localhost:5432/bookstore | bookstore / bookstore |

---

## Revisión final: ¿Qué construimos?

=== "Módulos"
    ```
    ✅ common/    — módulo OPEN con PagedResult
    ✅ catalog/   — CQRS: command/ query/ internal/
    ✅ orders/    — allowedDependencies={"catalog","common"}, multi-ítem
    ✅ inventory/ — puramente reactivo, solo escucha eventos
    ```

=== "Reglas verificadas"
    ```
    ✅ ModularityTest — boundaries verificados automáticamente
    ✅ Sin acceso a tipos internos entre módulos
    ✅ Sin dependencias circulares
    ✅ Dependencias explícitas declaradas
    ✅ Módulos OPEN/CLOSED correctamente definidos
    ```

=== "Eventos"
    ```
    ✅ CatalogEvents (sealed) — sync CQRS interno
    ✅ OrderCreatedEvent — cross-module con @Externalized
    ✅ Event Publication Registry — outbox automático
    ✅ Mensajes en RabbitMQ — externalización garantizada
    ✅ Reintentos automáticos — republish-on-restart
    ```

=== "Tests"
    ```
    ✅ ModularityTest — arquitectura
    ✅ @ApplicationModuleTest — catalog en aislamiento
    ✅ @ApplicationModuleTest + @MockitoBean — orders en aislamiento
    ✅ AssertablePublishedEvents — eventos verificados
    ✅ Scenario — flujos event-driven
    ✅ @SpringBootTest — smoke test de contexto completo
    ```

=== "Infraestructura"
    ```
    ✅ Flyway — migraciones versionadas con schemas por módulo
    ✅ Docker Compose — Postgres + RabbitMQ + Zipkin
    ✅ Testcontainers — tests con infraestructura real
    ✅ Zipkin — tracing entre módulos
    ✅ C4 docs — generados desde el código
    ✅ Spotless — formato de código consistente
    ✅ Taskfile — todos los comandos en un solo lugar
    ```

---

## Próximos pasos

**Agregar el módulo `notifications`**

Es el ejercicio más natural después del workshop. Solo escucha `OrderCreatedEvent` y envía un email de confirmación. Aplicas todo lo aprendido en un módulo nuevo sin tocar ningún otro.

```
notifications/
├── NotificationService.java
└── OrderNotificationHandler.java  ← @ApplicationModuleListener
```

**Extraer `inventory` como microservicio**

Gracias a los boundaries:
1. `inventory` no depende de ningún interno de otros módulos
2. `OrderCreatedEvent` ya llega a RabbitMQ vía `@Externalized`
3. Solo conviertes `OrderEventsInventoryHandler` en un consumer de RabbitMQ independiente

La extracción es operacional, no una reescritura. Ese es el valor real del monolito modular.

---

## Checklist de la Parte 6

- [ ] `spring-modulith-actuator` agregado al `pom.xml` con `scope=runtime`
- [ ] `micrometer-tracing-bridge-brave` y `zipkin-reporter-brave` en el `pom.xml`
- [ ] `compose.yml` tiene el servicio `zipkin` con imagen `openzipkin/zipkin:3`
- [ ] `application.properties` con `management.endpoints.web.exposure.include=health,info,metrics,modulith`
- [ ] `application.properties` con `management.tracing.sampling.probability=1.0`
- [ ] `application.properties` con `management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans`
- [ ] `spring.docker.compose.enabled=false` configurado si gestionas Docker manualmente
- [ ] `writesDocumentationSnippets()` agregado a `ModularityTest`
- [ ] `target/spring-modulith-docs/` generado con `.puml` de cada módulo ✅
- [ ] `/actuator/modulith` responde con la estructura de módulos ✅
- [ ] `/actuator/health` muestra Postgres y RabbitMQ UP ✅
- [ ] Traza visible en Zipkin después de crear una orden ✅
- [ ] Spotless configurado en `pom.xml`
- [ ] `Taskfile.yml` creado con todos los comandos
- [ ] `mvn test` completa al 100% verde ✅

---

**¡Workshop completado!**

Pasaste de un monolito acoplado con package-by-layer a un sistema modular con boundaries verificados, CQRS, Outbox Pattern, externalización de eventos, tests en aislamiento, documentación C4 generada automáticamente y observabilidad con Zipkin.

Recursos adicionales en la página de [Referencias](referencias.md).

---

**Anterior**: [Parte 5 — Testing en Aislamiento](parte_5.md) &nbsp;&nbsp;&nbsp;&nbsp; [Inicio](index.md)
