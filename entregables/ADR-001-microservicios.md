# ADR-001: Arquitectura de Microservicios con Patrón Event-Driven

> **Proyecto:** Conecta360  
> **ID:** ADR-001  
> **Estado:** Aprobado ✅  
> **Fecha:** 2026-05-11  
> **Decisión tomada por:** Ariel Montero (Arquitecto de Software)  
> **Revisado por:** Carlos Fuentes (Seguridad), Diana Rivas (Datos), Eduardo Lara (PM), Beatriz Salcedo (BA)  

---

## 🎭 Contexto de la Decisión (Conversación del Equipo)

**Ariel (Arq):** Tenemos un sistema con 10+ millones de habitantes potenciales de Costa Verde, cinco dependencias gubernamentales con necesidades completamente diferentes y un requerimiento de 500,000 solicitudes diarias. El monolito es una trampa aquí.

**Eduardo (PM):** Pero los microservicios también tienen un costo de complejidad. ¿Cómo justificamos eso al cliente?

**Beatriz (BA):** Yo lo veo desde el negocio: las instituciones deben poder actualizarse independientemente. Si Salud quiere añadir telemedicina no puede bloquear a Obras Públicas.

**Carlos (Seg):** Además, el aislamiento de servicios es clave para seguridad. Si el Chatbot se ve comprometido, no quiero que tenga acceso a la base de datos de casos completa.

**Diana (AD):** El event-driven nos da el desacoplamiento que necesitamos para el módulo analítico. No quiero que los reportes afecten el rendimiento del core de gestión de casos.

---

## 1. Contexto y Problema

El sistema Conecta360 debe atender a más de 10 millones de ciudadanos, integrar múltiples instituciones gubernamentales autónomas, soportar hasta 500,000 solicitudes diarias y garantizar un SLA del 99.9%. Adicionalmente, cada módulo tiene ciclos de vida, equipos y requerimientos de escalabilidad distintos.

## 2. Decisión

**Se adopta una arquitectura de microservicios con comunicación híbrida:**
- **Síncrona (REST/gRPC):** Para operaciones que requieren respuesta inmediata (creación de caso, consulta de estado)
- **Asíncrona (Event-Driven / Kafka):** Para operaciones de notificación, analítica, sincronización con sistemas legados

## 3. Alternativas Evaluadas

| Opción | Ventajas | Desventajas | Score |
|--------|---------|-------------|:---:|
| **Monolito modular** | Simplicidad operacional, transacciones ACID | No escala por módulo independiente, despliegues acoplados | 4/10 |
| **SOA (Service-Oriented Architecture)** | Más maduro que microservicios, ESB probado | ESB = single point of failure, alto licenciamiento | 5/10 |
| **Microservicios + Event-Driven** ✅ | Escalabilidad independiente, resiliencia, desacoplamiento | Mayor complejidad operacional, requiere DevOps maduro | 9/10 |
| **Serverless puro** | Sin gestión de servidores, auto-scale | Cold starts inaceptables para 500k req/día, vendor lock-in | 6/10 |

## 4. Consecuencias

### Positivas
- Cada microservicio escala de forma independiente según carga
- Fallos aislados: un servicio caído no colapsa el sistema completo
- Despliegues independientes por dominio funcional
- Tecnologías heterogéneas donde sea necesario (Go para core, Python para AI)
- Event sourcing facilita auditoría completa (RNF de trazabilidad)

### Negativas / Riesgos asumidos
- Mayor complejidad operacional → **Mitigación:** Kubernetes + observabilidad (OpenTelemetry)
- Consistencia eventual en lugar de inmediata → **Mitigación:** Outbox Pattern + idempotency keys
- Latencia de red entre servicios → **Mitigación:** gRPC para comunicación interna crítica

## 5. Patrones Complementarios Adoptados

| Patrón | Servicio | Justificación |
|--------|---------|---------------|
| **Outbox Pattern** | Case Service | Garantía at-least-once en publicación de eventos |
| **CQRS** | Analytics / Case | Separar lecturas de escrituras para rendimiento |
| **Circuit Breaker** | Todos los clientes HTTP | Resiliencia frente a fallos de dependencias |
| **API Gateway** | Kong | Centralizar auth, rate limiting y routing |
| **Anti-Corruption Layer** | Integración legados | Aislar el modelo de dominio de sistemas externos |
| **Saga (Choreography)** | Flujo de creación de caso | Coordinar múltiples servicios sin 2PC |

## 6. Revisión

Esta decisión debe revisarse si:
- El equipo de DevOps no alcanza madurez en Kubernetes en 3 meses
- El volumen de solicitudes cae significativamente del estimado
- El proyecto incorpora un proveedor de PaaS con oferta serverless que resuelva las limitaciones identificadas

---

*ADR-001 aprobado por el equipo técnico de Conecta360*
