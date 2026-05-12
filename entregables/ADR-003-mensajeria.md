# ADR-003: Mensajería Asíncrona — Apache Kafka con Schema Registry

> **Proyecto:** Conecta360  
> **ID:** ADR-003  
> **Estado:** Aprobado ✅  
> **Fecha:** 2026-05-11  
> **Área:** Arquitectura de Software / Integraciones  

---

## 📐 Contexto de la Decisión

El PRD menciona explícitamente "Kafka o RabbitMQ" como opción de mensajería asincrónica. El sistema requiere desacoplar productores y consumidores para los flujos de creación de casos, notificaciones, sincronización con sistemas legados y alimentación del módulo analítico.

La decisión consideró dos factores clave: (1) el **volumen** esperado de 500,000 solicitudes diarias con picos de 10x hace a Kafka significativamente superior en throughput; (2) la **retención de eventos** de Kafka permite reconstruir proyecciones analíticas y aplicar replay ante fallos, algo imposible con RabbitMQ en su configuración estándar. Los ACLs granulares por tópic garantizan aislamiento entre servicios (ninguna restricción de acceso dependiente solo de la capa de aplicación).

---

## 1. Contexto y Problema

El sistema requiere un bus de mensajería para desacoplar productores y consumidores de eventos: creación de casos, actualizaciones de estado, notificaciones, sincronización con sistemas legados y alimentación del módulo analítico. El volumen esperado es de 500,000 solicitudes diarias, con picos de hasta 10x en eventos críticos.

## 2. Decisión

**Se adopta Apache Kafka 3.x con Confluent Schema Registry:**
- Kafka para todos los eventos del dominio de alto volumen
- Schema Registry con Avro/JSON Schema para compatibilidad de contratos
- **Tópicos definidos** por dominio con naming convention clara
- ACLs por topic para aislamiento de servicios

## 3. Alternativas Evaluadas

| Opción | Ventajas | Desventajas | Score |
|--------|---------|-------------|:---:|
| **Apache Kafka** ✅ | Alto throughput, retención, replay, ACLs, Schema Registry | Mayor complejidad, requires ZooKeeper/KRaft | 9/10 |
| **RabbitMQ** | Simple, bien conocido, AMQP estándar | Mensajes efímeros por defecto, menor throughput para big data | 7/10 |
| **AWS SQS/SNS** | Serverless, sin gestión | Vendor lock-in, no soporta replay completo, costo escala | 6/10 |
| **Google Pub/Sub** | Serverless, al menos 7 días de retención | Vendor lock-in, modelo de precios imprevisible | 6/10 |
| **NATS JetStream** | Ultra-ligero, bajo latencia | Ecosistema más pequeño, menor soporte institucional | 5/10 |

## 4. Definición de Topics

```
# Naming convention: {dominio}.{entidad}.{evento}

# Domain: cases
cases.case.created          # Un caso fue creado
cases.case.updated          # Estado o datos del caso fueron actualizados
cases.case.assigned         # El caso fue asignado a un funcionario
cases.case.reassigned       # El caso fue reasignado
cases.case.resolved         # El caso fue marcado como resuelto
cases.case.closed           # El caso fue cerrado definitivamente
cases.case.escalated        # El caso fue escalado por SLA

# Domain: notifications
notifications.notification.requested    # Se solicita enviar una notificación
notifications.notification.sent         # La notificación fue enviada al proveedor
notifications.notification.delivered    # El proveedor confirmó entrega
notifications.notification.failed       # El envío falló (retry logic)

# Domain: analytics
analytics.kpi.snapshot      # Snapshot periódico de KPIs para DWH

# Domain: audit
audit.event.recorded        # Cualquier evento de auditoría del sistema
```

## 5. ACLs por Servicio

| Servicio | Topics — Produce | Topics — Consume |
|---------|------------------|-----------------|
| Case Service | cases.case.* | — |
| Notification Service | notifications.notification.* | cases.case.*, notifications.notification.requested |
| Analytics Service | analytics.kpi.* | cases.case.*, notifications.notification.*, audit.event.* |
| Audit Service | audit.event.* | — |
| Institution Service | — | cases.case.created (para SLA tracking) |

## 6. Schema de Eventos (ejemplo Avro)

```json
{
  "namespace": "conecta360.cases",
  "type": "record",
  "name": "CaseCreated",
  "fields": [
    {"name": "event_id", "type": "string"},
    {"name": "case_id", "type": "string"},
    {"name": "case_number", "type": "string"},
    {"name": "citizen_id", "type": "string"},
    {"name": "category_code", "type": "string"},
    {"name": "institution_id", "type": "string"},
    {"name": "priority", "type": {"type": "enum", "name": "Priority",
      "symbols": ["low", "medium", "high", "critical"]}},
    {"name": "channel", "type": "string"},
    {"name": "sla_deadline", "type": "string", "doc": "ISO 8601"},
    {"name": "occurred_at", "type": "string", "doc": "ISO 8601"}
  ]
}
```

## 7. Consecuencias

### Positivas
- Throughput > 1,000,000 mensajes/segundo en configuración estándar
- Retención configurable (30 días default) para replay y reconstrucción
- Schema Registry garantiza compatibilidad hacia atrás entre versiones
- ACLs granulares por topic para aislamiento de servicios
- Ecosistema maduro: Kafka Connect para integración con sistemas legados

### Negativas / Riesgos asumidos
- Complejidad operacional de Kafka → **Mitigación:** Kubernetes Operator (Strimzi) o MSK en AWS
- Overhead del Schema Registry → **Mitigación:** Caché local de schemas en cada servicio
- Sin garantía de orden entre particiones → **Mitigación:** Usar `case_id` como partition key para orden por caso

## 8. Configuración Recomendada

```properties
# Replication factor mínimo para producción
default.replication.factor=3
min.insync.replicas=2

# Retención
log.retention.hours=720  # 30 días

# Compresión
compression.type=lz4

# Producer: durabilidad
acks=all
enable.idempotence=true
```

---

*Conecta360 v1.0 — ADR-003*
