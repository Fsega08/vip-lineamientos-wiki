# Inventario de Servicios y Dependencias

[Volver al indice](index.md)

---

## 1. Objetivo

Centralizar el inventario de todos los microservicios de la plataforma VIP con sus dependencias, endpoints, tecnologias y estado actual. Este documento es la fuente de verdad para entender la topologia de la capa de interoperabilidad.

---

## 2. Inventario de Microservicios

### 2.1 Dominio: Catastro

| Microservicio | Namespace | Componente | Responsabilidad | Estado |
|---------------|-----------|------------|-----------------|--------|
| vip-catastro-service-consulta | vip-catastro | service | Consultas de predios, NPN, avaluos, intervinientes | Activo |
| vip-catastro-service-mutacion | vip-catastro | service | Operaciones de mutacion catastral (transferencias, actualizaciones) | Activo |

### 2.2 Dominio: Geoespacial

| Microservicio | Namespace | Componente | Responsabilidad | Estado |
|---------------|-----------|------------|-----------------|--------|
| vip-geo-service-geometrias | vip-geo | service | Operaciones PostGIS, validacion de geometrias, transformacion de coordenadas | Activo |

### 2.3 Dominio: Urbanismo

| Microservicio | Namespace | Componente | Responsabilidad | Estado |
|---------------|-----------|------------|-----------------|--------|
| vip-urbanismo-service-licencias | vip-urbanismo | service | Gestion de licencias urbanisticas, uso de suelo | Planificado |

### 2.4 Dominio: Hacienda

| Microservicio | Namespace | Componente | Responsabilidad | Estado |
|---------------|-----------|------------|-----------------|--------|
| vip-hacienda-service-predial | vip-hacienda | service | Impuesto predial, liquidacion tributaria | Planificado |

### 2.5 Plataforma

| Microservicio | Namespace | Componente | Responsabilidad | Estado |
|---------------|-----------|------------|-----------------|--------|
| vip-gateway-gateway-proxy | vip-gateway | gateway | API Gateway, enrutamiento, autenticacion (Kong) | Activo |
| vip-notificaciones-service-notificacion | vip-notificaciones | service | Emails, SMS, notificaciones push | Planificado |
| vip-archivos-service-archivo | vip-archivos | service | Gestion de archivos S3 | Planificado |

---

## 3. Matriz de Dependencias

### 3.1 Dependencias Internas (Infraestructura)

| Microservicio | PostgreSQL | PostGIS | Redis | RabbitMQ | Keycloak |
|---------------|:----------:|:-------:|:-----:|:--------:|:--------:|
| vip-catastro-service-consulta | Si | No | Si | Si | Si |
| vip-catastro-service-mutacion | Si | No | Si | Si | Si |
| vip-geo-service-geometrias | Si | Si | Si | Si | Si |
| vip-urbanismo-service-licencias | Si | No | Si | Si | Si |
| vip-hacienda-service-predial | Si | No | Si | Si | Si |
| vip-gateway-gateway-proxy | Si | No | Si | No | Si |

### 3.2 Dependencias Externas (Backends)

| Microservicio | IGAC | SNR | Hacienda Municipal | Catastro Backend |
|---------------|:----:|:---:|:------------------:|:----------------:|
| vip-catastro-service-consulta | Si | Si | No | Si |
| vip-catastro-service-mutacion | Si | No | No | Si |
| vip-geo-service-geometrias | Si | No | No | No |
| vip-urbanismo-service-licencias | No | No | No | No |
| vip-hacienda-service-predial | No | No | Si | Si |

### 3.3 Dependencias entre Microservicios

| Microservicio | Depende de | Tipo | Descripcion |
|---------------|------------|------|-------------|
| vip-catastro-service-mutacion | vip-catastro-service-consulta | Sincrono | Valida existencia del predio antes de mutar |
| vip-catastro-service-mutacion | vip-geo-service-geometrias | Sincrono | Valida geometria en mutaciones con componente espacial |
| vip-catastro-service-consulta | vip-catastro-service-mutacion | Evento | Escucha `vip.catastro.predio.actualizado` para invalidar cache |
| vip-hacienda-service-predial | vip-catastro-service-consulta | Sincrono | Consulta datos del predio para liquidacion |

