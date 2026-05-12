# 📋 Listado de Requerimientos — Conecta360

> **Proyecto:** Sistema Integral de Atención Ciudadana "Conecta360"  
> **País:** Costa Verde  
> **Versión:** 1.0  
> **Fecha:** 2026-05-11  

---

## 🎭 Sesión de Análisis — Conversación del Equipo

> Esta tabla fue construida mediante una sesión colaborativa de análisis entre los siguientes roles:
>
> - **🏛️ Ariel Montero** — Arquitecto de Software
> - **📊 Beatriz Salcedo** — Analista de Sistemas / Business Analyst
> - **🔒 Carlos Fuentes** — Experto en Seguridad

---

### Extracto de la sesión de análisis

**Beatriz (BA):** Empecemos por mapear los módulos del PRD. El punto 4 nos da cuatro módulos: Atención Ciudadana, Gestión Institucional, Analítico e Integraciones. Voy a catalogarlos uno a uno.

**Ariel (Arq):** Estoy de acuerdo. Pero mientras los enumeramos, necesito que marquemos cuáles implican estado persistente, cuáles son en tiempo real y cuáles pueden ser eventualmente consistentes. Eso impacta la arquitectura.

**Carlos (Seg):** Y yo necesito saber qué requerimientos tocan datos de ciudadanos o datos de auditoría. Ahí aplican controles de acceso, cifrado y trazabilidad obligatorios.

**Beatriz:** Perfecto. Acordemos tres criterios de análisis: *Complejidad técnica*, *Sensibilidad de datos* y *Criticidad de negocio*.

**Ariel:** Añadiría un cuarto: *Dependencias externas*, porque el SSO, WhatsApp y redes sociales son integraciones que no controlamos.

**Carlos:** Absolutamente. Un ciudadano que envía su número de identidad por WhatsApp está exponiendo datos PII. Necesitamos que eso quede marcado en la tabla.

**Beatriz:** Entonces la tabla tiene: ID, Módulo, Requerimiento, Tipo (Funcional/No Funcional), Complejidad, Sensibilidad de datos, Criticidad de negocio, Observaciones de seguridad y Estado de análisis.

---

## 📌 Sección A — Requerimientos Funcionales (Punto 4)

