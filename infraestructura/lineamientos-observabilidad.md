# Lineamientos de Observabilidad

[Volver al indice](/)

---

## 1. Objetivo

Definir las practicas estandar de observabilidad para los microservicios de la plataforma VIP, garantizando visibilidad sobre el estado, rendimiento y comportamiento de la capa de interoperabilidad en todos los ambientes.

---

## 2. Stack de Observabilidad

| Componente | Herramienta | Proposito |
|------------|-------------|-----------|
| Metricas | Prometheus | Recoleccion y almacenamiento de metricas de microservicios, K8s e infraestructura |
| Dashboards | Grafana | Visualizacion de metricas, alertas y estado del cluster |
| Logs | Loki + Promtail | Agregacion centralizada de logs con busqueda por labels |
| Tracing | Zipkin (opcional) | Tracing distribuido para seguir requests entre microservicios |

---

## 3. Metricas que Cada Microservicio Debe Exponer

### 3.1 Endpoint

Todos los microservicios Spring Boot deben exponer metricas en:

```
/actuator/prometheus
```

### 3.2 Configuracion Spring Boot

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus
  metrics:
    tags:
      application: "${spring.application.name}"
      environment: "${plataforma.environment:dev}"
```

### 3.3 Metricas Obligatorias

| Metrica | Tipo | Descripcion |
|---------|------|-------------|
| `http_server_requests_seconds` | Histogram | Latencia de requests HTTP por endpoint, metodo y status |
| `jvm_memory_used_bytes` | Gauge | Uso de memoria JVM por area (heap, non-heap) |
| `jvm_threads_states_threads` | Gauge | Threads activos por estado |
| `db_pool_active_connections` | Gauge | Conexiones activas al pool de base de datos |
| `spring_rabbitmq_listener_*` | Counter | Mensajes procesados/fallidos por listener |
| `cache_gets_total` | Counter | Hits y misses de cache Redis |
| `process_cpu_usage` | Gauge | Uso de CPU del proceso |

### 3.4 Metricas Personalizadas Recomendadas

Cada microservicio deberia agregar metricas de negocio:

```java
@Timed(value = "vip.catastro.consulta.predio", description = "Tiempo de consulta de predio")
public ResponseEntity<?> consultarPredio(RequestDTO request) { ... }
```

---

## 4. Alertas Base

Las siguientes alertas deben estar configuradas para todos los ambientes:

| Alerta | Condicion | Severidad |
|--------|-----------|-----------|
| PodCrashLoopBackOff | Pod reiniciandose mas de 5 veces en 15 min | critical |
| HighCPUUsage | CPU > 80% por mas de 10 min | warning |
| HighMemoryUsage | Memoria > 85% por mas de 10 min | warning |
| DiskPressure | Disco > 90% en algun nodo | critical |
| KongHighLatency | Latencia p99 > 5s por mas de 5 min | warning |
| PostgreSQLDown | Conexion a RDS perdida | critical |
| VaultSealed | Vault en estado sealed | critical |
| RabbitMQQueueBacklog | Mensajes en cola > 1000 por mas de 10 min | warning |
| RedisHighMemory | Memoria Redis > 80% | warning |

---

## 5. Formato de Logs

### 5.1 Estructura JSON Obligatoria

Todos los microservicios deben emitir logs en formato **JSON estructurado** para facilitar la indexacion en Loki.

```json
{
  "timestamp": "2026-04-06T08:45:00.123-05:00",
  "level": "INFO",
  "logger": "com.vortexbird.vip.catastro.service.ConsultaService",
  "message": "Consulta de predio exitosa",
  "service": "vip-catastro-service-consulta",
  "traceId": "abc123def456",
  "spanId": "789ghi",
  "correlationId": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "usuario.prueba",
  "duration_ms": 145
}
```

### 5.2 Campos Obligatorios

| Campo | Descripcion |
|-------|-------------|
| `timestamp` | ISO 8601 con zona horaria Colombia (-05:00) |
| `level` | ERROR, WARN, INFO, DEBUG |
| `logger` | Clase Java que genera el log |
| `message` | Descripcion del evento |
| `service` | Nombre del microservicio (patron de 4 segmentos) |
| `traceId` | ID de tracing distribuido (Zipkin/OpenTelemetry) |
| `correlationId` | `Id_Mensaje` del [Canonical](contratos/canonical.md) |

### 5.3 Campos Prohibidos en Logs

Nunca loguear:

- Tokens de autenticacion (JWT, Bearer)
- Numeros de documento de identidad
- Passwords o secretos
- Datos financieros sensibles
- Bodies completos de request/response en produccion

### 5.4 Configuracion Logback (Spring Boot)

```xml
<appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
  <encoder class="net.logstash.logback.encoder.LogstashEncoder">
    <customFields>{"service":"${spring.application.name}"}</customFields>
  </encoder>
</appender>
```

---

## 6. Retencion por Ambiente

| Componente | Dev | QA | Produccion |
|------------|-----|-----|------------|
| Prometheus (metricas) | 7 dias | 15 dias | 30 dias |
| Loki (logs) | 7 dias | 15 dias | 30 dias |
| Grafana (dashboards) | Persistente | Persistente | Persistente |

---

## 7. Dashboards Esperados

Cada microservicio desplegado debe tener visibilidad minima en Grafana:

| Dashboard | Metricas incluidas |
|-----------|-------------------|
| **VIP Overview** | Requests/s totales, latencia p50/p95/p99, tasa de errores, pods activos |
| **Por Microservicio** | CPU, memoria, threads, pool de conexiones, cache hit rate |
| **RabbitMQ** | Mensajes publicados/consumidos, backlog por cola, consumers activos |
| **Redis** | Hit rate, memoria usada, keys activas, evictions |
| **Kong** | Requests por ruta, latencia por servicio, rate limiting triggers |

---

## 8. Consideraciones Generales

- Todos los microservicios exponen metricas en `/actuator/prometheus` y Prometheus los scrapea automaticamente via annotations de pod.
- El `correlationId` (= `Id_Mensaje` del Canonical) debe propagarse en **toda la cadena**: Kong → microservicio → backend → RabbitMQ → consumer.
- En produccion, el nivel de log debe ser **WARN**. Solo bajar a INFO o DEBUG temporalmente para diagnostico.
- Los dashboards de Grafana usan timezone **America/Bogota**.
- Las alertas deben tener receivers configurados (Slack, Teams, PagerDuty) para que no se pierdan.
