# ⚠️ Matriz de Riesgo — Conecta360

> **Proyecto:** Sistema Integral de Atención Ciudadana "Conecta360"  
> **País:** Costa Verde  
> **Versión:** 1.0  
> **Fecha:** 2026-05-11  

---

## 📐 Metodología de Evaluación

Cada desafío del PRD §6 fue analizado desde las perspectivas de arquitectura de software, análisis de sistemas, seguridad, modelado de datos y gestión de proyectos, para convertirlo en un riesgo cuantificable con probabilidad, impacto, estrategia de mitigación y plan de contingencia.

Se utiliza la escala estándar: Probabilidad (1=Raro → 5=Casi seguro) × Impacto (1=Insignificante → 5=Catastrófico). Score = P × I.

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
| R-01 | **Desafío 1** — Integración heterogénea | Las dependencias institucionales con sistemas legacy sin APIs bloquean la integración, generando retrasos de hasta 6 meses por institución | 4 | 5 | 20 | 🔴 Crítico | (1) Inventario de APIs existentes semana 1. (2) Definir patrón Adapter/ACL por sistema legacy. (3) Priorizar instituciones con mayor volumen de casos. (4) Proveer SDK de integración estándar. | Implementar scraping estructurado como puente temporal mientras se desarrolla el adapter. Prever presupuesto de contingencia del 20% por cada integración legada. | Arquitectura | Mensual |
| R-02 | **Desafío 2** — Interoperabilidad semántica | Diferentes instituciones usan vocabularios distintos para las mismas entidades (caso, expediente, solicitud, ticket), causando inconsistencias en el modelo de datos | 5 | 4 | 20 | 🔴 Crítico | (1) Crear Diccionario de Datos Canónico en sprint 1. (2) Taller de alineación semántica con representantes de cada institución. (3) Modelo de dominio compartido versionado. | Usar ontología neutra como árbitro (ej. schema.org adaptado). Mantener mapeos de traducción por institución. | Datos | Quincenal |
| R-03 | **Desafío 3** — Alta disponibilidad geográfica | Un desastre natural o fallo eléctrico en el centro de datos principal deja el sistema completamente inoperativo | 3 | 5 | 15 | 🔴 Crítico | (1) Arquitectura multi-región activo-pasivo mínimo. (2) RTO < 4h, RPO < 1h definidos en contrato. (3) Runbook de failover documentado y practicado trimestralmente. | Activación de región secundaria en < 15 minutos mediante proceso automatizado (IaC). Modo degradado sin módulo analítico. | Arquitectura | Trimestral |
| R-04 | **Desafío 4** — Gestión de identidades complejas | Un funcionario de la institución A accede a casos de la institución B por error de configuración de roles (cross-tenant leakage) | 3 | 5 | 15 | 🔴 Crítico | (1) RBAC + ABAC desde sprint 1. (2) Tests automatizados de aislamiento de tenant en pipeline CI/CD. (3) Revisión de permisos por auditoría interna trimestral. | Revocar sesiones activas de forma masiva. Notificar a DPA (Autoridad de Protección de Datos). Plan de comunicación a ciudadanos afectados. | Seguridad | Quincenal |
| R-05 | **Desafío 5** — Datos sensibles | Brecha de seguridad que expone datos personales de ciudadanos (PII) por vulnerabilidad en la API pública | 3 | 5 | 15 | 🔴 Crítico | (1) SAST y DAST en pipeline CI/CD. (2) Pen testing semestral por tercero certificado. (3) Cifrado AES-256 en reposo y TLS 1.3 en tránsito. (4) WAF en perímetro. | Plan de respuesta a incidentes documentado (< 72h de notificación per GDPR-equiv). Forensics team en standby. | Seguridad | Mensual |
| R-06 | **Desafío 6** — Balance centralización vs. autonomía | Las instituciones rechazan el sistema por sentir que pierden control de sus datos y procesos propios | 4 | 4 | 16 | 🔴 Crítico | (1) Modelo de datos con autonomía institucional garantizada (aislamiento lógico). (2) Governance board con representantes institucionales. (3) Dashboards propios por institución. | Modo "federado" donde cada institución controla su instancia con sincronización central limitada. | Negocio / PM | Mensual |
| R-07 | **Desafío 7** — Cambio cultural | Los funcionarios rechazan adoptar el nuevo sistema y continúan usando canales informales (papel, correo personal, WhatsApp), dejando casos sin registrar | 5 | 3 | 15 | 🔴 Crítico | (1) Plan de Gestión del Cambio formal con sponsor ejecutivo. (2) Capacitación por roles (no masiva). (3) Quick wins visibles en 30 días. (4) KPIs de adopción monitoreados. | Campaña de incentivos por adopción. Escalado a nivel gerencial si resistencia > 30%. | Negocio / PM | Semanal (primeros 3 meses) |
| R-08 | Riesgo adicional identificado | Scope creep: cada institución solicita funcionalidades personalizadas que inflan el alcance y retrasan las entregas core | 4 | 4 | 16 | 🔴 Crítico | (1) MoSCoW por requerimiento. (2) Change Request Board formal. (3) Backlog versionado y priorizado públicamente. | Congelar alcance de release actual. Funcionalidades adicionales van a roadmap v2.0. | PM | Mensual |
| R-09 | Riesgo adicional identificado | Dependencia de terceros (WhatsApp Business API, redes sociales) que cambian sus políticas o precios unilateralmente | 3 | 3 | 9 | 🟡 Medio | (1) Patrón Adapter por canal externo. (2) Cláusula contractual de SLA con proveedores. (3) Canal alternativo siempre disponible (email como fallback). | Activar canal alternativo en < 24h. Notificar a ciudadanos del cambio temporal. | Arquitectura | Trimestral |
| R-10 | Riesgo adicional identificado | El chatbot con IA genera respuestas incorrectas o sesgadas que desinforman al ciudadano | 3 | 4 | 12 | 🟠 Alto | (1) Human-in-the-loop para casos de baja confianza (< 80% score). (2) Evaluación semanal de accuracy del modelo. (3) Opción siempre disponible de escalar a agente humano. | Desactivar respuestas automáticas del chatbot. Modo solo-FAQ de bajo riesgo. | Arquitectura / Negocio | Semanal |
| R-11 | Riesgo adicional identificado | Falta de financiamiento o cambio de gobierno que paralice el proyecto | 2 | 5 | 10 | 🟠 Alto | (1) Arquitectura modular que permite entregas parciales con valor independiente. (2) Documentación suficiente para handover a otro equipo. | Priorizar módulos de mayor impacto visible (RF-01, RF-02). Asegurar continuidad contractual. | PM | Trimestral |
| R-12 | Riesgo adicional identificado | El volumen de 500,000 solicitudes diarias es subestimado y el sistema colapsa en picos (elecciones, catástrofes) | 3 | 5 | 15 | 🔴 Crítico | (1) Load testing con 3x el volumen esperado antes de go-live. (2) Autoscaling configurado. (3) Circuit breakers y degraded mode. | Activar modo de cola: el ciudadano recibe confirmación inmediata, el procesamiento es asíncrono. | Arquitectura | Pre-release |

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

## 📋 Criterios de Gestión de Riesgos

| # | Criterio | Fecha |
|---|---------|-------|
| 1 | La Matriz de Riesgo debe revisarse en cada sprint review | 2026-05-11 |
| 2 | Cualquier nuevo riesgo identificado se documenta en este artefacto dentro de las 24h de identificado | 2026-05-11 |
| 3 | Los riesgos con score ≥ 15 requieren plan de mitigación aprobado antes de comenzar desarrollo | 2026-05-11 |
| 4 | El área de Seguridad tiene veto sobre cualquier diseño que no cumpla los estándares de seguridad del proyecto | 2026-05-11 |

---

*Conecta360 v1.0 — Documento de análisis de riesgos*
