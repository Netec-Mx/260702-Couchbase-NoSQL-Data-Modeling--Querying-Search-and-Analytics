---LAB_START---
LAB_ID: 15-00-01
---MARKDOWN---
# Uso de funciones y window functions en Analytics

## Metadatos

| Campo | Valor |
|---|---|
| **Duración estimada** | 70 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Aplicar (Apply) |
| **Servicio principal** | Couchbase Analytics |
| **Dataset requerido** | `travel-sample` |

---

## Descripción General

Este laboratorio explora las capacidades analíticas más avanzadas del servicio Analytics de Couchbase. En la primera parte se aplican funciones built-in de SQL++ para Analytics (cadenas, fechas, colecciones, condicionales) sobre el dataset `travel-sample` para transformar y enriquecer resultados analíticos. En la segunda parte se implementan Window Functions (`ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE`, `LAG`, `LEAD`, `FIRST_VALUE`, `LAST_VALUE`) y funciones de agregación con ventana (`SUM OVER`, `AVG OVER`) para análisis de rankings, series temporales y cálculos acumulativos. Finalmente, se analiza el plan de ejecución con `EXPLAIN` y se revisa la sintaxis de External Datasets y Remote Links.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio, el estudiante será capaz de:

- [ ] Aplicar funciones built-in avanzadas de Couchbase Analytics (`ARRAY_AGG`, `ARRAY_DISTINCT`, `DATE_DIFF_STR`, `REGEXP_CONTAINS`, `IFMISSING`, `OBJECT_KEYS`) para transformar documentos JSON en resultados analíticos enriquecidos.
- [ ] Implementar Window Functions (`ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE`, `LAG`, `LEAD`) con cláusulas `PARTITION BY` y `ORDER BY` para análisis de rankings y series temporales sobre colecciones `hotels` y `routes`.
- [ ] Utilizar funciones de agregación con ventana (`SUM OVER`, `AVG OVER`, `COUNT OVER`) con marcos de ventana `ROWS BETWEEN` para calcular totales y promedios acumulativos.
- [ ] Analizar el plan de ejecución de una consulta Analytics con la sentencia `EXPLAIN` e identificar oportunidades de optimización aplicando filtros tempranos y proyecciones reducidas.
- [ ] Describir la sintaxis de `CREATE EXTERNAL DATASET` y `CREATE LINK` para conectar Analytics con fuentes de datos externas y clústeres remotos.

---

## Prerrequisitos

### Conocimiento previo
- Haber completado la Práctica 14 o tener experiencia equivalente con Couchbase Analytics básico (Datasets, Dataverses, consultas SQL++ simples).
- Conocimiento sólido de funciones de agregación SQL estándar (`SUM`, `COUNT`, `AVG`, `MAX`, `MIN`).
- Comprensión básica de Window Functions de SQL estándar (se repasará en el laboratorio).
- Familiaridad con la estructura del dataset `travel-sample` (colecciones `hotels`, `routes`, `airports`, `airlines`, `landmarks`).

### Acceso y configuración requerida
- Couchbase Server 7.6.x en ejecución con el servicio **Analytics habilitado**.
- Dataset `travel-sample` cargado en el clúster (verificar en **Settings → Sample Buckets**).
- Acceso a la Web Console en `http://localhost:8091`.
- Acceso a terminal con `cbas` shell o `curl` disponible.
- Los Datasets de Analytics del Lab 14 creados (`TravelHotels`, `TravelRoutes`, `TravelAirports`, `TravelAirlines`). Si no existen, el Paso 1 de este lab los recrea.

---

## Entorno de Laboratorio

### Requisitos de hardware

| Recurso | Mínimo | Recomendado |
|---|---|---|
| RAM disponible | 16 GB | 32 GB |
| CPU | 4 núcleos x86_64 | 8 núcleos |
| Almacenamiento libre | 50 GB SSD | 100 GB SSD |
| Resolución de pantalla | 1280×800 | 1920×1080 |

### Requisitos de software

| Software | Versión | Notas |
|---|---|---|
| Couchbase Server | 7.6.x | Analytics Service habilitado |
| Navegador web | Chrome/Firefox/Edge 110+ | Para Web Console |
| `curl` | 7.x o superior | Para llamadas REST API |
| `cbas` shell | Incluido con Couchbase 7.6.x | Shell de Analytics |
| Editor de texto | VS Code 1.80+ o equivalente | Para preparar consultas |

### Verificación del entorno antes de comenzar

Abra una terminal y ejecute los siguientes comandos para confirmar que el entorno está listo:

```bash
# 1. Verificar que Couchbase está corriendo
curl -s -u Administrator:password \
  http://localhost:8091/pools/default | python3 -m json.tool | grep '"name"'

# 2. Verificar que el servicio Analytics responde
curl -s -u Administrator:password \
  http://localhost:8095/analytics/service \
  -d 'statement=SELECT+1+AS+ping' | python3 -m json.tool

# 3. Verificar que travel-sample está cargado
curl -s -u Administrator:password \
  http://localhost:8091/pools/default/buckets/travel-sample \
  | python3 -m json.tool | grep '"name"'
```

> **Nota:** Reemplace `Administrator:password` con sus credenciales reales. Si usa Docker, reemplace `localhost` con la IP del contenedor.

---

## Pasos del Laboratorio

---

### Paso 1 — Preparación del entorno Analytics: Verificar y recrear Datasets

**Objetivo:** Confirmar que los Datasets de Analytics necesarios existen y están sincronizados con `travel-sample`. Si no existen (por no haber completado el Lab 14), se crean en este paso.

#### Instrucciones

1. Abra la **Web Console** en `http://localhost:8091` e inicie sesión.

2. Navegue a **Analytics → Query Editor**.

3. Ejecute la siguiente consulta para listar los Datasets existentes en el dataverse `TravelAnalytics`:

```sql
SELECT dv.DataverseName, ds.DatasetName, ds.BucketName
FROM Metadata.`Dataverse` dv
JOIN Metadata.`Dataset` ds
  ON dv.DataverseName = ds.DataverseName
WHERE dv.DataverseName = "TravelAnalytics"
ORDER BY ds.DatasetName;
```

4. Si el dataverse o los datasets no existen, ejecute el siguiente bloque completo de configuración:

```sql
-- Crear dataverse si no existe
CREATE DATAVERSE TravelAnalytics IF NOT EXISTS;

USE TravelAnalytics;

-- Dataset de hoteles
CREATE DATASET IF NOT EXISTS TravelHotels
ON `travel-sample`.inventory.hotel;

-- Dataset de rutas
CREATE DATASET IF NOT EXISTS TravelRoutes
ON `travel-sample`.inventory.route;

-- Dataset de aeropuertos
CREATE DATASET IF NOT EXISTS TravelAirports
ON `travel-sample`.inventory.airport;

-- Dataset de aerolíneas
CREATE DATASET IF NOT EXISTS TravelAirlines
ON `travel-sample`.inventory.airline;

-- Conectar los datasets (sincronizar con los datos del bucket)
CONNECT LINK Local;
```

5. Espere aproximadamente 30–60 segundos para que la ingesta inicial se complete y luego verifique el conteo de documentos:

