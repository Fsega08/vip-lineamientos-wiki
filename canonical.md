# Canonical de Interoperabilidad

[Volver al indice](index.md)

---

> **Nota:** aunque los campos sean opcionales, la estructura siempre debe enviarse completa y la informacion del campo vacia.

## 1. Estructura de Solicitud (Request)

| No. | Campo Espanol | Campo Ingles | Descripcion | Obligatorio |
|-----|---------------|--------------|-------------|:-----------:|
| — | **Encabezado.\*** | header.\* | Informacion esencial para identificar y autenticar la transaccion | Si |
| 1 | Encabezado.Id_Mensaje | header.messageId | UUID unico por transaccion | Si |
| 2 | Encabezado.Fecha_Hora_Solicitud | header.timestamp | Fecha y hora en formato ISO8601 (`YYYY-MM-DDTHH:MM:SS`) | Si |
| 3 | Encabezado.Token_Autenticacion | header.authToken | Token de autenticacion (Keycloak) | Si |
| 4 | Encabezado.Url_Retorno | header.callbackUrl | Callback URL para respuesta asincrona | No |
| 5 | Encabezado.Tipo_Solicitud | header.requestType | Tipo de solicitud: `transfer`, `consulta`, `creacion`, `actualizacion`, `eliminacion` | Si |
| 6 | Encabezado.Id_Transaccion_Origen | header.transactionId | Id de seguimiento del sistema origen | No |
| 7 | Encabezado.Usuario_Login | header.userLogin | Usuario conectado o codigo de sistema consumidor | Si |
| — | **Dispositivo.\*** | deviceContext.\* | Informacion sobre el dispositivo origen | Si |
| 8 | Dispositivo.Id_Dispositivo | deviceContext.deviceId | Identificador unico del dispositivo | No |
| 9 | Dispositivo.Tipo_Dispositivo | deviceContext.deviceType | Tipo: `Mobile`, `Desktop` | No |
| 10 | Dispositivo.Version_Sistema_Operativo | deviceContext.osVersion | Version del SO | No |
| 11 | Dispositivo.Direccion_Ip | deviceContext.ipAddress | Direccion IP del dispositivo | Si |
| 12 | Dispositivo.Direccion_Mac | deviceContext.macAddress | Direccion MAC del dispositivo | No |
| — | **Geolocalizacion.\*** | deviceContext.geoLocation.\* | Ubicacion fisica del dispositivo | No |
| 13 | Geolocalizacion.Longitud | deviceContext.geoLocation.longitude | Coordenada de longitud | No |
| 14 | Geolocalizacion.Latitud | deviceContext.geoLocation.latitude | Coordenada de latitud | No |
| — | **Aplicacion_Cliente.\*** | clientApp.\* | Informacion de la aplicacion cliente | No |
| 15 | Aplicacion_Cliente.Nombre_Aplicacion | clientApp.appName | Nombre de la aplicacion | No |
| 16 | Aplicacion_Cliente.Version_Aplicacion | clientApp.appVersion | Version de la aplicacion | No |
| 17 | Aplicacion_Cliente.Agente_Usuario | clientApp.userAgent | User agent del cliente HTTP | No |
| — | **Datos_Operacion.\*** | operationData.\* | Datos especificos de cada operacion (estructura propia por servicio) | Si |

### Ejemplo de Request

```json
{
  "Encabezado": {
    "Id_Mensaje": "550e8400-e29b-41d4-a716-446655440000",
    "Fecha_Hora_Solicitud": "2026-04-06T08:30:00-05:00",
    "Token_Autenticacion": "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9",
    "Url_Retorno": "https://api.cliente.com/callback/respuesta",
    "Tipo_Solicitud": "Consulta",
    "Id_Transaccion_Origen": "TRX-20260406-000123",
    "Usuario_Login": "usuario.prueba"
  },
  "Dispositivo": {
    "Id_Dispositivo": "DEV-ABC-123456",
    "Tipo_Dispositivo": "desktop",
    "Version_Sistema_Operativo": "Windows 11",
    "Direccion_Ip": "192.168.1.10",
    "Direccion_Mac": "00:1A:2B:3C:4D:5E"
  },
  "Geolocalizacion": {
    "Longitud": "-76.5320",
    "Latitud": "3.4516"
  },
  "Aplicacion_Cliente": {
    "Nombre_Aplicacion": "Portal_Operaciones",
    "Version_Aplicacion": "1.0.0",
    "Agente_Usuario": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"
  },
  "Datos_Operacion": {}
}
```

---

## 2. Estructura de Respuesta (Response)

| No. | Campo Espanol | Campo Ingles | Descripcion | Obligatorio |
|-----|---------------|--------------|-------------|:-----------:|
| 1 | Id_Mensaje | messageId | UUID que enlaza la respuesta con la solicitud original | Si |
| 2 | Fecha_Respuesta | timestamp | Marca de tiempo de la respuesta en formato ISO 8601 | Si |
| 3 | Http_Estado | status | Codigo HTTP: 200, 400, 401, 403, 409, 500, 502, 503, 504 | Si |
| 4 | Errores[] | errors[] | Lista de errores. Vacia (`[]`) si Http_Estado es 200 | Si |
| 5 | Errores[].Tipo_Error | errorType | `VALIDACION` \| `NEGOCIO` \| `SEGURIDAD` \| `INTEGRACION` \| `SISTEMA` | Cond. |
| 6 | Errores[].Codigo_Error | errorCode | Codigo del catalogo por dominio (ej: VLD-001, CTR-003) | Cond. |
| 7 | Errores[].Ruta | path | JSON Pointer al campo con error. `null` si no aplica | Cond. |
| 8 | Errores[].Mensaje | message | Descripcion clara del error orientada al consumidor | Cond. |
| 9 | Errores[].Origen | errorOrigin | Microservicio o adaptador que origino el error | Cond. |
| 10 | Datos_Operacion.\* | operationData.\* | Datos de respuesta especificos del servicio. `{}` si hay error | Si |

### Ejemplo de Respuesta Exitosa

```json
{
  "Id_Mensaje": "550e8400-e29b-41d4-a716-446655440000",
  "Fecha_Respuesta": "2026-04-06T08:45:00-05:00",
  "Http_Estado": 200,
  "Errores": [],
  "Datos_Operacion": {
    "Id_Predio": "PRD-12345",
    "Numero_Predial": "05001010203040000000",
    "Direccion": "Calle 50 # 45-20"
  }
}
```

### Ejemplo de Respuesta con Error

```json
{
  "Id_Mensaje": "550e8400-e29b-41d4-a716-446655440000",
  "Fecha_Respuesta": "2026-04-06T08:45:00-05:00",
  "Http_Estado": 400,
  "Errores": [
    {
      "Tipo_Error": "VALIDACION",
      "Codigo_Error": "VLD-001",
      "Ruta": "/Datos_Operacion/Numero_Documento",
      "Mensaje": "El campo Numero_Documento es obligatorio.",
      "Origen": "vip-gateway"
    }
  ],
  "Datos_Operacion": {}
}
```
