# 📖 Diccionario de Datos Canónico — Conecta360

> **Proyecto:** Sistema Integral de Atención Ciudadana "Conecta360"  
> **País:** Costa Verde  
> **Versión:** 1.0  
> **Fecha:** 2026-05-11  
> **Entregable:** E-01  

---

## 📐 Propósito

Este documento establece el vocabulario canónico del sistema Conecta360. Su objetivo es garantizar que todas las áreas del proyecto (arquitectura, datos, seguridad, integraciones) utilicen los mismos nombres y definiciones para las entidades del dominio, eliminando la ambigüedad semántica entre instituciones.

**Entidades cubiertas** (según PRD §7.2): ciudadanos, casos, dependencias institucionales, flujos de atención, auditorías, notificaciones, adjuntos, políticas de SLA y categorías de servicio.

**Nomenclatura:** entidades en PascalCase, campos en snake_case inglés, con alias en español para claridad del negocio.

---

## 🏗️ Entidades del Sistema

### 1. `Citizen` — Ciudadano

> Persona natural que interactúa con el sistema para reportar casos o solicitar servicios.

| Campo | Tipo | Restricciones | PII | Alias (ES) | Descripción |
|-------|------|--------------|:---:|-----------|-------------|
| `citizen_id` | UUID v7 | PK, NOT NULL | No | ID Ciudadano | Identificador único interno del sistema |
| `national_id` | VARCHAR(20) | UNIQUE, NOT NULL | ✅ Sí | Cédula / DPI | Número de documento de identidad nacional |
| `national_id_type` | ENUM | ('cedula','pasaporte','extranjeria') | No | Tipo documento | Tipo de documento de identidad |
| `first_name` | VARCHAR(100) | NOT NULL | ✅ Sí | Nombre(s) | Nombre(s) del ciudadano |
| `last_name` | VARCHAR(100) | NOT NULL | ✅ Sí | Apellido(s) | Apellido(s) del ciudadano |
| `email` | VARCHAR(255) | UNIQUE, nullable | ✅ Sí | Correo electrónico | Dirección de email para notificaciones |
| `phone_number` | VARCHAR(20) | nullable | ✅ Sí | Teléfono | Número de teléfono para SMS/WhatsApp |
| `preferred_channel` | ENUM | ('email','sms','push','whatsapp') | No | Canal preferido | Canal preferido para notificaciones |
| `consent_notifications` | BOOLEAN | DEFAULT FALSE | No | Consentimiento notif. | Opt-in para recibir notificaciones |
| `consent_date` | TIMESTAMPTZ | nullable | No | Fecha consentimiento | Fecha en que se otorgó el consentimiento |
| `is_active` | BOOLEAN | DEFAULT TRUE | No | Activo | Si el ciudadano tiene cuenta activa |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT NOW() | No | Fecha creación | Timestamp de creación del registro |
| `updated_at` | TIMESTAMPTZ | NOT NULL | No | Última actualización | Timestamp de última modificación |
| `deleted_at` | TIMESTAMPTZ | nullable (soft delete) | No | Fecha eliminación | Soft delete para trazabilidad |

**Políticas de seguridad:**
- `national_id`, `email`, `phone_number` → cifrados con AES-256 en columna
- Retención: 5 años desde la última actividad
- Acceso: Solo el módulo de autenticación y el propio ciudadano autenticado

---

### 2. `User` — Usuario del Sistema (Funcionario / Operador)

> Persona que opera el sistema como funcionario institucional, operador de call center o administrador.

