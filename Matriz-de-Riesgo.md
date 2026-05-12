# ⚠️ Matriz de Riesgo — Conecta360

> **Proyecto:** Sistema Integral de Atención Ciudadana "Conecta360"  
> **País:** Costa Verde  
> **Versión:** 1.0  
> **Fecha:** 2026-05-11  

---

## 🎭 Sesión de Análisis de Riesgos — Las 5 Personas

> **Participantes:**
> - **🏛️ Ariel Montero** — Arquitecto de Software  
> - **📊 Beatriz Salcedo** — Analista de Sistemas / Business Analyst  
> - **🔒 Carlos Fuentes** — Experto en Seguridad  
> - **🗄️ Diana Rivas** — Arquitecta de Datos  
> - **📅 Eduardo Lara** — Project Manager  

---

### Extracto de la sesión de análisis de riesgos

**Eduardo (PM):** Bien equipo, vamos a atacar el punto 6 del PRD —los 7 desafíos técnicos y de negocio. Mi objetivo es convertir cada uno en un riesgo cuantificable con probabilidad, impacto, estrategia de mitigación y dueño. Sin eso, no puedo hablarle al cliente de plazos ni de presupuesto con honestidad.

**Ariel (Arq):** Los desafíos 1 y 3 los veo directamente en mi área. La integración heterogénea con sistemas sin API puede duplicar el tiempo de desarrollo. He visto proyectos así tardar el triple por la legacidad.

**Diana (AD):** El desafío 2 —interoperabilidad semántica— también es crítico desde datos. Si cada institución llama distinto a la misma entidad (un "caso" en salud es un "expediente" en obras públicas), el modelo de datos se fragmenta. Necesito un Diccionario de Datos canónico antes de construir cualquier cosa.

**Carlos (Seg):** El desafío 4 —gestión de identidades complejas— me preocupa mucho. Tenemos tres tipos de usuarios: ciudadanos, funcionarios y operadores. Si mezclamos sus permisos, un ciudadano podría ver expedientes de otro. Eso es violación de datos. Necesitamos un modelo RBAC/ABAC desde el día uno.

**Beatriz (BA):** Y el desafío 7 —cambio cultural— lo subestiman siempre. He visto sistemas técnicamente perfectos que fracasan porque los funcionarios siguen usando Excel. Necesitamos un plan de gestión del cambio real, no solo capacitación de un día.

**Eduardo (PM):** Perfecto. Usamos la escala estándar: Probabilidad (1=Raro, 2=Poco probable, 3=Posible, 4=Probable, 5=Casi seguro) e Impacto (1=Insignificante, 2=Menor, 3=Moderado, 4=Mayor, 5=Catastrófico). El score = P × I. 1-4 = Bajo, 5-9 = Medio, 10-14 = Alto, 15-25 = Crítico.

---

## 📊 Escala de Evaluación

| Probabilidad | Descripción | Impacto | Descripción |
|:---:|---|:---:|---|
| 1 | Raro (< 10%) | 1 | Insignificante |
| 2 | Poco probable (10–30%) | 2 | Menor |
| 3 | Posible (30–50%) | 3 | Moderado |
| 4 | Probable (50–70%) | 4 | Mayor |
| 5 | Casi seguro (> 70%) | 5 | Catastrófico |

| Score | Nivel | Color |
|:---:|---|---|
| 1–4 | 🟢 Bajo | Verde |
| 5–9 | 🟡 Medio | Amarillo |
| 10–14 | 🟠 Alto | Naranja |
| 15–25 | 🔴 Crítico | Rojo |

---

## 🗂️ Matriz de Riesgos

