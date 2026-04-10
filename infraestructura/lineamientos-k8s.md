# Lineamientos de Manifiestos Kubernetes

[Volver al indice](/)

---

## 1. Objetivo

Definir las convenciones y practicas estandar para los manifiestos de Kubernetes (EKS) de todos los microservicios de la plataforma VIP, incluyendo Deployments, Services, ConfigMaps, probes de salud y exposicion de documentacion Swagger.

---

## 2. Estructura de Manifiestos por Microservicio

Cada microservicio debe tener la siguiente estructura de manifiestos en su repositorio:

```
k8s/
├── deployment.yaml
├── service.yaml
├── configmap.yaml
├── hpa.yaml
└── ingress.yaml          # Solo si aplica (normalmente Kong maneja el ingress)
```

---

## 3. Deployment

### 3.1 Nombramiento

El nombre del Deployment debe coincidir con el nombre del microservicio del estandar [SF-NM](estandares/nombramiento-microservicios-colas.md).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: acm-catastro-service-consulta
  namespace: acm-catastro
  labels:
    app: acm-catastro-service-consulta
    domain: catastro
    team: interoperabilidad
    version: "1.0.0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: acm-catastro-service-consulta
  template:
    metadata:
      labels:
        app: acm-catastro-service-consulta
        domain: catastro
    spec:
      containers:
        - name: acm-catastro-service-consulta
          image: registry.vortexbird.com/vip/acm-catastro-service-consulta:1.0.0
          ports:
            - containerPort: 8080
              name: http
          envFrom:
            - configMapRef:
                name: acm-catastro-service-consulta-config
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 12
```

### 3.2 Labels Obligatorios

| Label | Descripcion | Ejemplo |
|-------|-------------|---------|
| `app` | Nombre del microservicio | `acm-catastro-service-consulta` |
| `domain` | Dominio funcional | `catastro`, `geo`, `gateway` |
| `team` | Equipo responsable | `interoperabilidad` |
| `version` | Version del despliegue | `1.0.0` |

---

## 4. Probes de Salud

### 4.1 Liveness Probe

Determina si el contenedor esta vivo. Si falla, Kubernetes reinicia el pod.

| Parametro | Valor |
|-----------|-------|
| Endpoint | `/actuator/health/liveness` |
| initialDelaySeconds | 30 |
| periodSeconds | 15 |
| timeoutSeconds | 5 |
| failureThreshold | 3 |

**Que valida:**
- La JVM responde.
- El proceso principal esta activo.
- No hay deadlocks.

### 4.2 Readiness Probe

Determina si el pod esta listo para recibir trafico. Si falla, Kubernetes lo remueve del Service (no recibe requests).

| Parametro | Valor |
|-----------|-------|
| Endpoint | `/actuator/health/readiness` |
| initialDelaySeconds | 15 |
| periodSeconds | 10 |
| timeoutSeconds | 5 |
| failureThreshold | 3 |

**Que valida:**
- Conexion a base de datos activa.
- Conexion a Redis activa.
- Conexion a RabbitMQ activa.
- Dependencias criticas disponibles.

### 4.3 Startup Probe

Protege contenedores con tiempos de arranque lentos. Mientras no pase el startup probe, los demas probes no se ejecutan.

| Parametro | Valor |
|-----------|-------|
| Endpoint | `/actuator/health/liveness` |
| initialDelaySeconds | 10 |
| periodSeconds | 5 |
| failureThreshold | 12 |

> Esto da hasta **70 segundos** para que la aplicacion arranque (10 + 12*5).

### 4.4 Configuracion Spring Boot

En `application.yml`:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  endpoint:
    health:
      probes:
        enabled: true
      show-details: when-authorized
      group:
        liveness:
          include: livenessState
        readiness:
          include: readinessState, db, redis, rabbit
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

---

## 5. Swagger / OpenAPI

### 5.1 Exposicion

Cada microservicio debe exponer su documentacion OpenAPI 3.0 en los siguientes endpoints:

| Endpoint | Descripcion |
|----------|-------------|
| `/swagger-ui.html` | Interfaz visual de Swagger UI |
| `/v3/api-docs` | Especificacion OpenAPI en JSON |
| `/v3/api-docs.yaml` | Especificacion OpenAPI en YAML |

### 5.2 Configuracion Spring Boot

En `application.yml`:

```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    operations-sorter: method
    tags-sorter: alpha
  info:
    title: "${spring.application.name}"
    version: "@project.version@"
    description: "API de interoperabilidad VIP"
