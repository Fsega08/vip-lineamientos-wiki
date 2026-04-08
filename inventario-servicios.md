# Inventario de Servicios y Dependencias

[Volver al indice](index.md)

---

## 1. Objetivo

Centralizar el inventario de todos los microservicios de la plataforma VIP con sus dependencias, endpoints, tecnologias y estado actual. Este documento es la fuente de verdad para entender la topologia de la capa de interoperabilidad.

---

## 2. Inventario de Microservicios

### 2.1 Dominio: Catastro

| Microservicio | Namespace | Responsabilidad | Estado |
|---------------|-----------|-----------------|--------|
| vip-catastro-consulta | vip-catastro | Consultas de predios, NPN, avaluos, intervinientes | Activo |
| vip-catastro-mutacion | vip-catastro | Operaciones de mutacion catastral (transferencias, actualizaciones) | Activo |

### 2.2 Dominio: Geoespacial

| Microservicio | Namespace | Responsabilidad | Estado |
|---------------|-----------|-----------------|--------|
| vip-geo-service | vip-geo | Operaciones PostGIS, validacion de geometrias, transformacion de coordenadas | Activo |

### 2.3 Dominio: Urbanismo

| Microservicio | Namespace | Responsabilidad | Estado |
|---------------|-----------|-----------------|--------|
| vip-urbanismo-licencias | vip-urbanismo | Gestion de licencias urbanisticas, uso de suelo | Planificado |

### 2.4 Dominio: Hacienda

| Microservicio | Namespace | Responsabilidad | Estado |
|---------------|-----------|-----------------|--------|
| vip-hacienda-predial | vip-hacienda | Impuesto predial, liquidacion tributaria | Planificado |

### 2.5 Plataforma

| Microservicio | Namespace | Responsabilidad | Estado |
|---------------|-----------|-----------------|--------|
| vip-gateway (Kong) | vip-gateway | API Gateway, enrutamiento, autenticacion | Activo |
| vip-notificacion-service | vip-notificaciones | Emails, SMS, notificaciones push | Planificado |
| vip-archivo-service | vip-archivos | Gestion de archivos S3 | Planificado |

---

## 3. Matriz de Dependencias

### 3.1 Dependencias Internas (Infraestructura)

| Microservicio | PostgreSQL | PostGIS | Redis | RabbitMQ | Keycloak |
|---------------|:----------:|:-------:|:-----:|:--------:|:--------:|
| vip-catastro-consulta | Si | No | Si | Si | Si |
| vip-catastro-mutacion | Si | No | Si | Si | Si |
| vip-geo-service | Si | Si | Si | Si | Si |
| vip-urbanismo-licencias | Si | No | Si | Si | Si |
| vip-hacienda-predial | Si | No | Si | Si | Si |
| vip-gateway (Kong) | Si | No | Si | No | Si |

### 3.2 Dependencias Externas (Backends)

| Microservicio | IGAC | SNR | Hacienda Municipal | Catastro Backend |
|---------------|:----:|:---:|:------------------:|:----------------:|
| vip-catastro-consulta | Si | Si | No | Si |
| vip-catastro-mutacion | Si | No | No | Si |
| vip-geo-service | Si | No | No | No |
| vip-urbanismo-licencias | No | No | No | No |
| vip-hacienda-predial | No | No | Si | Si |

### 3.3 Dependencias entre Microservicios

| Microservicio | Depende de | Tipo | Descripcion |
|---------------|------------|------|-------------|
| vip-catastro-mutacion | vip-catastro-consulta | Sincrono | Valida existencia del predio antes de mutar |
| vip-catastro-mutacion | vip-geo-service | Sincrono | Valida geometria en mutaciones con componente espacial |
| vip-catastro-consulta | vip-catastro-mutacion | Evento | Escucha `vip.catastro.predio.actualizado` para invalidar cache |
| vip-hacienda-predial | vip-catastro-consulta | Sincrono | Consulta datos del predio para liquidacion |

---

## 4. Endpoints por Microservicio

### vip-catastro-consulta

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| POST | `/vip/catastro/predios/consultar` | Consultar predio por NPN |
| POST | `/vip/catastro/predios/avaluo` | Consultar avaluo catastral |
| POST | `/vip/catastro/predios/intervinientes` | Consultar intervinientes del predio |
| GET | `/actuator/health/liveness` | Liveness probe |
| GET | `/actuator/health/readiness` | Readiness probe |
| GET | `/swagger-ui.html` | Documentacion Swagger |

### vip-catastro-mutacion

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| POST | `/vip/catastro/mutaciones/solicitar` | Solicitar mutacion catastral |
| POST | `/vip/catastro/mutaciones/estado` | Consultar estado de mutacion |
| GET | `/actuator/health/liveness` | Liveness probe |
| GET | `/actuator/health/readiness` | Readiness probe |
| GET | `/swagger-ui.html` | Documentacion Swagger |

### vip-geo-service

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
| vip-catastro-mutacion | `vip.catastro.mutacion.solicitada`, `vip.catastro.predio.actualizado` | — |
| vip-catastro-consulta | — | `vip.catastro.predio.actualizado` (invalida cache) |
| vip-geo-service | `vip.geo.validacion.completado`, `vip.geo.validacion.fallido` | `vip.geo.validacion.solicitada` |
| vip-notificacion-service | — | `vip.notificacion.email.enviado` |
| vip-archivo-service | `vip.archivo.procesamiento.completado` | `vip.archivo.procesamiento.iniciado` |

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

1. Registrarlo en la seccion **2. Inventario** con su dominio y namespace.
2. Actualizar la **Matriz de Dependencias** (seccion 3).
3. Documentar sus **Endpoints** (seccion 4).
4. Registrar sus **Colas** si aplica (seccion 5).
5. Actualizar el catalogo de [Namespaces](lineamientos-namespaces.md) si se crea uno nuevo.
6. Registrar las routes en [Kong](lineamientos-kong.md).
7. Actualizar el catalogo de [Microservicios](nombramiento-microservicios-colas.md) del SF-NM.
