# Manejo de Errores de Interoperabilidad

[Volver al indice](index.md)

---

## 1. Objetivo

Definir las estrategias y practicas estandar para el manejo de errores en los microservicios de la capa de interoperabilidad de la plataforma VIP (Vortex Integration Platform), implementada para la Alcaldia de Medellin. Se abarca la gestion de errores en terminos de validacion, negocio, seguridad, integracion con servicios externos y errores de sistema.

## 2. Alcance

Este estandar aplica a todos los microservicios que conforman la capa de interoperabilidad (middleware) de VIP:

- Servicios de consulta y gestion catastral (LADM-COL 4.1).
- Integraciones con backends externos (IGAC, SNR, Hacienda Municipal).
- Servicios geoespaciales (PostGIS).
- Orquestacion de procesos (Camunda).
- Gestion de identidad y autorizacion (Keycloak/OIDC).
- Cualquier nuevo microservicio que se integre a la plataforma VIP.

---

## 3. Estructura del Mensaje de Error

La estructura de respuesta estandar utiliza la convencion de nomenclatura del modelo LADM-COL (espanol, Snake_Case con mayuscula inicial). Todos los microservicios deben retornar sus respuestas utilizando esta estructura.

### 3.1 Descripcion de Campos

| Campo | Tipo | Descripcion |
|-------|------|-------------|
| Id_Mensaje | String (UUID v4) | Identificador unico de la solicitud/respuesta |
| Fecha_Hora_Solicitud | String (ISO 8601) | Marca de tiempo con zona horaria de Colombia (-05:00). Formato: `YYYY-MM-DDThh:mm:ss-05:00` |
| Http_Estado | Integer | Codigo de estado HTTP (200, 400, 401, 403, 409, 500, 502, 503, 504) |
| Errores | Array\<Error\> | Lista de errores. Vacia (`[]`) cuando Http_Estado es 200 |
| Errores[].Tipo_Error | String (Enum) | Clasificacion: `VALIDACION`, `NEGOCIO`, `SEGURIDAD`, `INTEGRACION`, `SISTEMA` |
| Errores[].Codigo_Error | String | Codigo unico del error segun catalogo por dominio (ej: VLD-001) |
| Errores[].Ruta | String \| null | JSON Pointer al campo con error. `null` si no aplica |
| Errores[].Mensaje | String | Descripcion clara y concisa del error |
| Errores[].Origen | String | Microservicio o adaptador que origino el error |
| Datos_Operacion | Object | Datos especificos de la respuesta. Estructura propia de cada servicio |

### 3.2 Ejemplo: Respuesta Exitosa

```
Http_Estado: 200
```

```json
{
  "Id_Mensaje": "550e8400-e29b-41d4-a716-446655440000",
  "Fecha_Hora_Solicitud": "2026-04-06T08:45:00-05:00",
  "Http_Estado": 200,
  "Errores": [],
  "Datos_Operacion": {
    "Id_Predio": "PRD-12345",
    "Numero_Predial": "05001010203040000000",
    "Direccion": "Calle 50 # 45-20"
  }
}
```

### 3.3 Ejemplo: Respuesta con Errores

```
Http_Estado: 409
```

```json
{
  "Id_Mensaje": "550e8400-e29b-41d4-a716-446655440000",
  "Fecha_Hora_Solicitud": "2026-04-06T08:45:00-05:00",
  "Http_Estado": 409,
  "Errores": [
    {
      "Tipo_Error": "VALIDACION",
      "Codigo_Error": "VLD-001",
      "Ruta": "/Datos_Operacion/Numero_Documento",
      "Mensaje": "El campo Numero_Documento es obligatorio.",
      "Origen": "vip-gateway"
    },
    {
      "Tipo_Error": "NEGOCIO",
      "Codigo_Error": "CTR-012",
      "Ruta": "/Datos_Operacion/Tipo_Solicitud",
      "Mensaje": "El tipo de solicitud no se encuentra parametrizado.",
      "Origen": "catastro-service"
    }
  ],
  "Datos_Operacion": {}
}
```

### 3.4 Ejemplo: Error de Integracion

```
Http_Estado: 502
```

```json
{
  "Id_Mensaje": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "Fecha_Hora_Solicitud": "2026-04-06T09:15:33-05:00",
  "Http_Estado": 502,
  "Errores": [
    {
      "Tipo_Error": "INTEGRACION",
      "Codigo_Error": "INT-001",
      "Ruta": null,
      "Mensaje": "El servicio externo snr-service no se encuentra disponible.",
      "Origen": "snr-adapter"
    }
  ],
  "Datos_Operacion": {}
}
```

