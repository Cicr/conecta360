# ADR-006: API Gateway — Kong OSS en Kubernetes

> **Proyecto:** Conecta360  
> **ID:** ADR-006  
> **Estado:** Aprobado ✅  
> **Fecha:** 2026-05-11  
> **Área:** Arquitectura de Software / Seguridad  

---

## 📐 Contexto de la Decisión

Con 10+ microservicios, centralizar autenticación, logging, rate limiting y routing en cada servicio de forma independiente genera deuda técnica e inconsistencias de seguridad. Se requiere un API Gateway como punto de entrada único.

Los requerimientos de seguridad estrictos exigen que el gateway sea la primera línea de defensa (OWASP Top 10), con rate limiting por identidad de usuario —no solo por IP—, y que los plugins de seguridad utilizados sean mantenidos oficialmente. Kong OSS satisface estos criterios con un ecosistema de plugins oficiales para JWT, OAuth2, rate limiting, CORS y logging, todos con soporte activo. Adicionalmente, corre nativamente en Kubernetes mediante su operador oficial, lo que permite configuración declarativa versionada como código.

---

## 1. Contexto y Problema

Con 10+ microservicios, cada uno necesita autenticación, logging, rate limiting y routing. Sin un API Gateway centralizado, cada servicio implementa estos controles de forma inconsistente, generando deuda técnica y riesgos de seguridad. El gateway es también el punto donde se aplican las políticas de multi-tenancy antes de que el request llegue al servicio.

## 2. Decisión

**Se adopta Kong OSS (Open Source) 3.x desplegado en Kubernetes via Kong Operator:**

| Función | Plugin Kong | Detalles |
|---------|------------|---------|
| Autenticación JWT | `jwt` | Validación de JWT emitidos por Auth Service |
| OAuth2 (ciudadanos) | `oauth2` | Integración con SSO Nacional |
| Rate Limiting | `rate-limiting-advanced` | Por consumer (citizen_id o user_id) |
| CORS | `cors` | Whitelist de orígenes permitidos |
| Logging | `http-log` | Logs estructurados a Loki/ELK |
| Tracing | `opentelemetry` | Distributed tracing con OTLP |
| Request Size Limit | `request-size-limiting` | Máx. 10MB para evitar DoS |
| IP Restriction | `ip-restriction` | Whitelist para APIs de administración |
| Response Transformer | `response-transformer` | Eliminar headers sensibles de respuesta |

## 3. Configuración de Rate Limiting

```yaml
# Rate limiting por ciudadano — endpoints de creación de casos
plugins:
  - name: rate-limiting-advanced
    config:
      limit:
        - 10          # 10 solicitudes
      window_size:
        - 60          # por minuto
      identifier: consumer
      strategy: redis
      redis:
        host: redis-cluster
        port: 6379
      hide_client_headers: false
      error_message: "Demasiadas solicitudes. Intente en 60 segundos."
```

## 4. Routing de Microservicios

| Route | Servicio Backend | Método(s) | Auth requerida |
|-------|-----------------|-----------|---------------|
| `/api/v1/cases` | Case Service | POST, GET | JWT (ciudadano o funcionario) |
| `/api/v1/cases/{id}` | Case Service | GET, PUT | JWT |
| `/api/v1/citizens` | Citizen Service | POST, GET, PUT | JWT (ciudadano = solo sus datos) |
| `/api/v1/notifications` | Notification Service | GET | JWT |
| `/api/v1/analytics/*` | Analytics Service | GET | JWT (supervisor, admin) |
| `/api/v1/institutions` | Institution Service | CRUD | JWT (admin) |
| `/api/v1/chat` | Chatbot Service | POST | JWT o anónimo |
| `/health` | Todos | GET | Sin auth |
| `/admin/*` | Kong Admin | ALL | IP Restriction + mTLS |

## 5. Alternativas Evaluadas

| Opción | Ventajas | Desventajas | Score |
|--------|---------|-------------|:---:|
| **Kong OSS** ✅ | Plugin ecosystem maduro, K8s native, declarativo | License Enterprise para algunas features avanzadas | 9/10 |
| **NGINX + Lua** | Ultra-performante, base conocida | Config imperativa, plugins a medida, mantenimiento | 6/10 |
| **AWS API Gateway** | Serverless, integración AWS nativa | Vendor lock-in, datos fuera de CR, costo escala | 5/10 |
| **Traefik** | Nativo K8s, auto-discovery | Ecosistema de plugins más pequeño para seguridad | 7/10 |
| **Envoy / Istio Ingress** | Altamente performante, parte de service mesh | Complejidad extrema, overkill para gateway solo | 6/10 |

## 6. Consecuencias

### Positivas
- Configuración declarativa (CRDs de Kubernetes): la config es código versionable
- Plugins oficiales mantenidos con parches de seguridad
- Observabilidad built-in: métricas Prometheus, trazas OTLP, logs estructurados
- Separación de concerns: cada microservicio se enfoca en lógica de negocio, no en auth/logging
- Kong Admin API facilita configuración programática desde pipelines CI/CD

### Negativas / Riesgos asumidos
- Kong es un single point of failure → **Mitigación:** Kong desplegado en 3+ réplicas con HPA, datos en PostgreSQL de Kong separado
- Actualización de Kong puede romper plugins → **Mitigación:** Ambiente de staging obligatorio, testing automatizado de routes en CI/CD
- Kong Enterprise tiene WAF nativo, Kong OSS no → **Mitigación:** Agregar ModSecurity como sidecar o usar el WAF del load balancer cloud

## 7. Política de Seguridad Obligatoria en el Gateway

```
┌─────────────────────────────────────────────┐
│           Request Flow — Kong               │
│                                             │
│  Request                                    │
│     │                                       │
│     ▼                                       │
│  1. TLS Termination (TLS 1.3 mínimo)        │
│     │                                       │
│     ▼                                       │
│  2. IP Allow/Block List                     │
│     │                                       │
│     ▼                                       │
│  3. Request Size Limiting (máx 10MB)        │
│     │                                       │
│     ▼                                       │
│  4. Rate Limiting (por consumer)            │
│     │                                       │
│     ▼                                       │
│  5. JWT/OAuth2 Validation                   │
│     │                                       │
│     ▼                                       │
│  6. CORS Policy Check                       │
│     │                                       │
│     ▼                                       │
│  7. Route to Microservice (mTLS interna)    │
│     │                                       │
│     ▼                                       │
│  8. Response: Strip sensitive headers       │
└─────────────────────────────────────────────┘
```

---

*Conecta360 v1.0 — ADR-006*