```sql
USE TravelAnalytics;

SELECT
    "TravelHotels"   AS dataset, COUNT(*) AS total FROM TravelHotels
UNION ALL
SELECT "TravelRoutes"    AS dataset, COUNT(*) AS total FROM TravelRoutes
UNION ALL
SELECT "TravelAirports"  AS dataset, COUNT(*) AS total FROM TravelAirports
UNION ALL
SELECT "TravelAirlines"  AS dataset, COUNT(*) AS total FROM TravelAirlines;
```

#### Salida esperada

```json
[
  { "dataset": "TravelHotels",  "total": 917  },
  { "dataset": "TravelRoutes",  "total": 24024 },
  { "dataset": "TravelAirports","total": 1968  },
  { "dataset": "TravelAirlines","total": 187   }
]
```

> Los totales exactos pueden variar ligeramente según la versión del sample bucket.

#### Verificación

✅ Los cuatro datasets devuelven conteos mayores a cero.
✅ No hay mensajes de error en la consola de Analytics.

---

### Paso 2 — Funciones de cadena avanzadas: Normalización y extracción de texto

**Objetivo:** Aplicar `REGEXP_CONTAINS`, `SPLIT`, `TRIM`, `STRING_UPPER` y `STRING_LENGTH` para limpiar y extraer información de campos de texto en la colección `hotels`.

#### Instrucciones

1. En el **Analytics Query Editor**, asegúrese de estar en el dataverse correcto:

```sql
USE TravelAnalytics;
```

2. Ejecute la siguiente consulta que combina múltiples funciones de cadena para analizar los hoteles:

```sql
USE TravelAnalytics;

SELECT
    h.name                                                    AS nombre_hotel,
    STRING_UPPER(h.country)                                   AS pais_mayusculas,
    STRING_LENGTH(TRIM(h.name))                               AS longitud_nombre,
    SPLIT(h.name, " ")[0]                                     AS primera_palabra_nombre,
    REGEXP_CONTAINS(h.name, "(?i)(hotel|inn|resort|lodge)")   AS es_establecimiento_conocido,
    IFMISSING(h.email, "sin_email")                           AS email,
    IFMISSING(h.phone, "sin_telefono")                        AS telefono
FROM TravelHotels h
WHERE h.country = "United States"
  AND REGEXP_CONTAINS(h.name, "(?i)(hotel|inn|resort|lodge)")
ORDER BY h.name
LIMIT 10;
```

3. Ahora ejecute una consulta que use `OBJECT_KEYS` para inspeccionar dinámicamente la estructura de los documentos de hotel y detectar qué campos opcionales están presentes:

```sql
USE TravelAnalytics;

SELECT
    h.name                                AS nombre_hotel,
    ARRAY_LENGTH(OBJECT_KEYS(h))          AS total_campos_documento,
    OBJECT_KEYS(h)                        AS campos_disponibles,
    TYPE(h.reviews)                       AS tipo_campo_reviews,
    ISNULL(h.email)                       AS email_es_null,
    ISMISSING(h.email)                    AS email_falta
FROM TravelHotels h
WHERE h.country = "United Kingdom"
LIMIT 5;
```

4. Examine los resultados. Observe cómo `OBJECT_KEYS` revela la estructura variable de los documentos y cómo `ISNULL` vs `ISMISSING` distinguen entre un campo presente con valor nulo y un campo ausente.

#### Salida esperada (extracto)

```json
[
  {
    "nombre_hotel": "Abbeyglen Castle Hotel",
    "pais_mayusculas": "UNITED KINGDOM",
    "longitud_nombre": 22,
    "primera_palabra_nombre": "Abbeyglen",
    "es_establecimiento_conocido": true,
    "email": "info@abbeyglen.ie",
    "telefono": "+353 95 21201"
  }
]
```

#### Verificación

✅ La columna `es_establecimiento_conocido` devuelve `true` para todos los resultados (dado el filtro `WHERE`).
✅ `IFMISSING` devuelve `"sin_email"` para hoteles sin campo `email` (no genera error).
✅ `OBJECT_KEYS` devuelve un arreglo de strings con los nombres de campos del documento.

---

### Paso 3 — Funciones de fecha/hora y colecciones: Análisis temporal de hoteles

**Objetivo:** Utilizar `DATE_DIFF_STR`, `DATE_ADD_STR`, `NOW_STR`, `ARRAY_AGG`, `ARRAY_DISTINCT` y `ARRAY_FLATTEN` para análisis temporal y de colecciones anidadas.

#### Instrucciones

1. Los hoteles en `travel-sample` contienen un arreglo `reviews` con subdocumentos que incluyen `date` y `ratings`. Ejecute primero una consulta exploratoria para entender la estructura:

```sql
USE TravelAnalytics;

SELECT h.name, h.reviews[0] AS primera_resena
FROM TravelHotels h
WHERE ARRAY_LENGTH(h.reviews) > 0
LIMIT 3;
```

2. Ahora aplique funciones de fecha para calcular la antigüedad de las reseñas más recientes:

```sql
USE TravelAnalytics;

SELECT
    h.name                                                          AS hotel,
    h.country                                                       AS pais,
    ARRAY_LENGTH(h.reviews)                                         AS total_resenas,
    -- Fecha de la reseña más reciente (última en el arreglo)
    h.reviews[ARRAY_LENGTH(h.reviews) - 1].date                     AS fecha_ultima_resena,
    -- Días transcurridos desde la última reseña hasta hoy
    DATE_DIFF_STR(
        NOW_STR(),
        h.reviews[ARRAY_LENGTH(h.reviews) - 1].date,
        "day"
    )                                                               AS dias_desde_ultima_resena,
    -- Rating promedio de todas las reseñas
    ROUND(
        AVG(r.ratings.Overall)
        FOR r IN h.reviews
        END,
        2
    )                                                               AS rating_promedio
FROM TravelHotels h
WHERE ARRAY_LENGTH(h.reviews) >= 3
  AND h.country = "United States"
ORDER BY dias_desde_ultima_resena ASC
LIMIT 10;
```

3. Use `ARRAY_AGG` y `ARRAY_DISTINCT` para recolectar valores únicos de un campo a través de múltiples documentos:

```sql
USE TravelAnalytics;

SELECT
    h.country                                           AS pais,
    COUNT(*)                                            AS total_hoteles,
    ARRAY_DISTINCT(ARRAY_AGG(h.city))                   AS ciudades_unicas,
    ARRAY_LENGTH(ARRAY_DISTINCT(ARRAY_AGG(h.city)))     AS num_ciudades_unicas,
    ROUND(AVG(h.price), 2)                              AS precio_promedio
FROM TravelHotels h
WHERE h.country IN ("United States", "United Kingdom", "France")
  AND h.price IS NOT NULL
GROUP BY h.country
ORDER BY total_hoteles DESC;
```

4. Demuestre `ARRAY_FLATTEN` con un ejemplo que aplana arreglos de amenidades anidadas:

```sql
USE TravelAnalytics;

-- Primero explorar la estructura de amenities
SELECT h.name, h.amenities
FROM TravelHotels h
WHERE h.amenities IS NOT NULL
LIMIT 3;
```

