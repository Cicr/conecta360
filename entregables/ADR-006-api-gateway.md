# ADR-006: API Gateway — Kong OSS en Kubernetes

> **Proyecto:** Conecta360  
> **ID:** ADR-006  
> **Estado:** Aprobado ✅  
> **Fecha:** 2026-05-11  
> **Decisión tomada por:** Ariel Montero (Arquitecto de Software)  
> **Revisado por:** Carlos Fuentes (Seg), Eduardo Lara (PM)  

---

## 🎭 Contexto de la Decisión

**Ariel (Arq):** Necesitamos un API Gateway que sea el punto de entrada único para los 10 microservicios. Las funciones mínimas son: routing, autenticación JWT, rate limiting y terminación TLS.

**Carlos (Seg):** Y WAF. No quiero que OWASP Top 10 llegue a los microservicios. El gateway debe ser la primera línea de defensa. También necesito que el rate limiting sea por ciudadano, no solo por IP.

**Eduardo (PM):** ¿Kong, NGINX o un gateway cloud?

**Ariel (Arq):** Kong tiene el mejor ecosistema de plugins para lo que necesitamos: JWT, OAuth2, rate limiting, CORS, logging. Y corre en Kubernetes nativamente con el operador oficial. NGINX haría lo básico pero necesitaríamos codificar mucho.

**Carlos (Seg):** ¿Kong tiene plugins de seguridad certificados? Necesito algo que pueda auditar, no un plugin de comunidad que nadie mantiene.

**Ariel (Arq):** Los plugins que necesitamos (JWT, OAuth2, rate-limiting, IP restriction) son todos plugins oficiales de Kong, mantenidos por la empresa. Puedo presentar el análisis de cada uno.

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

*ADR-006 aprobado por el equipo técnico de Conecta360*
