# Template: Documentacion de Servicio

[Volver al indice](/) | [Guia de Documentacion](guias/guia-documentacion.md)

---

> **Instrucciones:** Copiar este template y llenar cada seccion al documentar un servicio nuevo o un control de cambios. Las secciones marcadas con (*) son obligatorias.

---

## 1. Datos Generales (*)

| Campo | Valor |
|-------|-------|
| **Nombre del servicio** | `vip-{namespace}-{componente}-{funcionalidad}` |
| **Tipo** | Nuevo desarrollo / Control de cambios |
| **Dominio** | Catastro / Geoespacial / Urbanismo / Hacienda / Plataforma |
| **Responsable funcional** | Nombre |
| **Fecha** | YYYY-MM-DD |
| **Version** | 1.0 |

### Descripcion (*)

*Explicar de manera general la finalidad y el funcionamiento del servicio.*

---

## 2. Diagrama de Secuencia (*)

*Incluir diagrama generado en [sequencediagram.org](https://sequencediagram.org) que muestre:*
- *Flujo principal (camino feliz)*
- *Caminos alternos y reglas de negocio*
- *Reintentos y manejo de fallos*
- *Consumo de catalogos y dependencias externas*

```
(Pegar imagen o link al diagrama)
```

---

## 3. Request (*)

### 3.1 Endpoint

| Metodo | Ruta | Descripcion |
|--------|------|-------------|
| POST | `/vip/{dominio}/v1/{recurso}` | Descripcion de la operacion |

### 3.2 Campos de Datos_Operacion

| No. | Campo | Tipo | Obligatorio | Min | Max | Descripcion |
|-----|-------|------|:-----------:|:---:|:---:|-------------|
| 1 | Campo_Ejemplo | String | Si | 1 | 50 | Descripcion del campo |
| 2 | Otro_Campo | Integer | No | 0 | 999999 | Descripcion del campo |
| 3 | Fecha_Ejemplo | Date | Si | — | — | Formato ISO 8601 (YYYY-MM-DD) |
| 4 | Activo | Boolean | No | — | — | `true` / `false` |

> **Recordar:** verificar tipos de datos contra el servicio backend a mediar. Ver [tips de campos](guias/guia-documentacion.md).

### 3.3 Ejemplo de Request Completo

```json
{
  "Encabezado": {
    "Id_Mensaje": "550e8400-e29b-41d4-a716-446655440000",
    "Fecha_Hora_Solicitud": "2026-04-06T08:30:00-05:00",
    "Url_Retorno": "",
    "Tipo_Solicitud": "consulta",
    "Id_Transaccion_Origen": "",
    "Usuario_Login": "usuario.prueba"
  },
  "Dispositivo": {
    "Id_Dispositivo": "",
    "Tipo_Dispositivo": "desktop",
    "Version_Sistema_Operativo": "",
    "Direccion_Ip": "192.168.1.10",
    "Direccion_Mac": ""
  },
  "Geolocalizacion": { "Longitud": "", "Latitud": "" },
  "Aplicacion_Cliente": {
    "Nombre_Aplicacion": "Portal_Operaciones",
    "Version_Aplicacion": "1.0.0",
    "Agente_Usuario": ""
  },
  "Datos_Operacion": {
    "Campo_Ejemplo": "valor",
    "Otro_Campo": 12345
  }
}
```

---

## 4. Response (*)

### 4.1 Campos de Datos_Operacion (Response)

| No. | Campo | Tipo | Descripcion |
|-----|-------|------|-------------|
| 1 | Campo_Respuesta | String | Descripcion |
| 2 | Otro_Dato | Integer | Descripcion |

> Los campos del response no se validan salvo que se solicite transformacion. Los datos se envian tal como los entrega el backend.

### 4.2 Ejemplo de Response Exitoso

```json
{
  "Id_Mensaje": "550e8400-e29b-41d4-a716-446655440000",
  "Fecha_Respuesta": "2026-04-06T08:45:00-05:00",
  "Http_Estado": 200,
  "Errores": [],
  "Datos_Operacion": {
    "Campo_Respuesta": "valor",
    "Otro_Dato": 67890
  }
}
```

---

## 5. Consumos Backend (*)

### 5.1 Servicio(s) a consumir

| Backend | URL / Endpoint | Auth | Metodo |
|---------|---------------|:----:|--------|
| Nombre del backend | URL o ruta | Si/No | POST/GET |

### 5.2 Curl de Consumo (*)

```bash
curl -X POST 'https://backend.example.com/api/endpoint' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer {token}' \
  -d '{
    "campo": "valor"
  }'
```

> **Importante:** siempre incluir el curl, incluso si es un control de cambios.

### 5.3 Verificaciones previas

- [ ] El cluster tiene alcance al servicio backend
- [ ] Se confirmo si requiere token de autenticacion
- [ ] Se valido si se necesitan campos adicionales al body

---

## 6. Manejo de Errores (*)

### 6.1 Definicion de errores

| Escenario | Es error? | Tipo_Error | Codigo | Mensaje |
|-----------|:---------:|------------|--------|---------|
| Backend retorna vacio | No / Si | — / NEGOCIO | — / BIZ-002 | — / El registro no fue encontrado |
| Campo X invalido | Si | VALIDACION | VLD-002 | El campo X no cumple con el formato requerido |
| Backend no disponible | Si | INTEGRACION | INT-001 | El servicio externo no se encuentra disponible |

### 6.2 Mensajes personalizados

*Indicar si se requieren mensajes de error diferentes a los del [catalogo estandar](contratos/manejo-errores.md).*

### 6.3 Reintentos y reversos

*Indicar si se requieren reintentos ante fallos del backend y/o acciones de reverso.*

---

## 7. Acuerdos de Nivel de Servicio (SLA)

| Aspecto | Valor |
|---------|-------|
| Tiempo de respuesta esperado | < X segundos |
| Concurrencia esperada | X usuarios simultaneos |
| Volumen estimado | X transacciones/dia |

---

## 8. Data de Prueba (*)

*La data debe permitir validar todos los caminos del servicio, incluyendo campos no obligatorios.*

| Escenario | Datos Clave | Resultado Esperado |
|-----------|-------------|-------------------|
| Camino feliz | Campo_Ejemplo = "ABC123" | HTTP 200, datos completos |
| Campo invalido | Campo_Ejemplo = "" | HTTP 400, VLD-001 |
| Backend sin datos | Campo_Ejemplo = "NOEXISTE" | HTTP 200, Registros: [] (o HTTP 409, BIZ-002) |
| Backend no disponible | — (simular timeout) | HTTP 502, INT-001 |

---

## 9. Control de Cambios

*Solo aplica si es un control de cambios. Indicar claramente que se modifica, agrega o elimina.*

| Cambio | Tipo | Descripcion |
|--------|------|-------------|
| Campo_Nuevo en request | Agregar | Nuevo campo opcional para filtrar por tipo |
| Campo_Viejo en response | Eliminar | Se remueve campo deprecado |
