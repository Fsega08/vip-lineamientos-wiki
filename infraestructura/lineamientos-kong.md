# Lineamientos de Kong API Gateway

[Volver al indice](/)

---

## 1. Objetivo

Definir las convenciones y practicas estandar para la configuracion de Kong como API Gateway en la plataforma VIP, garantizando un punto de entrada unificado, seguro y observable para todos los microservicios de la capa de interoperabilidad.

---

## 2. Arquitectura General

Kong actua como unico punto de entrada (ingress) a la capa de interoperabilidad. Todos los consumidores externos acceden a los microservicios exclusivamente a traves de Kong.

```
Consumidor → Kong (API Gateway) → Microservicio VIP → Backend
```

---

## 3. Nombramiento de Services y Routes

### 3.1 Services

El nombre del service en Kong debe coincidir con el nombre del microservicio definido en el estandar [SF-NM](estandares/nombramiento-microservicios-colas.md).

**Patron:**

```
{prefijo}-{namespace}-{componente}-{funcionalidad}
```

| Service Kong | Microservicio Destino | Puerto |
|--------------|----------------------|--------|
| `vip-catastro-service-consulta` | vip-catastro-service-consulta | 8080 |
| `vip-catastro-service-mutacion` | vip-catastro-service-mutacion | 8080 |
| `vip-geo-service-geometrias` | vip-geo-service-geometrias | 8080 |
| `vip-urbanismo-service-licencias` | vip-urbanismo-service-licencias | 8080 |
| `vip-hacienda-service-predial` | vip-hacienda-service-predial | 8080 |

**URL del upstream:**

```
http://{service-name}.{namespace}.svc.cluster.local:{puerto}
```

Ejemplo:

```
http://vip-catastro-service-consulta.vip-catastro.svc.cluster.local:8080
```

### 3.2 Routes

**Patron de rutas:**

```
/{prefijo}/{dominio}/{version}/{recurso}
```

| Route | Service | Metodos |
|-------|---------|---------|
| `/vip/catastro/v1/predios` | vip-catastro-service-consulta | GET, POST |
| `/vip/catastro/v1/mutaciones` | vip-catastro-service-mutacion | POST, PUT |
| `/vip/geo/v1/geometrias` | vip-geo-service-geometrias | GET, POST |
| `/vip/urbanismo/v1/licencias` | vip-urbanismo-service-licencias | GET, POST |
| `/vip/hacienda/v1/predial` | vip-hacienda-service-predial | GET, POST |

**Reglas:**

- Todas las rutas inician con `/{prefijo}/{version}/`. Ver [Versionamiento de API](estandares/versionamiento-api.md).
- Los recursos se nombran en **plural** y **minusculas**.
- No incluir verbos en la ruta (el metodo HTTP indica la accion).
- Usar path parameters para identificadores: `/vip/catastro/v1/predios/{npn}`.

---

## 4. Plugins Estandar

### 4.1 Autenticacion — OIDC (Keycloak)

Todos los routes deben tener habilitado el plugin de autenticacion contra Keycloak.

| Parametro | Valor |
|-----------|-------|
| Plugin | `openid-connect` |
| Issuer | URL del realm de Keycloak |
| Scopes | Segun servicio |
| Token Header | `Authorization: Bearer {token}` |

**Excepcion:** rutas de healthcheck (`/health`, `/ready`) no requieren autenticacion.

### 4.2 Rate Limiting

| Parametro | Valor por Defecto |
|-----------|-------------------|
| Plugin | `rate-limiting` |
| Requests por minuto | 60 (configurable por route) |
| Politica | `local` en dev, `redis` en produccion |
| Header de respuesta | `X-RateLimit-Remaining` |

### 4.3 Request/Response Transformation

| Parametro | Valor |
|-----------|-------|
| Plugin | `request-transformer` |
| Headers a inyectar | `X-Request-Id` (UUID), `X-Correlation-Id` |

El header `X-Correlation-Id` debe propagarse en toda la cadena de microservicios para trazabilidad de extremo a extremo. Corresponde al campo `Id_Mensaje` del [Canonical](contratos/canonical.md).

### 4.4 Logging

| Parametro | Valor |
|-----------|-------|
| Plugin | `http-log` o `tcp-log` |
| Destino | Servicio de logging centralizado |
| Campos | Request ID, Consumer, Route, Latencia, Status |

### 4.5 CORS

| Parametro | Valor |
|-----------|-------|
| Plugin | `cors` |
| Origins | Configurar por ambiente |
| Methods | GET, POST, PUT, DELETE, OPTIONS |
| Headers | Authorization, Content-Type, X-Request-Id |
| Max Age | 3600 |

---

## 5. Configuracion por Ambiente

| Aspecto | Desarrollo | QA | Produccion |
|---------|------------|-----|------------|
| Rate Limiting | Deshabilitado | 120 req/min | 60 req/min |
| OIDC | Keycloak dev | Keycloak QA | Keycloak prod |
| Rate Limiting Policy | `local` | `local` | `redis` |
| Logging Level | Debug | Info | Warn |
| CORS Origins | `*` | Dominio QA | Dominio produccion |

---

## 6. Manejo de Errores en Kong

Cuando Kong no puede enrutar una solicitud, debe retornar la estructura de error estandar del [SF-IN](contratos/manejo-errores.md):

### Gateway no disponible (503)

```json
{
  "Id_Mensaje": null,
  "Fecha_Hora_Solicitud": "2026-04-06T09:00:00-05:00",
  "Http_Estado": 503,
  "Errores": [
    {
      "Tipo_Error": "SISTEMA",
      "Codigo_Error": "SYS-002",
      "Ruta": null,
      "Mensaje": "El servicio no esta disponible temporalmente.",
      "Origen": "vip-gateway-gateway-proxy"
    }
  ],
  "Datos_Operacion": {}
}
```

### Rate limit excedido (429)

```json
{
  "Id_Mensaje": null,
  "Fecha_Hora_Solicitud": "2026-04-06T09:00:00-05:00",
  "Http_Estado": 429,
  "Errores": [
    {
      "Tipo_Error": "SISTEMA",
      "Codigo_Error": "BIZ-005",
      "Ruta": null,
      "Mensaje": "Se han realizado demasiadas solicitudes. Intente mas tarde.",
      "Origen": "vip-gateway-gateway-proxy"
    }
  ],
  "Datos_Operacion": {}
}
```

---

## 7. Consideraciones Generales

- Kong debe desplegarse en el namespace `vip-gateway` (ver [Namespaces](infraestructura/lineamientos-namespaces.md)).
- Los services de Kong apuntan a los ClusterIP services de Kubernetes, nunca a pods directamente.
- La configuracion de Kong debe gestionarse como **codigo** (declarativa) usando `deck` o CRDs de Kong Ingress Controller.
- Los certificados TLS se terminan en Kong (TLS termination). La comunicacion interna entre Kong y microservicios es HTTP dentro del cluster.
- Documentar toda ruta nueva en el [Inventario de Servicios](inventario/inventario-servicios.md).
