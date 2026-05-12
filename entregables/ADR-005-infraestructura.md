# ADR-005: Infraestructura — Kubernetes Multi-región con IaC (Terraform)

> **Proyecto:** Conecta360  
> **ID:** ADR-005  
> **Estado:** Aprobado ✅  
> **Fecha:** 2026-05-11  
> **Área:** Arquitectura de Infraestructura  

---

## 📐 Contexto de la Decisión

El RNF de disponibilidad (99.9% SLA — máximo 8.7h de downtime/año) y el volumen de 500,000 solicitudes diarias con picos de 10x hacen inviables las alternativas serverless y de VMs tradicionales sin orquestación. La opción multi-cloud queda descartada por regulación de soberanía de datos: los datos ciudadanos deben permanecer dentro del territorio de República Dominicana.

Kubernetes fue seleccionado sobre Serverless por: (1) ausencia de cold starts en 10+ microservicios siempre activos; (2) control total sobre la red (NetworkPolicy, mTLS); (3) portabilidad entre datacenters del gobierno sin vendor lock-in. La distribución RKE2 de Rancher está orientada a entornos regulados de gobierno y cuenta con certificación FIPS 140-2.

---

## 1. Contexto y Problema

El sistema debe garantizar 99.9% de disponibilidad (≤ 8.7h de downtime/año), soportar 500,000 solicitudes diarias con picos de 10x, permitir despliegues independientes de 10+ microservicios y garantizar la soberanía de los datos ciudadanos dentro del territorio de República Dominicana.

## 2. Decisión

**Se adopta Docker + Kubernetes como plataforma de orquestación en configuración multi-región activo-pasivo:**

| Componente | Tecnología | Justificación |
|-----------|-----------|---------------|
| Contenedorización | Docker | Estándar de empaquetado de microservicios |
| Orquestación | Kubernetes 1.29+ | Estándar de la industria, portabilidad entre plataformas |
| Distribución K8s | **Dependiente del proyecto** | On-prem (RKE2/k3s), nube gestionada (EKS/GKE/AKS) o hibrido |
| Proveedor Cloud | **Dependiente del proyecto** | La selección de cloud o datacenter on-prem es decisión del proyecto |
| IaC | **No aplica en esta fase** | No se usará IaC más allá de manifiestos Kubernetes (YAML/Helm) y Dockerfiles |
| CI/CD | GitLab CI + ArgoCD (recomendado) | GitOps: el repositorio es la fuente de verdad |
| Observabilidad | OpenTelemetry + Prometheus + Grafana + Loki + Jaeger | Stack open-source sin vendor lock-in |
| Backup y DR | Velero (K8s backup) + pg_basebackup | Backup automatizado de estados y BD |

> **Nota del Arquitecto:** IaC con Terraform/Terragrunt fue evaluado pero se descarta en esta fase. La infraestructura se gestiona únicamente con manifiestos Kubernetes declarativos (YAML/Helm charts). Si en fases futuras se requiere IaC completo, la herramienta se seleccionará según el proveedor de nube elegido por el proyecto.

## 3. Arquitectura Multi-región

```
┌─────────────────────────────────────────────────────────┐
│                    República Dominicana                          │
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
| **Image scanning** | Trivy o equivalente: bloquear imágenes con CVE críticos |
| **Secrets management** | A definir según proyecto: Kubernetes Secrets cifrados en etcd o servicio cloud nativo |
| **mTLS entre servicios** | Istio Service Mesh o Linkerd (recomendado) |
| **Auditoría K8s** | kube-audit habilitado, logs centralizados |
| **RBAC K8s** | Principio de mínimo privilegio en todos los ServiceAccounts |

## 8. Consecuencias

### Positivas
- Infraestructura portable: manifiestos Kubernetes reproducibles en cualquier distribución
- Rollback automático ante fallos de despliegue (ArgoCD)
- Escalado automático según carga real (HPA + KEDA para Kafka lag)
- Stack de observabilidad completo desde el día 1
- Sin vendor lock-in: la selección de cloud/distribución K8s queda abierta al proyecto

### Negativas / Riesgos asumidos
- Sin IaC formal, el aprovisionamiento inicial de infraestructura es manual o parcialmente automatizado → **Mitigación:** scripts de bootstrapping documentados para cada entorno
- Costo inicial de infraestructura en dos regiones → **Mitigación:** Región secundaria con capacidad reducida (activo-pasivo)
- Complejidad de Istio para mTLS → **Mitigación:** Iniciar con Linkerd (más simple) y migrar si es necesario

---

*Conecta360 v1.0 — ADR-005*
