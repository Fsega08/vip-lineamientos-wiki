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
- El prefijo es el mismo configurable del estandar SF-NM (`vip` para Alcaldia de Medellin).
- Un namespace por **dominio funcional**.
- Los servicios de infraestructura compartida van en un namespace dedicado `{prefijo}-infra`.

---

## 3. Catalogo de Namespaces

| Namespace | Tipo | Servicios que contiene |
|-----------|------|----------------------|
| `vip-gateway` | Plataforma | vip-gateway-gateway-proxy (Kong) |
| `vip-catastro` | Dominio | vip-catastro-service-consulta, vip-catastro-service-mutacion |
| `vip-geo` | Dominio | vip-geo-service-geometrias |
| `vip-urbanismo` | Dominio | vip-urbanismo-service-licencias |
| `vip-hacienda` | Dominio | vip-hacienda-service-predial |
| `vip-infra` | Infraestructura | Redis, RabbitMQ, PostgreSQL/PostGIS, Keycloak |
| `vip-observabilidad` | Plataforma | Prometheus, Grafana, Loki, Jaeger |
| `vip-archivos` | Plataforma | vip-archivos-service-archivo |
| `vip-notificaciones` | Plataforma | vip-notificaciones-service-notificacion |
| `vip-workflow` | Plataforma | Camunda, vip-workflow-engine-camunda |

---

## 4. Relacion Namespace — Microservicio — Service URL

La URL interna de un microservicio en el cluster se compone de:

```
http://{microservicio}.{namespace}.svc.cluster.local:{puerto}
```

| Microservicio | Namespace | URL Interna |
|---------------|-----------|-------------|
| vip-catastro-service-consulta | vip-catastro | `http://vip-catastro-service-consulta.vip-catastro.svc.cluster.local:8080` |
| vip-catastro-service-mutacion | vip-catastro | `http://vip-catastro-service-mutacion.vip-catastro.svc.cluster.local:8080` |
| vip-geo-service-geometrias | vip-geo | `http://vip-geo-service-geometrias.vip-geo.svc.cluster.local:8080` |
| vip-gateway-gateway-proxy | vip-gateway | `http://vip-gateway-gateway-proxy.vip-gateway.svc.cluster.local:80` |

---

## 5. Network Policies

Cada namespace debe tener Network Policies que restrinjan la comunicacion:

### 5.1 Regla por Defecto — Denegar Todo

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: vip-catastro
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
  namespace: vip-catastro
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: vip-gateway
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
  namespace: vip-catastro
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: vip-infra
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
  name: vip-catastro-quota
  namespace: vip-catastro
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
| vip-gateway | 2 | 4Gi | 10 |
| vip-catastro | 4 | 8Gi | 20 |
| vip-geo | 4 | 8Gi | 15 |
| vip-infra | 8 | 16Gi | 20 |
| vip-observabilidad | 4 | 8Gi | 15 |

> Los valores son iniciales y deben ajustarse segun el uso real observado en produccion.

---

## 7. Labels de Namespace

Cada namespace debe tener labels estandar para facilitar Network Policies y operaciones:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: vip-catastro
  labels:
    name: vip-catastro
    platform: vip
    type: domain        # domain | infra | platform | observability
    environment: dev    # dev | qa | prod
```

---

## 8. Consideraciones Generales

- No crear namespaces adicionales sin actualizar este catalogo y el [Inventario de Servicios](inventario/inventario-servicios.md).
- Los ambientes (dev, qa, prod) se separan por **cluster**, no por namespace. El mismo namespace `vip-catastro` existe en los 3 clusters.
- El namespace `default` de Kubernetes **no debe usarse** para cargas de trabajo VIP.
- Los namespaces de infraestructura (`vip-infra`) no deben contener microservicios de negocio.
