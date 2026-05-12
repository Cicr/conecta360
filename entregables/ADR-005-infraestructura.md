# ADR-005: Infraestructura — Kubernetes Multi-región con IaC (Terraform)

> **Proyecto:** Conecta360  
> **ID:** ADR-005  
> **Estado:** Aprobado ✅  
> **Fecha:** 2026-05-11  
> **Decisión tomada por:** Ariel Montero (Arquitecto de Software)  
> **Revisado por:** Carlos Fuentes (Seg), Eduardo Lara (PM), Diana Rivas (Datos)  

---

## 🎭 Contexto de la Decisión

**Eduardo (PM):** El PRD dice 99.9% SLA. Eso son máximo 8.7 horas de downtime al año. ¿Cómo lo garantizamos?

**Ariel (Arq):** Con arquitectura multi-región. Una región principal y una secundaria con failover automático. El tiempo de failover debe ser < 15 minutos para cumplir el RTO.

**Carlos (Seg):** Multi-región también tiene implicaciones de seguridad: los datos deben replicarse cifrados, y necesitamos decidir si ambas regiones están en el mismo país (por regulación de datos soberanos).

**Diana (AD):** Para Costa Verde, recomendaría que ambos centros de datos estén en el país. Los datos ciudadanos no deben salir de la jurisdicción nacional. Eso limita la elección de nube.

**Eduardo (PM):** ¿Kubernetes o serverless? ¿El equipo puede operar Kubernetes?

**Ariel (Arq):** Con los 10 microservicios que tenemos, Kubernetes nos da portabilidad entre nubes y control total. Serverless tendría cold starts y costos impredecibles a 500k solicitudes diarias. Usaremos Kubernetes con operadores para gestionar Kafka y PostgreSQL.

---

## 1. Contexto y Problema

El sistema debe garantizar 99.9% de disponibilidad (≤ 8.7h de downtime/año), soportar 500,000 solicitudes diarias con picos de 10x, permitir despliegues independientes de 10+ microservicios y garantizar la soberanía de los datos ciudadanos dentro del territorio de Costa Verde.

## 2. Decisión

**Se adopta Kubernetes en configuración multi-región activo-pasivo con IaC (Terraform):**

| Componente | Tecnología | Justificación |
|-----------|-----------|---------------|
| Orquestación de contenedores | Kubernetes 1.29+ | Estándar de la industria, portabilidad entre nubes |
| Distribución Kubernetes | RKE2 (Rancher) | Orientado a gobierno y sectores regulados |
| IaC | Terraform + Terragrunt | Infraestructura versionada y reproducible |
| Registry de contenedores | Harbor (self-hosted) | Control total sobre imágenes, scanning de vulnerabilidades |
| CI/CD | GitLab CI + ArgoCD | GitOps: el repositorio es la fuente de verdad |
| Observabilidad | OpenTelemetry + Prometheus + Grafana + Loki + Jaeger | Stack open-source sin vendor lock-in |
| Backup y DR | Velero (K8s backup) + pg_basebackup | Backup automatizado de estados y BD |

## 3. Arquitectura Multi-región

```
┌─────────────────────────────────────────────────────────┐
│                    Costa Verde                          │
│                                                         │
│  ┌─────────────────────┐    ┌─────────────────────────┐ │
│  │   Región NORTE      │    │   Región SUR (DR)        │ │
│  │   (ACTIVA)          │    │   (PASIVA - standby)     │ │
│  │                     │    │                         │ │
│  │  ┌───────────────┐  │    │  ┌───────────────────┐  │ │
│  │  │ K8s Cluster   │  │    │  │  K8s Cluster      │  │ │
│  │  │ (3 masters,   │  │    │  │  (3 masters,      │  │ │
│  │  │  6 workers)   │  │◄──►│  │   4 workers)      │  │ │
│  │  └───────────────┘  │    │  └───────────────────┘  │ │
│  │                     │    │                         │ │
│  │  ┌───────────────┐  │    │  ┌───────────────────┐  │ │
│  │  │ PostgreSQL    │  │    │  │  PostgreSQL        │  │ │
│  │  │ (Patroni)     │◄─┼────┼─►│  Standby (Patroni) │  │ │
│  │  │ Primary       │  │    │  │                   │  │ │
│  │  └───────────────┘  │    │  └───────────────────┘  │ │
│  │                     │    │                         │ │
│  │  ┌───────────────┐  │    │  ┌───────────────────┐  │ │
│  │  │ Kafka Cluster │  │    │  │  Kafka MirrorMaker │  │ │
│  │  │ (3 brokers)   │◄─┼────┼─►│  (replica)        │  │ │
│  │  └───────────────┘  │    │  └───────────────────┘  │ │
│  └─────────────────────┘    └─────────────────────────┘ │
│                                                         │
│         Global Load Balancer (DNS failover)             │
└─────────────────────────────────────────────────────────┘
```

