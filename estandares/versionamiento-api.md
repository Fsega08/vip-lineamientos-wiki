# Lineamientos de Versionamiento de API

[Volver al indice](/)

---

## 1. Objetivo

Definir la estrategia de versionamiento para las APIs de la plataforma VIP, garantizando compatibilidad hacia atras y transiciones controladas entre versiones.

---

## 2. Estrategia: Versionamiento por Dominio en el Path

Las APIs de VIP usan **versionamiento por dominio en el path** del URL. La version va despues del dominio, permitiendo que cada dominio evolucione de forma independiente:

```
/{prefijo}/{dominio}/{version}/{recurso}
```

Ejemplos:

```
/vip/catastro/v1/predios
/vip/catastro/v2/predios
/vip/geo/v1/geometrias
```

> Catastro puede estar en v2 mientras geo sigue en v1. Cada dominio versiona a su propio ritmo.

### Por que version por dominio y no global

- Cada dominio (catastro, geo, urbanismo, hacienda) evoluciona a ritmos diferentes.
- Un breaking change en mutaciones catastrales no debe forzar nueva version en geo.
- En Kong se rutea por dominio — un route por `/{prefijo}/{dominio}/v{n}/` es mas natural.
- Los consumidores que solo usan un dominio no se ven afectados por cambios en otros.

### Por que path y no header

- Es visible en logs, Kong routes y Swagger sin necesidad de inspeccionar headers.
- Compatible con Kong routing nativo (un route por version).
- Los consumidores ven explicitamente que version estan usando.

---

## 3. Reglas de Versionamiento

### 3.1 Cuando crear una nueva version

| Cambio | Nueva version? | Ejemplo |
|--------|:--------------:|---------|
| Agregar campo **opcional** al response | No | Agregar `Fecha_Actualizacion` al response |
| Agregar campo **opcional** al request | No | Agregar filtro opcional `Tipo_Predio` |
| Eliminar campo del response | **Si** | Quitar `Codigo_Postal` |
| Cambiar tipo de dato de un campo | **Si** | Cambiar `Avaluo` de string a number |
| Cambiar nombre de un campo | **Si** | Renombrar `Nro_Predial` a `Numero_Predial` |
| Cambiar estructura del Canonical | **Si** | Reestructurar `Datos_Operacion` |
| Agregar nuevo endpoint | No | Agregar `/vip/catastro/v1/predios/historico` |
| Cambiar comportamiento de endpoint existente | **Si** | Cambiar logica de consulta que retorna datos diferentes |

### 3.2 Regla de compatibilidad

- **Agregar** campos opcionales al request o response → compatible, no requiere nueva version.
- **Eliminar, renombrar o cambiar tipo** de un campo existente → breaking change, requiere nueva version.

### 3.3 Convivencia de versiones

- Se soportan maximo **2 versiones simultaneas** por dominio (actual + anterior).
- La version anterior se mantiene por **6 meses** despues del lanzamiento de la nueva.
- Se debe notificar a los consumidores con minimo **30 dias** de anticipacion antes de deprecar una version.

---

## 4. Configuracion en Kong

Cada version se registra como un route independiente apuntando al mismo service o a un service diferente:

| Route | Service | Version |
|-------|---------|---------|
| `/vip/catastro/v1/predios` | vip-catastro-service-consulta | v1 |
| `/vip/catastro/v2/predios` | vip-catastro-service-consulta-v2 | v2 |
| `/vip/geo/v1/geometrias` | vip-geo-service-geometrias | v1 |

Si la nueva version es compatible internamente (misma logica, diferente contrato), el microservicio puede manejar ambas versiones con un path mapping interno.

---

## 5. Versionamiento de Imagenes Docker

Las imagenes Docker siguen **Semantic Versioning** (semver):

```
registry.vortexbird.com/vip/vip-catastro-service-consulta:1.2.3
```

| Segmento | Significado | Ejemplo |
|----------|-------------|---------|
| Major | Breaking change en la API | 1.x.x → 2.0.0 |
| Minor | Nuevo feature compatible | 1.1.x → 1.2.0 |
| Patch | Bug fix | 1.2.0 → 1.2.1 |

- En **dev**: tags mutables (se puede sobreescribir `1.2.3`).
- En **QA y prod**: tags **inmutables** (una vez publicado, no se modifica).
- Nunca usar `latest` en QA o produccion.

---

## 6. Headers de Respuesta

Toda respuesta de VIP incluye headers informativos de version:

| Header | Valor | Ejemplo |
|--------|-------|---------|
| `X-API-Version` | Version del endpoint | `v1` |
| `X-Service-Version` | Version del microservicio (semver) | `1.2.3` |

Estos headers ayudan a diagnosticar problemas cuando hay multiples versiones conviviendo.
