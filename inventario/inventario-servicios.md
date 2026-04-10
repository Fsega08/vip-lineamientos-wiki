# Inventario de Servicios y Dependencias

[Volver al indice](/)

---

## 1. Objetivo

Centralizar el inventario de todos los microservicios de la plataforma VIP con sus dependencias, endpoints, tecnologias y estado actual. Este documento es la fuente de verdad para entender la topologia de la capa de interoperabilidad.

---

## 2. Inventario de Microservicios

### 2.1 Dominio: Catastro

| Microservicio | Namespace | Componente | Responsabilidad | Estado |
|---------------|-----------|------------|-----------------|--------|
| acm-catastro-service-consulta | acm-catastro | service | Consultas de predios, NPN, avaluos, intervinientes | Activo |
| acm-catastro-service-mutacion | acm-catastro | service | Operaciones de mutacion catastral (transferencias, actualizaciones) | Activo |

### 2.2 Dominio: Geoespacial

| Microservicio | Namespace | Componente | Responsabilidad | Estado |
|---------------|-----------|------------|-----------------|--------|
| acm-geo-service-geometrias | acm-geo | service | Operaciones PostGIS, validacion de geometrias, transformacion de coordenadas | Activo |

### 2.3 Dominio: Urbanismo

| Microservicio | Namespace | Componente | Responsabilidad | Estado |
|---------------|-----------|------------|-----------------|--------|
| acm-urbanismo-service-licencias | acm-urbanismo | service | Gestion de licencias urbanisticas, uso de suelo | Planificado |

### 2.4 Dominio: Hacienda

| Microservicio | Namespace | Componente | Responsabilidad | Estado |
|---------------|-----------|------------|-----------------|--------|
| acm-hacienda-service-predial | acm-hacienda | service | Impuesto predial, liquidacion tributaria | Planificado |

### 2.5 Plataforma

| Microservicio | Namespace | Componente | Responsabilidad | Estado |
|---------------|-----------|------------|-----------------|--------|
| acm-gateway-gateway-proxy | acm-gateway | gateway | API Gateway, enrutamiento, autenticacion (Kong) | Activo |
| acm-notificaciones-service-notificacion | acm-notificaciones | service | Emails, SMS, notificaciones push | Planificado |
| acm-archivos-service-archivo | acm-archivos | service | Gestion de archivos S3 | Planificado |

---

## 3. Matriz de Dependencias

### 3.1 Dependencias Internas (Infraestructura)

| Microservicio | PostgreSQL | PostGIS | Redis | RabbitMQ | Keycloak |
|---------------|:----------:|:-------:|:-----:|:--------:|:--------:|
| acm-catastro-service-consulta | Si | No | Si | Si | Si |
| acm-catastro-service-mutacion | Si | No | Si | Si | Si |
| acm-geo-service-geometrias | Si | Si | Si | Si | Si |
| acm-urbanismo-service-licencias | Si | No | Si | Si | Si |
| acm-hacienda-service-predial | Si | No | Si | Si | Si |
| acm-gateway-gateway-proxy | Si | No | Si | No | Si |

### 3.2 Dependencias Externas (Backends)

| Microservicio | IGAC | SNR | Hacienda Municipal | Catastro Backend |
|---------------|:----:|:---:|:------------------:|:----------------:|
| acm-catastro-service-consulta | Si | Si | No | Si |
| acm-catastro-service-mutacion | Si | No | No | Si |
| acm-geo-service-geometrias | Si | No | No | No |
| acm-urbanismo-service-licencias | No | No | No | No |
| acm-hacienda-service-predial | No | No | Si | Si |

### 3.3 Dependencias entre Microservicios

| Microservicio | Depende de | Tipo | Descripcion |
|---------------|------------|------|-------------|
| acm-catastro-service-mutacion | acm-catastro-service-consulta | Sincrono | Valida existencia del predio antes de mutar |
| acm-catastro-service-mutacion | acm-geo-service-geometrias | Sincrono | Valida geometria en mutaciones con componente espacial |
| acm-catastro-service-consulta | acm-catastro-service-mutacion | Evento | Escucha `acm.catastro.predio.actualizado` para invalidar cache |
| acm-hacienda-service-predial | acm-catastro-service-consulta | Sincrono | Consulta datos del predio para liquidacion |

---

## 4. Endpoints por Microservicio

### acm-catastro-service-consulta

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| POST | `/acm/catastro/v1/predios/consultar` | Consultar predio por NPN |
| POST | `/acm/catastro/v1/predios/avaluo` | Consultar avaluo catastral |
| POST | `/acm/catastro/v1/predios/intervinientes` | Consultar intervinientes del predio |
| GET | `/actuator/health/liveness` | Liveness probe |
| GET | `/actuator/health/readiness` | Readiness probe |
| GET | `/swagger-ui.html` | Documentacion Swagger |

### acm-catastro-service-mutacion

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| POST | `/acm/catastro/v1/mutaciones/solicitar` | Solicitar mutacion catastral |
| POST | `/acm/catastro/v1/mutaciones/estado` | Consultar estado de mutacion |
| GET | `/actuator/health/liveness` | Liveness probe |
| GET | `/actuator/health/readiness` | Readiness probe |
| GET | `/swagger-ui.html` | Documentacion Swagger |

### acm-geo-service-geometrias

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| POST | `/acm/geo/v1/geometrias/validar` | Validar geometria OGC |
| POST | `/acm/geo/v1/geometrias/transformar` | Transformar sistema de referencia |
| POST | `/acm/geo/v1/municipio/limites` | Consultar limites del municipio |
| GET | `/actuator/health/liveness` | Liveness probe |
| GET | `/actuator/health/readiness` | Readiness probe |
| GET | `/swagger-ui.html` | Documentacion Swagger |

---

## 5. Colas RabbitMQ por Microservicio

| Microservicio | Publica en | Consume de |
|---------------|------------|------------|
| acm-catastro-service-mutacion | `acm.catastro.mutacion.solicitada`, `acm.catastro.predio.actualizado` | — |
| acm-catastro-service-consulta | — | `acm.catastro.predio.actualizado` (invalida cache) |
| acm-geo-service-geometrias | `acm.geo.validacion.completado`, `acm.geo.validacion.fallido` | `acm.geo.validacion.solicitada` |
| acm-notificaciones-service-notificacion | — | `acm.notificacion.email.enviado` |
| acm-archivos-service-archivo | `acm.archivo.procesamiento.completado` | `acm.archivo.procesamiento.iniciado` |

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
5. Actualizar el catalogo de [Namespaces](infraestructura/lineamientos-namespaces.md) si se crea uno nuevo.
6. Registrar las routes en [Kong](infraestructura/lineamientos-kong.md).
7. Actualizar el catalogo de [Microservicios](estandares/nombramiento-microservicios-colas.md) del SF-NM.