```sql
USE TravelAnalytics;

-- Contar amenidades únicas por país usando ARRAY_FLATTEN y ARRAY_DISTINCT
SELECT
    h.country                                                   AS pais,
    ARRAY_DISTINCT(
        ARRAY_FLATTEN(ARRAY_AGG(h.amenities), 1)
    )                                                           AS todas_amenidades_unicas,
    ARRAY_LENGTH(
        ARRAY_DISTINCT(ARRAY_FLATTEN(ARRAY_AGG(h.amenities), 1))
    )                                                           AS num_amenidades_distintas
FROM TravelHotels h
WHERE h.amenities IS NOT NULL
  AND h.country IN ("United States", "United Kingdom")
GROUP BY h.country;
```

#### Salida esperada (extracto — Paso 3.3)

```json
[
  {
    "pais": "United States",
    "total_hoteles": 328,
    "num_ciudades_unicas": 89,
    "precio_promedio": 145.23
  },
  {
    "pais": "United Kingdom",
    "total_hoteles": 194,
    "num_ciudades_unicas": 67,
    "precio_promedio": 132.87
  }
]
```

#### Verificación

✅ `DATE_DIFF_STR` devuelve valores numéricos positivos (días desde la reseña).
✅ `ARRAY_DISTINCT(ARRAY_AGG(...))` no genera duplicados en la lista de ciudades.
✅ `ARRAY_FLATTEN` aplana correctamente arreglos anidados (profundidad 1).

---

### Paso 4 — Introducción a Window Functions: ROW_NUMBER y RANK

**Objetivo:** Comprender la sintaxis de Window Functions en SQL++ para Analytics e implementar `ROW_NUMBER()` y `RANK()` con `PARTITION BY` y `ORDER BY` para rankear hoteles por país según su precio.

#### Instrucciones

1. Revise la sintaxis general de una Window Function en SQL++ para Analytics:

```
función_ventana() OVER (
    [PARTITION BY columna1, columna2, ...]
    [ORDER BY columna3 [ASC|DESC], ...]
    [ROWS BETWEEN inicio AND fin]
)
```

2. Implemente `ROW_NUMBER()` para asignar un número de fila único dentro de cada país, ordenado por precio descendente:

```sql
USE TravelAnalytics;

SELECT
    h.name                                                          AS hotel,
    h.country                                                       AS pais,
    h.city                                                          AS ciudad,
    h.price                                                         AS precio,
    ROW_NUMBER() OVER (
        PARTITION BY h.country
        ORDER BY h.price DESC
    )                                                               AS fila_por_pais
FROM TravelHotels h
WHERE h.country IN ("United States", "United Kingdom", "France")
  AND h.price IS NOT NULL
ORDER BY h.country, fila_por_pais;
```

3. Ahora compare `RANK()` vs `DENSE_RANK()` para entender cómo manejan los empates en el precio:

```sql
USE TravelAnalytics;

SELECT
    h.name                                                          AS hotel,
    h.country                                                       AS pais,
    h.price                                                         AS precio,
    RANK() OVER (
        PARTITION BY h.country
        ORDER BY h.price DESC
    )                                                               AS rank_precio,
    DENSE_RANK() OVER (
        PARTITION BY h.country
        ORDER BY h.price DESC
    )                                                               AS dense_rank_precio,
    ROW_NUMBER() OVER (
        PARTITION BY h.country
        ORDER BY h.price DESC
    )                                                               AS row_num_precio
FROM TravelHotels h
WHERE h.country = "United States"
  AND h.price IS NOT NULL
ORDER BY rank_precio
LIMIT 20;
```

4. Use `ROW_NUMBER()` en una subconsulta para obtener **solo el hotel más caro de cada país** (patrón top-N por partición):

```sql
USE TravelAnalytics;

SELECT ranked.hotel, ranked.pais, ranked.precio, ranked.ciudad
FROM (
    SELECT
        h.name    AS hotel,
        h.country AS pais,
        h.city    AS ciudad,
        h.price   AS precio,
        ROW_NUMBER() OVER (
            PARTITION BY h.country
            ORDER BY h.price DESC
        ) AS rn
    FROM TravelHotels h
    WHERE h.price IS NOT NULL
) AS ranked
WHERE ranked.rn = 1
ORDER BY ranked.precio DESC
LIMIT 15;
```

#### Salida esperada (extracto — Paso 4.3, primeras filas para "United States")

```json
[
  { "hotel": "The Grand Luxury", "pais": "United States", "precio": 899.0,
    "rank_precio": 1, "dense_rank_precio": 1, "row_num_precio": 1 },
  { "hotel": "Oceanview Resort", "pais": "United States", "precio": 899.0,
    "rank_precio": 1, "dense_rank_precio": 1, "row_num_precio": 2 },
  { "hotel": "Mountain Lodge",   "pais": "United States", "precio": 750.0,
    "rank_precio": 3, "dense_rank_precio": 2, "row_num_precio": 3 }
]
```

> **Observación clave:** Con empate en precio=899, `RANK` salta al 3 (gap), `DENSE_RANK` continúa en 2 (sin gap), y `ROW_NUMBER` siempre asigna valores únicos.

#### Verificación

✅ `RANK()` muestra gaps cuando hay empates (salta números).
✅ `DENSE_RANK()` no tiene gaps aunque haya empates.
✅ La subconsulta con `WHERE rn = 1` devuelve exactamente un hotel por país.

---

### Paso 5 — Window Functions avanzadas: NTILE, LAG y LEAD

**Objetivo:** Implementar `NTILE()` para segmentación en cuartiles y `LAG`/`LEAD` para comparar valores entre filas consecutivas en una serie de datos.

#### Instrucciones

1. Use `NTILE(4)` para segmentar los hoteles de Estados Unidos en cuatro cuartiles de precio:

```sql
USE TravelAnalytics;

SELECT
    h.name                                                          AS hotel,
    h.city                                                          AS ciudad,
    h.price                                                         AS precio,
    NTILE(4) OVER (
        ORDER BY h.price ASC
    )                                                               AS cuartil_precio,
    CASE
        NTILE(4) OVER (ORDER BY h.price ASC)
        WHEN 1 THEN "Económico"
        WHEN 2 THEN "Moderado"
        WHEN 3 THEN "Superior"
        WHEN 4 THEN "Premium"
    END                                                             AS segmento
FROM TravelHotels h
WHERE h.country = "United States"
  AND h.price IS NOT NULL
ORDER BY h.price ASC;
```

2. Agregue estadísticas por segmento usando la consulta anterior como subconsulta:

```sql
USE TravelAnalytics;

SELECT
    segmentado.segmento,
    segmentado.cuartil_precio,
    COUNT(*)                                AS total_hoteles,
    ROUND(MIN(segmentado.precio), 2)        AS precio_minimo,
    ROUND(MAX(segmentado.precio), 2)        AS precio_maximo,
    ROUND(AVG(segmentado.precio), 2)        AS precio_promedio
FROM (
    SELECT
        h.name   AS hotel,
        h.price  AS precio,
        NTILE(4) OVER (ORDER BY h.price ASC) AS cuartil_precio,
        CASE NTILE(4) OVER (ORDER BY h.price ASC)
            WHEN 1 THEN "Económico"
            WHEN 2 THEN "Moderado"
            WHEN 3 THEN "Superior"
            WHEN 4 THEN "Premium"
        END AS segmento
    FROM TravelHotels h
    WHERE h.country = "United States"
      AND h.price IS NOT NULL
) AS segmentado
GROUP BY segmentado.segmento, segmentado.cuartil_precio
ORDER BY segmentado.cuartil_precio;
```

3. Implemente `LAG` y `LEAD` para comparar el precio de cada hotel con el hotel anterior y siguiente en el ranking de precio dentro de su país:

```sql
USE TravelAnalytics;

SELECT
    h.name                                                              AS hotel,
    h.country                                                           AS pais,
    h.price                                                             AS precio_actual,
    LAG(h.price, 1, 0) OVER (
        PARTITION BY h.country
        ORDER BY h.price ASC
    )                                                                   AS precio_anterior,
    LEAD(h.price, 1, 0) OVER (
        PARTITION BY h.country
        ORDER BY h.price ASC
    )                                                                   AS precio_siguiente,
    -- Diferencia con el hotel anterior en el mismo país
    h.price - LAG(h.price, 1, h.price) OVER (
        PARTITION BY h.country
        ORDER BY h.price ASC
    )                                                                   AS diff_con_anterior
FROM TravelHotels h
WHERE h.country IN ("United States", "France")
  AND h.price IS NOT NULL
ORDER BY h.country, h.price ASC
LIMIT 20;
```

4. Analice los resultados. Los parámetros de `LAG(campo, offset, default)`:
   - `campo`: el campo a observar en la fila anterior.
   - `offset`: cuántas filas atrás mirar (1 = fila inmediatamente anterior).
   - `default`: valor a usar cuando no existe fila anterior (primera fila de la partición).

#### Salida esperada (extracto — Paso 5.2)

```json
[
  { "segmento": "Económico",  "cuartil_precio": 1, "total_hoteles": 82, "precio_minimo": 45.0,  "precio_maximo": 99.0,  "precio_promedio": 74.5  },
  { "segmento": "Moderado",   "cuartil_precio": 2, "total_hoteles": 82, "precio_minimo": 100.0, "precio_maximo": 149.0, "precio_promedio": 124.3 },
  { "segmento": "Superior",   "cuartil_precio": 3, "total_hoteles": 82, "precio_minimo": 150.0, "precio_maximo": 249.0, "precio_promedio": 193.7 },
  { "segmento": "Premium",    "cuartil_precio": 4, "total_hoteles": 82, "precio_minimo": 250.0, "precio_maximo": 899.0, "precio_promedio": 412.1 }
]
```

#### Verificación

✅ `NTILE(4)` distribuye los hoteles en cuatro grupos aproximadamente iguales.
✅ `LAG` devuelve el valor por defecto (`0`) en la primera fila de cada partición.
✅ `diff_con_anterior` es `0` en la primera fila de cada partición (precio - precio = 0 con el default).

---

### Paso 6 — FIRST_VALUE, LAST_VALUE y funciones de agregación con ventana

**Objetivo:** Implementar `FIRST_VALUE`, `LAST_VALUE`, y funciones de agregación acumulativas (`SUM OVER`, `AVG OVER`, `COUNT OVER`) con marcos de ventana `ROWS BETWEEN`.

#### Instrucciones

1. Use `FIRST_VALUE` y `LAST_VALUE` para comparar cada hotel con el más barato y el más caro dentro de su país:

```sql
USE TravelAnalytics;

SELECT
    h.name                                                              AS hotel,
    h.country                                                           AS pais,
    h.price                                                             AS precio,
    FIRST_VALUE(h.price) OVER (
        PARTITION BY h.country
        ORDER BY h.price ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    )                                                                   AS precio_minimo_pais,
    LAST_VALUE(h.price) OVER (
        PARTITION BY h.country
        ORDER BY h.price ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    )                                                                   AS precio_maximo_pais,
    ROUND(
        (h.price - FIRST_VALUE(h.price) OVER (
            PARTITION BY h.country
            ORDER BY h.price ASC
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
        )) * 100.0 /
        NULLIF(
            LAST_VALUE(h.price) OVER (
                PARTITION BY h.country
                ORDER BY h.price ASC
                ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
            ) -
            FIRST_VALUE(h.price) OVER (
                PARTITION BY h.country
                ORDER BY h.price ASC
                ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
            ),
            0
        ),
        1
    )                                                                   AS posicion_relativa_pct
FROM TravelHotels h
WHERE h.country IN ("United States", "France")
  AND h.price IS NOT NULL
ORDER BY h.country, h.price ASC
LIMIT 15;
```

> **Nota importante:** `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` es necesario para que `LAST_VALUE` vea toda la partición, no solo las filas hasta la actual (comportamiento por defecto).

2. Implemente un total acumulativo de rutas por aerolínea usando `SUM OVER` con `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`:

```sql
USE TravelAnalytics;

-- Primero: contar rutas por aerolínea
SELECT
    r.airline                                                       AS aerolinea,
    COUNT(*)                                                        AS total_rutas,
    SUM(COUNT(*)) OVER (
        ORDER BY COUNT(*) DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    )                                                               AS total_acumulativo,
    ROUND(
        SUM(COUNT(*)) OVER (
            ORDER BY COUNT(*) DESC
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) * 100.0 /
        SUM(COUNT(*)) OVER (),
        2
    )                                                               AS porcentaje_acumulativo
FROM TravelRoutes r
WHERE r.airline IS NOT NULL
GROUP BY r.airline
ORDER BY total_rutas DESC
LIMIT 20;
```

3. Use `AVG OVER` con una ventana deslizante para calcular el promedio móvil de precios de hoteles (ordenados por precio):

```sql
USE TravelAnalytics;

SELECT
    h.name                                                          AS hotel,
    h.country                                                       AS pais,
    h.price                                                         AS precio,
    ROUND(
        AVG(h.price) OVER (
            PARTITION BY h.country
            ORDER BY h.price ASC
            ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING
        ),
        2
    )                                                               AS promedio_movil_5,
    COUNT(*) OVER (
        PARTITION BY h.country
    )                                                               AS total_hoteles_pais
FROM TravelHotels h
WHERE h.country = "United States"
  AND h.price IS NOT NULL
ORDER BY h.price ASC
LIMIT 15;
```

#### Salida esperada (extracto — Paso 6.2, primeras filas)

```json
[
  { "aerolinea": "AA",  "total_rutas": 2765, "total_acumulativo": 2765,  "porcentaje_acumulativo": 11.51 },
  { "aerolinea": "UA",  "total_rutas": 2498, "total_acumulativo": 5263,  "porcentaje_acumulativo": 21.91 },
  { "aerolinea": "DL",  "total_rutas": 2341, "total_acumulativo": 7604,  "porcentaje_acumulativo": 31.65 },
  { "aerolinea": "WN",  "total_rutas": 1987, "total_acumulativo": 9591,  "porcentaje_acumulativo": 39.93 }
]
```