| Campo | Tipo | Restricciones | PII | Alias (ES) | Descripción |
|-------|------|--------------|:---:|-----------|-------------|
| `user_id` | UUID v7 | PK, NOT NULL | No | ID Usuario | Identificador único del usuario interno |
| `username` | VARCHAR(100) | UNIQUE, NOT NULL | No | Nombre de usuario | Login username |
| `email` | VARCHAR(255) | UNIQUE, NOT NULL | ✅ Sí | Correo institucional | Email institucional del funcionario |
| `first_name` | VARCHAR(100) | NOT NULL | ✅ Sí | Nombre(s) | Nombre(s) |
| `last_name` | VARCHAR(100) | NOT NULL | ✅ Sí | Apellido(s) | Apellido(s) |
| `institution_id` | UUID | FK → Institution | No | ID Institución | Institución a la que pertenece |
| `role` | ENUM | ('admin','supervisor','agent','operator','readonly') | No | Rol | Rol del usuario en el sistema |
| `is_active` | BOOLEAN | DEFAULT TRUE | No | Activo | Estado del usuario |
| `last_login_at` | TIMESTAMPTZ | nullable | No | Último acceso | Última sesión iniciada |
| `password_hash` | VARCHAR(255) | NOT NULL | ✅ Sí | Hash contraseña | Hash bcrypt de la contraseña (si aplica) |
| `mfa_enabled` | BOOLEAN | DEFAULT FALSE | No | MFA habilitado | Autenticación multifactor activada |
| `created_at` | TIMESTAMPTZ | NOT NULL | No | Fecha creación | |
| `updated_at` | TIMESTAMPTZ | NOT NULL | No | Última actualización | |

**Políticas de seguridad:**
- `password_hash` → bcrypt con factor 12 mínimo
- MFA obligatorio para roles `admin` y `supervisor`
- Sesiones con JWT, expiración 1h, refresh 24h

---

### 3. `Institution` — Dependencia Institucional

> Entidad gubernamental que gestiona un área de servicios (salud, transporte, energía, etc.)

| Campo | Tipo | Restricciones | PII | Alias (ES) | Descripción |
|-------|------|--------------|:---:|-----------|-------------|
| `institution_id` | UUID v7 | PK, NOT NULL | No | ID Institución | Identificador único |
| `code` | VARCHAR(20) | UNIQUE, NOT NULL | No | Código | Código corto único (ej. SALUD, ENERGIA) |
| `name` | VARCHAR(200) | NOT NULL | No | Nombre | Nombre completo de la institución |
| `description` | TEXT | nullable | No | Descripción | Descripción del área de competencia |
| `parent_institution_id` | UUID | FK → Institution, nullable | No | Institución padre | Para jerarquías (ministerio > dirección) |
| `contact_email` | VARCHAR(255) | nullable | No | Email contacto | Email de contacto institucional |
| `contact_phone` | VARCHAR(20) | nullable | No | Teléfono contacto | |
| `sla_policy_id` | UUID | FK → SlaPolicy | No | Política SLA | SLA por defecto de la institución |
| `is_active` | BOOLEAN | DEFAULT TRUE | No | Activo | |
| `created_at` | TIMESTAMPTZ | NOT NULL | No | Fecha creación | |
| `updated_at` | TIMESTAMPTZ | NOT NULL | No | Última actualización | |

---

### 4. `ServiceCategory` — Categoría de Servicio

> Taxonomía canónica de tipos de solicitudes/incidencias del sistema.

| Campo | Tipo | Restricciones | PII | Alias (ES) | Descripción |
|-------|------|--------------|:---:|-----------|-------------|
| `category_id` | UUID v7 | PK, NOT NULL | No | ID Categoría | Identificador único |
| `code` | VARCHAR(50) | UNIQUE, NOT NULL | No | Código | Código canónico (ej. VIA_BACHE, SEG_ROBO) |
| `name` | VARCHAR(200) | NOT NULL | No | Nombre | Nombre de la categoría |
| `description` | TEXT | nullable | No | Descripción | Descripción detallada |
| `parent_category_id` | UUID | FK → ServiceCategory, nullable | No | Categoría padre | Para categorías jerárquicas |
| `default_institution_id` | UUID | FK → Institution | No | Institución por defecto | Institución que gestiona esta categoría |
| `priority_level` | ENUM | ('low','medium','high','critical') | No | Nivel de prioridad | Prioridad base de la categoría |
| `is_active` | BOOLEAN | DEFAULT TRUE | No | Activo | |
| `created_at` | TIMESTAMPTZ | NOT NULL | No | Fecha creación | |

---

### 5. `Case` — Caso / Solicitud / Incidencia

> Entidad central del sistema. Representa cualquier interacción ciudadana que requiere gestión institucional.