| ID | Desafío (PRD §6) | Riesgo | Probabilidad (P) | Impacto (I) | Score (P×I) | Nivel | Estrategia de Mitigación | Plan de Contingencia | Dueño | Revisión |
|----|-----------------|--------|:---:|:---:|:---:|---|---|---|---|---|
| R-01 | **Desafío 1** — Integración heterogénea | Las dependencias institucionales con sistemas legacy sin APIs bloquean la integración, generando retrasos de hasta 6 meses por institución | 4 | 5 | 20 | 🔴 Crítico | (1) Inventario de APIs existentes semana 1. (2) Definir patrón Adapter/ACL por sistema legacy. (3) Priorizar instituciones con mayor volumen de casos. (4) Proveer SDK de integración estándar. | Implementar scraping estructurado como puente temporal mientras se desarrolla el adapter. Prever presupuesto de contingencia del 20% por cada integración legada. | Ariel (Arq) | Mensual |
| R-02 | **Desafío 2** — Interoperabilidad semántica | Diferentes instituciones usan vocabularios distintos para las mismas entidades (caso, expediente, solicitud, ticket), causando inconsistencias en el modelo de datos | 5 | 4 | 20 | 🔴 Crítico | (1) Crear Diccionario de Datos Canónico en sprint 1. (2) Taller de alineación semántica con representantes de cada institución. (3) Modelo de dominio compartido versionado. | Usar ontología neutra como árbitro (ej. schema.org adaptado). Mantener mapeos de traducción por institución. | Diana (AD) | Quincenal |
| R-03 | **Desafío 3** — Alta disponibilidad geográfica | Un desastre natural o fallo eléctrico en el centro de datos principal deja el sistema completamente inoperativo | 3 | 5 | 15 | 🔴 Crítico | (1) Arquitectura multi-región activo-pasivo mínimo. (2) RTO < 4h, RPO < 1h definidos en contrato. (3) Runbook de failover documentado y practicado trimestralmente. | Activación de región secundaria en < 15 minutos mediante proceso automatizado (IaC). Modo degradado sin módulo analítico. | Ariel (Arq) | Trimestral |
| R-04 | **Desafío 4** — Gestión de identidades complejas | Un funcionario de la institución A accede a casos de la institución B por error de configuración de roles (cross-tenant leakage) | 3 | 5 | 15 | 🔴 Crítico | (1) RBAC + ABAC desde sprint 1. (2) Tests automatizados de aislamiento de tenant en pipeline CI/CD. (3) Revisión de permisos por auditoría interna trimestral. | Revocar sesiones activas de forma masiva. Notificar a DPA (Autoridad de Protección de Datos). Plan de comunicación a ciudadanos afectados. | Carlos (Seg) | Quincenal |
| R-05 | **Desafío 5** — Datos sensibles | Brecha de seguridad que expone datos personales de ciudadanos (PII) por vulnerabilidad en la API pública | 3 | 5 | 15 | 🔴 Crítico | (1) SAST y DAST en pipeline CI/CD. (2) Pen testing semestral por tercero certificado. (3) Cifrado AES-256 en reposo y TLS 1.3 en tránsito. (4) WAF en perímetro. | Plan de respuesta a incidentes documentado (< 72h de notificación per GDPR-equiv). Forensics team en standby. | Carlos (Seg) | Mensual |
| R-06 | **Desafío 6** — Balance centralización vs. autonomía | Las instituciones rechazan el sistema por sentir que pierden control de sus datos y procesos propios | 4 | 4 | 16 | 🔴 Crítico | (1) Modelo de datos con autonomía institucional garantizada (aislamiento lógico). (2) Governance board con representantes institucionales. (3) Dashboards propios por institución. | Modo "federado" donde cada institución controla su instancia con sincronización central limitada. | Beatriz (BA) + Eduardo (PM) | Mensual |
| R-07 | **Desafío 7** — Cambio cultural | Los funcionarios rechazan adoptar el nuevo sistema y continúan usando canales informales (papel, correo personal, WhatsApp), dejando casos sin registrar | 5 | 3 | 15 | 🔴 Crítico | (1) Plan de Gestión del Cambio formal con sponsor ejecutivo. (2) Capacitación por roles (no masiva). (3) Quick wins visibles en 30 días. (4) KPIs de adopción monitoreados. | Campaña de incentivos por adopción. Escalado a nivel gerencial si resistencia > 30%. | Beatriz (BA) + Eduardo (PM) | Semanal (primeros 3 meses) |
| R-08 | Riesgo adicional identificado | Scope creep: cada institución solicita funcionalidades personalizadas que inflan el alcance y retrasan las entregas core | 4 | 4 | 16 | 🔴 Crítico | (1) MoSCoW por requerimiento. (2) Change Request Board formal. (3) Backlog versionado y priorizado públicamente. | Congelar alcance de release actual. Funcionalidades adicionales van a roadmap v2.0. | Eduardo (PM) | Mensual |
| R-09 | Riesgo adicional identificado | Dependencia de terceros (WhatsApp Business API, redes sociales) que cambian sus políticas o precios unilateralmente | 3 | 3 | 9 | 🟡 Medio | (1) Patrón Adapter por canal externo. (2) Cláusula contractual de SLA con proveedores. (3) Canal alternativo siempre disponible (email como fallback). | Activar canal alternativo en < 24h. Notificar a ciudadanos del cambio temporal. | Ariel (Arq) | Trimestral |
| R-10 | Riesgo adicional identificado | El chatbot con IA genera respuestas incorrectas o sesgadas que desinforman al ciudadano | 3 | 4 | 12 | 🟠 Alto | (1) Human-in-the-loop para casos de baja confianza (< 80% score). (2) Evaluación semanal de accuracy del modelo. (3) Opción siempre disponible de escalar a agente humano. | Desactivar respuestas automáticas del chatbot. Modo solo-FAQ de bajo riesgo. | Ariel (Arq) + Beatriz (BA) | Semanal |
| R-11 | Riesgo adicional identificado | Falta de financiamiento o cambio de gobierno que paralice el proyecto | 2 | 5 | 10 | 🟠 Alto | (1) Arquitectura modular que permite entregas parciales con valor independiente. (2) Documentación suficiente para handover a otro equipo. | Priorizar módulos de mayor impacto visible (RF-01, RF-02). Asegurar continuidad contractual. | Eduardo (PM) | Trimestral |
| R-12 | Riesgo adicional identificado | El volumen de 500,000 solicitudes diarias es subestimado y el sistema colapsa en picos (elecciones, catástrofes) | 3 | 5 | 15 | 🔴 Crítico | (1) Load testing con 3x el volumen esperado antes de go-live. (2) Autoscaling configurado. (3) Circuit breakers y degraded mode. | Activar modo de cola: el ciudadano recibe confirmación inmediata, el procesamiento es asíncrono. | Ariel (Arq) | Pre-release |

