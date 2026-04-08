# Estandar de Nombramiento de Microservicios y Colas

[Volver al indice](index.md)

---

## 1. Objetivo

Definir la convencion estandar de nombramiento para microservicios, exchanges, colas (queues), routing keys y dead letter queues (DLQ) de RabbitMQ en la plataforma VIP, garantizando consistencia, trazabilidad y reutilizacion entre proyectos.

---

## 2. Prefijo de Proyecto

| Prefijo | Cliente | Proyecto |
|---------|---------|----------|
| acm | Alcaldia de Medellin | Vortex Integration Platform Alcaldia Medellin |

---

## 3. Nombramiento de Microservicios

### 3.1 Patron

```
{prefijo}-{dominio}-{responsabilidad}
```

### 3.2 Reglas

- Todo en **minusculas**, formato **kebab-case** (separado por guiones).
- Maximo **3 segmentos**: prefijo, dominio y responsabilidad.
- El prefijo se obtiene de la configuracion del ambiente (`application.properties`).
- El dominio identifica el modulo funcional (catastro, geo, urbanismo, hacienda, etc.).
- La responsabilidad describe la funcion especifica del microservicio (consulta, mutacion, service, engine).
- Si el microservicio es de proposito general dentro del dominio, usar `service` como responsabilidad.

### 3.3 Catalogo de Microservicios

| Microservicio | Dominio | Responsabilidad |
|---------------|---------|-----------------|
| {prefijo}-catastro-consulta | Catastro | Consultas de predios, NPN, avaluos |
| {prefijo}-catastro-mutacion | Catastro | Operaciones de mutacion catastral |
| {prefijo}-geo-service | Geoespacial | Operaciones PostGIS, validacion de geometrias |
| {prefijo}-urbanismo-licencias | Urbanismo | Gestion de licencias urbanisticas |
| {prefijo}-hacienda-predial | Hacienda | Impuesto predial, liquidacion |
| {prefijo}-gateway | Plataforma | API Gateway / punto de entrada |
| {prefijo}-notificacion-service | Plataforma | Emails, SMS, notificaciones push |
| {prefijo}-archivo-service | Plataforma | Gestion de archivos S3 |

---

## 4. Nombramiento de Colas RabbitMQ

### 4.1 Componentes y Patrones

La mensajeria se organiza con exchanges de tipo **Topic**. Cada dominio tiene su propio exchange y las colas se vinculan mediante routing keys especificas.

| Componente | Patron | Ejemplo |
|------------|--------|---------|
| Exchange | `{prefijo}.{dominio}` | `vip.catastro` |
| Queue | `{prefijo}.{dominio}.{accion}.queue` | `vip.catastro.mutacion.queue` |
| Routing Key | `{prefijo}.{dominio}.{accion}.{evento}` | `vip.catastro.mutacion.solicitada` |
| Dead Letter Queue | `{prefijo}.{dominio}.{accion}.dlq` | `vip.catastro.mutacion.dlq` |

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

- El prefijo es **configurable por ambiente** y no debe hardcodearse en el codigo fuente.
- Al agregar un nuevo microservicio o cola, se debe actualizar el catalogo en este documento.
- Las DLQ deben configurarse con politicas de reintento (retry count, backoff) definidas por el equipo de infraestructura.
- Los exchanges de tipo Topic permiten que un consumidor se suscriba a todos los eventos de un dominio usando el wildcard `#` (ej: `vip.catastro.#`).
- La convencion de microservicios aplica tambien al nombre del **artefacto Maven**, al nombre del **contenedor Docker** y al **Deployment en Kubernetes (EKS)**.
