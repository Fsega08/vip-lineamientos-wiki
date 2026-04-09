# Patrones de Comunicacion: Sincrono vs Asincrono

[Volver al indice](/)

---

## 1. Objetivo

Definir cuando usar comunicacion sincrona (REST) y cuando asincrona (RabbitMQ) en la plataforma VIP, garantizando que cada interaccion use el patron adecuado segun sus requisitos de latencia, consistencia y acoplamiento.

---

## 2. Regla General

| Tipo de operacion | Patron | Protocolo |
|-------------------|--------|-----------|
| Consultas | **Sincrono** | REST (HTTP) |
| Escrituras simples (un solo backend) | **Sincrono** | REST (HTTP) |
| Escrituras que tocan multiples backends | **Asincrono** | RabbitMQ |
| Procesos de larga duracion | **Asincrono** | RabbitMQ |
| Invalidacion de cache | **Asincrono** | RabbitMQ (evento) |
| Notificaciones (email, SMS) | **Asincrono** | RabbitMQ |
| Procesamiento de archivos | **Asincrono** | RabbitMQ |

---

## 3. Comunicacion Sincrona (REST)

### Cuando usar

- El consumidor necesita la respuesta **inmediatamente** para continuar.
- La operacion involucra **un solo backend** y se completa en menos de **5 segundos**.
- Es una consulta de datos (GET semantico).

### Ejemplos en VIP

| Operacion | Microservicio | Backend |
|-----------|---------------|---------|
| Consultar predio por NPN | vip-catastro-service-consulta | Catastro BD |
| Consultar avaluo catastral | vip-catastro-service-consulta | Catastro BD |
| Validar geometria OGC | vip-geo-service-geometrias | PostGIS |
| Consultar intervinientes | vip-catastro-service-consulta | IGAC |

### Timeout

- Kong: **30 segundos** (timeout global del gateway).
- Microservicio → Backend: **10 segundos** (timeout del adaptador).
- Si el backend no responde en 10s → error `INT-002` (Timeout).

---

## 4. Comunicacion Asincrona (RabbitMQ)

### Cuando usar

- La operacion toca **multiples backends** y necesita orquestacion.
- El procesamiento puede tardar **mas de 5 segundos**.
- Se necesita **reintento automatico** ante fallos.
- El consumidor puede consultar el estado despues (polling o callback).
- Es un efecto colateral que no necesita respuesta inmediata (notificacion, invalidacion de cache).

### Ejemplos en VIP

| Operacion | Exchange | Routing Key | Consumer |
|-----------|----------|-------------|----------|
| Solicitar mutacion catastral | vip.catastro | vip.catastro.mutacion.solicitada | vip-catastro-service-mutacion |
| Invalidar cache de predio | vip.catastro | vip.catastro.predio.actualizado | vip-catastro-service-consulta |
| Enviar notificacion email | vip.notificacion | vip.notificacion.email.enviado | vip-notificaciones-service-notificacion |
| Procesar archivo cargado | vip.archivo | vip.archivo.procesamiento.iniciado | vip-archivos-service-archivo |
| Validacion geoespacial batch | vip.geo | vip.geo.validacion.solicitada | vip-geo-service-geometrias |

### Reintentos

- Maximo **3 reintentos** con backoff exponencial: 1s, 2s, 4s.
- Si falla despues de los 3 reintentos → el mensaje va a la **DLQ** con TTL de 24 horas.
- Los mensajes en DLQ deben revisarse manualmente o reprocesarse con herramientas de monitoreo.

### Respuesta al consumidor

Cuando una operacion es asincrona, el microservicio retorna inmediatamente un **HTTP 202 (Accepted)** con un ID de seguimiento:

```json
{
  "Id_Mensaje": "550e8400-e29b-41d4-a716-446655440000",
  "Fecha_Respuesta": "2026-04-06T08:45:00-05:00",
  "Http_Estado": 202,
  "Errores": [],
  "Datos_Operacion": {
    "Id_Transaccion": "TRX-20260406-000456",
    "Estado": "solicitado",
    "Mensaje": "La solicitud de mutacion fue recibida y esta en proceso."
  }
}
```

El consumidor puede consultar el estado despues con el `Id_Transaccion`, o recibir el resultado via el `Url_Retorno` del Canonical (callback).

---

## 5. Patron Mixto: Request-Reply Asincrono

Para operaciones que son demasiado lentas para ser sincronas pero donde el consumidor quiere esperar el resultado:

1. El consumidor envia request con `Url_Retorno` en el Canonical.
2. El microservicio publica en RabbitMQ y retorna **HTTP 202**.
3. El consumer procesa la operacion.
4. Al completar, el consumer invoca el `Url_Retorno` con el resultado.

Esto es util para mutaciones catastrales que involucran validaciones con IGAC + SNR + PostGIS.

---

## 6. Decision: Cuando NO usar asincrono

- Si solo quieres desacoplar por "buena practica" pero la operacion tarda < 2 segundos → usa sincrono. La complejidad del async no se justifica.
- Si necesitas consistencia transaccional fuerte (ACID) entre dos escrituras → usa sincrono con compensacion, no colas.
- Si el consumidor no puede manejar una respuesta 202 → usa sincrono.
