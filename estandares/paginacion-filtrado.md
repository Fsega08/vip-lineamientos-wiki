# Lineamientos de Paginacion y Filtrado

[Volver al indice](/)

---

## 1. Objetivo

Definir el estandar de paginacion y filtrado para los endpoints de consulta de la plataforma VIP, garantizando respuestas predecibles y eficientes.

---

## 2. Paginacion

### 2.1 Estrategia: Offset / Limit

VIP usa paginacion basada en **offset y limit**, por ser la mas compatible con los backends relacionales (PostgreSQL) y predecible para los consumidores.

### 2.2 Campos de Paginacion en el Request

Los campos de paginacion van dentro de `Datos_Operacion`:

```json
{
  "Encabezado": { ... },
  "Datos_Operacion": {
    "Pagina": 1,
    "Tamanio_Pagina": 20,
    "Ordenar_Por": "Numero_Predial",
    "Direccion_Orden": "ASC"
  }
}
```

| Campo | Tipo | Obligatorio | Default | Descripcion |
|-------|------|:-----------:|---------|-------------|
| `Pagina` | Integer | No | 1 | Numero de pagina (inicia en 1) |
| `Tamanio_Pagina` | Integer | No | 20 | Registros por pagina |
| `Ordenar_Por` | String | No | Varia por servicio | Campo por el cual ordenar |
| `Direccion_Orden` | String | No | `ASC` | `ASC` o `DESC` |

### 2.3 Limites

| Parametro | Valor |
|-----------|-------|
| `Tamanio_Pagina` minimo | 1 |
| `Tamanio_Pagina` maximo | 100 |
| `Tamanio_Pagina` default | 20 |

Si el consumidor envia un `Tamanio_Pagina` mayor a 100, el microservicio debe retornar error `VLD-003` (excede longitud permitida).

### 2.4 Campos de Paginacion en el Response

```json
{
  "Id_Mensaje": "...",
  "Http_Estado": 200,
  "Errores": [],
  "Datos_Operacion": {
    "Paginacion": {
      "Pagina_Actual": 1,
      "Tamanio_Pagina": 20,
      "Total_Registros": 153,
      "Total_Paginas": 8
    },
    "Registros": [
      { "Numero_Predial": "05001010203040000000", "Direccion": "Calle 50 # 45-20" },
      { "Numero_Predial": "05001010203040000001", "Direccion": "Carrera 30 # 10-15" }
    ]
  }
}
```

| Campo | Tipo | Descripcion |
|-------|------|-------------|
| `Paginacion.Pagina_Actual` | Integer | Pagina retornada |
| `Paginacion.Tamanio_Pagina` | Integer | Registros por pagina usados |
| `Paginacion.Total_Registros` | Integer | Total de registros que coinciden con el filtro |
| `Paginacion.Total_Paginas` | Integer | Total de paginas disponibles |
| `Registros` | Array | Lista de resultados de la pagina actual |

---

## 3. Filtrado

### 3.1 Campos de Filtro en el Request

Los filtros van dentro de `Datos_Operacion` como campos del servicio. No se usa query string — todo va en el body siguiendo el Canonical.

```json
{
  "Datos_Operacion": {
    "Numero_Predial": "0500101020304%",
    "Tipo_Predio": "Urbano",
    "Pagina": 1,
    "Tamanio_Pagina": 20
  }
}
```

### 3.2 Reglas de Filtrado

- Los filtros son **opcionales**. Si no se envian, se retornan todos los registros (paginados).
- Los campos de tipo String soportan busqueda parcial con `%` (like). Ejemplo: `"Numero_Predial": "050010%"`.
- Los campos de tipo Date soportan rangos con campos `_Desde` / `_Hasta`. Ejemplo: `"Fecha_Avaluo_Desde": "2025-01-01"`, `"Fecha_Avaluo_Hasta": "2026-01-01"`.
- Los filtros se combinan con **AND** logico.

### 3.3 Respuesta Vacia

Si el filtro no retorna resultados, el response es exitoso (HTTP 200) con `Registros` vacio:

```json
{
  "Http_Estado": 200,
  "Errores": [],
  "Datos_Operacion": {
    "Paginacion": {
      "Pagina_Actual": 1,
      "Tamanio_Pagina": 20,
      "Total_Registros": 0,
      "Total_Paginas": 0
    },
    "Registros": []
  }
}
```

> **Importante:** una consulta sin resultados **NO es un error**. No se debe retornar HTTP 404 ni error `BIZ-002` para consultas vacias. El 404 y BIZ-002 son para cuando se busca un registro especifico por ID y no existe.

---

## 4. Consideraciones

- La paginacion es obligatoria en todo endpoint que retorne listas. No se permiten endpoints que retornen todos los registros sin paginacion.
- El `Total_Registros` puede omitirse si el conteo es costoso (ej: consultas geoespaciales). En ese caso, se retorna `null` y el consumidor navega con "pagina siguiente / anterior".
- El cache de Redis debe incluir los parametros de paginacion y filtrado en el `hash_request` de la key. Ejemplo: `vip:catastro:consulta:{hash(filtros+pagina+tamanio)}`.
