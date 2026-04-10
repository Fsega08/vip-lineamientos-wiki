# Preguntas Frecuentes (FAQ)

[Volver al indice](/)

---

## Contratos y Canonical

**Campo no obligatorio es lo mismo que campo vacio?**

No. Un campo **no obligatorio** permite enviar `null` (el campo existe con valor nulo). Un campo **vacio** (`""`) es un string de longitud 0, y solo esta permitido si la longitud minima del campo es 0. Si se envia `""` en un campo con longitud minima > 0, se retorna error `VLD-002`.

---

**Si el backend no retorna un campo opcional, que enviamos?**

Se envia como `null`. No se omite el campo del JSON — la estructura siempre va completa. Solo si hay una indicacion explicita en la documentacion del servicio se puede enviar como vacio (`""`) o no enviarlo.

---

**Cuando uso HTTP 409 vs HTTP 400?**

- **400** (`VALIDACION`): el error es de estructura o formato en la capa de interoperabilidad. El request ni siquiera llego al backend. Ejemplo: campo obligatorio faltante, formato de fecha invalido.
- **409** (`NEGOCIO`): el request es valido en estructura, pero el backend rechaza la operacion por logica de negocio. Ejemplo: predio no existe, predio en embargo, NPN con formato IGAC invalido.

---

**Puedo tener errores de tipo diferente en el mismo response?**

Si. El array `Errores` puede contener errores de tipos mixtos. Por ejemplo, un error `VALIDACION` y un error `NEGOCIO` en la misma respuesta. El `Tipo_Error` va dentro de cada error, no a nivel raiz.

---

**Una consulta que no retorna resultados es un error?**

No. Una consulta sin resultados retorna **HTTP 200** con `Registros: []` y `Total_Registros: 0`. El error `BIZ-002` ("El registro solicitado no fue encontrado") solo aplica cuando se busca un registro especifico por ID (ej: consultar un predio por NPN que no existe).

---

**Por que los campos del body van en espanol y no en ingles?**

Porque la plataforma VIP esta alineada con el modelo LADM-COL 4.1, que define sus entidades y atributos en espanol. Usar nomenclatura en espanol con `Snake_Case` y mayuscula inicial reduce la ambiguedad al mapear campos del modelo catastral. Los headers HTTP se mantienen en ingles por ser estandar del protocolo.

---

## Errores

**Que significa el campo `Ruta` en un error?**

Es un [JSON Pointer](https://datatracker.ietf.org/doc/html/rfc6901) que indica exactamente que campo del request causo el error. Ejemplo: `/Datos_Operacion/Numero_Documento` apunta al campo `Numero_Documento` dentro de `Datos_Operacion`. Esto permite que el frontend resalte el campo con error en el formulario.

---

**Que hago si el backend retorna un error que no esta en el catalogo?**

El adaptador debe clasificarlo en la categoria mas cercana (`BIZ`, `CTR`, `GEO`, etc.) y asignar un codigo existente si aplica. Si es un escenario nuevo, se debe agregar un codigo al catalogo en el documento de [Manejo de Errores](contratos/manejo-errores.md) y actualizar la wiki.

---

**Los errores de tipo SISTEMA exponen detalles tecnicos?**

Nunca. Los errores `SISTEMA` (SYS-001 a SYS-005) siempre retornan mensajes genericos como "Se presento un error tecnico durante el procesamiento de la solicitud". Los stack traces, nombres de tablas, IPs internas y versiones de software **nunca** se incluyen en la respuesta.

---

## Naming y Convenciones

**Como agrego un nuevo dominio a la plataforma?**

1. Definir el nombre del dominio (ej: `tributario`).
2. Crear el namespace `vip-tributario` siguiendo [Lineamientos de Namespaces](infraestructura/lineamientos-namespaces.md).
3. Registrar los microservicios con patron `vip-tributario-{componente}-{funcionalidad}`.
4. Registrar el exchange en RabbitMQ: `vip.tributario`.
5. Actualizar el [Inventario de Servicios](inventario/inventario-servicios.md).
6. Si aplica, definir un nuevo prefijo de error (ej: `TRB`) en el catalogo de errores.
7. Registrar las routes en [Kong](infraestructura/lineamientos-kong.md).

---

**El prefijo `vip` es fijo?**

No. El prefijo es configurable por proyecto en `application.yml` bajo `plataforma.prefijo`. Para la Alcaldia de Medellin es `vip`. Para otros clientes puede ser diferente. Todo lo que usa el prefijo (microservicios, namespaces, queues, keys de Redis) se adapta automaticamente.

---

**Por que el patron de microservicios tiene 4 segmentos y no 3?**

El cuarto segmento (`componente`) permite diferenciar el tipo tecnico: `service` (REST), `adapter` (integracion con backend externo), `listener` (consumidor de colas), `worker` (batch), `engine` (workflow), `gateway`. Esto es clave cuando un mismo dominio tiene multiples tipos de artefactos.

---

## Cache

**Si cacheo una consulta paginada, como manejo las diferentes paginas?**

Los parametros de paginacion y filtrado se incluyen en el hash del request para generar la key de Redis. Ejemplo: `vip:catastro:consulta:{hash(filtros+pagina+tamanio)}`. Cada combinacion de filtros + pagina es una key diferente.

---

**Cuando se invalida el cache automaticamente?**

Cuando un microservicio modifica un dato, publica un evento en RabbitMQ (ej: `vip.catastro.predio.actualizado`). El microservicio de consulta escucha el evento y borra la key de Redis. El siguiente request traera el dato fresco del backend. Ver detalle en [Cache Redis](estandares/cache-redis.md).

---

## RabbitMQ

**Que pasa si un mensaje falla 3 veces?**

Despues de 3 reintentos con backoff exponencial (1s, 2s, 4s), el mensaje va a la **DLQ** (Dead Letter Queue) con un TTL de 24 horas. Los mensajes en DLQ deben investigarse y reprocesarse manualmente. Ver [Runbook - seccion 4](guias/runbook.md).

---

**Cuando uso sincrono (REST) vs asincrono (RabbitMQ)?**

Regla general: consultas siempre sincronas, escrituras simples sincronas, escrituras que tocan multiples backends asincronas. Ver detalle en [Comunicacion Sync vs Async](estandares/comunicacion-sync-async.md).

---

## Seguridad

**El microservicio valida el JWT?**

No la firma — eso lo hace Kong con el plugin OIDC. El microservicio solo extrae los claims del token (roles, usuario, scopes) para verificar si el usuario tiene permisos para la operacion especifica. Si Kong deja pasar el request, el token es valido.

---

**Donde se guardan los passwords de base de datos?**

En **Vault**. Nunca en variables de entorno en texto plano, ConfigMaps, ni en el repositorio de codigo. Los microservicios se autentican con Vault via ServiceAccount (IRSA en AWS) y obtienen los secretos en memoria.

---

## Infraestructura

**Puedo conectarme directamente a un microservicio sin pasar por Kong?**

No en QA/produccion. Las Network Policies solo permiten trafico entrante desde el namespace de Kong al puerto 8080. En dev las Network Policies estan deshabilitadas para facilitar debugging.

---

**Como veo los logs de un microservicio?**

Tres opciones:
1. **kubectl**: `kubectl logs -f -l app=vip-catastro-service-consulta -n vip-catastro`
2. **Grafana/Loki**: filtrar con `{namespace="vip-catastro", app="vip-catastro-service-consulta"}`
3. **Por correlation ID**: en Loki `{namespace="vip-catastro"} |= "550e8400-e29b..."` para seguir un request especifico

Ver mas en el [Runbook](guias/runbook.md).