---

## 4. Clasificacion de Tipos de Error

| Tipo_Error | HTTP | Descripcion |
|------------|------|-------------|
| VALIDACION | 400 | Errores de estructura, formato o campos requeridos propios de la capa de interoperabilidad |
| NEGOCIO | 409 | Errores de logica de negocio del dominio. Se envia el error del backend validando que sea diciente |
| SEGURIDAD | 401 / 403 | Errores de autenticacion (token invalido/expirado) o autorizacion (permisos insuficientes) desde Keycloak |
| INTEGRACION | 502 / 504 | Errores en la comunicacion con servicios externos (IGAC, SNR, backends de la Alcaldia) |
| SISTEMA | 500 - 599 | Errores inesperados de infraestructura, base de datos o logica interna. No se exponen detalles tecnicos |

### 4.1 Reglas de Clasificacion

- Si son errores de validacion del microservicio (estructura, formato, campos requeridos) → **VALIDACION** con `Http_Estado 400`.
- Si son errores dicientes del backend (logica de negocio) → **NEGOCIO** con `Http_Estado 409`. Se devuelven los errores del backend dentro del array `Errores` con su codigo y mensaje.
- Si son errores de autenticacion o autorizacion provenientes de Keycloak → **SEGURIDAD** con `Http_Estado 401` o `403`.
- Si son errores de comunicacion con servicios externos → **INTEGRACION** con `Http_Estado 502` o `504`.
- Si son errores tecnicos internos → **SISTEMA**. Deben traducirse mediante el adaptador para evitar exponer detalles de infraestructura.

---

## 5. Catalogo de Codigos de Error por Dominio

Cada codigo se compone de un **prefijo de dominio** (3 letras) + guion + **numero secuencial** de 3 digitos. El catalogo es dinamico y se ampliara con nuevos modulos.

### 5.1 Prefijos por Dominio

| Prefijo | Dominio | Alcance |
|---------|---------|---------|
| VLD | Validacion | Estructura, formato, campos requeridos en la capa de interoperabilidad |
| BIZ | Negocio General | Errores genericos de logica de negocio |
| CTR | Catastro | Dominio catastral, predios, mutaciones, LADM-COL |
| GEO | Geoespacial | Geometrias, PostGIS, sistemas de referencia |
| SEC | Seguridad | Autenticacion y autorizacion (Keycloak/OIDC) |
| INT | Integracion | Comunicacion con servicios externos (IGAC, SNR, etc.) |
| SYS | Sistema | Infraestructura, base de datos, errores internos |
| URB | Urbanismo | Licencias urbanisticas, uso de suelo *(futuro)* |
| HAC | Hacienda | Impuesto predial, facturacion tributaria *(futuro)* |

### 5.2 Validacion (VLD)

| Codigo | HTTP | Mensaje |
|--------|------|---------|
| VLD-001 | 400 | El campo {campo} es obligatorio. |
| VLD-002 | 400 | El campo {campo} no cumple con el formato requerido. |
| VLD-003 | 400 | El valor del campo {campo} excede la longitud permitida. |
| VLD-004 | 400 | El campo {campo} contiene caracteres no permitidos. |
| VLD-005 | 400 | El campo {campo} debe ser un valor numerico. |
| VLD-006 | 400 | El campo {campo} debe ser una fecha valida en formato ISO 8601. |
| VLD-007 | 400 | El body de la solicitud esta vacio o es invalido. |
| VLD-008 | 400 | El parametro de ruta {param} es obligatorio. |
| VLD-009 | 400 | El parametro de consulta {param} no es valido. |

### 5.3 Negocio General (BIZ)

| Codigo | HTTP | Mensaje |
|--------|------|---------|
| BIZ-001 | 409 | La operacion presento un error durante la ejecucion. |
| BIZ-002 | 409 | El registro solicitado no fue encontrado. |
| BIZ-003 | 409 | El registro ya existe y no puede ser duplicado. |
| BIZ-004 | 409 | El estado actual del registro no permite esta operacion. |
| BIZ-005 | 409 | Se han realizado demasiadas solicitudes. Intente mas tarde. |

### 5.4 Catastro (CTR)

| Codigo | HTTP | Mensaje |
|--------|------|---------|
| CTR-001 | 409 | El Numero Predial Nacional (NPN) no cumple con el formato IGAC. |
| CTR-002 | 409 | El predio no se encuentra registrado en el sistema catastral. |
| CTR-003 | 409 | El predio se encuentra en estado de embargo. |
| CTR-004 | 409 | La informacion del avaluo catastral no se encuentra vigente. |
| CTR-005 | 409 | El tipo de mutacion catastral no es valido para el predio. |
| CTR-006 | 409 | El predio no cuenta con geometria registrada. |