---

## 📈 Mapa de Calor de Riesgos

```
         IMPACTO →
         1      2      3      4      5
P  5 |         |      | R-07  | R-02  |       |
R     |         |      |       |       |       |
O  4 |         |      | R-01  | R-06  | R-01  |
B     |         |      |       | R-08  |       |
A  3 |         |      | R-09  | R-10  | R-03  |
B     |         |      |       |       | R-04  |
I  2 |         |      |       |       | R-05  |
L     |         |      |       |       | R-11  |
I  1 |         |      |       |       |       |
D
A
D
```

> **Nota:** El mapa es aproximado. Riesgos en zona ≥ 15 requieren plan de mitigación activo y revisión mensual.

---

## 🔑 Riesgos Críticos — Resumen Ejecutivo

| # | Riesgo | Score | Acción inmediata requerida |
|---|--------|:---:|---|
| 1 | Integración con sistemas legacy sin APIs | 20 🔴 | Inventario y plan de adapters en semana 1 |
| 2 | Falta de vocabulario compartido entre instituciones | 20 🔴 | Taller semántico y Diccionario Canónico sprint 1 |
| 3 | Scope creep por demandas institucionales | 16 🔴 | Establecer Change Request Board antes del kickoff |
| 4 | Rechazo de instituciones por pérdida de autonomía | 16 🔴 | Arquitectura de autonomía + governance board |
| 5 | Brecha de seguridad de datos PII | 15 🔴 | SAST/DAST desde sprint 1, cifrado en todos los entornos |

---

## 📋 Acuerdos del Equipo

| # | Acuerdo | Responsable | Fecha |
|---|---------|-------------|-------|
| 1 | La Matriz de Riesgo será revisada en cada sprint review | Eduardo (PM) | 2026-05-11 |
| 2 | Cualquier nuevo riesgo identificado se documenta en este artefacto dentro de las 24h de identificado | Todos | 2026-05-11 |
| 3 | Los riesgos con score ≥ 15 requieren plan de mitigación aprobado por el equipo antes de comenzar desarrollo | Equipo | 2026-05-11 |
| 4 | Carlos es el CISO del proyecto y tiene veto sobre cualquier diseño que no cumpla los estándares de seguridad | Carlos (Seg) | 2026-05-11 |

---

*Documento generado por sesión colaborativa de las 5 personas — Conecta360 v1.0*
