# Referencias

## Documentación Oficial

| Recurso | URL |
|---|---|
| Spring Modulith Reference | [docs.spring.io/spring-modulith](https://docs.spring.io/spring-modulith/reference/index.html) |
| Spring Modulith Javadoc | [docs.spring.io/spring-modulith/api](https://docs.spring.io/spring-modulith/api/) |
| Spring Boot 3.4 Reference | [docs.spring.io/spring-boot/reference](https://docs.spring.io/spring-boot/reference/index.html) |
| Spring Boot Testcontainers | [docs.spring.io/spring-boot/reference/testing/testcontainers](https://docs.spring.io/spring-boot/reference/testing/testcontainers.html) |
| Testcontainers Java | [java.testcontainers.org](https://java.testcontainers.org/) |
| Flyway Docs | [flywaydb.org/documentation](https://flywaydb.org/documentation/) |

## Código Fuente del Workshop

| Recurso | URL |
|---|---|
| Proyecto final (bookstore-modulith) | [github.com/geovannymcode/bookstore-modulith](https://github.com/geovannymcode/bookstore-modulith) |
| Esta guía (MkDocs) | [geolabs-spring-modulith.github.io](https://geolabs-spring-modulith.github.io) |

## Lecturas Complementarias

### Spring Modulith

- **"Modular Monoliths" — Oliver Drotbohm** (autor de Spring Modulith)  
  Charla fundacional sobre el diseño de monolitos modulares con Spring.  
  [YouTube: GOTO 2023](https://www.youtube.com/watch?v=1eVGHMK_4MQ)

- **"From Monolith to Modules" — Blog de Spring**  
  Guía oficial de migración progresiva de un monolito a módulos.  
  [spring.io/blog](https://spring.io/blog/2022/10/21/introducing-spring-modulith)

### CQRS

- **"CQRS Documents" — Greg Young**  
  El paper original que introdujo CQRS como patrón formal.  
  [cqrs.files.wordpress.com](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf)

- **"Clarified CQRS" — Udi Dahan**  
  Una perspectiva más pragmática y aplicada del patrón.  
  [udidahan.com](http://udidahan.com/2009/12/09/clarified-cqrs/)

### Eventos y Outbox Pattern

- **"Transactional Outbox Pattern" — microservices.io**  
  Explicación detallada del patrón que Spring Modulith implementa con el Event Publication Registry.  
  [microservices.io/patterns/data/transactional-outbox](https://microservices.io/patterns/data/transactional-outbox.html)

- **"Event-Driven Architecture" — Martin Fowler**  
  El ensayo clásico sobre los diferentes tipos de event-driven architecture.  
  [martinfowler.com/articles/201701-event-driven](https://martinfowler.com/articles/201701-event-driven.html)

### Arquitectura de Módulos

- **"Modular Monolith: A Primer" — Kamil Grzybek**  
  Serie de artículos muy detallada sobre cómo implementar un monolito modular en la práctica.  
  [kamilgrzybek.com](https://www.kamilgrzybek.com/blog/posts/modular-monolith-primer)

- **"Package by Feature, not Layer" — javapractices.com**  
  El argumento clásico para package-by-feature.  
  [javapractices.com](http://www.javapractices.com/topic/TopicAction.do?Id=205)

## Herramientas Mencionadas en el Workshop

| Herramienta | Versión | URL |
|---|---|---|
| Taskfile | 3+ | [taskfile.dev](https://taskfile.dev) |
| PlantUML | — | [plantuml.com](https://plantuml.com) |
| PlantUML IntelliJ Plugin | — | [plugins.jetbrains.com](https://plugins.jetbrains.com/plugin/7017-plantuml-integration) |
| Spring Debugger Plugin (IntelliJ) | — | [plugins.jetbrains.com/plugin/25302](https://plugins.jetbrains.com/plugin/25302-spring-debugger) |
| Zipkin | 3 | [zipkin.io](https://zipkin.io) |
| RabbitMQ | 3.13 | [rabbitmq.com](https://www.rabbitmq.com) |

## Proyectos de Referencia

Estos proyectos sirvieron de inspiración y referencia para el diseño del workshop:

- **spring-modulith-workshop (Siva Prasad Reddy)**  
  Workshop en inglés con enfoque en refactoring desde package-by-layer.  
  [github.com/sivaprasadreddy/spring-modulith-workshop](https://github.com/sivaprasadreddy/spring-modulith-workshop)

- **spring-modulith-examples (Oliver Drotbohm)**  
  Ejemplos oficiales del equipo de Spring Modulith.  
  [github.com/spring-projects/spring-modulith](https://github.com/spring-projects/spring-modulith)

## Comunidad

- **BAQJUG — Barranquilla Java User Group**  
  Organiza eventos mensuales sobre Java, Spring y arquitectura.  
  [barranquillajug.com](https://barranquillajug.com) | [@barranquillaJUG](https://twitter.com/barranquillajug)

- **Geovanny Mendoza — Autor del Workshop**  
  Senior Backend Engineer, Vaadin Champion, organizador de BarranquillaJUG.  
  [geovannycode.com](https://geovannycode.com) | [@geovannycode](https://twitter.com/geovannycode)

---

## Agradecimientos

Este workshop fue construido sobre el trabajo de la comunidad Java y del equipo de Spring.
Un agradecimiento especial a Oliver Drotbohm por diseñar Spring Modulith,
y a Siva Prasad Reddy por su workshop de referencia en inglés.

---

*Última actualización: Mayo 2026 — Workshop BarranquillaJUG*