| Campo | Tipo | Restricciones | PII | Alias (ES) | Descripción |
|-------|------|--------------|:---:|-----------|-------------|
| `case_id` | UUID v7 | PK, NOT NULL | No | ID Caso | Identificador único interno |
| `case_number` | VARCHAR(20) | UNIQUE, NOT NULL | No | Número de caso | Número legible único (ej. C360-2026-000001) |
| `citizen_id` | UUID | FK → Citizen, NOT NULL | No | ID Ciudadano | Ciudadano que reportó el caso |
| `category_id` | UUID | FK → ServiceCategory, NOT NULL | No | ID Categoría | Tipo/categoría del caso |
| `institution_id` | UUID | FK → Institution, NOT NULL | No | ID Institución | Institución actualmente responsable |
| `assigned_user_id` | UUID | FK → User, nullable | No | ID Agente asignado | Funcionario asignado al caso |
| `status` | ENUM | ('received','assigned','in_progress','pending_info','resolved','closed','cancelled') | No | Estado | Estado actual del caso |
| `priority` | ENUM | ('low','medium','high','critical') | No | Prioridad | Prioridad calculada del caso |
| `title` | VARCHAR(300) | NOT NULL | No | Título | Título/asunto del caso |
| `description` | TEXT | NOT NULL | ⚠️ Puede contener PII | Descripción | Descripción detallada por el ciudadano |
| `location_address` | VARCHAR(500) | nullable | ⚠️ PII indirecta | Dirección | Dirección del incidente |
| `location_lat` | DECIMAL(10,8) | nullable | No | Latitud | Coordenada geográfica |
| `location_lng` | DECIMAL(11,8) | nullable | No | Longitud | Coordenada geográfica |
| `channel` | ENUM | ('web','mobile','phone','social','in_person') | No | Canal de entrada | Canal por el que se reportó el caso |
| `source_reference` | VARCHAR(200) | nullable | No | Referencia origen | ID externo (ej. ID tweet, ID WhatsApp) |
| `sla_deadline` | TIMESTAMPTZ | NOT NULL | No | Fecha límite SLA | Fecha límite de resolución según SLA |
| `resolved_at` | TIMESTAMPTZ | nullable | No | Fecha resolución | Cuando se marcó como resuelto |
| `closed_at` | TIMESTAMPTZ | nullable | No | Fecha cierre | Cuando se cerró definitivamente |
| `satisfaction_score` | SMALLINT | CHECK(1-5), nullable | No | Puntuación satisfacción | Calificación del ciudadano (1-5) |
| `created_at` | TIMESTAMPTZ | NOT NULL | No | Fecha creación | |
| `updated_at` | TIMESTAMPTZ | NOT NULL | No | Última actualización | |

**Políticas de seguridad:**
- `description` y `location_address` → acceso restringido; solo el agente asignado y su supervisor
- `case_number` es el único dato que se comunica al ciudadano como referencia externa

---

### 6. `CaseEvent` — Flujo de Atención / Historial del Caso

> Registro inmutable de todos los cambios de estado y acciones sobre un caso.

| Campo | Tipo | Restricciones | PII | Alias (ES) | Descripción |
|-------|------|--------------|:---:|-----------|-------------|
| `event_id` | UUID v7 | PK, NOT NULL | No | ID Evento | Identificador único del evento |
| `case_id` | UUID | FK → Case, NOT NULL | No | ID Caso | Caso al que pertenece |
| `event_type` | ENUM | ('created','status_changed','assigned','reassigned','commented','resolved','closed','escalated') | No | Tipo evento | Tipo de acción realizada |
| `from_status` | VARCHAR(50) | nullable | No | Estado anterior | Estado previo del caso |
| `to_status` | VARCHAR(50) | nullable | No | Estado nuevo | Estado nuevo del caso |
| `from_user_id` | UUID | FK → User, nullable | No | Usuario origen | Quién realizó la acción (null si es sistema) |
| `comment` | TEXT | nullable | No | Comentario | Nota del funcionario o agente |
| `is_public` | BOOLEAN | DEFAULT FALSE | No | Visible al ciudadano | Si el comentario es visible al ciudadano |
| `metadata` | JSONB | nullable | No | Metadatos | Información adicional del evento |
| `created_at` | TIMESTAMPTZ | NOT NULL | No | Fecha evento | Timestamp inmutable del evento |