#### Verificación

✅ `FIRST_VALUE` devuelve el mismo valor para todos los hoteles del mismo país (el mínimo precio).
✅ El total acumulativo en la última fila del Paso 6.2 iguala al `SUM(COUNT(*)) OVER ()` (total general).
✅ `AVG OVER` con `ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING` produce un promedio móvil de hasta 5 valores.

---

### Paso 7 — Análisis del plan de ejecución con EXPLAIN y optimización

**Objetivo:** Usar la sentencia `EXPLAIN` para analizar el plan de ejecución de una consulta Analytics, identificar operaciones costosas y aplicar técnicas de optimización como filtros tempranos y proyecciones reducidas.

#### Instrucciones

1. Ejecute `EXPLAIN` sobre una consulta con Window Function para ver su plan de ejecución:

```sql
USE TravelAnalytics;

EXPLAIN
SELECT
    h.name,
    h.country,
    h.price,
    RANK() OVER (PARTITION BY h.country ORDER BY h.price DESC) AS ranking
FROM TravelHotels h
WHERE h.price IS NOT NULL
ORDER BY h.country, ranking;
```

2. Observe el plan de ejecución en la respuesta. Busque los siguientes operadores clave:

| Operador | Significado |
|---|---|
| `datasource-scan` | Escaneo completo del dataset (costoso si no hay filtro) |
| `assign` | Evaluación de expresiones y funciones |
| `window` | Operación de Window Function |
| `sort` | Ordenamiento (costoso en datasets grandes) |
| `stream-select` | Proyección de campos |
| `distribute-result` | Distribución de resultados al cliente |

3. Compare el plan de una consulta **sin filtro** vs **con filtro** para ver el impacto:

```sql
USE TravelAnalytics;

-- Plan SIN filtro (escaneo completo)
EXPLAIN
SELECT h.name, h.country, h.price
FROM TravelHotels h
ORDER BY h.price DESC
LIMIT 10;
```

```sql
USE TravelAnalytics;

-- Plan CON filtro early (reduce filas antes del sort)
EXPLAIN
SELECT h.name, h.country, h.price
FROM TravelHotels h
WHERE h.country = "United States"
  AND h.price > 100
ORDER BY h.price DESC
LIMIT 10;
```

4. Aplique las siguientes técnicas de optimización y compare los tiempos de ejecución:

```sql
USE TravelAnalytics;

-- Versión NO optimizada: selecciona todos los campos, sin filtros tempranos
SELECT *
FROM TravelHotels h
WHERE h.type = "hotel"
ORDER BY h.price DESC;
```

```sql
USE TravelAnalytics;

-- Versión OPTIMIZADA: proyección mínima + filtros específicos + LIMIT
SELECT h.name, h.country, h.city, h.price
FROM TravelHotels h
WHERE h.type = "hotel"
  AND h.price IS NOT NULL
  AND h.country IS NOT NULL
ORDER BY h.price DESC
LIMIT 100;
```

5. Documente los tiempos de ejecución observados en la Web Console (visible en la pestaña **Results** después de ejecutar cada consulta) en la siguiente tabla:

| Consulta | Tiempo de ejecución | Filas procesadas | Observación |
|---|---|---|---|
| Sin filtro, `SELECT *` | ___ ms | ~917 | Escaneo completo + sort completo |
| Con filtros + proyección + LIMIT | ___ ms | ~328 | Filtro early reduce sort |

#### Salida esperada (fragmento del plan EXPLAIN — Paso 7.1)

```json
{
  "plans": {
    "plan": {
      "operator": "distribute-result",
      "inputs": [{
        "operator": "sort",
        "inputs": [{
          "operator": "window",
          "inputs": [{
            "operator": "assign",
            "inputs": [{
              "operator": "datasource-scan",
              "dataset": "TravelHotels",
              "filter": "price IS NOT NULL"
            }]
          }]
        }]
      }]
    }
  }
}
```

#### Verificación

✅ El plan `EXPLAIN` muestra el operador `datasource-scan` como la hoja del árbol de ejecución.
✅ La versión optimizada (con filtros + proyección + LIMIT) ejecuta más rápido que la versión sin optimizar.
✅ Se puede identificar el operador `window` en el plan de consultas con Window Functions.

---

### Paso 8 — External Datasets: Configuración para archivos locales

**Objetivo:** Crear un archivo JSON local de prueba y configurar un External Dataset en Analytics para consultarlo, demostrando la capacidad de integrar datos externos (data lake) con datos del bucket.

#### Instrucciones

1. Primero, cree un archivo JSON de prueba que simule datos externos de un sistema de reservas. Abra una terminal y ejecute:

```bash
# Crear directorio para datos externos de Analytics
mkdir -p /tmp/analytics_external_data

# Crear archivo JSON con datos de reservas simuladas
cat > /tmp/analytics_external_data/reservations.json << 'EOF'
{"reservation_id": "RES001", "hotel_name": "Grand Hotel NYC", "country": "United States", "check_in": "2024-03-15", "check_out": "2024-03-18", "total_amount": 897.50, "currency": "USD", "status": "confirmed"}
{"reservation_id": "RES002", "hotel_name": "Paris Boutique", "country": "France", "check_in": "2024-04-01", "check_out": "2024-04-05", "total_amount": 1240.00, "currency": "EUR", "status": "confirmed"}
{"reservation_id": "RES003", "hotel_name": "London Bridge Inn", "country": "United Kingdom", "check_in": "2024-03-20", "check_out": "2024-03-22", "total_amount": 560.00, "currency": "GBP", "status": "cancelled"}
{"reservation_id": "RES004", "hotel_name": "Miami Beach Resort", "country": "United States", "check_in": "2024-05-10", "check_out": "2024-05-17", "total_amount": 2100.00, "currency": "USD", "status": "confirmed"}
{"reservation_id": "RES005", "hotel_name": "Barcelona Suites", "country": "Spain", "check_in": "2024-06-01", "check_out": "2024-06-04", "total_amount": 780.00, "currency": "EUR", "status": "pending"}
EOF

# Verificar que el archivo se creó correctamente
wc -l /tmp/analytics_external_data/reservations.json
cat /tmp/analytics_external_data/reservations.json
```

2. En el **Analytics Query Editor**, cree el External Dataset apuntando al archivo local:

```sql
USE TravelAnalytics;

-- Crear External Dataset para leer archivos JSON locales
CREATE EXTERNAL DATASET IF NOT EXISTS ExternalReservations
(
    reservation_id  STRING,
    hotel_name      STRING,
    country         STRING,
    check_in        STRING,
    check_out       STRING,
    total_amount    DOUBLE,
    currency        STRING,
    status          STRING
)
USING localfs
WITH {
    "path": "/tmp/analytics_external_data",
    "format": "json",
    "include": "*.json"
};
```

3. Consulte el External Dataset directamente:

```sql
USE TravelAnalytics;

SELECT *
FROM ExternalReservations
ORDER BY total_amount DESC;
```

4. Realice un análisis combinando el External Dataset con los datos internos de `TravelHotels`:

```sql
USE TravelAnalytics;

SELECT
    er.reservation_id,
    er.hotel_name                                                   AS hotel_reservado,
    er.country,
    er.check_in,
    er.check_out,
    DATE_DIFF_STR(er.check_out, er.check_in, "day")                 AS noches,
    er.total_amount,
    er.currency,
    er.status,
    h.price                                                         AS precio_por_noche_catalogo,
    ROUND(
        er.total_amount /
        NULLIF(DATE_DIFF_STR(er.check_out, er.check_in, "day"), 0),
        2
    )                                                               AS precio_real_por_noche
FROM ExternalReservations er
LEFT JOIN TravelHotels h
    ON STRING_UPPER(TRIM(er.hotel_name)) = STRING_UPPER(TRIM(h.name))
WHERE er.status = "confirmed"
ORDER BY er.total_amount DESC;
```

5. Revise la sintaxis para S3 (solo conceptual — no ejecutar en el laboratorio sin credenciales AWS):

```sql
-- EJEMPLO CONCEPTUAL: External Dataset en Amazon S3
-- NO ejecutar sin credenciales AWS válidas

CREATE EXTERNAL DATASET ExternalS3Reservations
(
    reservation_id STRING,
    hotel_name     STRING,
    total_amount   DOUBLE
)
USING s3
WITH {
    "region":          "us-east-1",
    "serviceEndpoint": "s3.amazonaws.com",
    "container":       "mi-bucket-analytics",
    "prefix":          "reservations/2024/",
    "format":          "json",
    "accessKeyId":     "AKIAIOSFODNN7EXAMPLE",
    "secretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
};
```

#### Salida esperada (Paso 8.3)

```json
[
  { "reservation_id": "RES004", "hotel_name": "Miami Beach Resort",  "country": "United States", "check_in": "2024-05-10", "check_out": "2024-05-17", "total_amount": 2100.0, "currency": "USD", "status": "confirmed" },
  { "reservation_id": "RES002", "hotel_name": "Paris Boutique",      "country": "France",        "check_in": "2024-04-01", "check_out": "2024-04-05", "total_amount": 1240.0, "currency": "EUR", "status": "confirmed" },
  { "reservation_id": "RES001", "hotel_name": "Grand Hotel NYC",     "country": "United States", "check_in": "2024-03-15", "check_out": "2024-03-18", "total_amount": 897.5,  "currency": "USD", "status": "confirmed" }
]
```

#### Verificación

✅ El archivo `/tmp/analytics_external_data/reservations.json` contiene exactamente 5 líneas (5 documentos JSON).
✅ `SELECT * FROM ExternalReservations` devuelve 5 filas sin errores.
✅ El `LEFT JOIN` con `TravelHotels` no genera errores (puede devolver `null` en `precio_por_noche_catalogo` si el nombre no coincide exactamente).

---

### Paso 9 — Remote Links: Sintaxis y configuración conceptual

**Objetivo:** Comprender la sintaxis de `CREATE LINK` para conectar el servicio Analytics local con un clúster Couchbase remoto, y verificar la configuración de un link local.

#### Instrucciones

1. Primero, inspeccione el link `Local` que viene preconfigurado en Couchbase Analytics:

```sql
-- Listar todos los links disponibles
SELECT lnk.LinkName, lnk.DataverseName, lnk.IsActive, lnk.Type
FROM Metadata.`Link` lnk;
```

2. Verifique el estado del link `Local`:

```sql
-- Verificar estado del link Local
SELECT VALUE lnk
FROM Metadata.`Link` lnk
WHERE lnk.LinkName = "Local";
```

3. Revise la sintaxis completa para crear un Remote Link a un clúster Couchbase externo (ejecute esta consulta para ver que la sintaxis es válida, pero use un host ficticio):

```sql
-- SINTAXIS de Remote Link (no ejecutar en producción sin clúster remoto real)
-- Esta consulta fallará con "connection refused" si el host no existe,
-- lo cual es el comportamiento esperado en este laboratorio

CREATE LINK TravelAnalytics.RemoteClusterLink
TYPE couchbase
WITH {
    "hostname":             "remote-couchbase-host:8091",
    "username":             "remote_user",
    "password":             "remote_password",
    "encryption":           "none"
};
```

> **Nota:** En un entorno real con clúster remoto disponible, este comando crearía el link. Para este laboratorio, el comando fallará con un error de conexión, lo cual es esperado.

4. Revise la sintaxis para usar un Remote Link en una consulta (solo referencia):

```sql
-- SINTAXIS: Crear dataset usando un Remote Link
-- (Solo referencia — requiere link activo con clúster remoto)

CREATE DATASET TravelAnalytics.RemoteHotels
ON `travel-sample`.inventory.hotel
AT TravelAnalytics.RemoteClusterLink;

-- Consulta que usaría el dataset remoto
SELECT COUNT(*) AS total_hoteles_remotos
FROM TravelAnalytics.RemoteHotels;
```

5. Documente las diferencias entre los tipos de links disponibles:

```sql
-- Consulta informativa: tipos de links en Couchbase Analytics
SELECT
    "Local"    AS tipo_link,
    "Acceso directo al clúster local donde corre Analytics" AS descripcion,
    "Automático, no requiere configuración" AS configuracion
UNION ALL
SELECT
    "Remote (Couchbase)" AS tipo_link,
    "Acceso a un clúster Couchbase externo via red" AS descripcion,
    "Requiere hostname, credenciales y configuración de encriptación" AS configuracion
UNION ALL
SELECT
    "External (S3/GCS/Azure)" AS tipo_link,
    "Acceso a almacenamiento de objetos en la nube" AS descripcion,
    "Requiere credenciales de cloud provider y configuración de región" AS configuracion;
```

#### Salida esperada (Paso 9.1)

```json
[
  {
    "LinkName": "Local",
    "DataverseName": "Metadata",
    "IsActive": true,
    "Type": "INTERNAL"
  }
]
```

#### Verificación

✅ El link `Local` existe y tiene `IsActive: true`.
✅ La consulta del Paso 9.5 devuelve las tres filas de tipos de links sin error.
✅ El intento de crear el Remote Link falla con un error de conexión (no con un error de sintaxis), confirmando que la sintaxis es correcta.

---

### Paso 10 — Consulta analítica integrada: Caso de uso completo

**Objetivo:** Integrar funciones built-in avanzadas, Window Functions y datos externos en una consulta analítica compleja que simula un reporte ejecutivo real.

#### Instrucciones

1. Ejecute la consulta analítica integrada que combina todo lo aprendido en el laboratorio:

