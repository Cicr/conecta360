# 📦 Listado de Entregables — Conecta360

> **Proyecto:** Sistema Integral de Atención Ciudadana "Conecta360"  
> **País:** República Dominicana  
> **Versión:** 1.0  
> **Fecha:** 2026-05-11  
> **Basado en:** PRD §7 — Entregables  

---

## 📐 Criterios de Definición

Cada entregable fue definido con los siguientes atributos:

- **Alcance** claro y acotado al PRD §7
- **Formato** de entrega especificado (Markdown, YAML, PlantUML)
- **Criterios de aceptación** verificables
- **Trazabilidad** directa al requerimiento o sección del PRD que justifica su existencia

> **Orden de generación:** El Diccionario de Datos precede al Diagrama ER; ambos preceden a los diagramas C4.

---

## 📋 Tabla de Entregables

| ID | Entregable | Descripción | Categoría PRD | Formato | Responsable Principal | Colaboradores | Criterios de Aceptación | Estado |
|----|-----------|-------------|--------------|---------|----------------------|---------------|------------------------|--------|
| **E-01** | Diccionario de Datos Canónico | Definición de todas las entidades del dominio: ciudadanos, casos, dependencias, flujos, auditorías, notificaciones. Incluye nombre canónico, tipo de dato, restricciones y mapeos entre instituciones. | §7.2 | `diccionario-datos.md` | Datos | Arquitectura, Negocio | Todas las entidades del §7.2 cubiertas. | ✅ Completado |
| **E-02** | Diagrama de Base de Datos (ER) | Modelo entidad-relación de las entidades principales con cardinalidades, claves primarias/foráneas y esquemas sugeridos. PlantUML. | §7.2 | `diagrama-base-datos.puml` | Datos | Arquitectura | Diagrama renderizable. Cubre todas las entidades del Diccionario de Datos. | ✅ Completado |
| **E-03** | Diagrama C4 — Nivel 1 (Contexto) | Vista de contexto del sistema Conecta360: actores externos, sistemas externos y el sistema en su conjunto. PlantUML C4. | §7.1 | `c4-context.puml` | Arquitectura | Seguridad, Datos | Todos los actores del PRD incluidos. | ✅ Completado |
| **E-04** | Diagrama C4 — Nivel 2 (Contenedores) | Vista de contenedores: API Gateway, microservicios, bases de datos, colas de mensajería, frontend. PlantUML C4. | §7.1 | `c4-containers.puml` | Arquitectura | Seguridad, Datos | Cubre capas: presentación, aplicación, integración, datos. | ✅ Completado |
| **E-05** | Diagrama C4 — Nivel 3 (Componentes) | Vista de componentes del microservicio de Atención Ciudadana (caso principal). PlantUML C4. | §7.1 | `c4-components.puml` | Arquitectura | Seguridad | Al menos 1 microservicio detallado. Incluye interacciones de seguridad. | ✅ Completado |
| **E-06** | ADR-001: Patrón de arquitectura (Microservicios + Event-Driven) | Architectural Decision Record justificando la elección de microservicios y event-driven sobre monolito o SOA. | §7.3 | `ADR-001-microservicios.md` | Arquitectura | PM | Contexto, decisión, alternativas, consecuencias documentadas. | ✅ Completado |
| **E-07** | ADR-002: Base de datos (PostgreSQL multi-tenant con RLS) | ADR justificando PostgreSQL con Row-Level Security sobre otras opciones (MongoDB, Oracle, schema-per-tenant). | §7.3 | `ADR-002-base-de-datos.md` | Datos | Arquitectura, Seguridad | Alternativas evaluadas. Impacto de seguridad y rendimiento documentado. | ✅ Completado |
| **E-08** | ADR-003: Mensajería asíncrona (Apache Kafka) | ADR justificando Kafka sobre RabbitMQ, AWS SQS u otras opciones para el bus de eventos. | §7.3 | `ADR-003-mensajeria.md` | Arquitectura | Datos | Carga esperada analizada (500k req/día). Pros/contras documentados. | ✅ Completado |
| **E-09** | ADR-004: Autenticación y autorización (OAuth2 + RBAC/ABAC) | ADR justificando OAuth2/OIDC con RBAC + ABAC sobre otras soluciones de IAM. | §7.3 | `ADR-004-autenticacion.md` | Seguridad | Arquitectura | Justificación de seguridad completa. Casos de roles documentados. | ✅ Completado |
| **E-10** | ADR-005: Infraestructura y nube (Kubernetes + Multi-región) | ADR justificando Kubernetes sobre serverless u otras opciones para cumplir el 99.9% SLA. | §7.3 | `ADR-005-infraestructura.md` | Arquitectura | PM | RTO y RPO definidos. Estrategia multi-región documentada. | ✅ Completado |
| **E-11** | ADR-006: API Gateway (Kong) | ADR justificando Kong como API Gateway sobre Nginx, AWS API GW u otras opciones. | §7.3 | `ADR-006-api-gateway.md` | Arquitectura | Seguridad | Capacidades de seguridad (rate limiting, WAF, authn) documentadas. | ✅ Completado |
| **E-12** | Especificación OpenAPI 3.1 — APIs REST | Definición Swagger/OpenAPI de los endpoints principales: casos, ciudadanos, dependencias, notificaciones, autenticación. Con ejemplos de payloads JSON. | §7.4 | `openapi.yaml` | Negocio | Arquitectura, Seguridad | Todos los RF- funcionales cubiertos. Ejemplos de request/response incluidos. | ✅ Completado |
| **E-13** | Especificación AsyncAPI 2.6 — Eventos asíncronos | Definición de eventos del bus de mensajería: `case.created`, `case.updated`, `case.closed`, `notification.sent`. Con ejemplos de payloads. | §7.4 | `asyncapi.yaml` | Negocio | Arquitectura, Datos | Todos los eventos de integración documentados. Schemas Avro/JSON definidos. | ✅ Completado |
| **E-14** | Listado de Requerimientos | Tabla de requerimientos funcionales y no funcionales con análisis técnico. | §4, §5 | `Listado-Requerimientos.md` | Negocio | Arquitectura, Seguridad | Todos los RF y RNF del PRD documentados. Trazabilidad clara. | ✅ Completado |
| **E-15** | Matriz de Riesgo | Matriz con probabilidad, impacto, score y estrategias de mitigación para todos los desafíos del PRD. | §6 | `Matriz-de-Riesgo.md` | PM | Todos | Todos los desafíos del §6 cubiertos. Score calculado. | ✅ Completado |

