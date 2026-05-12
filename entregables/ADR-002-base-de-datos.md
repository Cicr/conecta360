# ADR-002: Base de Datos — PostgreSQL 16 con Row-Level Security (Multi-tenant)

> **Proyecto:** Conecta360  
> **ID:** ADR-002  
> **Estado:** Aprobado ✅  
> **Fecha:** 2026-05-11  
> **Área:** Arquitectura de Datos  

---

## 📐 Contexto de la Decisión

El PRD establece como requerimiento no funcional la **multitenencia con aislamiento lógico de datos** (RNF-05). Los modelos clásicos de multitenencia fueron evaluados: schema-per-tenant, database-per-tenant y Row-Level Security (RLS) en schema único.

Las entidades del dominio presentan relaciones relacionales claras (ciudadano → caso → eventos → notificaciones) con requerimiento de consistencia ACID, lo que descarta motores NoSQL. El RLS de PostgreSQL aplica el aislamiento a nivel de motor de base de datos, lo que provee defensa en profundidad más allá de los controles de la capa de aplicación. Con particionamiento correcto sobre `institution_id` y rangos de fecha en tablas de alto volumen, el modelo es operacionalmente viable para 50 instituciones y 10+ millones de casos.

---

## 1. Contexto y Problema

El sistema debe almacenar datos de múltiples instituciones gubernamentales con aislamiento lógico garantizado. Los datos incluyen información PII de ciudadanos, casos con historial inmutable y registros de auditoría. Se requiere ACID, búsqueda flexible y cumplimiento de seguridad de nivel gobierno.

## 2. Decisión

**Se adopta PostgreSQL 16 como base de datos relacional principal con:**
- **Row-Level Security (RLS)** para aislamiento multi-tenant por institución
- **Schema único** con columna `institution_id` en todas las entidades relevantes
- **Particionamiento por rango de fecha** para tablas de alto volumen (CaseEvent, AuditLog, Notification)
- **Réplica de lectura** para CQRS (Read Model)
- **Cifrado de columnas** con pgcrypto para campos PII
- **Object Storage:** no seleccionado — la elección (autoalojado o nube) es dependiente de los recursos del proyecto
- **Gestión de secretos:** no seleccionada — la estrategia es dependiente del manejo del proyecto

## 3. Alternativas Evaluadas

| Opción | Ventajas | Desventajas | Score |
|--------|---------|-------------|:---:|
| **Schema-per-tenant (PostgreSQL)** | Aislamiento total, fácil backup por tenant | 50+ schemas difíciles de mantener, migraciones costosas | 6/10 |
| **Database-per-tenant** | Aislamiento total, independencia total | 50+ bases de datos, imposible de gestionar | 3/10 |
| **PostgreSQL + RLS** ✅ | Aislamiento en motor, single schema, flexible | Requiere SET LOCAL en cada conexión | 9/10 |
| **MongoDB** | Esquema flexible, JSON nativo | Sin ACID completo en transacciones multi-doc, joins costosos | 5/10 |
| **Oracle** | Maduro, multi-tenant nativo | Costo de licenciamiento prohibitivo para gobierno | 4/10 |
| **CockroachDB** | Distribuido, PostgreSQL compatible | Complejidad operacional, costo en cloud | 6/10 |

## 4. Implementación de Multi-tenancy con RLS

```sql
-- Activar RLS en la tabla Case
ALTER TABLE cases ENABLE ROW LEVEL SECURITY;

-- Política de aislamiento por institución
CREATE POLICY tenant_isolation ON cases
    USING (institution_id = current_setting('app.current_tenant')::UUID);

-- Cada conexión del microservicio establece el tenant
SET LOCAL app.current_tenant = '123e4567-e89b-12d3-a456-426614174000';
```

## 5. Estrategia de Particionamiento

```sql
-- CaseEvent: particionado por mes (append-only, alto volumen)
CREATE TABLE case_events (
    event_id UUID,
    case_id UUID,
    created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE case_events_2026_05
    PARTITION OF case_events
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

## 6. Campos PII Cifrados

| Campo | Entidad | Método | Gestión de clave |
|-------|---------|--------|-----------------|
| `national_id` | Citizen | AES-256-GCM (pgcrypto) | A definir según proyecto |
| `email` | Citizen | AES-256-GCM | A definir según proyecto |
| `phone_number` | Citizen | AES-256-GCM | A definir según proyecto |
| `password_hash` | User | bcrypt (factor 12) | N/A (hash unidireccional) |

> **Nota del Arquitecto:** La herramienta de gestión de secretos (HashiCorp Vault, AWS Secrets Manager, Kubernetes Secrets cifrados u otra) se determinará según los recursos y restricciones operacionales del proyecto.

## 7. Consecuencias

### Positivas
- ACID garantizado en todas las transacciones
- RLS aplicado a nivel de motor (defensa en profundidad)
- Herramientas maduras: pg_dump, pgAdmin, Flyway/Liquibase
- Extensiones: pgcrypto (cifrado), pg_partman (particionado), pg_stat_statements (performance)
- Compatible con herramientas de BI estándar

### Negativas / Riesgos asumidos
- Requiere `SET LOCAL` en cada conexión del pool → **Mitigación:** middleware de infraestructura en cada servicio
- Escalabilidad vertical antes que horizontal → **Mitigación:** read replicas + particionado + caché Redis
- pgcrypto agrega latencia en campos cifrados → **Mitigación:** índices sobre hashes para búsquedas

## 8. Herramientas de Gestión

| Herramienta | Uso |
|-------------|-----|
| **Flyway** | Migraciones de base de datos versionadas |
| **pgAdmin 4** | Administración visual de la BD |
| **pg_partman** | Gestión automática de particiones |
| **Patroni** | Alta disponibilidad y failover automático |
| **pgBouncer** | Connection pooling |

---

*Conecta360 v1.0 — ADR-002*