| ID | Módulo | Requerimiento | Tipo | Complejidad Técnica | Sensibilidad de Datos | Criticidad de Negocio | Observación Arquitectura | Observación Seguridad | Estado |
|----|--------|---------------|------|--------------------|-----------------------|----------------------|--------------------------|-----------------------|--------|
| RF-01 | 4.1 – Atención Ciudadana | Registro de incidencias y solicitudes (bache, queja, solicitud de documento, reporte de robo) | Funcional | Alta | Alta — datos personales + geolocalización | Crítica | Requiere API REST con validación de esquema, persistencia en BD relacional + evento publicado en bus de mensajería | Datos PII del ciudadano deben cifrarse en reposo (AES-256) y en tránsito (TLS 1.3). Auditar cada creación. | Analizado ✅ |
| RF-02 | 4.1 – Atención Ciudadana | Seguimiento del caso mediante número único universal | Funcional | Media | Media — identificador de caso + historial | Crítica | UUID v7 (ordenado por tiempo) como número de caso. Endpoint GET `/cases/{id}` con paginación de historial | El número de caso NO debe ser secuencial predecible. Usar UUID o hash opaco. | Analizado ✅ |
| RF-03 | 4.1 – Atención Ciudadana | Notificaciones automáticas (correo, SMS o push) al haber avances | Funcional | Media | Media — correo o número de teléfono del ciudadano | Alta | Patrón Observer/Event-Driven. Publicar evento `case.status.updated` a cola de mensajería. Notification Service consume. | Opt-in obligatorio por canal. No enviar notificaciones sin consentimiento explícito del ciudadano (GDPR-equivalente local). | Analizado ✅ |
| RF-04 | 4.1 – Atención Ciudadana | Chatbot con IA para atención inicial y clasificación de casos | Funcional | Muy Alta | Alta — conversación contiene datos personales potenciales | Alta | Integración con LLM externo (OpenAI/Azure AI) o modelo propio. Patrón BFF (Backend for Frontend) para canal web/móvil. | Las conversaciones deben anonimizarse o cifrarse. No almacenar texto libre sin política de retención. | Analizado ✅ |
| RF-05 | 4.2 – Gestión Institucional | Derivación automática de casos según categoría o dependencia | Funcional | Alta | Baja — datos de gestión interna | Crítica | Motor de reglas (Drools, o tabla de enrutamiento configurable). Evento `case.created` dispara asignación automática. | Solo funcionarios autorizados pueden ver casos de su dependencia. RBAC estricto por institución. | Analizado ✅ |
| RF-06 | 4.2 – Gestión Institucional | SLA configurables por tipo de solicitud | Funcional | Media | Baja | Alta | Servicio de configuración (Config Service) con SLA por categoría. Scheduler que evalúa SLA cada N minutos. | Integridad de la configuración: solo admins pueden modificar SLAs. Log de auditoría por cambio de configuración. | Analizado ✅ |
| RF-07 | 4.2 – Gestión Institucional | Panel para supervisores con métricas de productividad y cumplimiento | Funcional | Alta | Media — datos agregados de funcionarios | Alta | CQRS: las lecturas van a un Read Model (Elasticsearch o BI store). Evitar queries pesadas en BD principal. | El panel debe mostrar datos anonimizados cuando el volumen sea bajo (privacidad diferencial básica). | Analizado ✅ |
| RF-08 | 4.2 – Gestión Institucional | Reasignación automática en caso de inactividad o congestión | Funcional | Alta | Baja | Alta | Scheduler con heartbeat por agente. Threshold de inactividad configurable. Evento `case.reassigned`. | Registro de auditoría por cada reasignación: quién, cuándo, motivo. | Analizado ✅ |
| RF-09 | 4.3 – Módulo Analítico | Dashboard central con KPIs globales y por institución | Funcional | Alta | Media | Media | Capa de BI separada. Refresco periódico (near-real-time con streaming o batch por ETL). | Los KPIs no deben exponer datos individuales. Agregaciones mínimas de 5 registros (privacidad). | Analizado ✅ |
| RF-10 | 4.3 – Módulo Analítico | Reportes de desempeño (tiempos promedio, casos cerrados, satisfacción ciudadana) | Funcional | Media | Media | Media | Exportación asíncrona. Job de generación + notificación por webhook/email cuando esté listo. | Los reportes descargados deben ser firmados digitalmente o con hash para detectar manipulación. | Analizado ✅ |
| RF-11 | 4.3 – Módulo Analítico | Exportación de datos a Power BI o Data Warehouse | Funcional | Alta | Alta — datos masivos de ciudadanos | Media | Conector OData o API de exportación con paginación por cursor. CDC (Change Data Capture) para sincronización. | Requiere acuerdo de procesamiento de datos (DPA) con el proveedor de Power BI. Enmascarar PII en exports. | Analizado ✅ |
| RF-12 | 4.4 – Integraciones | SSO con plataforma de autenticación ciudadana nacional (OpenID Connect) | Funcional | Alta | Muy Alta — identidad del ciudadano | Crítica | API Gateway con validación de JWT. Identity Provider federado. Refresh token rotation. | Implementar PKCE obligatorio. Token binding cuando sea posible. Blacklist de tokens revocados. | Analizado ✅ |
| RF-13 | 4.4 – Integraciones | Sistema de mensajería (correo, SMS, push notifications) | Funcional | Media | Alta — datos de contacto | Alta | Notification Service desacoplado. Patrón Outbox para garantizar entrega al menos una vez. | Canal de mensajería debe ser TLS. API keys de proveedores (Twilio, SendGrid) en vault (HashiCorp/AWS Secrets). | Analizado ✅ |
| RF-14 | 4.4 – Integraciones | Integración con redes sociales (Facebook, Twitter/X, WhatsApp Business) | Funcional | Muy Alta | Alta — datos de usuarios de terceros | Alta | Canal Adapter por plataforma. Gateway de integración (MuleSoft / Apache Camel / custom). | Cumplir políticas de uso de APIs de cada red social. WhatsApp: cumplir GDPR/CCPA. Revisar ley local de protección de datos. | Analizado ✅ |
| RF-15 | 4.4 – Integraciones | APIs de sistemas internos (salud, obras públicas, energía) | Funcional | Alta | Alta — datos institucionales sensibles | Crítica | Anti-Corruption Layer (ACL) por sistema legado. Circuit Breaker (Resilience4j / Polly). Timeout por defecto: 5s. | mTLS para comunicación entre servicios internos. Autenticación por certificado para sistemas legados. | Analizado ✅ |