```sql
USE TravelAnalytics;

-- Reporte ejecutivo: Top hoteles por país con métricas de ranking,
-- segmentación por precio, comparación con promedio del país
-- y clasificación de calidad basada en reseñas

SELECT
    ranked.pais,
    ranked.hotel,
    ranked.ciudad,
    ranked.precio,
    ranked.rating_promedio,
    ranked.total_resenas,
    ranked.ranking_precio_en_pais,
    ranked.ranking_rating_en_pais,
    ranked.segmento_precio,
    ranked.precio_vs_promedio_pais,
    ranked.precio_anterior_en_ranking,
    ranked.diferencia_con_anterior
FROM (
    SELECT
        h.country                                                       AS pais,
        h.name                                                          AS hotel,
        h.city                                                          AS ciudad,
        h.price                                                         AS precio,

        -- Rating promedio calculado desde el arreglo de reseñas
        ROUND(
            AVG(r.ratings.Overall) FOR r IN h.reviews END,
            2
        )                                                               AS rating_promedio,

        ARRAY_LENGTH(h.reviews)                                         AS total_resenas,

        -- Ranking de precio dentro del país (más caro = rank 1)
        RANK() OVER (
            PARTITION BY h.country
            ORDER BY h.price DESC
        )                                                               AS ranking_precio_en_pais,

        -- Ranking de rating dentro del país (mejor rating = rank 1)
        RANK() OVER (
            PARTITION BY h.country
            ORDER BY AVG(r.ratings.Overall) FOR r IN h.reviews END DESC
        )                                                               AS ranking_rating_en_pais,

        -- Segmentación en cuartiles de precio por país
        CASE NTILE(4) OVER (PARTITION BY h.country ORDER BY h.price ASC)
            WHEN 1 THEN "Económico"
            WHEN 2 THEN "Moderado"
            WHEN 3 THEN "Superior"
            WHEN 4 THEN "Premium"
        END                                                             AS segmento_precio,

        -- Diferencia porcentual con el precio promedio del país
        ROUND(
            (h.price - AVG(h.price) OVER (PARTITION BY h.country)) * 100.0
            / NULLIF(AVG(h.price) OVER (PARTITION BY h.country), 0),
            1
        )                                                               AS precio_vs_promedio_pais,

        -- Precio del hotel anterior en el ranking (LAG)
        LAG(h.price, 1, h.price) OVER (
            PARTITION BY h.country
            ORDER BY h.price DESC
        )                                                               AS precio_anterior_en_ranking,

        -- Diferencia de precio con el hotel anterior
        LAG(h.price, 1, h.price) OVER (
            PARTITION BY h.country
            ORDER BY h.price DESC
        ) - h.price                                                     AS diferencia_con_anterior

    FROM TravelHotels h
    WHERE h.country IN ("United States", "United Kingdom", "France")
      AND h.price IS NOT NULL
      AND ARRAY_LENGTH(h.reviews) > 0
) AS ranked
WHERE ranked.ranking_precio_en_pais <= 5   -- Solo top 5 más caros por país
ORDER BY ranked.pais, ranked.ranking_precio_en_pais;
```

2. Analice el plan de ejecución de esta consulta compleja:

```sql
USE TravelAnalytics;

EXPLAIN
SELECT
    h.country,
    h.name,
    h.price,
    RANK() OVER (PARTITION BY h.country ORDER BY h.price DESC) AS rk
FROM TravelHotels h
WHERE h.country IN ("United States", "United Kingdom", "France")
  AND h.price IS NOT NULL
ORDER BY h.country, rk
LIMIT 15;
```

3. Observe en el plan cuántos operadores `window` aparecen y en qué orden se ejecutan las operaciones.

#### Salida esperada (extracto — primeras filas)

```json
[
  {
    "pais": "France",
    "hotel": "Château Frontenac",
    "ciudad": "Paris",
    "precio": 650.0,
    "rating_promedio": 4.8,
    "total_resenas": 12,
    "ranking_precio_en_pais": 1,
    "ranking_rating_en_pais": 2,
    "segmento_precio": "Premium",
    "precio_vs_promedio_pais": 187.3,
    "precio_anterior_en_ranking": 650.0,
    "diferencia_con_anterior": 0.0
  }
]
```

#### Verificación

✅ La consulta devuelve exactamente 5 filas por país (filtro `ranking_precio_en_pais <= 5`).
✅ La columna `diferencia_con_anterior` es `0.0` para el hotel de `ranking_precio_en_pais = 1` (primer hotel de cada partición, LAG usa el default = precio actual).
✅ La columna `segmento_precio` muestra `"Premium"` para los hoteles más caros.

---

## Validación y Pruebas Finales

Ejecute las siguientes consultas de validación para confirmar que todos los pasos del laboratorio se completaron correctamente:

```sql
USE TravelAnalytics;

-- Validación 1: Confirmar que los 4 datasets están activos
SELECT ds.DatasetName, ds.BucketName
FROM Metadata.`Dataset` ds
WHERE ds.DataverseName = "TravelAnalytics"
ORDER BY ds.DatasetName;
-- ESPERADO: 4 filas (TravelAirlines, TravelAirports, TravelHotels, TravelRoutes)
```

```sql
USE TravelAnalytics;

-- Validación 2: Window Function básica funciona correctamente
SELECT COUNT(*) AS total_con_rank
FROM (
    SELECT
        h.name,
        RANK() OVER (PARTITION BY h.country ORDER BY h.price DESC) AS rk
    FROM TravelHotels h
    WHERE h.price IS NOT NULL
) AS t
WHERE t.rk = 1;
-- ESPERADO: número igual al total de países distintos con precio no nulo
```

```sql
USE TravelAnalytics;

-- Validación 3: External Dataset accesible
SELECT COUNT(*) AS total_reservas_externas
FROM ExternalReservations;
-- ESPERADO: 5
```

```sql
USE TravelAnalytics;

-- Validación 4: Funciones de fecha funcionan correctamente
SELECT
    DATE_DIFF_STR("2024-12-31", "2024-01-01", "day") AS dias_en_2024,
    DATE_ADD_STR("2024-01-01", 365, "day")            AS un_anio_despues,
    NOW_STR()                                          AS timestamp_actual;
-- ESPERADO: dias_en_2024 = 365, un_anio_despues = "2025-01-01T..."
```

```sql
USE TravelAnalytics;

-- Validación 5: Funciones de colección funcionan correctamente
SELECT
    ARRAY_LENGTH([1, 2, 3, 4, 5])                          AS longitud,
    ARRAY_DISTINCT([1, 2, 2, 3, 3, 3])                     AS sin_duplicados,
    ARRAY_FLATTEN([[1, 2], [3, 4], [5]], 1)                 AS aplanado,
    OBJECT_KEYS({"a": 1, "b": 2, "c": 3})                  AS claves;
-- ESPERADO: longitud=5, sin_duplicados=[1,2,3], aplanado=[1,2,3,4,5], claves=["a","b","c"]
```

---

## Resolución de Problemas

### Problema 1: Error "Window function not supported" o resultados incorrectos con LAST_VALUE

**Síntomas:**
- La consulta con `LAST_VALUE` devuelve el mismo valor que `FIRST_VALUE` (no el valor esperado del extremo de la partición).
- O bien, la consulta falla con un error indicando que el marco de ventana no es válido.

