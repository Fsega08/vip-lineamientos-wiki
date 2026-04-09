# Guia de Documentacion de Servicios

[Volver al indice](/)

---

> Lineamientos clave para la documentacion de servicios de interoperabilidad.
>
> *"El trabajo en equipo se fortalece cuando el conocimiento se comparte; una buena documentacion convierte el esfuerzo individual en crecimiento colectivo."*

---

## 1. Datos Generales del Servicio

### Nombre del Servicio

Identifica el servicio y permite distinguir si se trata de un desarrollo nuevo o un control de cambios.

### Descripcion del Servicio

Explica de manera general la finalidad y el funcionamiento del servicio.

---

## 2. Diagrama de Secuencia

Representa el flujo de interaccion entre los sistemas durante la ejecucion de un proceso.

**Tener en cuenta:**
- Reglas de negocio y caminos alternos.
- Reintentos y manejo de fallos del servicio.
- Consumo de catalogos y dependencias externas.

**Herramienta sugerida:** [sequencediagram.org](https://sequencediagram.org)

---

## 3. Acuerdos de Nivel de Servicio (SLA)

| Aspecto | Descripcion |
|---------|-------------|
| **Rendimiento** | Capacidad de procesar transacciones por segundo y mantener tiempos de respuesta optimos |
| **Concurrencia** | Manejo de multiples usuarios o procesos simultaneos sin errores |
| **Volumen de transacciones** | Capacidad de soportar altos picos de transacciones sin degradar el servicio |

---

## 4. Control de Cambios

En caso de tratarse de un control de cambios, se debe indicar claramente (de forma resaltada) que se solicita modificar, anadir o eliminar.

---

## 5. Nomenclatura de Campos

- Todos los campos de request y response deben nombrarse en base al formato **LADM_COL Version 4.1**, salvo indicacion explicita.
- Los nombres de los campos deben ser en **Espanol**, con **Snake_Case** coherentes y consistentes entre servicios.
- Si un campo representa el mismo concepto funcional, debe mantener el mismo nombre en todos los servicios.

**Ejemplo:**
```
Datos_Operacion.Tipo_Planta
→ Debe usarse de la misma forma en todos los servicios que hagan referencia al tipo de planta
```

---

## 6. Canonical

Por defecto, el canonical de interoperabilidad aplica a todos los servicios mediados. La mayoria de los campos del canonical son opcionales.

En la documentacion se debe indicar explicitamente si algun campo del canonical es **obligatorio** y requerido por el servicio.

Ver: [Canonical de Interoperabilidad](contratos/canonical.md)

---

## 7. Campos del Request — Validaciones por Tipo de Dato

| Tipo | Validacion |
|------|------------|
| **String** | Longitud minima y maxima |
| **Integer** | Rango de numeros minimos y maximos |
| **Date** | Formato de fecha `AAAA-MM-DD` (ISO 8601) |
| **Boolean** | No tienen validaciones adicionales |
| **Decimal** | Para la longitud contar la coma y los decimales |

### Tips importantes

- Verificar que los tipos de datos del request y response coincidan con los del servicio a mediar.
- **Campo no obligatorio ≠ campo vacio:**
  - No obligatorio permite enviar `null`.
  - Para permitir valores vacios (`""`), la longitud minima debe ser `0`.
- Los campos de tipo `Integer` **no aceptan valores vacios** (`""`).
- Los campos tipo `boolean` van en **minuscula** y sin comillas. Ejemplo: `true`, `false`.

### Array vs Objeto

Antes de documentar o consumir un servicio, verificar si el backend entrega la informacion como **Array** u **Objeto**.

Esta definicion debe respetarse y mantenerse en la documentacion y en la implementacion.

En BIAN, la identificacion es:
- **Array:** `OperationData.Array[].item`
- **Objeto:** `OperationData.Objeto.item`

> Una definicion incorrecta puede generar errores de consumo e interpretacion del servicio.

---

## 8. Campos del Response

- Los campos **no se validan** salvo que se solicite una transformacion o regla especifica.
- Por interoperabilidad, los datos se envian tal como los entrega el backend.
- Si un campo es opcional y el backend no lo retorna, lo enviamos como `null`, salvo indicacion para enviarlo como vacio o no enviarlo en la respuesta.

---

## 9. Consumos Backend

### Servicios Core o Backend

- Verificar si el cluster tiene alcance al servicio.
- Confirmar si el consumo requiere token de autenticacion.
- Validar si es necesario enviar campos adicionales al body del request.
- **Anadir el curl de consumo** (siempre, sin importar si es un control de cambios).

---

## 10. Manejo de Errores

- Definir que se considera error y que no lo es (Ej.: si el backend retorna vacio en una consulta, ¿se debe responder vacio o enviar error?).
- Definir si se requieren mensajes de error personalizados.
- Definir si son necesarios reintentos y/o reversos ante fallos.

Ver detalle completo: [Manejo de Errores](contratos/manejo-errores.md)

---

## 11. Data de Prueba

- La data debe permitir validar **todos los caminos** del servicio.
- La data debe incluir **todos los campos** del servicio, incluso los no obligatorios.
- Si el servicio es de consumo unico, se debe proporcionar suficiente data para desarrollar y validar su funcionamiento.
- En servicios que implican transferencias, la data debe contar con **fondos suficientes** para evitar reprocesos durante el desarrollo.

---

## 12. No Conformidades — Como Reportar una NC

Al reportar una No Conformidad (NC), incluir:

1. **Nombre del servicio** en el titulo y resumen del caso.
2. **Enlace a la documentacion** en SharePoint.
3. **Curl utilizado** donde se evidencio la novedad (Data).
4. **Descripcion del caso:**
   - Accion realizada
   - Resultado obtenido
   - Resultado esperado
