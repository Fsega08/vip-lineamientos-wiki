# Estandar de Nombramiento de Microservicios y Colas

[Volver al indice](/)

---

## 1. Objetivo

Definir la convencion estandar de nombramiento para microservicios, exchanges, colas (queues), routing keys y dead letter queues (DLQ) de RabbitMQ en la plataforma VIP, garantizando consistencia, trazabilidad y reutilizacion entre proyectos.

---

## 2. Prefijo de Proyecto

| Prefijo | Cliente | Proyecto |
|---------|---------|----------|
| vip | Alcaldia de Medellin | Vortex Integration Platform Alcaldia Medellin |

El prefijo es configurable por proyecto en `application.yml` bajo `plataforma.prefijo`.

---

## 3. Nombramiento de Microservicios

### 3.1 Patron

```
{prefijo}-{namespace}-{componente}-{funcionalidad}
```

| Segmento | Descripcion | Ejemplo |
|----------|-------------|---------|
| `prefijo` | Identificador del proyecto, configurable por ambiente | `acm` |
| `namespace` | Dominio funcional. Coincide con el namespace de K8s (sin prefijo) | `catastro`, `geo`, `gateway` |
| `componente` | Tipo tecnico del artefacto (ver tabla 3.2) | `service`, `adapter`, `gateway` |
| `funcionalidad` | Funcion especifica que cumple | `consulta`, `mutacion`, `validacion` |

### 3.2 Tipos de Componente

| Componente | Descripcion | Uso |
|------------|-------------|-----|
| `service` | Microservicio REST que expone logica de negocio o consulta | Servicios principales de cada dominio |
| `adapter` | Adaptador hacia un backend o sistema externo | Integraciones con IGAC, SNR, Hacienda Municipal |
| `listener` | Consumidor de eventos o colas de mensajeria | Procesamiento asincrono de eventos RabbitMQ |
| `worker` | Proceso batch o de background | Tareas programadas, procesamiento masivo |
| `engine` | Motor de workflow o reglas de negocio | Camunda, motores de decision |
| `gateway` | API Gateway o punto de entrada | Kong, BFF |

### 3.3 Reglas

- Todo en **minusculas**, formato **kebab-case** (separado por guiones).
- Siempre **4 segmentos**: prefijo, namespace, componente y funcionalidad.
- El prefijo se obtiene de la configuracion del ambiente (`application.yml` bajo `plataforma.prefijo`).
- El namespace identifica el dominio funcional y **debe coincidir** con el namespace de Kubernetes (sin el prefijo). Ej: namespace K8s `acm-catastro` → segmento `catastro`.
- El componente indica el tipo tecnico del artefacto (ver tabla 3.2).
- La funcionalidad describe la responsabilidad especifica del microservicio.

### 3.4 Catalogo de Microservicios

| Microservicio | Namespace | Componente | Funcionalidad |
|---------------|-----------|------------|---------------|
| acm-catastro-service-consulta | acm-catastro | service | Consultas de predios, NPN, avaluos |
| acm-catastro-service-mutacion | acm-catastro | service | Operaciones de mutacion catastral |
| acm-geo-service-geometrias | acm-geo | service | Operaciones PostGIS, validacion de geometrias |
| acm-urbanismo-service-licencias | acm-urbanismo | service | Gestion de licencias urbanisticas |
| acm-hacienda-service-predial | acm-hacienda | service | Impuesto predial, liquidacion |
| acm-gateway-gateway-proxy | acm-gateway | gateway | API Gateway / punto de entrada (Kong) |
| acm-notificaciones-service-notificacion | acm-notificaciones | service | Emails, SMS, notificaciones push |
| acm-archivos-service-archivo | acm-archivos | service | Gestion de archivos S3 |

---

## 4. Nombramiento de Colas RabbitMQ

### 4.1 Componentes y Patrones

La mensajeria se organiza con exchanges de tipo **Topic**. Cada dominio tiene su propio exchange y las colas se vinculan mediante routing keys especificas.

> **Nota:** Las colas y routing keys mantienen su propia convencion con dot notation (separados por puntos) independiente del patron de 4 segmentos de los microservicios.

| Componente | Patron | Ejemplo |
|------------|--------|---------|
| Exchange | `{prefijo}.{dominio}` | `acm.catastro` |
| Queue | `{prefijo}.{dominio}.{accion}.queue` | `acm.catastro.mutacion.queue` |
| Routing Key | `{prefijo}.{dominio}.{accion}.{evento}` | `acm.catastro.mutacion.solicitada` |
| Dead Letter Queue | `{prefijo}.{dominio}.{accion}.dlq` | `acm.catastro.mutacion.dlq` |

### 4.2 Reglas

- Todo en **minusculas**, separado por **puntos** (dot notation) para colas y routing keys.
- Las queues siempre terminan con el sufijo `.queue`.
- Las Dead Letter Queues siempre terminan con el sufijo `.dlq`.
- Cada exchange es de tipo **Topic**, lo que permite suscripciones flexibles con wildcards (`*` y `#`).
- Los eventos en las routing keys se expresan en **participio pasado en espanol** (solicitada, completado, fallido).

### 4.3 Eventos Estandar

Los siguientes eventos estan predefinidos como vocabulario comun para las routing keys:

| Evento | Descripcion |
|--------|-------------|
| solicitado | Un recurso o accion ha sido solicitado y esta pendiente de procesamiento |
| procesando | El recurso esta siendo procesado activamente |
| completado | La operacion se completo exitosamente |
| fallido | La operacion fallo y no pudo completarse |
| actualizado | Un recurso existente fue modificado |
| eliminado | Un recurso fue eliminado del sistema |
| reintentando | La operacion fallo y se esta reintentando automaticamente |

### 4.4 Catalogo de Colas

| Exchange | Queue | Routing Key | DLQ |
|----------|-------|-------------|-----|
| {prefijo}.catastro | {prefijo}.catastro.mutacion.queue | ...mutacion.solicitada | ...mutacion.dlq |
| {prefijo}.catastro | {prefijo}.catastro.avaluo.queue | ...avaluo.actualizado | ...avaluo.dlq |
| {prefijo}.archivo | {prefijo}.archivo.procesamiento.queue | ...procesamiento.iniciado | ...procesamiento.dlq |
| {prefijo}.notificacion | {prefijo}.notificacion.email.queue | ...email.enviado | ...email.dlq |
| {prefijo}.workflow | {prefijo}.workflow.tarea.queue | ...tarea.asignada | ...tarea.dlq |
| {prefijo}.geo | {prefijo}.geo.validacion.queue | ...validacion.solicitada | ...validacion.dlq |

---

## 5. Consideraciones Generales

- El prefijo es **configurable por ambiente** en `application.yml` bajo `plataforma.prefijo` y no debe hardcodearse en el codigo fuente.
- Al agregar un nuevo microservicio o cola, se debe actualizar el catalogo en este documento.
- Las DLQ deben configurarse con politicas de reintento: maximo **3 reintentos** con **backoff exponencial** (1s, 2s, 4s). Mensajes fallidos van a DLQ con TTL de **24 horas**.
- Los exchanges de tipo Topic permiten que un consumidor se suscriba a todos los eventos de un dominio usando el wildcard `#` (ej: `acm.catastro.#`).
- La convencion de 4 segmentos aplica tambien al nombre del **artefacto Maven**, al nombre del **contenedor Docker** y al **Deployment en Kubernetes (EKS)**.
- El segmento `namespace` del nombre del microservicio debe coincidir con el namespace de K8s (ver [Lineamientos de Namespaces](infraestructura/lineamientos-namespaces.md)).