## 4. SLA, RTO y RPO

| Métrica | Target | Mecanismo |
|---------|--------|-----------|
| **Disponibilidad** | 99.9% | Multi-región + failover automático |
| **RTO** (Recovery Time Objective) | < 15 minutos | Failover automático DNS + K8s health checks |
| **RPO** (Recovery Point Objective) | < 5 minutos | PostgreSQL streaming replication sync |
| **Tiempo de despliegue** | < 10 minutos | Rolling update en K8s |
| **Rollback** | < 5 minutos | ArgoCD rollback automático |

## 5. Estrategia de Escalabilidad

```yaml
# HPA (Horizontal Pod Autoscaler) — Case Service
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: case-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: case-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: External
    external:
      metric:
        name: kafka_consumer_group_lag
      target:
        type: Value
        value: "1000"
```

## 6. Alternativas Evaluadas

| Opción | Ventajas | Desventajas | Score |
|--------|---------|-------------|:---:|
| **Kubernetes multi-región** ✅ | Control total, portabilidad, escalabilidad granular | Complejidad operacional | 9/10 |
| **AWS ECS / Fargate** | Serverless containers, sin gestión K8s | Vendor lock-in, costo escala, datos fuera de CR | 5/10 |
| **Serverless (Lambda/Cloud Functions)** | Sin gestión de servidores | Cold starts, costo en picos, no cumple RTO | 4/10 |
| **VMs tradicionales (sin contenedores)** | Simple, conocido | Sin elasticidad, despliegues lentos, desperdicio de recursos | 3/10 |

## 7. Consideraciones de Seguridad en Infraestructura

| Control | Implementación |
|---------|---------------|
| **Network Policies** | K8s NetworkPolicy: solo comunicación autorizada entre pods |
| **Pod Security Standards** | Todos los pods en modo `restricted` |
| **Image scanning** | Harbor + Trivy: bloquear imágenes con CVE críticos |
| **Secrets management** | External Secrets Operator + HashiCorp Vault |
| **mTLS entre servicios** | Istio Service Mesh o Linkerd |
| **Auditoría K8s** | kube-audit habilitado, logs centralizados |
| **RBAC K8s** | Principio de mínimo privilegio en todos los ServiceAccounts |

## 8. Consecuencias

### Positivas
- Infraestructura reproducible al 100% con Terraform
- Rollback automático ante fallos de despliegue (ArgoCD)
- Escalado automático según carga real (HPA + KEDA para Kafka lag)
- Stack de observabilidad completo desde el día 1
- Sin vendor lock-in: migración entre clouds posible

### Negativas / Riesgos asumidos
- Curva de aprendizaje de Kubernetes para el equipo → **Mitigación:** RKE2 simplifica gestión; plan de capacitación 4 semanas
- Costo inicial de infraestructura en dos regiones → **Mitigación:** Región secundaria con capacidad reducida (activo-pasivo vs activo-activo)
- Complejidad de Istio para mTLS → **Mitigación:** Iniciar con Linkerd (más simple) y migrar si es necesario

---

*ADR-005 aprobado por el equipo técnico de Conecta360*
