# Wiki de Lineamientos VIP

**Vortex Integration Platform (VIP)** — Alcaldia de Medellin  
Plataforma de interoperabilidad para el dominio catastral LADM-COL 4.1.

---

## Perfil Funcional

Si documentas servicios, defines contratos de API o necesitas entender la estructura de los mensajes:

| Documento | Para que lo necesitas |
|-----------|----------------------|
| [Canonical](contratos/canonical.md) | Estructura de Request y Response que usan todos los servicios |
| [Manejo de Errores](contratos/manejo-errores.md) | Tipos de error, codigos y mensajes que retorna la plataforma |
| [Guia de Documentacion](guias/guia-documentacion.md) | Como documentar un servicio nuevo o un control de cambios |
| [Inventario de Servicios](inventario/inventario-servicios.md) | Que servicios existen, sus endpoints y dependencias |

## Perfil Tecnico

Si desarrollas, configuras infraestructura o despliegas microservicios:

| Documento | Para que lo necesitas |
|-----------|----------------------|
| [Nombramiento de Microservicios y Colas](estandares/nombramiento-microservicios-colas.md) | Patron de 4 segmentos, colas RabbitMQ, eventos |
| [Cache Redis](estandares/cache-redis.md) | Keys, TTL, estrategias de invalidacion |
| [Kong API Gateway](infraestructura/lineamientos-kong.md) | Routes, plugins, autenticacion, rate limiting |
| [Manifiestos K8s](infraestructura/lineamientos-k8s.md) | Deployments, probes, Swagger, HPA |
| [Namespaces](infraestructura/lineamientos-namespaces.md) | Convencion de namespaces, Network Policies |
| [Observabilidad](infraestructura/lineamientos-observabilidad.md) | Metricas, logs, alertas, dashboards Grafana |
| [Seguridad](infraestructura/lineamientos-seguridad.md) | Keycloak, Vault, TLS, proteccion de datos |

## Referencia Rapida

| Concepto | Patron |
|----------|--------|
| Microservicio | `{prefijo}-{namespace}-{componente}-{funcionalidad}` |
| Redis Key | `{prefijo}:{dominio}:{entidad}:{identificador}` |
| RabbitMQ Queue | `{prefijo}.{dominio}.{accion}.queue` |
| Routing Key | `{prefijo}.{dominio}.{accion}.{evento}` |
| Namespace K8s | `{prefijo}-{dominio}` |
| Route Kong | `/{prefijo}/{dominio}/{recurso}` |

---

**Cliente:** Alcaldia de Medellin  
**Proveedor:** Vortexbird S.A.S.  
**Version:** 1.0  
**Fecha:** Abril 2026