**Nota de arquitectura:** Esta tabla es append-only. Nunca se actualiza ni elimina un registro.

---

### 7. `Notification` — Notificación

> Registro de notificaciones enviadas al ciudadano por correo, SMS o push.

| Campo | Tipo | Restricciones | PII | Alias (ES) | Descripción |
|-------|------|--------------|:---:|-----------|-------------|
| `notification_id` | UUID v7 | PK, NOT NULL | No | ID Notificación | Identificador único |
| `citizen_id` | UUID | FK → Citizen, NOT NULL | No | ID Ciudadano | Destinatario de la notificación |
| `case_id` | UUID | FK → Case, nullable | No | ID Caso | Caso relacionado (si aplica) |
| `channel` | ENUM | ('email','sms','push','whatsapp') | No | Canal | Canal de envío |
| `status` | ENUM | ('pending','sent','delivered','failed','bounced') | No | Estado | Estado del envío |
| `subject` | VARCHAR(300) | nullable | No | Asunto | Asunto del mensaje |
| `body` | TEXT | NOT NULL | ⚠️ Puede contener PII | Cuerpo | Contenido del mensaje |
| `external_id` | VARCHAR(200) | nullable | No | ID externo | ID del proveedor (Twilio, SendGrid, etc.) |
| `sent_at` | TIMESTAMPTZ | nullable | No | Fecha envío | Cuando fue enviado al proveedor |
| `delivered_at` | TIMESTAMPTZ | nullable | No | Fecha entrega | Confirmación de entrega |
| `retry_count` | SMALLINT | DEFAULT 0 | No | Intentos | Número de reintentos |
| `created_at` | TIMESTAMPTZ | NOT NULL | No | Fecha creación | |

---

### 8. `Attachment` — Adjunto

> Archivo asociado a un caso (foto, documento, video).

| Campo | Tipo | Restricciones | PII | Alias (ES) | Descripción |
|-------|------|--------------|:---:|-----------|-------------|
| `attachment_id` | UUID v7 | PK, NOT NULL | No | ID Adjunto | Identificador único |
| `case_id` | UUID | FK → Case, NOT NULL | No | ID Caso | Caso al que pertenece |
| `uploaded_by_citizen_id` | UUID | FK → Citizen, nullable | No | ID Ciudadano | Ciudadano que subió el archivo |
| `uploaded_by_user_id` | UUID | FK → User, nullable | No | ID Usuario | Funcionario que subió el archivo |
| `file_name` | VARCHAR(255) | NOT NULL | No | Nombre archivo | Nombre original del archivo |
| `file_type` | VARCHAR(100) | NOT NULL | No | Tipo MIME | Tipo MIME del archivo |
| `file_size_bytes` | BIGINT | NOT NULL | No | Tamaño | Tamaño en bytes |
| `storage_path` | VARCHAR(1000) | NOT NULL | No | Ruta almacenamiento | Ruta en object storage (S3/GCS) |
| `is_virus_scanned` | BOOLEAN | DEFAULT FALSE | No | Escaneado | Si pasó por antivirus |
| `scan_result` | VARCHAR(50) | nullable | No | Resultado scan | 'clean', 'infected', 'pending' |
| `created_at` | TIMESTAMPTZ | NOT NULL | No | Fecha subida | |

---

### 9. `SlaPolicy` — Política de SLA

> Configuración de tiempos de respuesta por tipo de solicitud.

