# ADR-004: Autenticación y Autorización — OAuth2/OIDC con RBAC + ABAC

> **Proyecto:** Conecta360  
> **ID:** ADR-004  
> **Estado:** Aprobado ✅  
> **Fecha:** 2026-05-11  
> **Área:** Seguridad / IAM  

---

## 📐 Contexto de la Decisión

El sistema gestiona tres tipos de identidades radicalmente diferentes que no pueden compartir el mismo mecanismo de autenticación: (1) **ciudadanos**, que se identifican con el SSO Nacional (documento de identidad nacional vía OIDC) y no deben tener contraseñas locales en Conecta360; (2) **funcionarios**, que usan credenciales institucionales o LDAP; (3) **servicios** (M2M), que requieren Client Credentials flow.

Para autorización, RBAC solo no es suficiente: un agente del Departamento Norte no debe ver casos del Departamento Sur aunque tengan el mismo rol. Se requieren atributos contextuales (ABAC). El modelo de triple defensa —API Gateway, microservicio, RLS en BD— garantiza que ningún error en una capa comprometa el aislamiento total. El JWT interno emitido por el Auth Service propaga el tenant_id e institution_id para activar el RLS de PostgreSQL.

---

## 1. Contexto y Problema

El sistema maneja tres tipos de identidades con modelos de autenticación distintos:
1. **Ciudadanos:** Se autentican con el SSO Nacional (OpenID Connect con documento de identidad)
2. **Funcionarios/Operadores:** Se autentican con credenciales institucionales o directorio LDAP
3. **Servicios (M2M):** Comunicación entre microservicios

Adicionalmente, la autorización debe contemplar:
- Roles institucionales (quién puede hacer qué)
- Atributos contextuales (a qué datos puede acceder: por institución, región, turno)

## 2. Decisión

**Se adopta un modelo híbrido de autenticación y autorización:**

### Autenticación
| Actor | Protocolo | IdP |
|-------|-----------|-----|
| Ciudadano | OpenID Connect (PKCE) | SSO Nacional (externo) |
| Funcionario | OAuth2 + MFA | Auth Service interno + LDAP institucional |
| M2M | OAuth2 Client Credentials | Auth Service interno |

### Autorización
- **RBAC (Role-Based Access Control):** Define qué operaciones puede hacer cada rol
- **ABAC (Attribute-Based Access Control):** Filtra datos por atributos contextuales (institution_id, region_id)
- **Enforcement:** API Gateway (grueso) + microservicio (fino) + RLS en DB (capa base)

## 3. Definición de Roles (RBAC)

| Rol | Descripción | Permisos principales |
|-----|-------------|---------------------|
| `admin` | Administrador global del sistema | CRUD completo en todas las entidades |
| `institution_admin` | Admin de una institución | CRUD en su institución y usuarios |
| `supervisor` | Supervisor institucional | Ver todos los casos de su institución, reasignar, reportes |
| `agent` | Agente/funcionario | Ver y gestionar casos asignados a él |
| `operator` | Operador de call center | Crear casos, consultar estado, sin acceso a datos internos |
| `readonly` | Consultor / auditor | Solo lectura en su institución |
| `citizen` | Ciudadano | Solo sus propios casos y datos |

## 4. Atributos ABAC

| Atributo | Tipo | Uso |
|----------|------|-----|
| `institution_id` | UUID | Filtra datos por institución |
| `region_id` | UUID | Filtra por región geográfica (futuro) |
| `allowed_categories` | Array[String] | Categorías de caso que puede ver/gestionar |
| `shift` | ENUM | Horario laboral activo (para asignación automática) |

## 5. Estructura del JWT Interno

```json
{
  "sub": "user-uuid-123",
  "iss": "conecta360-auth-service",
  "aud": ["conecta360-api"],
  "exp": 1747000000,
  "iat": 1746996400,
  "jti": "unique-jwt-id",
  "actor_type": "user",
  "institution_id": "inst-uuid-456",
  "role": "supervisor",
  "allowed_categories": ["VIA", "ENERGIA"],
  "mfa_verified": true,
  "session_id": "session-uuid-789"
}
```

## 6. Alternativas Evaluadas

| Opción | Ventajas | Desventajas | Score |
|--------|---------|-------------|:---:|
| **OAuth2 + OIDC + RBAC + ABAC** ✅ | Estándar, flexible, gradual | Mayor complejidad de implementación | 9/10 |
| **SAML 2.0** | Estándar en gobierno, maduro | XML verboso, peor para SPA/apps modernas | 6/10 |
| **Keycloak (completo)** | Listo para usar, RBAC/ABAC integrado | Dependencia externa pesada, curva de aprendizaje | 7/10 |
| **Solo API Keys** | Simple | No viable para ciudadanos, sin granularidad | 2/10 |
| **mTLS solamente** | Máxima seguridad M2M | No escalable para usuarios humanos | 4/10 |

## 7. Flujo de Autenticación — Ciudadano

```
Ciudadano                Web Portal           Auth Service         SSO Nacional
    |                        |                     |                     |
    |── Login (cédula) ──►   |                     |                     |
    |                        |── OIDC Auth Request ►|                     |
    |                        |                     |── Redirect ──────►  |
    |◄── Redirect ────────── |                     |                    |
    |── Credenciales ──────────────────────────────────────────────────► |
    |                        |                     |◄── ID Token (JWT) – |
    |                        |                     |                     |
    |                        |◄── JWT Interno ──── |                     |
    |◄── JWT + Cookie ─────  |                     |                     |
```

## 8. Consecuencias

### Positivas
- Defensa en profundidad: API Gateway + servicio + BD
- Ciudadanos no crean contraseñas en Conecta360 (reducción de superficie de ataque)
- JWT stateless reduce latencia de validación
- ABAC permite control granular sin explosión de roles

### Negativas / Riesgos asumidos
- Complejidad de ABAC en implementación → **Mitigación:** Comenzar con RBAC, añadir ABAC incrementalmente
- JWT revocation en logout → **Mitigación:** Blacklist en Redis con TTL = tiempo expiración JWT
- Dependencia del SSO Nacional → **Mitigación:** Modo degradado con login local para funcionarios si el SSO falla

## 9. Requerimientos de Seguridad Obligatorios

| Control | Implementación |
|---------|---------------|
| MFA obligatorio | Para roles `admin` y `supervisor` |
| PKCE | Obligatorio para flujos de browser (ciudadano y funcionario) |
| Token rotation | Refresh tokens de un solo uso |
| JWT expiración | Access token: 1h, Refresh token: 24h |
| Secure + HttpOnly + SameSite | Atributos de cookie para JWT |
| Rate limiting auth | Máx. 5 intentos fallidos → bloqueo temporal 15min |
| Gestión de secretos y claves | A definir según recursos del proyecto — opciones: Kubernetes Secrets (cifrado en etcd), servicios cloud nativos u otros |

> **Nota del Arquitecto:** HashiCorp Vault fue evaluado como opción de gestión de secretos. La selección final de herramienta se delega al proyecto según sus restricciones operacionales y presupuestarias.

---

*Conecta360 v1.0 — ADR-004*