---

## 📌 Sección B — Requerimientos No Funcionales (Punto 5)

| ID | Categoría | Requisito | Complejidad de Implementación | Impacto Arquitectónico | Observación Arquitectura | Observación Seguridad | Estado |
|----|-----------|-----------|-------------------------------|------------------------|---------------------------|-----------------------|--------|
| RNF-01 | Disponibilidad | 99.9% SLA, con failover automático entre regiones | Alta | Muy Alto | Multi-región activo-activo o activo-pasivo. Health checks en API Gateway. Circuit Breakers. | El failover no debe exponer endpoints de administración. WAF activo en ambas regiones. | Analizado ✅ |
| RNF-02 | Escalabilidad | Hasta 500,000 solicitudes diarias concurrentes | Muy Alta | Muy Alto | ~5.7 req/seg promedio, pero con picos de hasta 10x. Kubernetes HPA + KEDA para escalado basado en cola. | Bajo carga extrema, no deshabilitar controles de seguridad (rate limiting, WAF). | Analizado ✅ |
| RNF-03 | Seguridad | Cifrado AES-256, OAuth2 / OpenID Connect, segregación por entidad | Alta | Alto | Integrar con Vault para gestión de claves. Implementar rotación de claves sin downtime. | Revisión anual de certificados y claves. Pruebas de penetración semestrales. DAST/SAST en pipeline CI/CD. | Analizado ✅ |
| RNF-04 | Rendimiento | < 1.5 segundos de tiempo promedio de respuesta | Alta | Alto | APM (Datadog / New Relic / OpenTelemetry). Caché (Redis) para lecturas frecuentes. CDN para assets. | Los caché no deben servir datos de un ciudadano a otro. Cache-key incluir tenant/user ID. | Analizado ✅ |
| RNF-05 | Multitenencia | Soporte multi-institución con aislamiento lógico de datos | Muy Alta | Muy Alto | Schema-per-tenant o Row-Level Security (RLS) en PostgreSQL. Middleware de resolución de tenant. | Pruebas de cross-tenant data leakage obligatorias antes de cada release. | Analizado ✅ |
| RNF-06 | Integrabilidad | APIs REST y mensajería asíncrona (Kafka o RabbitMQ) | Alta | Alto | Kafka para eventos de alto volumen. RabbitMQ para comandos point-to-point. Schema Registry (Avro/Protobuf). | Los topics de Kafka deben tener ACLs. Sin consumers anónimos. | Analizado ✅ |

---

## 📝 Acuerdos del Equipo

| # | Acuerdo | Responsable | Fecha |
|---|---------|-------------|-------|
| 1 | Todos los requerimientos que toquen datos PII se marcan como "Sensibilidad Alta" y requieren revisión de seguridad antes de diseño técnico | Carlos (Seg) | 2026-05-11 |
| 2 | Los requerimientos RF-12 (SSO) y RF-15 (APIs internas) son los de mayor riesgo arquitectónico y se abordan primero en el backlog técnico | Ariel (Arq) | 2026-05-11 |
| 3 | El BA será responsable de mantener este documento actualizado conforme avance el proyecto | Beatriz (BA) | 2026-05-11 |
| 4 | Cualquier cambio de requerimiento debe pasar por un proceso de Change Request documentado | Equipo | 2026-05-11 |

---

*Documento generado por sesión colaborativa del equipo técnico — Conecta360 v1.0*