---

## 4. Endpoints por Microservicio

### vip-catastro-service-consulta

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| POST | `/vip/catastro/predios/consultar` | Consultar predio por NPN |
| POST | `/vip/catastro/predios/avaluo` | Consultar avaluo catastral |
| POST | `/vip/catastro/predios/intervinientes` | Consultar intervinientes del predio |
| GET | `/actuator/health/liveness` | Liveness probe |
| GET | `/actuator/health/readiness` | Readiness probe |
| GET | `/swagger-ui.html` | Documentacion Swagger |

### vip-catastro-service-mutacion

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| POST | `/vip/catastro/mutaciones/solicitar` | Solicitar mutacion catastral |
| POST | `/vip/catastro/mutaciones/estado` | Consultar estado de mutacion |
| GET | `/actuator/health/liveness` | Liveness probe |
| GET | `/actuator/health/readiness` | Readiness probe |
| GET | `/swagger-ui.html` | Documentacion Swagger |

### vip-geo-service-geometrias

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| POST | `/vip/geo/geometrias/validar` | Validar geometria OGC |
| POST | `/vip/geo/geometrias/transformar` | Transformar sistema de referencia |
| POST | `/vip/geo/municipio/limites` | Consultar limites del municipio |
| GET | `/actuator/health/liveness` | Liveness probe |
| GET | `/actuator/health/readiness` | Readiness probe |
| GET | `/swagger-ui.html` | Documentacion Swagger |

---

## 5. Colas RabbitMQ por Microservicio

| Microservicio | Publica en | Consume de |
|---------------|------------|------------|
| vip-catastro-service-mutacion | `vip.catastro.mutacion.solicitada`, `vip.catastro.predio.actualizado` | — |
| vip-catastro-service-consulta | — | `vip.catastro.predio.actualizado` (invalida cache) |
| vip-geo-service-geometrias | `vip.geo.validacion.completado`, `vip.geo.validacion.fallido` | `vip.geo.validacion.solicitada` |
| vip-notificaciones-service-notificacion | — | `vip.notificacion.email.enviado` |
| vip-archivos-service-archivo | `vip.archivo.procesamiento.completado` | `vip.archivo.procesamiento.iniciado` |

---

## 6. Stack Tecnologico

| Componente | Tecnologia | Version |
|------------|------------|---------|
| Runtime | Java | 17 LTS |
| Framework | Spring Boot | 3.x |
| API Gateway | Kong | 3.x |
| Base de datos | PostgreSQL | 15+ |
| Extension espacial | PostGIS | 3.x |
| Cache | Redis | 7.x |
| Mensajeria | RabbitMQ | 3.12+ |
| Identidad | Keycloak | 22+ |
| Orquestacion | Kubernetes (EKS) | 1.28+ |
| Workflow | Camunda | 8.x |
| Almacenamiento | AWS S3 | — |
| Documentacion API | OpenAPI 3.0 / Swagger UI | — |

---

## 7. Instrucciones de Actualizacion

Al agregar un nuevo microservicio a la plataforma:

1. Registrarlo en la seccion **2. Inventario** con su dominio, namespace y componente.
2. Actualizar la **Matriz de Dependencias** (seccion 3).
3. Documentar sus **Endpoints** (seccion 4).
4. Registrar sus **Colas** si aplica (seccion 5).
5. Actualizar el catalogo de [Namespaces](lineamientos-namespaces.md) si se crea uno nuevo.
6. Registrar las routes en [Kong](lineamientos-kong.md).
7. Actualizar el catalogo de [Microservicios](nombramiento-microservicios-colas.md) del SF-NM.