```

### 5.3 Anotaciones Estandar

Cada endpoint debe documentarse con las anotaciones de OpenAPI:

```java
@Operation(
    summary = "Consultar predio por NPN",
    description = "Consulta la informacion catastral de un predio por su Numero Predial Nacional"
)
@ApiResponses({
    @ApiResponse(responseCode = "200", description = "Consulta exitosa"),
    @ApiResponse(responseCode = "400", description = "Error de validacion"),
    @ApiResponse(responseCode = "409", description = "Error de negocio"),
    @ApiResponse(responseCode = "502", description = "Error de integracion")
})
```

### 5.4 Acceso via Kong

Kong debe enrutar las peticiones de Swagger sin autenticacion:

| Route | Service | Auth |
|-------|---------|------|
| `/acm/catastro/v1/consulta/swagger-ui.html` | acm-catastro-service-consulta | No |
| `/acm/catastro/v1/consulta/v3/api-docs` | acm-catastro-service-consulta | No |

---

## 6. Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: acm-catastro-service-consulta
  namespace: acm-catastro
  labels:
    app: acm-catastro-service-consulta
spec:
  type: ClusterIP
  selector:
    app: acm-catastro-service-consulta
  ports:
    - name: http
      port: 8080
      targetPort: 8080
      protocol: TCP
```

**Reglas:**
- Tipo siempre **ClusterIP** (el acceso externo se maneja via Kong).
- El nombre del Service debe coincidir con el nombre del Deployment.
- Puerto estandar: **8080** para HTTP.

---

## 7. ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: acm-catastro-service-consulta-config
  namespace: acm-catastro
data:
  SPRING_PROFILES_ACTIVE: "dev"
  PLATAFORMA_PREFIJO: "acm"
  PLATAFORMA_CACHE_TTL_PARAMETRICOS: "86400"
  PLATAFORMA_CACHE_TTL_CONSULTA: "3600"
  RABBITMQ_HOST: "rabbitmq.acm-infra.svc.cluster.local"
  REDIS_HOST: "redis.acm-infra.svc.cluster.local"
```

**Reglas:**
- Nombre: `{nombre-microservicio}-config`.
- Variables sensibles (passwords, tokens) van en **Secrets**, no en ConfigMaps.
- El prefijo `PLATAFORMA_PREFIJO` se inyecta desde el ConfigMap para mantenerlo configurable.

---

## 8. HPA (Horizontal Pod Autoscaler)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: acm-catastro-service-consulta
  namespace: acm-catastro
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: acm-catastro-service-consulta
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

| Parametro | Valor por Defecto |
|-----------|-------------------|
| minReplicas | 2 |
| maxReplicas | 6 (ajustable por servicio) |
| CPU target | 70% |
| Memory target | 80% |

---

## 9. Resources por Perfil

| Perfil | CPU Request | CPU Limit | Memory Request | Memory Limit |
|--------|-------------|-----------|----------------|--------------|
| Ligero (consulta) | 250m | 500m | 512Mi | 1Gi |
| Medio (mutacion, orquestacion) | 500m | 1000m | 1Gi | 2Gi |
| Pesado (geo, procesamiento) | 1000m | 2000m | 2Gi | 4Gi |

---

## 10. Consideraciones Generales

- Todos los manifiestos deben versionarse en el repositorio del microservicio bajo `k8s/`.
- Usar **image tags especificos** (nunca `latest`) para garantizar reproducibilidad.
- El registry de imagenes sigue el patron: `registry.vortexbird.com/vip/{nombre-microservicio}:{version}`.
- Cada microservicio debe exponer metricas Prometheus en `/actuator/prometheus` para observabilidad.
- Los Secrets se gestionan con AWS Secrets Manager o Sealed Secrets, nunca en texto plano en el repositorio.
