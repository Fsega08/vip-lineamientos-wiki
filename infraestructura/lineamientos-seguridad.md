# Lineamientos de Seguridad

[Volver al indice](/)

---

## 1. Objetivo

Definir las practicas de seguridad para la capa de interoperabilidad de VIP, cubriendo autenticacion, gestion de secretos, politicas de red y proteccion de datos sensibles.

---

## 2. Autenticacion con Keycloak

### 2.1 Flujo de Autenticacion

```
1. Cliente solicita token → Keycloak (OIDC)
2. Keycloak valida credenciales → retorna JWT (access_token + refresh_token)
3. Cliente envia request a Kong con header: Authorization: Bearer {access_token}
4. Kong valida JWT contra Keycloak (plugin openid-connect)
5. Si valido → Kong reenvia al microservicio con claims del token
6. Microservicio extrae claims (roles, usuario, scopes) del token
```

### 2.2 Estructura del Token

El JWT contiene los claims necesarios para autorizacion:

| Claim | Descripcion | Ejemplo |
|-------|-------------|---------|
| `sub` | Identificador unico del usuario | `f47ac10b-58cc-4372-a567-0e02b2c3d479` |
| `preferred_username` | Nombre de usuario | `usuario.prueba` |
| `realm_access.roles` | Roles del usuario en el realm | `["catastro-consulta", "catastro-mutacion"]` |
| `resource_access` | Roles por cliente/servicio | `{"acm-catastro": {"roles": ["lector"]}}` |
| `exp` | Expiracion del token (epoch) | `1712400000` |

### 2.3 Validacion en el Microservicio

- **Kong valida** la firma y expiracion del token. Si es invalido, retorna error `SEC-001` o `SEC-003`.
- **El microservicio valida** los roles y permisos especificos de la operacion. Si no tiene permisos, retorna `SEC-002` o `SEC-004`.
- El microservicio **nunca** valida la firma del JWT (eso lo hace Kong).

### 2.4 Endpoints Sin Autenticacion

Los siguientes endpoints estan exentos de autenticacion en Kong:

- `/actuator/health/liveness` — Probe de Kubernetes
- `/actuator/health/readiness` — Probe de Kubernetes
- `/swagger-ui.html` — Documentacion OpenAPI
- `/v3/api-docs` — Especificacion OpenAPI

---

## 3. Gestion de Secretos con Vault

### 3.1 Principio

**Vault es la fuente de verdad para todos los secretos.** Los microservicios no deben tener passwords, tokens ni API keys en:

- Variables de entorno en texto plano
- ConfigMaps de Kubernetes
- Archivos `application.yml` comiteados en el repositorio
- Imagenes Docker

### 3.2 Patron de Acceso

```
1. Microservicio se autentica con Vault via ServiceAccount (IRSA en AWS)
2. Vault valida la identidad del pod contra Kubernetes
3. Vault retorna los secretos solicitados
4. Microservicio usa los secretos en memoria (nunca los persiste en disco)
```

### 3.3 Secretos que Deben Estar en Vault

| Secreto | Ejemplo |
|---------|---------|
| Passwords de base de datos | PostgreSQL, Redis |
| API keys de servicios externos | IGAC, SNR |
| Credenciales de Keycloak | Client secret del realm |
| Certificados TLS | Comunicacion con backends |
| Tokens de integracion | Webhooks, notificaciones |

---

## 4. Politicas de Red por Ambiente

### 4.1 Estrategia

| Ambiente | Network Policies | Justificacion |
|----------|-----------------|---------------|
| **Dev** | Deshabilitadas | Facilitar desarrollo y debugging sin restricciones de red |
| **QA** | Habilitadas | Validar que el servicio funciona con restricciones antes de produccion |
| **Prod** | Habilitadas (obligatorias) | Minimo privilegio, solo comunicacion autorizada |

### 4.2 Reglas Base (QA/Prod)

1. **Deny-all por defecto** — Todo trafico ingress/egress bloqueado.
2. **Allow desde Kong** — Solo Kong puede enviar trafico a los microservicios (puerto 8080).
3. **Allow hacia infra** — Microservicios pueden conectarse a PostgreSQL (5432), Redis (6379), RabbitMQ (5672).
4. **Allow intra-namespace** — Pods dentro del mismo namespace pueden comunicarse.
5. **Allow Prometheus scrape** — Prometheus puede scrapear metricas de cualquier pod.
6. **Allow DNS** — Todos los pods pueden resolver DNS via kube-system.

### 4.3 Comunicacion entre Dominios

Si un microservicio necesita comunicarse con otro de un namespace diferente (ej: `acm-hacienda-service-predial` → `acm-catastro-service-consulta`), se debe crear una NetworkPolicy explicita. Ver [matriz de comunicacion](infraestructura/lineamientos-namespaces.md).

---

## 5. Proteccion de Datos Sensibles

### 5.1 Datos que Nunca se Cachean

Ver detalle en [Cache Redis](estandares/cache-redis.md):

- Datos transaccionales activos (mutaciones, liquidaciones en curso)
- Datos financieros en proceso
- Datos sensibles sin cifrar (documentos de identidad, informacion de habeas data)
- Tokens de autenticacion

### 5.2 Datos que Nunca se Loguean

- Tokens JWT (access_token, refresh_token)
- Numeros de documento de identidad
- Passwords o credenciales
- Datos financieros sensibles (montos de liquidacion, saldos)
- Bodies completos de request/response en produccion

### 5.3 Datos que Nunca se Exponen en Errores

Los errores de tipo `SISTEMA` **nunca** deben incluir:

- Stack traces o trazas de excepcion
- Nombres de tablas o columnas de base de datos
- IPs internas del cluster
- Versiones de software o frameworks
- Rutas del filesystem

Ver [Manejo de Errores](contratos/manejo-errores.md) — el adaptador del framework traduce estos errores a mensajes genericos.

---

## 6. TLS y Comunicacion

| Tramo | Protocolo | Responsable |
|-------|-----------|-------------|
| Cliente → Kong | HTTPS (TLS termination) | Kong con certificado ACM/Let's Encrypt |
| Kong → Microservicio | HTTP | Dentro del cluster (red privada) |
| Microservicio → RDS | TLS | Certificado RDS (SSL obligatorio) |
| Microservicio → Redis | TLS | Encryption in-transit habilitado |
| Microservicio → RabbitMQ | AMQPS | TLS habilitado |

---

## 7. Consideraciones Generales

- Todo microservicio que acceda a datos debe autenticarse primero via Keycloak.
- Los secretos tienen rotacion programada: passwords de BD cada 90 dias, tokens de integracion cada 30 dias.
- Los ambientes de dev y QA pueden compartir instancias de infra con **prefijos diferentes**. Produccion es siempre dedicado.
- Las imagenes Docker deben escanearse con herramientas de vulnerabilidades (Trivy, Snyk) antes del despliegue.
- Pre-commit hooks en los repositorios deben incluir deteccion de secretos (trufflehog, gitleaks).
