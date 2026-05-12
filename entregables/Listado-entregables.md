# 📦 Listado de Entregables — Conecta360

> **Proyecto:** Sistema Integral de Atención Ciudadana "Conecta360"  
> **País:** Costa Verde  
> **Versión:** 1.0  
> **Fecha:** 2026-05-11  
> **Basado en:** PRD §7 — Entregables  

---

## 🎭 Sesión de Planificación de Entregables — Las 5 Personas

**Eduardo (PM):** Basándonos en el §7 del PRD, tenemos 4 categorías de entregables. Necesito que cada uno esté definido con alcance, responsable y criterios de aceptación claros.

**Ariel (Arq):** El §7.1 y §7.2 son mi responsabilidad principal. Para el C4 usaré PlantUML. Para la base de datos primero necesito que Diana genere el Diccionario de Datos.

**Diana (AD):** Correcto. El orden natural es: Diccionario de Datos → Diagrama ER → luego el C4 puede referenciarlos. No al revés.

**Carlos (Seg):** En el §7.3, cada decisión tecnológica que toque criptografía, autenticación o control de acceso necesita un ADR propio con justificación de seguridad.

**Beatriz (BA):** Y el §7.4 debe estar alineado con los requerimientos que documentamos. Los endpoints del Swagger deben trazar directamente a los RF- de nuestra tabla.

---

## 📋 Tabla de Entregables

| ID | Entregable | Descripción | Categoría PRD | Formato | Responsable Principal | Colaboradores | Criterios de Aceptación | Estado |
|----|-----------|-------------|--------------|---------|----------------------|---------------|------------------------|--------|
| **E-01** | Diccionario de Datos Canónico | Definición de todas las entidades del dominio: ciudadanos, casos, dependencias, flujos, auditorías, notificaciones. Incluye nombre canónico, tipo de dato, restricciones y mapeos entre instituciones. | §7.2 | `diccionario-datos.md` | Diana (AD) | Ariel, Beatriz | Todas las entidades del §7.2 cubiertas. Revisado por BA y Arquitecto. | 📝 Pendiente |
| **E-02** | Diagrama de Base de Datos (ER) | Modelo entidad-relación de las entidades principales con cardinalidades, claves primarias/foráneas y esquemas sugeridos. PlantUML. | §7.2 | `diagrama-base-datos.puml` | Diana (AD) | Ariel | Diagrama renderizable. Cubre todas las entidades del Diccionario de Datos. | 📝 Pendiente |
| **E-03** | Diagrama C4 — Nivel 1 (Contexto) | Vista de contexto del sistema Conecta360: actores externos, sistemas externos y el sistema en su conjunto. PlantUML C4. | §7.1 | `c4-context.puml` | Ariel (Arq) | Carlos, Diana | Aprobado por BA. Todos los actores del PRD incluidos. | 📝 Pendiente |
| **E-04** | Diagrama C4 — Nivel 2 (Contenedores) | Vista de contenedores: API Gateway, microservicios, bases de datos, colas de mensajería, frontend. PlantUML C4. | §7.1 | `c4-containers.puml` | Ariel (Arq) | Carlos, Diana | Cubre capas: presentación, aplicación, integración, datos. Aprobado por equipo. | 📝 Pendiente |
| **E-05** | Diagrama C4 — Nivel 3 (Componentes) | Vista de componentes del microservicio de Atención Ciudadana (caso principal). PlantUML C4. | §7.1 | `c4-components.puml` | Ariel (Arq) | Carlos | Al menos 1 microservicio detallado. Incluye interacciones de seguridad. | 📝 Pendiente |
| **E-06** | ADR-001: Patrón de arquitectura (Microservicios + Event-Driven) | Architectural Decision Record justificando la elección de microservicios y event-driven sobre monolito o SOA. | §7.3 | `ADR-001-microservicios.md` | Ariel (Arq) | Eduardo | Contexto, decisión, alternativas, consecuencias documentadas. | 📝 Pendiente |
| **E-07** | ADR-002: Base de datos (PostgreSQL multi-tenant con RLS) | ADR justificando PostgreSQL con Row-Level Security sobre otras opciones (MongoDB, Oracle, schema-per-tenant). | §7.3 | `ADR-002-base-de-datos.md` | Diana (AD) | Ariel, Carlos | Alternativas evaluadas. Impacto de seguridad y rendimiento documentado. | 📝 Pendiente |
| **E-08** | ADR-003: Mensajería asíncrona (Apache Kafka) | ADR justificando Kafka sobre RabbitMQ, AWS SQS u otras opciones para el bus de eventos. | §7.3 | `ADR-003-mensajeria.md` | Ariel (Arq) | Diana | Carga esperada analizada (500k req/día). Pros/contras documentados. | 📝 Pendiente |
| **E-09** | ADR-004: Autenticación y autorización (OAuth2 + RBAC/ABAC) | ADR justificando OAuth2/OIDC con RBAC + ABAC sobre otras soluciones de IAM. | §7.3 | `ADR-004-autenticacion.md` | Carlos (Seg) | Ariel | Justificación de seguridad completa. Casos de roles documentados. | 📝 Pendiente |
| **E-10** | ADR-005: Infraestructura y nube (Kubernetes + Multi-región) | ADR justificando Kubernetes sobre serverless u otras opciones para cumplir el 99.9% SLA. | §7.3 | `ADR-005-infraestructura.md` | Ariel (Arq) | Eduardo | RTO y RPO definidos. Estrategia multi-región documentada. | 📝 Pendiente |
| **E-11** | ADR-006: API Gateway (Kong) | ADR justificando Kong como API Gateway sobre Nginx, AWS API GW u otras opciones. | §7.3 | `ADR-006-api-gateway.md` | Ariel (Arq) | Carlos | Capacidades de seguridad (rate limiting, WAF, authn) documentadas. | 📝 Pendiente |
| **E-12** | Especificación OpenAPI 3.1 — APIs REST | Definición Swagger/OpenAPI de los endpoints principales: casos, ciudadanos, dependencias, notificaciones, autenticación. Con ejemplos de payloads JSON. | §7.4 | `openapi.yaml` | Beatriz (BA) | Ariel, Carlos | Todos los RF- funcionales cubiertos. Validado con swagger-cli. Ejemplos de request/response incluidos. | 📝 Pendiente |
| **E-13** | Especificación AsyncAPI 2.6 — Eventos asíncronos | Definición de eventos del bus de mensajería: `case.created`, `case.updated`, `case.closed`, `notification.sent`. Con ejemplos de payloads. | §7.4 | `asyncapi.yaml` | Beatriz (BA) | Ariel, Diana | Todos los eventos de integración documentados. Schemas Avro/JSON definidos. | 📝 Pendiente |
| **E-14** | Listado de Requerimientos | Tabla de requerimientos funcionales y no funcionales con análisis del equipo técnico. | §4, §5 | `Listado-Requerimientos.md` | Beatriz (BA) | Ariel, Carlos | Todos los RF y RNF del PRD documentados. Trazabilidad clara. | ✅ Completado |
| **E-15** | Matriz de Riesgo | Matriz con probabilidad, impacto, score y estrategias de mitigación para todos los desafíos del PRD. | §6 | `Matriz-de-Riesgo.md` | Eduardo (PM) | Todos | Todos los desafíos del §6 cubiertos. Score calculado. Dueño por riesgo. | ✅ Completado |

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

*Documento generado por sesión colaborativa de las 5 personas — Conecta360 v1.0*
