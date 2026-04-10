# Wiki de Lineamientos VIP

**Vortex Integration Platform (VIP)** — Alcaldia de Medellin  
Plataforma de interoperabilidad para el dominio catastral LADM-COL 4.1.

**Nuevo aqui?** Empieza por el [Overview de VIP](overview.md) | [Glosario](glosario.md) | [FAQ](faq.md)

---

## Perfil Funcional

Si documentas servicios, defines contratos de API o necesitas entender la estructura de los mensajes:

| Documento | Para que lo necesitas |
|-----------|----------------------|
| [Canonical](contratos/canonical.md) | Estructura de Request y Response que usan todos los servicios |
| [Manejo de Errores](contratos/manejo-errores.md) | Tipos de error, codigos y mensajes que retorna la plataforma |
| [Paginacion y Filtrado](estandares/paginacion-filtrado.md) | Como paginar y filtrar resultados en consultas |
| [Guia de Documentacion](guias/guia-documentacion.md) | Como documentar un servicio nuevo o un control de cambios |
| [Template de Servicio](guias/template-servicio.md) | Template listo para copiar y llenar |
| [Inventario de Servicios](inventario/inventario-servicios.md) | Que servicios existen, sus endpoints y dependencias |

## Perfil Tecnico

Si desarrollas, configuras infraestructura o despliegas microservicios:

| Documento | Para que lo necesitas |
|-----------|----------------------|
| [Nombramiento de Microservicios y Colas](estandares/nombramiento-microservicios-colas.md) | Patron de 4 segmentos, colas RabbitMQ, eventos |
| [Comunicacion Sync vs Async](estandares/comunicacion-sync-async.md) | Cuando usar REST y cuando RabbitMQ |
| [Versionamiento de API](estandares/versionamiento-api.md) | Estrategia de versionamiento, semver, tags Docker |
| [Cache Redis](estandares/cache-redis.md) | Keys, TTL, estrategias de invalidacion |
| [CI/CD](estandares/lineamientos-cicd.md) | Pipeline, testing, scan de seguridad, deploy |
| [Kong API Gateway](infraestructura/lineamientos-kong.md) | Routes, plugins, autenticacion, rate limiting |
| [Manifiestos K8s](infraestructura/lineamientos-k8s.md) | Deployments, probes, Swagger, HPA |
| [Namespaces](infraestructura/lineamientos-namespaces.md) | Convencion de namespaces, Network Policies |
| [Observabilidad](infraestructura/lineamientos-observabilidad.md) | Metricas, logs, alertas, dashboards Grafana |
| [Seguridad](infraestructura/lineamientos-seguridad.md) | Keycloak, Vault, TLS, proteccion de datos |

## Referencia Rapida

| Concepto | Patron |
|----------|--------|
| Microservicio | `{prefijo}-{namespace}-{componente}-{funcionalidad}` |
| API Route | `/{prefijo}/{dominio}/{version}/{recurso}` |
| Redis Key | `{prefijo}:{dominio}:{entidad}:{identificador}` |
| RabbitMQ Queue | `{prefijo}.{dominio}.{accion}.queue` |
| Routing Key | `{prefijo}.{dominio}.{accion}.{evento}` |
| Namespace K8s | `{prefijo}-{dominio}` |
| Docker Image | `registry/vip/{microservicio}:{semver}` |

---

**Cliente:** Alcaldia de Medellin  
**Proveedor:** Vortexbird S.A.S.  
**Version:** 1.0  
**Fecha:** Abril 2026
