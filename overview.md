# Vortex Integration Platform (VIP)

[Volver al indice](/)

---

## Que es VIP

VIP (Vortex Integration Platform) es la plataforma de interoperabilidad desarrollada por Vortexbird S.A.S. para la Alcaldia de Medellin. Actua como **capa middleware** entre los sistemas consumidores (portales, aplicaciones moviles, sistemas internos) y los backends del dominio catastral.

```
┌─────────────┐     ┌─────────────────────────────────────────────┐     ┌──────────────┐
│             │     │              VIP (Middleware)                │     │              │
│ Consumidor  │────▶│  Kong ──▶ Microservicio ──▶ Adaptador       │────▶│   Backend    │
│ (Portal,    │◀────│  (GW)  ◀── (Logica)    ◀── (Integracion)   │◀────│   (IGAC,     │
│  App movil) │     │                                             │     │    SNR,      │
│             │     │  Keycloak   Redis   RabbitMQ   PostGIS      │     │    Hacienda) │
└─────────────┘     └─────────────────────────────────────────────┘     └──────────────┘
```

### Que hace VIP

- **Estandariza** la comunicacion: todos los servicios usan el mismo [Canonical](contratos/canonical.md) de request/response.
- **Traduce** errores de los backends a un formato coherente y seguro via el [catalogo de errores](contratos/manejo-errores.md).
- **Valida** estructura, formato y campos requeridos antes de llegar al backend.
- **Cachea** datos parametricos y consultas frecuentes con [Redis](estandares/cache-redis.md).
- **Orquesta** procesos asincronos con [RabbitMQ](estandares/nombramiento-microservicios-colas.md).
- **Protege** los servicios con autenticacion [Keycloak](infraestructura/lineamientos-seguridad.md) y rate limiting via [Kong](infraestructura/lineamientos-kong.md).

### Que NO hace VIP

- No es el sistema catastral. VIP **media** entre consumidores y el sistema catastral.
- No almacena datos maestros. Los datos viven en los backends (IGAC, SNR, Hacienda).
- No reemplaza la logica de negocio del backend. Solo traduce, valida y enruta.

---

## Contexto: LADM-COL 4.1

La plataforma VIP opera en el contexto del modelo **LADM-COL (Land Administration Domain Model — Colombia) version 4.1**, el estandar colombiano de catastro multiproposito.

Esto impacta directamente en:

- **Nomenclatura de campos**: todos los campos del Canonical usan nombres en espanol con formato `Snake_Case` con mayuscula inicial, alineados con las entidades de LADM-COL.
- **Dominios funcionales**: catastro, geoespacial, urbanismo, hacienda — los mismos dominios del modelo LADM-COL.
- **Identificadores**: el NPN (Numero Predial Nacional) de 30 digitos asignado por IGAC es el identificador principal de un predio.

---

## Arquitectura de Alto Nivel

```
                        ┌─────────────────────────────────┐
                        │         CONSUMIDORES             │
                        │  Portal Web, App Movil, Batch    │
                        └──────────────┬──────────────────┘
                                       │ HTTPS
                                       ▼
                        ┌──────────────────────────────┐
                        │     Kong API Gateway          │
                        │  - Autenticacion (Keycloak)   │
                        │  - Rate Limiting              │
                        │  - Correlation ID             │
                        │  - TLS Termination            │
                        └──────────────┬───────────────┘
                                       │ HTTP (interno)
                    ┌──────────────────┼──────────────────┐
                    ▼                  ▼                   ▼
          ┌─────────────────┐ ┌────────────────┐ ┌────────────────┐
          │ vip-catastro-   │ │ vip-catastro-  │ │ vip-geo-       │
          │ service-consulta│ │ service-       │ │ service-       │
          │                 │ │ mutacion       │ │ geometrias     │
          └────────┬────────┘ └───────┬────────┘ └───────┬────────┘
                   │                  │                   │
          ┌────────┴──────────────────┴───────────────────┴────────┐
          │                    INFRAESTRUCTURA                      │
          │                                                        │
          │  PostgreSQL/PostGIS    Redis    RabbitMQ    Keycloak    │
          └────────┬──────────────────────────────────────────┬────┘
                   │                                          │
          ┌────────┴────────┐                        ┌────────┴────────┐
          │ Backends         │                        │ Backends         │
          │ Internos         │                        │ Externos         │
          │ (Catastro BD)    │                        │ (IGAC, SNR)      │
          └─────────────────┘                        └─────────────────┘
```

### Flujo de una solicitud tipica

1. El consumidor se autentica con **Keycloak** y obtiene un JWT.
2. Envia request al endpoint de **Kong** con el JWT en el header `Authorization`.
3. Kong valida el token, inyecta `X-Correlation-Id` y enruta al microservicio.
4. El microservicio valida la estructura del request contra el **Canonical**.
5. Si la validacion falla → retorna error tipo `VALIDACION` (HTTP 400).
6. Si pasa → consulta **Redis** para ver si hay datos cacheados.
7. Si hay cache hit → retorna la respuesta cacheada.
8. Si no → invoca al **backend** (IGAC, SNR, BD catastral) via adaptador.
9. El adaptador traduce la respuesta del backend al formato del Canonical.
10. Si el backend retorna error → el adaptador lo clasifica y traduce via el catalogo de errores.
11. Si es exitoso → cachea en Redis (si aplica) y retorna la respuesta al consumidor.

---

## Componentes Clave

| Componente | Tecnologia | Proposito | Doc |
|------------|------------|-----------|-----|
| API Gateway | Kong | Punto de entrada unico, auth, rate limiting | [Kong](infraestructura/lineamientos-kong.md) |
| Microservicios | Spring Boot 3 / Java 17 | Logica de interoperabilidad | [Naming](estandares/nombramiento-microservicios-colas.md) |
| Identidad | Keycloak | Autenticacion OIDC, roles | [Seguridad](infraestructura/lineamientos-seguridad.md) |
| Cache | Redis | Reducir carga a backends | [Cache](estandares/cache-redis.md) |
| Mensajeria | RabbitMQ | Procesos asincronos, invalidacion de cache | [Colas](estandares/nombramiento-microservicios-colas.md) |
| BD Espacial | PostGIS | Validacion de geometrias, consultas espaciales | — |
| Orquestacion | Kubernetes (EKS) | Despliegue, escalamiento, probes | [K8s](infraestructura/lineamientos-k8s.md) |
| Observabilidad | Prometheus + Grafana + Loki | Metricas, dashboards, logs | [Observabilidad](infraestructura/lineamientos-observabilidad.md) |
| Secretos | Vault | Gestion segura de passwords y tokens | [Seguridad](infraestructura/lineamientos-seguridad.md) |

---

## Entidades Externas

| Entidad | Sigla | Rol |
|---------|-------|-----|
| Instituto Geografico Agustin Codazzi | IGAC | Autoridad catastral nacional, provee datos de predios y NPN |
| Superintendencia de Notariado y Registro | SNR | Registro de propiedad, tradicion y libertad de inmuebles |
| Hacienda Municipal | — | Impuesto predial, liquidacion tributaria |
| Alcaldia de Medellin | — | Cliente, consumidor principal de los servicios VIP |