---

## 📁 Estructura de Archivos

```
conecta360/
├── PRD.md
├── Listado-Requerimientos.md          ← E-14 ✅
├── Matriz-de-Riesgo.md               ← E-15 ✅
└── entregables/
    ├── Listado-entregables.md         ← Este archivo
    ├── diccionario-datos.md           ← E-01
    ├── diagrama-base-datos.puml       ← E-02
    ├── c4-context.puml                ← E-03
    ├── c4-containers.puml             ← E-04
    ├── c4-components.puml             ← E-05
    ├── ADR-001-microservicios.md      ← E-06
    ├── ADR-002-base-de-datos.md       ← E-07
    ├── ADR-003-mensajeria.md          ← E-08
    ├── ADR-004-autenticacion.md       ← E-09
    ├── ADR-005-infraestructura.md     ← E-10
    ├── ADR-006-api-gateway.md         ← E-11
    ├── openapi.yaml                   ← E-12
    └── asyncapi.yaml                  ← E-13
```

---

## 📊 Resumen de Estado

| Estado | Cantidad | Entregables |
|--------|:---:|---|
| ✅ Completado | 2 | E-14, E-15 |
| 📝 Pendiente | 13 | E-01 al E-13 |
| 🔄 En progreso | 0 | — |
| **Total** | **15** | |

---

*Conecta360 v1.0 — Índice maestro de entregables*