| Campo | Tipo | Restricciones | PII | Alias (ES) | Descripción |
|-------|------|--------------|:---:|-----------|-------------|
| `sla_policy_id` | UUID v7 | PK, NOT NULL | No | ID Política | Identificador único |
| `name` | VARCHAR(200) | NOT NULL | No | Nombre | Nombre de la política |
| `category_id` | UUID | FK → ServiceCategory, nullable | No | ID Categoría | Categoría a la que aplica (null = default) |
| `institution_id` | UUID | FK → Institution, nullable | No | ID Institución | Institución (null = global) |
| `priority` | ENUM | ('low','medium','high','critical') | No | Prioridad | Nivel de prioridad |
| `response_time_hours` | INTEGER | NOT NULL | No | Horas primera respuesta | Tiempo máximo para primera respuesta |
| `resolution_time_hours` | INTEGER | NOT NULL | No | Horas resolución | Tiempo máximo para resolución |
| `escalation_time_hours` | INTEGER | NOT NULL | No | Horas escalación | Tiempo antes de escalar automáticamente |
| `is_active` | BOOLEAN | DEFAULT TRUE | No | Activo | |
| `created_at` | TIMESTAMPTZ | NOT NULL | No | Fecha creación | |
| `updated_at` | TIMESTAMPTZ | NOT NULL | No | Última actualización | |

---

### 10. `AuditLog` — Registro de Auditoría

> Registro inmutable de todas las operaciones sensibles del sistema para trazabilidad completa.

| Campo | Tipo | Restricciones | PII | Alias (ES) | Descripción |
|-------|------|--------------|:---:|-----------|-------------|
| `audit_id` | UUID v7 | PK, NOT NULL | No | ID Auditoría | Identificador único |
| `entity_type` | VARCHAR(100) | NOT NULL | No | Tipo entidad | Nombre de la entidad modificada |
| `entity_id` | UUID | NOT NULL | No | ID Entidad | ID del registro afectado |
| `action` | ENUM | ('create','read','update','delete','login','logout','export') | No | Acción | Tipo de operación |
| `actor_type` | ENUM | ('citizen','user','system') | No | Tipo actor | Quién realizó la acción |
| `actor_id` | UUID | NOT NULL | No | ID Actor | ID del actor |
| `ip_address` | INET | NOT NULL | No | IP | Dirección IP de origen |
| `user_agent` | TEXT | nullable | No | User Agent | Agente del cliente |
| `institution_id` | UUID | FK → Institution, nullable | No | ID Institución | Institución del actor |
| `before_state` | JSONB | nullable | No | Estado anterior | Snapshot del registro antes del cambio |
| `after_state` | JSONB | nullable | No | Estado nuevo | Snapshot del registro después del cambio |
| `metadata` | JSONB | nullable | No | Metadatos | Información adicional contextual |
| `created_at` | TIMESTAMPTZ | NOT NULL | No | Fecha | Timestamp inmutable del evento |

**Nota de seguridad:** Esta tabla es append-only, no actualizable ni eliminable. Acceso restringido solo a rol `admin` y auditor externo.

---

## 📊 Resumen de Entidades

| # | Entidad | Registros esperados (año 1) | PII | Append-only |
|---|---------|:---:|:---:|:---:|
| 1 | Citizen | ~2,000,000 | ✅ | No |
| 2 | User | ~5,000 | ✅ | No |
| 3 | Institution | ~50 | No | No |
| 4 | ServiceCategory | ~200 | No | No |
| 5 | Case | ~10,000,000 | Indirecto | No |
| 6 | CaseEvent | ~50,000,000 | No | ✅ |
| 7 | Notification | ~30,000,000 | Indirecto | No |
| 8 | Attachment | ~5,000,000 | Potencial | No |
| 9 | SlaPolicy | ~500 | No | No |
| 10 | AuditLog | ~100,000,000 | No | ✅ |

---

## 🔑 Mapeo de Sinónimos por Institución

| Término Canónico | Salud | Obras Públicas | Energía | Seguridad | Municipal |
|-----------------|-------|---------------|---------|-----------|-----------|
| `Case` | Expediente | Reporte | Avería | Incidente | Solicitud |
| `ServiceCategory` | Especialidad | Tipo de obra | Tipo de falla | Tipo de delito | Trámite |
| `assigned_user` | Médico/Enf. | Inspector | Técnico | Agente | Funcionario |
| `resolved` | Alta | Subsanado | Restablecido | Cerrado | Aprobado |

---

*Diccionario de Datos generado por Diana Rivas — Arquitecta de Datos — Conecta360 v1.0*
