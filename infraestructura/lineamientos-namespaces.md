# Lineamientos de Nombramiento de Namespaces

[Volver al indice](/)

---

## 1. Objetivo

Definir la convencion estandar de nombramiento de namespaces en Kubernetes (EKS) para la plataforma VIP, garantizando aislamiento logico por dominio, claridad operativa y alineacion con el estandar de nombramiento [SF-NM](estandares/nombramiento-microservicios-colas.md).

---

## 2. Patron

```
{prefijo}-{dominio|funcion}
```

**Reglas:**

- Todo en **minusculas**, formato **kebab-case**.
- El prefijo es el mismo configurable del estandar SF-NM (`acm` para Alcaldia de Medellin).
- Un namespace por **dominio funcional**.
- Los servicios de infraestructura compartida van en un namespace dedicado `{prefijo}-infra`.

---

## 3. Catalogo de Namespaces

### 3.1 Namespaces de Dominio (estandarizados)

Estos namespaces siguen el patron `{prefijo}-{dominio}` y son obligatorios para todos los despliegues VIP. Contienen los microservicios de negocio.

| Namespace | Tipo | Servicios que contiene |
|-----------|------|----------------------|
| `acm-catastro` | Dominio | acm-catastro-service-consulta, acm-catastro-service-mutacion |
| `acm-geo` | Dominio | acm-geo-service-geometrias |
| `acm-urbanismo` | Dominio | acm-urbanismo-service-licencias |
| `acm-hacienda` | Dominio | acm-hacienda-service-predial |
| `acm-archivos` | Plataforma | acm-archivos-service-archivo |
| `acm-notificaciones` | Plataforma | acm-notificaciones-service-notificacion |

### 3.2 Namespaces de Infraestructura (definidos por el despliegue)

Estos namespaces contienen componentes de infraestructura compartida. Sus nombres pueden variar segun la herramienta de despliegue (Terraform, Helm, etc.) y no deben contener microservicios de negocio.

| Namespace | Componente | Notas |
|-----------|------------|-------|
| `kong` | API Gateway (Kong) | Ingress controller, routes, plugins |
| `keycloak` | Gestion de identidad | Autenticacion OIDC, roles, realms |
| `vault` | Gestion de secretos | KMS auto-unseal, almacenamiento seguro |
| `rabbitmq-system` | Mensajeria | RabbitMQ Cluster Operator, exchanges, queues |
| `redis` | Cache | Standalone (dev) o replication con sentinel (qa/prod) |
| `monitoring` | Observabilidad | Prometheus, Grafana, Loki, Promtail |
| `keda` | Autoscaling por eventos | Escalamiento basado en metricas de colas/HTTP |

---

## 4. Relacion Namespace — Microservicio — Service URL

La URL interna de un microservicio en el cluster se compone de:

```
http://{microservicio}.{namespace}.svc.cluster.local:{puerto}
```

| Microservicio | Namespace | URL Interna |
|---------------|-----------|-------------|
| acm-catastro-service-consulta | acm-catastro | `http://acm-catastro-service-consulta.acm-catastro.svc.cluster.local:8080` |
| acm-catastro-service-mutacion | acm-catastro | `http://acm-catastro-service-mutacion.acm-catastro.svc.cluster.local:8080` |
| acm-geo-service-geometrias | acm-geo | `http://acm-geo-service-geometrias.acm-geo.svc.cluster.local:8080` |
| acm-gateway-gateway-proxy | acm-gateway | `http://acm-gateway-gateway-proxy.acm-gateway.svc.cluster.local:80` |

---

## 5. Network Policies

Cada namespace debe tener Network Policies que restrinjan la comunicacion:

### 5.1 Regla por Defecto — Denegar Todo

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: acm-catastro
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

### 5.2 Permitir Trafico desde Kong

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-gateway
  namespace: acm-catastro
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: kong
      ports:
        - port: 8080
          protocol: TCP
```

### 5.3 Permitir Trafico hacia Infraestructura

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-to-infra
  namespace: acm-catastro
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: acm-infra
      ports:
        - port: 5432    # PostgreSQL
          protocol: TCP
        - port: 6379    # Redis
          protocol: TCP
        - port: 5672    # RabbitMQ
          protocol: TCP
```

### 5.4 Matriz de Comunicacion

| Origen → Destino | gateway | catastro | geo | infra | observabilidad |
|-------------------|:-------:|:--------:|:---:|:-----:|:--------------:|
| **gateway** | — | Si | Si | No | No |
| **catastro** | No | Si | Si | Si | Si |
| **geo** | No | No | — | Si | Si |
| **infra** | No | No | No | — | Si |

---

## 6. Resource Quotas por Namespace

Cada namespace debe tener cuotas de recursos para evitar que un dominio consuma todos los recursos del cluster:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: acm-catastro-quota
  namespace: acm-catastro
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"
```

| Namespace | CPU Requests | Memory Requests | Max Pods |
|-----------|-------------|-----------------|----------|
| acm-gateway | 2 | 4Gi | 10 |
| acm-catastro | 4 | 8Gi | 20 |
| acm-geo | 4 | 8Gi | 15 |
| acm-infra | 8 | 16Gi | 20 |
| acm-observabilidad | 4 | 8Gi | 15 |

> Los valores son iniciales y deben ajustarse segun el uso real observado en produccion.

---

## 7. Labels de Namespace

Cada namespace debe tener labels estandar para facilitar Network Policies y operaciones:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: acm-catastro
  labels:
    name: acm-catastro
    platform: vip
    type: domain        # domain | infra | platform | observability
    environment: dev    # dev | qa | prod
```

---

## 8. Consideraciones Generales

- No crear namespaces adicionales sin actualizar este catalogo y el [Inventario de Servicios](inventario/inventario-servicios.md).
- Los ambientes (dev, qa, prod) se separan por **cluster**, no por namespace. El mismo namespace `acm-catastro` existe en los 3 clusters.
- El namespace `default` de Kubernetes **no debe usarse** para cargas de trabajo VIP.
- Los namespaces de infraestructura (`acm-infra`) no deben contener microservicios de negocio.
