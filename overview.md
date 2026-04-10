# Vortex Integration Platform (VIP)

[Volver al indice](/)

---

## Que es VIP

VIP (Vortex Integration Platform) es la plataforma de interoperabilidad desarrollada por Vortexbird S.A.S. para la Alcaldia de Medellin. Actua como **capa middleware** entre los sistemas consumidores (portales, aplicaciones moviles, sistemas internos) y los backends del dominio catastral.

```mermaid
graph LR
    C[Consumidor<br>Portal, App] -->|HTTPS| K[Kong<br>API Gateway]
    K -->|HTTP| MS[Microservicio<br>VIP]
    MS -->|HTTP| AD[Adaptador]
    AD -->|Protocolo<br>backend| B[Backend<br>IGAC, SNR]
    MS --- R[(Redis<br>Cache)]
    MS --- RMQ[[RabbitMQ<br>Mensajeria]]
    K --- KC[Keycloak<br>Auth]

    style K fill:#1565C0,color:#fff
    style MS fill:#2E7D32,color:#fff
    style AD fill:#F57F17,color:#fff
    style R fill:#C62828,color:#fff
    style RMQ fill:#FF6F00,color:#fff
    style KC fill:#6A1B9A,color:#fff
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

```mermaid
graph TB
    subgraph Consumidores
        PW[Portal Web]
        AM[App Movil]
        BT[Batch]
    end

    subgraph VIP Gateway
        KONG[Kong API Gateway<br>Auth · Rate Limit · TLS · Correlation ID]
    end

    subgraph Microservicios VIP
        CC[vip-catastro<br>service-consulta]
        CM[vip-catastro<br>service-mutacion]
        GS[vip-geo<br>service-geometrias]
    end

    subgraph Infraestructura
        PG[(PostgreSQL<br>PostGIS)]
        RD[(Redis)]
        RQ[[RabbitMQ]]
        KC[Keycloak]
        VT[Vault]
    end

    subgraph Backends Externos
        IGAC[IGAC]
        SNR[SNR]
        HAC[Hacienda<br>Municipal]
    end

    PW & AM & BT -->|HTTPS| KONG
    KONG -->|HTTP| CC & CM & GS
    KONG -.->|OIDC| KC

    CC & CM & GS --> PG
    CC & CM & GS --> RD
    CM --> RQ
    RQ --> CC

    CC --> IGAC & SNR
    CM --> IGAC
    GS --> PG

    style KONG fill:#1565C0,color:#fff
    style CC fill:#2E7D32,color:#fff
    style CM fill:#2E7D32,color:#fff
    style GS fill:#2E7D32,color:#fff
    style RD fill:#C62828,color:#fff
    style RQ fill:#FF6F00,color:#fff
    style KC fill:#6A1B9A,color:#fff
```

---

## Flujo de una Solicitud Tipica

```mermaid
sequenceDiagram
    participant C as Consumidor
    participant K as Kong
    participant MS as Microservicio
    participant RD as Redis
    participant BE as Backend

    C->>K: POST /vip/catastro/v1/predios<br>Authorization: Bearer {JWT}
    K->>K: Validar JWT (Keycloak)
    K->>K: Inyectar X-Correlation-Id

    alt Token invalido
        K-->>C: 401 SEC-001 / SEC-003
    end

    K->>MS: Forward request + headers
    MS->>MS: Validar Canonical (estructura, campos)

    alt Validacion falla
        MS-->>C: 400 VALIDACION (VLD-001..009)
    end

    MS->>RD: GET vip:catastro:predio:{npn}

    alt Cache HIT
        RD-->>MS: Datos cacheados
        MS-->>C: 200 + Datos_Operacion
    end

    MS->>BE: Consultar backend (IGAC/SNR/BD)

    alt Backend error
        BE-->>MS: Error o timeout
        MS-->>C: 502 INTEGRACION (INT-001..004)
    end

    BE-->>MS: Respuesta exitosa
    MS->>RD: SET vip:catastro:predio:{npn} TTL 1h
    MS->>MS: Mapear a Canonical
    MS-->>C: 200 + Datos_Operacion
```

---

## Flujo de Errores

```mermaid
flowchart TD
    REQ[Request entrante] --> VAL{Validacion<br>Canonical}
    VAL -->|Falla| E400[400 VALIDACION<br>VLD-001..009]
    VAL -->|OK| AUTH{Autorizacion<br>roles}
    AUTH -->|Falla| E403[403 SEGURIDAD<br>SEC-002, SEC-004]
    AUTH -->|OK| BACK{Invocar<br>Backend}
    BACK -->|Timeout| E502[502 INTEGRACION<br>INT-001..004]
    BACK -->|Error negocio| E409[409 NEGOCIO<br>BIZ/CTR/GEO]
    BACK -->|Error interno| E500[500 SISTEMA<br>SYS-001..005]
    BACK -->|OK| OK[200 Exitoso<br>Datos_Operacion]

    style E400 fill:#F44336,color:#fff
    style E403 fill:#FF9800,color:#fff
    style E502 fill:#9C27B0,color:#fff
    style E409 fill:#FF5722,color:#fff
    style E500 fill:#795548,color:#fff
    style OK fill:#4CAF50,color:#fff
```

---

## Flujo de Invalidacion de Cache

```mermaid
sequenceDiagram
    participant CM as vip-catastro<br>service-mutacion
    participant RQ as RabbitMQ
    participant CC as vip-catastro<br>service-consulta
    participant RD as Redis

    CM->>CM: Mutacion exitosa en backend
    CM->>RQ: Publish vip.catastro.predio.actualizado
    RQ->>CC: Consume evento
    CC->>RD: DEL vip:catastro:predio:{npn}
    Note over CC,RD: Siguiente consulta<br>traera dato fresco
```

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
