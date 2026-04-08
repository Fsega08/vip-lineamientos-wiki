# Estandar de Manejo de Cache — Redis

[Volver al indice](index.md)

---

## 1. Objetivo

Definir las estrategias, convenciones de nombramiento y politicas de Time-To-Live (TTL) para el uso de Redis como capa de cache dentro de la plataforma VIP. Este estandar busca optimizar el rendimiento de los microservicios, reducir la carga sobre backends y servicios externos, y garantizar la consistencia de datos en toda la plataforma.

---

## 2. Convencion de Nombramiento de Keys

### 2.1 Patron

```
{prefijo}:{dominio}:{entidad}:{identificador}
```

El separador dos puntos (`:`) es el estandar de Redis, permite agrupar y visualizar jerarquicamente las keys en herramientas como Redis Commander o Redis Insight.

### 2.2 Reglas

- Todo en **minusculas**, separado por **dos puntos** (`:`).
- El prefijo es configurable por proyecto.
- El dominio corresponde a los dominios definidos en el estandar de nombramiento (SF-NM).
- La entidad identifica el tipo de dato cacheado.
- El identificador es la llave natural o UUID del registro.
- **No almacenar datos sensibles sin cifrar** (documentos de identidad, claves, tokens de larga duracion).
- **Toda key debe tener un TTL definido.** No se permiten keys sin expiracion.

### 2.3 Ejemplos de Keys

| Key | Descripcion | TTL |
|-----|-------------|-----|
| `vip:catastro:predio:{npn}` | Datos del predio por NPN (Numero Predial Nacional) | 1 hora |
| `vip:catastro:avaluo:{id_predio}` | Avaluo catastral vigente | 24 horas |
| `vip:geo:municipio:{cod_municipio}:limites` | Geometria de limites del municipio | 7 dias |
| `vip:parametros:tipos-mutacion` | Lista de tipos de mutacion catastral | 24 horas |
| `vip:parametros:departamentos` | Lista de departamentos de Colombia | 30 dias |
| `vip:parametros:ciudades:{cod_depto}` | Ciudades de un departamento | 30 dias |
| `vip:catastro:consulta:{hash_request}` | Resultado cacheado de consulta frecuente | 30 min |

---

## 3. Politicas de TTL por Categoria

El TTL define el tiempo maximo que un dato permanece en cache antes de expirar automaticamente.

| Categoria | TTL | Descripcion |
|-----------|-----|-------------|
| Datos parametricos | 24h - 30 dias | Departamentos, ciudades, tipos de mutacion, catalogos. Cambian muy poco, alta frecuencia de consulta |
| Datos de consulta frecuente | 30 min - 4 horas | Predios, avaluos, resultados de busqueda. Balance entre frescura y carga al backend |
| Datos geoespaciales estaticos | 7 dias | Limites de municipio, capas base, poligonos de referencia. Practicamente no cambian |
| Datos transaccionales | **NO CACHEAR** | Mutaciones en proceso, liquidaciones activas, cualquier dato que requiera consistencia transaccional fuerte |

### Configuracion en application.properties

```properties
cache.redis.ttl.parametricos: 86400        # 24 horas (segundos)
cache.redis.ttl.parametricos-largo: 604800  # 7 dias
cache.redis.ttl.sesion: 900                 # 15 minutos
cache.redis.ttl.consulta: 3600              # 1 hora
cache.redis.ttl.geoespacial: 604800         # 7 dias
```

---

## 4. Estrategias de Invalidacion de Cache

### 4.1 TTL Pasivo (Expiracion Automatica)

La key expira automaticamente al cumplir su TTL. El siguiente request que necesite el dato lo consultara al backend y lo recacheara.

**Aplica para:**
- Datos parametricos (departamentos, ciudades, catalogos).
- Datos geoespaciales estaticos (limites de municipio).
- Cualquier dato donde un desfase de minutos u horas sea aceptable.

### 4.2 Invalidacion Activa por Evento

Cuando un microservicio modifica un dato, publica un evento en RabbitMQ. El servicio que cacheo el dato escucha el evento y borra la key correspondiente.

**Ejemplo:**

1. El microservicio `vip-catastro-service-mutacion` actualiza un predio.
2. Publica evento en routing key: `vip.catastro.predio.actualizado`
3. El microservicio `vip-catastro-service-consulta` escucha el evento.
4. Borra la key `vip:catastro:predio:{npn}` de Redis.
5. El siguiente request consultara el dato fresco al backend y lo recacheara.

**Aplica para:**
- Datos de consulta frecuente modificados por otros microservicios.
- Predios, avaluos, datos de intervinientes.
- Cualquier dato donde la consistencia debe ser cercana al tiempo real.

### 4.3 Cache-Aside con Lock (Thundering Herd)

Cuando una key expira y multiples requests simultaneos intentan consultar el mismo dato, se utiliza un **lock distribuido con SETNX** de Redis para que solo un request consulte al backend y los demas esperen el resultado. Evita sobrecargar el backend.

**Aplica para:**
- Consultas geoespaciales costosas (PostGIS).
- Datos que se consultan masivamente y son costosos de obtener.
- Endpoints con alta concurrencia.

---

## 5. Datos que NO se Deben Cachear

| Tipo de Dato | Razon |
|--------------|-------|
| Datos transaccionales activos | Mutaciones en proceso, liquidaciones tributarias activas, cualquier operacion que requiera consistencia transaccional fuerte (ACID) |
| Datos financieros en proceso | Calculos de impuesto predial en curso, pagos pendientes de confirmacion, saldos en proceso de actualizacion |
| Respuestas de escritura | Resultados de operaciones POST, PUT, DELETE. Solo se cachean respuestas de consulta (GET) |
| Datos sensibles sin cifrar | Numeros de documento de identidad, informacion personal protegida por ley de habeas data, tokens de larga duracion |
| Datos con cambio frecuente | Contadores en tiempo real, datos de geolocalizacion en movimiento, estados de proceso que cambian cada segundo |

---

## 6. Monitoreo y Observabilidad

| Metrica | Descripcion | Criterio |
|---------|-------------|----------|
| Hit Rate | Porcentaje de requests resueltos desde cache | Debe mantenerse por encima del **80%** |
| Memory Usage | Memoria utilizada por Redis | Configurar `maxmemory-policy` como `allkeys-lru` |
| Eviction Count | Keys eliminadas por presion de memoria | Debe ser cercano a **cero** en condiciones normales |
| Key Count | Numero total de keys activas | Permite dimensionar la instancia y detectar fugas |
| Latency | Tiempo de respuesta de Redis | Debe mantenerse por debajo de **1ms** |

---

## 7. Consideraciones Generales

- Toda key de Redis debe tener un TTL configurado. No se permiten keys permanentes.
- La convencion de keys usa el mismo prefijo configurable definido en el estandar SF-NM.
- Se recomienda **Redis Cluster** para produccion con alta disponibilidad.
- La serializacion debe usar **JSON** (no serializacion binaria de Java) para facilitar depuracion.
- En Spring Boot, configurar **Spring Cache con RedisCacheManager** y los TTL por categoria.
- La invalidacion activa por evento requiere que los productores publiquen en RabbitMQ.
- Dev y QA pueden compartir instancia de Redis con prefijos diferentes. Produccion usa instancia dedicada.
