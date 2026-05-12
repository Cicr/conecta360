# ADR-004: Autenticación y Autorización — OAuth2/OIDC con RBAC + ABAC

> **Proyecto:** Conecta360  
> **ID:** ADR-004  
> **Estado:** Aprobado ✅  
> **Fecha:** 2026-05-11  
> **Decisión tomada por:** Carlos Fuentes (Experto en Seguridad)  
> **Revisado por:** Ariel Montero (Arq), Diana Rivas (Datos), Eduardo Lara (PM), Beatriz Salcedo (BA)  

---

## 🎭 Contexto de la Decisión

**Carlos (Seg):** Este es el ADR más crítico para mí. El sistema tiene tres tipos de identidades radicalmente diferentes: ciudadanos que se autentican con su documento de identidad nacional, funcionarios que usan credenciales institucionales y el sistema mismo que necesita M2M (machine-to-machine). No podemos mezclarlos.

**Ariel (Arq):** El PRD dice explícitamente OAuth2 / OpenID Connect. ¿Y para autorización?

**Carlos (Seg):** RBAC solo no es suficiente. Un agente de Salud en el Departamento Norte no debería poder ver casos del Departamento Sur. Eso requiere atributos — ABAC. La combinación RBAC + ABAC es la industria estándar para sistemas de gobierno.

**Beatriz (BA):** ¿Cómo impacta esto al ciudadano? ¿Tiene que crear una cuenta adicional?

**Carlos (Seg):** No. El ciudadano se autentica exclusivamente con el SSO Nacional (su cédula). Conecta360 solo valida el token emitido por ese SSO. Nunca almacenamos contraseñas de ciudadanos.

**Diana (AD):** Desde datos, ¿cómo propagamos el tenant (institución) en cada request para el RLS?

**Carlos (Seg):** El Auth Service emite un JWT interno que incluye los claims de tenant_id, institution_id y roles. Ese JWT es validado por cada microservicio.

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

---

*ADR-004 aprobado por el equipo técnico de Conecta360*