**Causa:**
El comportamiento por defecto del marco de ventana en SQL++ para Analytics cuando se usa `ORDER BY` es `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. Esto significa que `LAST_VALUE` solo "ve" desde el inicio de la partición hasta la fila actual, no hasta el final de la partición. Por eso devuelve el valor de la fila actual en lugar del último valor de toda la partición.

**Solución:**
Siempre especificar explícitamente el marco de ventana completo cuando se usa `LAST_VALUE`:

```sql
-- INCORRECTO (comportamiento por defecto — LAST_VALUE = valor actual):
LAST_VALUE(h.price) OVER (
    PARTITION BY h.country
    ORDER BY h.price ASC
)

-- CORRECTO (marco explícito para ver toda la partición):
LAST_VALUE(h.price) OVER (
    PARTITION BY h.country
    ORDER BY h.price ASC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

Aplique este patrón a todas las consultas con `LAST_VALUE` en el laboratorio y vuelva a ejecutar.

---

### Problema 2: External Dataset devuelve cero filas o error "path not found"

**Síntomas:**
- `SELECT COUNT(*) FROM ExternalReservations` devuelve `0`.
- O la consulta falla con un error como `"External dataset path not found"` o `"Permission denied"`.

**Causa:**
El External Dataset de tipo `localfs` requiere que el path especificado en `WITH {"path": "..."}` sea accesible por el proceso de Couchbase Server en el sistema operativo. Si Couchbase corre dentro de Docker, el path `/tmp/analytics_external_data` del host no es visible dentro del contenedor a menos que se haya montado como volumen. Adicionalmente, el proceso de Couchbase puede no tener permisos de lectura sobre el directorio.

**Solución:**

**Caso A — Couchbase en Docker:** Monte el directorio como volumen al iniciar el contenedor, o cópielo dentro del contenedor:

```bash
# Copiar el archivo dentro del contenedor Docker
docker cp /tmp/analytics_external_data/reservations.json \
    <nombre_contenedor>:/tmp/analytics_external_data/reservations.json

# O bien, al iniciar el contenedor, agregar el volumen:
# docker run -v /tmp/analytics_external_data:/tmp/analytics_external_data ...
```

**Caso B — Permisos insuficientes:** Otorgue permisos de lectura al directorio:

```bash
# Verificar quién ejecuta Couchbase
ps aux | grep couchbase

# Dar permisos de lectura universales al directorio de prueba
chmod -R 755 /tmp/analytics_external_data
ls -la /tmp/analytics_external_data/

# Recrear el External Dataset con el path corregido
# (En Analytics Query Editor):
```

```sql
USE TravelAnalytics;

DROP DATASET ExternalReservations IF EXISTS;

CREATE EXTERNAL DATASET ExternalReservations
(
    reservation_id STRING,
    hotel_name     STRING,
    country        STRING,
    check_in       STRING,
    check_out      STRING,
    total_amount   DOUBLE,
    currency       STRING,
    status         STRING
)
USING localfs
WITH {
    "path":    "/tmp/analytics_external_data",
    "format":  "json",
    "include": "*.json"
};
```

---

## Limpieza del Entorno

Ejecute los siguientes comandos para limpiar los recursos creados en este laboratorio y dejar el entorno en un estado ordenado:

```sql
-- Limpiar External Dataset creado en el Paso 8
USE TravelAnalytics;

DROP DATASET ExternalReservations IF EXISTS;
```

```sql
-- Opcional: eliminar el Remote Link si se creó exitosamente
-- (Solo si el link fue creado en el Paso 9)
USE TravelAnalytics;

DROP LINK RemoteClusterLink IF EXISTS;
```

```bash
# Limpiar archivos temporales del sistema de archivos
rm -rf /tmp/analytics_external_data
echo "Directorio de datos externos eliminado."
```

```sql
-- Verificación final: estado del dataverse después de la limpieza
USE TravelAnalytics;

SELECT ds.DatasetName, ds.DataverseName
FROM Metadata.`Dataset` ds
WHERE ds.DataverseName = "TravelAnalytics"
ORDER BY ds.DatasetName;
-- ESPERADO: Solo los 4 datasets originales (TravelAirlines, TravelAirports, TravelHotels, TravelRoutes)
-- ExternalReservations NO debe aparecer
```

> **Nota:** Los cuatro datasets principales (`TravelHotels`, `TravelRoutes`, `TravelAirports`, `TravelAirlines`) se conservan intencionalmente, ya que pueden ser necesarios para laboratorios posteriores.

---

## Resumen

En este laboratorio se exploraron las capacidades analíticas más avanzadas de Couchbase Analytics mediante la aplicación práctica de dos grandes categorías de funcionalidades:

**Funciones Built-in Avanzadas:**
- Se aplicaron funciones de cadena (`REGEXP_CONTAINS`, `SPLIT`, `STRING_UPPER`, `TRIM`) para normalizar y extraer información de campos de texto en documentos de hotel.
- Se utilizaron `DATE_DIFF_STR` y `NOW_STR` para calcular la antigüedad de reseñas directamente en el motor analítico.
- `ARRAY_AGG`, `ARRAY_DISTINCT` y `ARRAY_FLATTEN` permitieron operar sobre colecciones anidadas para análisis de amenidades y ciudades únicas.
- `IFMISSING`, `ISNULL` y `ISMISSING` demostraron ser esenciales para manejar el esquema flexible de documentos JSON en colecciones NoSQL reales.

**Window Functions:**
- `ROW_NUMBER`, `RANK` y `DENSE_RANK` permitieron implementar rankings por partición con diferente manejo de empates.
- `NTILE(4)` facilitó la segmentación de hoteles en cuartiles de precio para análisis de mercado.
- `LAG` y `LEAD` habilitaron comparaciones entre filas consecutivas, fundamentales para análisis de tendencias.
- `FIRST_VALUE` y `LAST_VALUE` con marcos explícitos (`ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`) permitieron comparar cada fila con los extremos de su partición.
- Las funciones de agregación con ventana (`SUM OVER`, `AVG OVER`) con `ROWS BETWEEN` implementaron totales acumulativos y promedios móviles.

**Optimización y Fuentes Externas:**
- La sentencia `EXPLAIN` reveló el árbol de operadores del plan de ejecución, permitiendo identificar escaneos completos costosos.
- Los filtros tempranos y las proyecciones reducidas demostraron mejoras medibles en el tiempo de ejecución.
- Los External Datasets (`localfs` y conceptualmente S3) abrieron la posibilidad de integrar datos de un data lake sin moverlos al bucket.
- La sintaxis de Remote Links permitió comprender cómo federar consultas entre múltiples clústeres Couchbase.

### Recursos Adicionales

| Recurso | URL |
|---|---|
| Referencia de Window Functions en SQL++ Analytics | https://docs.couchbase.com/server/current/analytics/window-functions.html |
| Funciones integradas de SQL++ para Analytics | https://docs.couchbase.com/server/current/analytics/sql-plus-plus-functions.html |
| External Datasets en Couchbase Analytics | https://docs.couchbase.com/server/current/analytics/external-datasets.html |
| Remote Links en Couchbase Analytics | https://docs.couchbase.com/server/current/analytics/remote-links.html |
| EXPLAIN en Couchbase Analytics | https://docs.couchbase.com/server/current/analytics/explain.html |
| Guía de optimización de consultas Analytics | https://docs.couchbase.com/server/current/analytics/query-optimization.html |

---
LAB_END---