### 5.5 Geoespacial (GEO)

| Codigo | HTTP | Mensaje |
|--------|------|---------|
| GEO-001 | 409 | La geometria proporcionada no es valida segun estandar OGC. |
| GEO-002 | 409 | La geometria esta fuera de los limites del municipio. |
| GEO-003 | 409 | Error en la transformacion del sistema de referencia de coordenadas. |
| GEO-004 | 409 | La geometria presenta auto-interseccion. |

### 5.6 Seguridad (SEC)

| Codigo | HTTP | Mensaje |
|--------|------|---------|
| SEC-001 | 401 | No se encuentra autorizado para ejecutar la operacion. |
| SEC-002 | 403 | Acceso denegado para ejecutar la operacion. |
| SEC-003 | 401 | El token de autenticacion ha expirado. |
| SEC-004 | 403 | El usuario no tiene el rol requerido para este recurso. |

### 5.7 Integracion (INT)

| Codigo | HTTP | Mensaje |
|--------|------|---------|
| INT-001 | 502 | El servicio externo {servicio} no se encuentra disponible. |
| INT-002 | 504 | Timeout en la comunicacion con el servicio {servicio}. |
| INT-003 | 502 | El servicio externo {servicio} retorno una respuesta inesperada. |
| INT-004 | 502 | Error de conexion con el servicio {servicio}. |

### 5.8 Sistema (SYS)

| Codigo | HTTP | Mensaje |
|--------|------|---------|
| SYS-001 | 500 | Se presento un error tecnico durante el procesamiento de la solicitud. |
| SYS-002 | 503 | El servicio no esta disponible temporalmente. |
| SYS-003 | 504 | Timeout. No se obtuvo una respuesta a tiempo del servidor. |
| SYS-004 | 500 | Error de conexion con la base de datos. |
| SYS-005 | 500 | Error inesperado en la logica interna del microservicio. |

---

## 6. Uso del Adaptador del Framework

El adaptador definido en el framework de VIP actua como intermediario para estandarizar y gestionar la estructura de errores entre la capa de interoperabilidad y los backends.

### 6.1 Responsabilidades del Adaptador

- Categorizar el `Tipo_Error` segun el origen de la respuesta del backend.
- Asignar el `Codigo_Error` correspondiente del catalogo por dominio.
- Traducir mensajes tecnicos o no dicientes del backend a mensajes claros para el consumidor.
- Identificar el `Origen` del error (nombre del microservicio o adaptador que lo detecto).
- Garantizar que errores de tipo `SISTEMA` nunca expongan detalles de infraestructura, trazas de excepciones o informacion sensible.
- Mapear los codigos HTTP del backend al `Http_Estado` estandar de VIP.

### 6.2 Flujo de Gestion de Errores

1. El microservicio recibe la solicitud y ejecuta validaciones de estructura y formato. Si falla, retorna error tipo `VALIDACION`.
2. Si la validacion es exitosa, se invoca el backend o servicio externo correspondiente.
3. El adaptador recibe la respuesta del backend y evalua si es exitosa o contiene errores.
4. Si el backend retorna un error, el adaptador lo clasifica (`NEGOCIO`, `SEGURIDAD`, `INTEGRACION` o `SISTEMA`), le asigna un `Codigo_Error` del catalogo y traduce el mensaje si es necesario.
5. Se construye la respuesta estandar con la estructura del Canonical y se retorna al consumidor.

---

## 7. Consideraciones Generales

- La nomenclatura de los campos sigue la convencion **Snake_Case con mayuscula inicial**, alineada con el modelo LADM-COL.
- El catalogo de errores es dinamico y se actualizara a medida que se integren nuevos modulos.
- Los mensajes de error deben ser claros y orientados al consumidor del servicio, evitando exponer detalles tecnicos internos.
- El campo `Ruta` utiliza la notacion **JSON Pointer** para permitir que el frontend identifique exactamente el campo con error.
- El campo `Origen` permite trazabilidad de extremo a extremo en una arquitectura con multiples microservicios e integraciones.
- El `Http_Estado` se incluye en el body de la respuesta ademas del header HTTP para soportar escenarios de comunicacion asincronica (colas RabbitMQ, eventos) donde se pierde el header.
- Los prefijos `URB` (Urbanismo) y `HAC` (Hacienda) estan reservados para futuras integraciones.
