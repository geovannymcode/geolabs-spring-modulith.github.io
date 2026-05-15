# Parte 6 — C4 Docs, Observabilidad y Docker

**Duración**: 20 minutos  
**Objetivo**: Documentación de arquitectura desde el código, observabilidad con Actuator y Zipkin, y automatización con Taskfile

---

## ¿Qué hacemos aquí?

Los tests pasan, la arquitectura está verificada. Quedan tres cosas:

1. Documentación C4 generada automáticamente desde el código
2. Actuator + Zipkin funcionando de verdad (con las dependencias correctas)
3. Taskfile con Spotless para centralizar los comandos del proyecto

---

## Paso 1: El pom.xml completo de esta parte

Este es el `pom.xml` final del workshop. Tiene las dependencias de Spring Modulith Actuator, tracing con Zipkin, y Spotless con Palantir Java Format.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>4.0.6</version>
        <relativePath/>
    </parent>

    <groupId>com.geovannycode</groupId>
    <artifactId>bookstore</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>jar</packaging>
    <name>bookstore</name>

    <properties>
        <java.version>25</java.version>
        <spring-modulith.version>2.0.6</spring-modulith.version>
        <testcontainers.version>1.20.5</testcontainers.version>
        <spotless.version>3.2.0</spotless.version>
        <palantir-java-format.version>2.85.0</palantir-java-format.version>
    </properties>

    <dependencies>
        <!-- ── Web ────────────────────────────────────────────── -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- ── Persistencia ───────────────────────────────────── -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-database-postgresql</artifactId>
        </dependency>
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- ── Mensajería ─────────────────────────────────────── -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-amqp</artifactId>
        </dependency>

        <!-- ── Dev ────────────────────────────────────────────── -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-docker-compose</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>

        <!-- ── Spring Modulith ────────────────────────────────── -->
        <dependency>
            <groupId>org.springframework.modulith</groupId>
            <artifactId>spring-modulith-starter-core</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.modulith</groupId>
            <artifactId>spring-modulith-starter-jdbc</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.modulith</groupId>
            <artifactId>spring-modulith-events-amqp</artifactId>
        </dependency>

        <!--
            spring-modulith-actuator: activa el endpoint /actuator/modulith.
            Sin esta dependencia el endpoint no existe aunque tengas
            spring-boot-starter-actuator en el classpath. Son proyectos
            independientes — hay que conectarlos explícitamente.
            scope=runtime porque no necesitamos compilar contra él.
        -->
        <dependency>
            <groupId>org.springframework.modulith</groupId>
            <artifactId>spring-modulith-actuator</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!--
            Tracing con Zipkin — dos dependencias que trabajan juntas:
            - micrometer-tracing-bridge-brave: puente entre Micrometer
              (el sistema de métricas de Spring Boot) y Brave (la librería
              de tracing de Zipkin). Sin este puente, Spring no sabe cómo
              generar trazas en el formato que Zipkin entiende.
            - zipkin-reporter-brave: el que realmente envía las trazas
              al servidor Zipkin vía HTTP.
        -->
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-tracing-bridge-brave</artifactId>
        </dependency>
        <dependency>
            <groupId>io.zipkin.reporter2</groupId>
            <artifactId>zipkin-reporter-brave</artifactId>
        </dependency>

        <!-- ── Test ───────────────────────────────────────────── -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-testcontainers</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>postgresql</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.testcontainers</groupId>
            <artifactId>rabbitmq</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.modulith</groupId>
            <artifactId>spring-modulith-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.testcontainers</groupId>
                <artifactId>testcontainers-bom</artifactId>
                <version>${testcontainers.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>org.springframework.modulith</groupId>
                <artifactId>spring-modulith-bom</artifactId>
                <version>${spring-modulith.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>

            <!--
                Spotless con Palantir Java Format.

                Palantir es una alternativa a Google Java Format — produce
                código más legible para los estándares de equipos modernos.
                La versión 2.85.0 soporta Java 25.

                La ejecución en la fase "compile" hace que el build falle
                si el código no está formateado. Para formatear automáticamente
                usa: mvn spotless:apply (o task format con Taskfile).
            -->
            <plugin>
                <groupId>com.diffplug.spotless</groupId>
                <artifactId>spotless-maven-plugin</artifactId>
                <version>${spotless.version}</version>
                <configuration>
                    <java>
                        <importOrder/>
                        <removeUnusedImports/>
                        <palantirJavaFormat>
                            <version>${palantir-java-format.version}</version>
                        </palantirJavaFormat>
                        <formatAnnotations/>
                    </java>
                </configuration>
                <executions>
                    <execution>
                        <phase>compile</phase>
                        <goals>
                            <goal>check</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## Paso 2: Configurar application.properties

```properties
# ── Actuator ──────────────────────────────────────────────────────────
# "modulith" requiere spring-modulith-actuator en el classpath.
# Sin esa dependencia este endpoint devuelve 404.
management.endpoints.web.exposure.include=health,info,metrics,modulith
management.endpoint.health.show-details=always
management.info.env.enabled=true

info.app.name=Bookstore Modulith
info.app.version=1.0.0
info.app.description=Workshop BarranquillaJUG — Spring Modulith

# ── Zipkin ────────────────────────────────────────────────────────────
# 1.0 samplea el 100% de las peticiones — solo en desarrollo.
# En producción usar 0.1 (10%) para no saturar.
management.tracing.sampling.probability=1.0
management.zipkin.tracing.endpoint=http://localhost:9411/api/v2/spans

# spring-boot-docker-compose gestiona Docker automáticamente.
# Si prefieres levantar Docker tú mismo, descomenta esta línea:
# spring.docker.compose.enabled=false
```

---

## Paso 3: Agregar Zipkin al Docker Compose

Zipkin necesita correr para recibir las trazas. La app las envía de forma "fire and forget" — si Zipkin no está, las trazas se pierden pero la app sigue funcionando.

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
      - "9411:9411"   # UI: http://localhost:9411
```

---

## Paso 4: Por qué Zipkin no muestra nada

Si abriste `http://localhost:9411`, seleccionaste el servicio `bookstore` y ejecutaste la consulta sin resultados, hay una razón concreta: la app arrancó antes de que Zipkin estuviera listo.

La solución más directa cuando usas `spring-boot-docker-compose` es desactivarlo y levantar Docker manualmente:

```properties
# application.properties
spring.docker.compose.enabled=false
```

Luego el flujo siempre es:

```bash
# 1. Levanta todos los servicios (incluido Zipkin)
docker compose up -d

# 2. Espera a que Zipkin esté listo (unos segundos)
sleep 5

# 3. Arranca la app
mvn spring-boot:run
```

Después de arrancar, haz una petición real:

```bash
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerName": "Geo Mendoza",
    "customerEmail": "geo@barranquillajug.com",
    "customerPhone": "+57 300 1234567",
    "deliveryAddress": "Barranquilla",
    "items": [{"productCode": "P001", "quantity": 1}]
  }'
```

Luego en Zipkin:

1. Abre `http://localhost:9411`
2. En el dropdown de servicio selecciona `bookstore`
3. Haz clic en **Ejecutar Consulta**

Deberías ver la traza del `POST /api/orders` con spans separados para la transacción de `orders` y el handler asíncrono de `inventory`. El span del handler aparece después del commit — eso confirma visualmente que `@ApplicationModuleListener` ejecuta post-commit en su propia transacción.

!!! tip "El servicio no aparece en el dropdown"
    Zipkin solo muestra servicios que enviaron al menos una traza. Verifica en los logs de la app que aparece `Tracing is enabled`. Si no aparece, revisa que `micrometer-tracing-bridge-brave` está en el `pom.xml`.

---

## Paso 5: Verificar /actuator/modulith

```bash
curl http://localhost:8080/actuator/modulith | python3 -m json.tool
```

Respuesta:

```json
{
  "modules": [
    {
      "name": "catalog",
      "basePackage": "com.geovannycode.bookstore.catalog",
      "type": "CLOSED"
    },
    {
      "name": "orders",
      "basePackage": "com.geovannycode.bookstore.orders",
      "type": "CLOSED",
      "allowedDependencies": ["catalog", "common"]
    },
    {
      "name": "inventory",
      "basePackage": "com.geovannycode.bookstore.inventory",
      "type": "CLOSED"
    },
    {
      "name": "common",
      "basePackage": "com.geovannycode.bookstore.common",
      "type": "OPEN"
    }
  ]
}
```

Si el endpoint devuelve 404, falta `spring-modulith-actuator` en el `pom.xml`.

---

## Paso 6: Documentación C4 con ModularityTest

Spring Modulith analiza el bytecode y genera diagramas PlantUML. Los diagramas nunca están desactualizados porque vienen del código real.

Actualiza `ModularityTest.java`:

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
     * Genera diagramas PlantUML en target/spring-modulith-docs/
     *
     * - components.puml: vista global de todos los módulos
     * - catalog.puml, orders.puml, inventory.puml: canvas por módulo
     * - aggregating-document.adoc: documento AsciiDoc completo
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

```bash
mvn test -Dtest=ModularityTest
ls target/spring-modulith-docs/
```

### Visualizar los diagramas

Tienes instalado `plantuml4idea` (Vojtěch Krása). Abre cualquier `.puml` en `target/spring-modulith-docs/` e IntelliJ renderiza el diagrama automáticamente en el panel de preview. Si no aparece, usa `Alt+D`.

---

## Paso 7: Taskfile

Taskfile centraliza todos los comandos. Sin él cada persona del equipo tiene sus propios alias y flags de Maven que nadie más recuerda.

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
    desc: "Baja la infraestructura y borra los volúmenes"
    cmds:
      - docker compose down -v

  infra:logs:
    desc: "Logs de todos los contenedores"
    cmds:
      - docker compose logs -f

  test:
    desc: "Ejecuta todos los tests"
    cmds:
      - mvn test

  test:modulith:
    desc: "Verifica la arquitectura modular"
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

  build:
    desc: "Compila y empaqueta el JAR sin tests"
    cmds:
      - mvn package -DskipTests

  clean:
    desc: "Limpia el directorio target"
    cmds:
      - mvn clean

  format:
    desc: "Formatea el código con Spotless (Palantir Java Format)"
    cmds:
      - mvn spotless:apply

  format:check:
    desc: "Verifica el formato sin modificar el código"
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

# Verificar
task --version
```

### Cómo funciona Spotless en este proyecto

El `pom.xml` usa Palantir Java Format 2.85.0, que soporta Java 25. El plugin corre en la fase `compile` con el goal `check` — eso significa que `mvn compile` falla si el código no está formateado correctamente.

Para formatear antes de compilar:

```bash
task format        # equivale a mvn spotless:apply
task format:check  # equivale a mvn spotless:check
```

La diferencia entre los dos: `apply` modifica los archivos, `check` solo informa si hay algo que cambiar. Útil en CI para rechazar PRs con código sin formatear.

---

## Flujo de trabajo con Taskfile

```bash
# Primera vez
task infra:up
task test

# Ciclo diario
task format              # formatea antes de trabajar
task test:catalog        # test rápido del módulo que estás tocando
task test:modulith       # verifica que no rompiste boundaries

# Antes de hacer commit
task format:check        # falla si hay algo sin formatear
task test                # todos los tests en verde
task docs                # actualiza los diagramas C4

# Demo
task infra:up
task demo
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

## Resumen final

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
    ```

=== "Eventos"
    ```
    ✅ CatalogEvents (sealed) — sync CQRS interno
    ✅ OrderCreatedEvent — @Externalized a RabbitMQ
    ✅ Event Publication Registry — outbox automático
    ✅ @ApplicationModuleListener — post-commit, transacción propia
    ```

=== "Tests"
    ```
    ✅ ModularityTest — arquitectura
    ✅ @ApplicationModuleTest catalog — con @Sql en ambas tablas CQRS
    ✅ @ApplicationModuleTest orders — @MockitoBean + AssertablePublishedEvents
    ✅ Scenario — inventory en aislamiento
    ✅ BookstoreApplicationTests — smoke test
    ```

=== "Infraestructura"
    ```
    ✅ Flyway V1-V5 — schemas por módulo + multi-ítem
    ✅ Docker Compose — Postgres + RabbitMQ + Zipkin
    ✅ Zipkin — trazas entre módulos
    ✅ /actuator/modulith — arquitectura en runtime
    ✅ C4 docs — generados desde el bytecode
    ✅ Spotless + Palantir — formato de código consistente
    ✅ Taskfile — todos los comandos en un lugar
    ```

---

## Próximos pasos

**Agregar el módulo `notifications`** — el ejercicio más directo después del workshop. Solo escucha `OrderCreatedEvent` con `@ApplicationModuleListener` y envía un email. No toca ningún otro módulo.

**Extraer `inventory` como microservicio** — gracias a que no tiene dependencias internas de otros módulos y `OrderCreatedEvent` ya llega a RabbitMQ, la extracción es cambiar el handler por un consumer de RabbitMQ independiente. No hay que reescribir nada.

---

## Checklist de la Parte 6

- [ ] `pom.xml` actualizado con `spring-modulith-actuator`, tracing y Spotless con Palantir
- [ ] `compose.yml` tiene el servicio `zipkin`
- [ ] `application.properties` con `modulith` en endpoints y Zipkin configurado
- [ ] `spring.docker.compose.enabled=false` si gestionas Docker manualmente
- [ ] `writesDocumentationSnippets()` en `ModularityTest`
- [ ] `target/spring-modulith-docs/` generado con `.puml` de cada módulo ✅
- [ ] `/actuator/modulith` responde con la estructura de módulos ✅
- [ ] `/actuator/health` muestra Postgres y RabbitMQ UP ✅
- [ ] Traza visible en Zipkin después de crear una orden ✅
- [ ] `Taskfile.yml` creado con todos los comandos
- [ ] `task format` funciona sin errores ✅
- [ ] `mvn test` completa al 100% verde ✅

---

**¡Workshop completado!**

Pasaste de un monolito acoplado con package-by-layer a un sistema modular con boundaries verificados, CQRS, Outbox Pattern, eventos externalizados a RabbitMQ, tests en aislamiento, documentación C4 generada desde el código y observabilidad con Zipkin.

Recursos adicionales en la página de [Referencias](referencias.md).

---

**Anterior**: [Parte 5 — Testing en Aislamiento](parte_5.md) &nbsp;&nbsp;&nbsp;&nbsp; [Inicio](index.md)
