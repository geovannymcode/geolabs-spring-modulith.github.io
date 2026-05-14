# Parte 2 — Boundaries y Reglas Modulares

**Duración**: 25 minutos  
**Objetivo**: Entender y demostrar todas las reglas que Spring Modulith puede verificar

---

## ¿Qué hacemos aquí?

La Parte 1 nos enseñó a resolver violaciones. Esta parte las provoca a propósito para entender el modelo mental detrás de cada regla.

1. Demostrar cómo se ve una **violación de boundary** en tiempo real
2. Provocar y resolver una **dependencia circular**
3. Entender cuándo y cómo usar **Named Interfaces**
4. Ver cómo `allowedDependencies` protege los boundaries en equipos grandes

El objetivo no es solo aprender a arreglar — es entender por qué existen estas reglas.

---

## Los tres tipos de violaciones

Spring Modulith detecta tres categorías de problemas:

| Tipo | Descripción | Ejemplo |
|---|---|---|
| **Tipo no expuesto** | Un módulo accede a un tipo interno de otro | `orders → catalog.command.ProductRepository` |
| **Dependencia circular** | A depende de B y B depende de A | `catalog → orders → catalog` |
| **Dependencia no declarada** | Un módulo usa otro que no está en `allowedDependencies` | `orders → inventory` cuando solo se declaró `catalog, common` |

Vamos a provocar y resolver cada uno.

---

## Experimento 1: Violación de boundary en acción

### Paso 1: Introducir una violación intencionalmente

Modifica temporalmente `OrderService` para que acceda a `ProductCommandService` directamente en lugar de `CatalogApi`:

```java
// orders/domain/OrderService.java — MODIFICACIÓN TEMPORAL SOLO PARA EL EXPERIMENTO
import com.geovannycode.bookstore.catalog.command.ProductCommandService; // ← INCORRECTO

@Service
public class OrderService {
    private final ProductCommandService commandService; // ← acceso a interno de catalog
    ...
}
```

### Paso 2: Ejecutar ModularityTest

```bash
mvn test -Dtest=ModularityTest
```

Resultado:
```
org.springframework.modulith.core.Violations:
 - Module 'orders' depends on non-exposed type
   com.geovannycode.bookstore.catalog.command.ProductCommandService
   within module 'catalog'!

   com.geovannycode.bookstore.orders.domain.OrderService
   →
   com.geovannycode.bookstore.catalog.command.ProductCommandService
```

!!! info "Cómo leer el mensaje"
    Spring Modulith no solo dice qué está mal — muestra la cadena de dependencia exacta: qué clase en qué módulo accede a qué clase en qué otro módulo. Puedes ir directamente al código problemático sin buscar.

### Paso 3: Revertir

```java
// Vuelve a usar CatalogApi ✅
private final CatalogApi catalogApi;
```

!!! tip "Soporte en IntelliJ IDEA Ultimate"
    Con el plugin de Spring instalado, la violación aparece directamente en el editor antes de ejecutar el test — la línea de import se subraya en rojo con el mensaje de Spring Modulith.

---

## Experimento 2: Dependencia circular

Las dependencias circulares son uno de los problemas más difíciles de detectar manualmente. Spring Modulith los encuentra en milisegundos.

### Paso 1: Crear una circular intencionalmente

Agrega temporalmente en `CatalogApi` una referencia a algo del módulo `orders`:

```java
// catalog/CatalogApi.java — MODIFICACIÓN TEMPORAL
import com.geovannycode.bookstore.orders.OrderCreatedEvent; // ← crea la circular

@Service
public class CatalogApi {

    public void triggerCircular() {
        // Esto hace que catalog → orders, pero orders ya depende de catalog.
        // Resultado: catalog → orders → catalog
        OrderCreatedEvent event = new OrderCreatedEvent(
                null, null, null, null, 0, null, null  // 7 parámetros del record
        );
    }
}
```

La cadena de dependencias queda así:
```
orders → CatalogApi (que está en catalog)
catalog → OrderCreatedEvent (que está en orders)
```

Ambos módulos se dependen mutuamente. Eso es una circular.

### Paso 2: Ejecutar ModularityTest

```bash
mvn test -Dtest=ModularityTest
```

Resultado:
```
org.springframework.modulith.core.Violations:
 - Cycle detected:
   Slice catalog →
   Slice orders →
   Slice catalog
```

El mensaje muestra el ciclo completo. En un proyecto con 15 módulos, detectar esto manualmente puede llevar horas. Spring Modulith lo hace en una línea.

### Paso 3: ¿Por qué las circulares son peligrosas?

Imagina que `catalog` necesita publicar un `CatalogUpdatedEvent` que `orders` escucha, y `orders` necesita consultar `catalog` para validar productos. Parece razonable, pero el resultado es:

```
Si catalog falla al arrancar → orders no arranca
Si orders falla al arrancar → catalog no arranca
```

Deadlock de inicialización. La solución es que uno de los dos módulos escuche un evento en lugar de hacer una llamada directa. Nunca llames hacia atrás en el grafo de dependencias.

### Paso 4: Revertir

```java
// Elimina triggerCircular() y el import de OrderCreatedEvent en CatalogApi
```

---

## Regla: Named Interfaces

### ¿Cuándo los necesitas?

Solo las clases en la **raíz del módulo** son visibles para otros. Hay casos donde quieres exponer algo de un sub-paquete sin moverlo a la raíz.

Ejemplo: supón que creces y quieres organizar los eventos en su propio sub-paquete:

```
orders/
├── OrderCreatedEvent.java       ← en la raíz, es público ✅
└── domain/
    └── events/
        └── OrderShippedEvent.java  ← en sub-paquete, es privado ❌
```

Si `inventory` necesita escuchar `OrderShippedEvent`, tienes dos opciones: moverlo a la raíz del módulo o usar `@NamedInterface`.

### Implementando un Named Interface

```java
// orders/domain/events/OrderShippedEvent.java
package com.geovannycode.bookstore.orders.domain.events;

public record OrderShippedEvent(String orderNumber, String trackingCode) {}
```

Para exponerlo sin moverlo a la raíz, crea un `package-info.java` en ese sub-paquete:

```java
// orders/domain/events/package-info.java

/**
 * Named Interface "order-events": expone los eventos de orders
 * para que otros módulos puedan suscribirse.
 */
@NamedInterface("order-events")
package com.geovannycode.bookstore.orders.domain.events;

import org.springframework.modulith.NamedInterface;
```

Ahora `inventory` puede importar `OrderShippedEvent` y Spring Modulith lo permite sin reportarlo como violación.

!!! note "En nuestra Bookstore no lo necesitamos"
    `OrderCreatedEvent` está en la raíz de `orders/` directamente — es parte del contrato público del módulo. Los Named Interfaces son más útiles cuando tienes muchos eventos y quieres organizarlos en un sub-paquete dedicado sin exponerlos todos desde la raíz.

---

## Regla: Dependencias explícitas con `allowedDependencies`

### El problema

En la Parte 1 declaramos las dependencias de `orders` como `{"catalog", "common"}`. Pero ¿qué pasaría si no hubiéramos declarado nada?

Por defecto, cualquier módulo puede depender de cualquier otro siempre que use solo tipos públicos. Eso significa que sin `allowedDependencies`, alguien podría agregar mañana `InventoryService` a `OrderService` y el test no fallaría — mientras `InventoryService` sea público.

`allowedDependencies` cierra esa puerta: declara el contrato arquitectónico por escrito en el código.

### Cómo lo tenemos declarado

En el `orders/package-info.java` que creamos en la Parte 1:

```java
// orders/package-info.java
@ApplicationModule(allowedDependencies = {"catalog", "common"})
package com.geovannycode.bookstore.orders;

import org.springframework.modulith.ApplicationModule;
```

- `catalog`: `OrderService` consulta productos a través de `CatalogApi`
- `common`: `OrderRestController` usa `PagedResult`

Cualquier dependencia fuera de esas dos genera una violación.

### Demostración: intentar acceder a inventory desde orders

Modifica temporalmente `OrderService`:

```java
// orders/domain/OrderService.java — SOLO PARA DEMOSTRACIÓN
import com.geovannycode.bookstore.inventory.InventoryService; // ← prohibido

@Service
public class OrderService {
    private final InventoryService inventoryService; // ← no está en allowedDependencies
}
```

Ejecuta:

```bash
mvn test -Dtest=ModularityTest
```

Resultado:
```
org.springframework.modulith.core.Violations:
 - Module 'orders' depends on module 'inventory' via
   com.geovannycode.bookstore.orders.domain.OrderService
   →
   com.geovannycode.bookstore.inventory.InventoryService.
   Allowed targets: catalog, common.
```

El mensaje incluye "Allowed targets: catalog, common" — le recuerda al desarrollador exactamente cuál es el contrato declarado para ese módulo.

### Revertir

```java
// Elimina la referencia a InventoryService
```

---

## El mapa de dependencias de la Bookstore

Después de esta parte, el grafo de módulos queda así:

```
┌─────────┐    usa API pública    ┌─────────────┐
│ orders  │ ─────────────────────▶│   catalog   │
│         │                       └─────────────┘
│         │ publica evento
│         │─────────────────────▶ ┌─────────────┐
└─────────┘                       │  inventory  │
                                   └─────────────┘

┌──────────────────────────────────────────────────┐
│                    common (OPEN)                 │
│             PagedResult, utilidades              │
└──────────────────────────────────────────────────┘
```

Lo que garantiza este diseño:

- `catalog` no depende de nadie — puede evolucionar libremente
- `inventory` no llama a nadie — solo reacciona a eventos
- `orders` solo habla con `catalog` via API pública
- Para extraer `inventory` como microservicio en el futuro no necesitas tocar `orders`

---

## Resumen de todas las reglas

| Regla | Cómo se activa | Qué previene |
|---|---|---|
| Solo API pública accesible | Por defecto | Acceso a implementaciones internas |
| Módulo OPEN | `@ApplicationModule(type = OPEN)` | Sub-paquetes inaccesibles |
| Named Interface | `@NamedInterface` en `package-info.java` del sub-paquete | Sub-paquetes específicos inaccesibles |
| Sin circulares | Por defecto | A → B → A |
| Dependencias explícitas | `allowedDependencies = {...}` | Dependencias no declaradas |

Todas se verifican con un solo test — `ModularityTest`. Sin herramientas externas.

---

## Checklist de la Parte 2

- [ ] Experimento 1: violación de boundary provocada, vista y revertida
- [ ] Experimento 2: dependencia circular provocada, vista y revertida
- [ ] Named Interfaces entendidos — cuándo usarlos vs mover a la raíz
- [ ] Demostración de `allowedDependencies` con inventory ejecutada y revertida
- [ ] `ModularityTest` pasa ✅

---

**Anterior**: [Parte 1 — Migración y Setup](parte_1.md) &nbsp;&nbsp;&nbsp;&nbsp; **Siguiente**: [Parte 3 — CQRS en el Módulo Catalog](parte_3.md)
