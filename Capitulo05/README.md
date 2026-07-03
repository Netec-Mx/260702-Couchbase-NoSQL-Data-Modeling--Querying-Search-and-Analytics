# Análisis de consultas con EXPLAIN y optimización básica

## 1. Metadatos

| Campo            | Detalle                                                                 |
|------------------|-------------------------------------------------------------------------|
| **Duración**     | 80 minutos                                                              |
| **Complejidad**  | Alta                                                                    |
| **Nivel Bloom**  | Aplicar                                                                 |
| **Dataset**      | `travel-sample` (debe estar pre-cargado)                                |
| **Servicio CB**  | Query Service, Index Service, Data Service                              |
| **Versión CB**   | Couchbase Server 7.6.x                                                  |

---

## 2. Descripción General

En este laboratorio recibirás un conjunto de **10 consultas SQL++ intencionalmente subóptimas** escritas sobre el dataset `travel-sample`. Para cada consulta aplicarás el ciclo diagnóstico completo: ejecutar la consulta en su estado original y registrar métricas de tiempo, interpretar el plan de ejecución con `EXPLAIN` identificando el operador más costoso, crear o modificar el índice apropiado, y verificar la mejora comparando los planes y métricas antes/después. Al finalizar construirás una **tabla comparativa de resultados** con la mejora porcentual de tiempo para cada caso.

---

## 3. Objetivos de Aprendizaje

- [ ] Interpretar la salida de `EXPLAIN` identificando operadores clave (`PrimaryScan`, `IndexScan3`, `Fetch`, `Filter`, `Order`, `Aggregate`) y el impacto de cada uno en el rendimiento.
- [ ] Comparar métricas de ejecución (`executionTime`, `resultCount`, documentos escaneados) antes y después de aplicar optimizaciones de índice.
- [ ] Aplicar técnicas de *index tuning*: índices cubrientes, índices con predicados de filtro y orden correcto de campos en índices compuestos.
- [ ] Identificar y corregir anti-patrones frecuentes: `SELECT *`, funciones en predicados `WHERE`, `ORDER BY` sin índice ordenado y `JOIN` sin índice en el campo de unión.
- [ ] Utilizar pushdowns de predicado, `ORDER BY`, `LIMIT` y `GROUP BY` para reducir el trabajo del motor de consultas.

---

## 4. Prerrequisitos

### Conocimiento previo

| Requisito | Detalle |
|-----------|---------|
| Lab 03-00-01 completado | Consultas SQL++ básicas sobre `travel-sample` |
| Lab 04-00-01 completado | Creación y gestión de índices secundarios en Couchbase |
| Conceptos de EXPLAIN | Familiaridad con planes de ejecución en SQL relacional es útil pero no obligatoria |
| Scopes y Collections | Entender la jerarquía `bucket.scope.collection` en Couchbase 7.x |

### Acceso requerido

| Recurso | Detalle |
|---------|---------|
| Couchbase Web Console | `http://localhost:8091` — credenciales de administrador |
| Dataset `travel-sample` | Cargado y con al menos las colecciones `hotel`, `airline`, `airport`, `route`, `landmark` |
| cbq o Query Editor | Acceso al Query Workbench de la Web Console o CLI `cbq` |
| Permisos | Rol `Full Admin` o `Query Manage Index` + `Query Select` sobre `travel-sample` |

---

## 5. Entorno de Laboratorio

### Hardware mínimo recomendado

| Componente | Mínimo | Recomendado |
|------------|--------|-------------|
| RAM | 8 GB | 16 GB |
| CPU | 4 núcleos x86_64 | 8 núcleos |
| Almacenamiento | 20 GB libres (SSD) | 50 GB SSD |
| Red | localhost con puertos 8091–8097 disponibles | — |

### Software requerido

| Software | Versión | Uso en este lab |
|----------|---------|-----------------|
| Couchbase Server | 7.6.x | Motor principal |
| Navegador web | Chrome/Firefox/Edge 110+ | Web Console y Query Workbench |
| `cbq` (opcional) | Incluido con CB 7.6.x | Ejecución de consultas por CLI |
| `curl` (opcional) | 7.x+ | Verificación via REST API |

### Verificación del entorno antes de comenzar

Ejecuta las siguientes comprobaciones en el **Query Workbench** (`http://localhost:8091` → pestaña **Query**):

```sql
-- 1. Verificar que travel-sample está disponible
SELECT RAW COUNT(*) FROM `travel-sample`.inventory.hotel;
-- Esperado: [~917]

-- 2. Verificar colecciones disponibles
SELECT RAW name FROM system:keyspaces
WHERE `path` LIKE '%travel-sample%'
ORDER BY name;

-- 3. Listar índices existentes sobre travel-sample
SELECT name, keyspace_id, state, `using`
FROM system:indexes
WHERE keyspace_id IN ("hotel","airline","airport","route","landmark")
   OR bucket_id = "travel-sample"
ORDER BY keyspace_id, name;
```

> **Nota para el instructor:** Si `travel-sample` no está cargado, ir a **Settings → Sample Buckets → travel-sample → Load**. El proceso tarda aproximadamente 2–3 minutos.

---

## 6. Instrucciones Paso a Paso

### Preparación: Habilitar métricas de consulta

Antes de comenzar, activa el **Query Profiling** en el Query Workbench para ver métricas detalladas en cada ejecución.

**Instrucciones:**

1. En la Web Console, ve a la pestaña **Query**.
2. Haz clic en el ícono de configuración (⚙) o en **Query Preferences**.
3. Establece **Profile** en `timings` y **Metrics** en `true`.
4. Alternativamente, agrega el parámetro en cada consulta:

```sql
-- Activar profiling en una consulta individual
SELECT h.name, h.city
FROM `travel-sample`.inventory.hotel AS h
WHERE h.country = "United States"
LIMIT 5;
-- Luego revisar la pestaña "Plan" y "Execution Plan" en el Workbench
```

5. Toma nota del formato de métricas que verás en la respuesta JSON:

```json
{
  "metrics": {
    "elapsedTime":   "45.123ms",
    "executionTime": "44.891ms",
    "resultCount":   5,
    "resultSize":    1240,
    "serviceLoad":   1,
    "sortCount":     0
  }
}
```

**Resultado esperado:** El Query Workbench muestra la pestaña **Plan** activa con el árbol de operadores visualizado.

---

### Paso 1 — Caso Base: Consulta sin índice secundario (PrimaryScan)

**Objetivo:** Observar el costo de una consulta que fuerza un escaneo del índice primario y entender por qué es el anti-patrón más costoso.

#### 1.1 Ejecutar la consulta original y registrar métricas

```sql
-- CONSULTA PROBLEMÁTICA #1: Sin predicado indexado, usa PrimaryScan
SELECT name, city, country, avg_rating
FROM `travel-sample`.inventory.hotel
WHERE avg_rating > 4.5
  AND free_breakfast = true;
```

**Instrucciones:**

1. Pega la consulta en el Query Workbench y ejecútala.
2. Registra en tu tabla de resultados:
   - `executionTime` (pestaña Metrics)
   - `resultCount`
   - El operador raíz que aparece en la pestaña **Plan**

#### 1.2 Analizar el plan con EXPLAIN

```sql
EXPLAIN
SELECT name, city, country, avg_rating
FROM `travel-sample`.inventory.hotel
WHERE avg_rating > 4.5
  AND free_breakfast = true;
```

3. Examina el JSON resultante. Busca el campo `"#operator"` en el nivel más alto del árbol.
4. Identifica si aparece `"PrimaryScan3"` — esto indica que Couchbase está escaneando **todos** los documentos del bucket.

**Salida esperada (fragmento):**

```json
{
  "plan": {
    "#operator": "Sequence",
    "~children": [
      {
        "#operator": "PrimaryScan3",
        "keyspace": "hotel",
        "namespace": "default",
        "using": "#primary"
      },
      {
        "#operator": "Fetch",
        "keyspace": "hotel"
      },
      {
        "#operator": "Filter",
        "condition": "(((`hotel`.`avg_rating`) > 4.5) and ((`hotel`.`free_breakfast`) = true))"
      },
      {
        "#operator": "InitialProject",
        "result_terms": [...]
      }
    ]
  }
}
```

#### 1.3 Crear el índice apropiado y re-ejecutar

```sql
-- Crear índice compuesto que cubra ambos predicados
CREATE INDEX idx_hotel_rating_breakfast
ON `travel-sample`.inventory.hotel(avg_rating, free_breakfast)
WHERE avg_rating IS NOT MISSING;
```

5. Espera a que el índice esté en estado `online` (verifica en **Indexes** o con `SELECT state FROM system:indexes WHERE name = 'idx_hotel_rating_breakfast'`).
6. Re-ejecuta la consulta original y registra las nuevas métricas.
7. Re-ejecuta `EXPLAIN` y confirma que ahora aparece `IndexScan3` en lugar de `PrimaryScan3`.

**Verificación:**

```sql
-- Confirmar que el índice está siendo utilizado
EXPLAIN
SELECT name, city, country, avg_rating
FROM `travel-sample`.inventory.hotel
WHERE avg_rating > 4.5
  AND free_breakfast = true;
-- Buscar: "#operator": "IndexScan3" con "index": "idx_hotel_rating_breakfast"
```

**Resultado esperado:** La mejora de tiempo debe ser del **60–80%** en este caso. Registra ambos valores en tu tabla comparativa.

---

### Paso 2 — Anti-patrón: Función en predicado WHERE

**Objetivo:** Demostrar que aplicar funciones sobre campos indexados en el `WHERE` impide el uso del índice.

#### 2.1 Consulta problemática con LOWER()

```sql
-- CONSULTA PROBLEMÁTICA #2: Función en predicado impide uso de índice
EXPLAIN
SELECT name, city, country
FROM `travel-sample`.inventory.hotel
WHERE LOWER(name) = "the westin new york grand central";
```

**Instrucciones:**

1. Ejecuta el `EXPLAIN` y observa que aparece `PrimaryScan3` o `IndexScan3` con un `Filter` residual que aplica `LOWER()` sobre cada documento.
2. Ejecuta la consulta sin `EXPLAIN` y registra el tiempo.

#### 2.2 Solución: Reescribir la consulta o crear índice funcional

```sql
-- OPCIÓN A: Reescribir la consulta (preferida cuando el dato es consistente)
SELECT name, city, country
FROM `travel-sample`.inventory.hotel
WHERE name = "The Westin New York Grand Central";

-- OPCIÓN B: Crear un índice sobre la expresión funcional
CREATE INDEX idx_hotel_name_lower
ON `travel-sample`.inventory.hotel(LOWER(name));

-- Consulta que aprovecha el índice funcional
EXPLAIN
SELECT name, city, country
FROM `travel-sample`.inventory.hotel
WHERE LOWER(name) = "the westin new york grand central";
```

3. Ejecuta ambas opciones con `EXPLAIN` y compara los planes.
4. Registra las métricas de la Opción B en tu tabla comparativa.

**Verificación:**

```sql
-- El plan de la Opción B debe mostrar IndexScan3 sobre idx_hotel_name_lower
-- con el span aplicado directamente sobre LOWER(name)
EXPLAIN
SELECT name, city, country
FROM `travel-sample`.inventory.hotel
WHERE LOWER(name) = "the westin new york grand central";
```

**Resultado esperado:** El plan muestra `IndexScan3` con `"index": "idx_hotel_name_lower"` y el span contiene el valor en minúsculas directamente.

---

### Paso 3 — Anti-patrón: SELECT * vs proyección específica

**Objetivo:** Comparar el costo de recuperar todos los campos versus solo los campos necesarios, y entender cuándo un índice cubriente elimina el operador `Fetch`.

#### 3.1 Consulta con SELECT *

```sql
-- CONSULTA PROBLEMÁTICA #3: SELECT * fuerza Fetch de documento completo
EXPLAIN
SELECT *
FROM `travel-sample`.inventory.hotel
WHERE country = "United States"
  AND avg_rating > 4.0
LIMIT 20;
```

**Instrucciones:**

1. Ejecuta el `EXPLAIN` y confirma la presencia del operador `Fetch`.
2. Ejecuta la consulta sin `EXPLAIN` y registra el tiempo.

#### 3.2 Crear índice cubriente y usar proyección específica

```sql
-- Crear índice cubriente para los campos que realmente necesitamos
CREATE INDEX idx_hotel_country_rating_covering
ON `travel-sample`.inventory.hotel(country, avg_rating, name, city)
WHERE country IS NOT MISSING;

-- Consulta optimizada con proyección específica
EXPLAIN
SELECT name, city, avg_rating
FROM `travel-sample`.inventory.hotel
WHERE country = "United States"
  AND avg_rating > 4.0
LIMIT 20;
```

3. Ejecuta el `EXPLAIN` de la consulta optimizada.
4. Verifica que el operador `Fetch` **no aparece** en el plan — esto confirma que el índice es cubriente.

**Salida esperada (plan optimizado):**

```json
{
  "#operator": "Sequence",
  "~children": [
    {
      "#operator": "IndexScan3",
      "index": "idx_hotel_country_rating_covering",
      "covers": ["cover((`hotel`.`country`))", "cover((`hotel`.`avg_rating`))", "..."]
    },
    {
      "#operator": "Limit",
      "expr": "20"
    },
    {
      "#operator": "InitialProject"
    }
  ]
}
```

**Verificación:** El plan NO debe contener `"#operator": "Fetch"`. Si `Fetch` está ausente, el índice es cubriente.

---

### Paso 4 — Anti-patrón: ORDER BY sin índice ordenado

**Objetivo:** Demostrar el costo del operador `Order` cuando Couchbase debe ordenar en memoria, y cómo un índice ordenado puede eliminar este paso.

#### 4.1 Consulta con ORDER BY sin soporte de índice

```sql
-- CONSULTA PROBLEMÁTICA #4: Ordenamiento en memoria
EXPLAIN
SELECT name, avg_rating, city
FROM `travel-sample`.inventory.hotel
WHERE country = "France"
ORDER BY avg_rating DESC
LIMIT 10;
```

**Instrucciones:**

1. Ejecuta el `EXPLAIN` y observa la presencia del operador `Order` en el plan.
2. Ejecuta la consulta y registra el tiempo, especialmente si hay muchos documentos para Francia.

#### 4.2 Crear índice con orden explícito

```sql
-- Índice con orden DESC que coincide con la consulta
CREATE INDEX idx_hotel_country_rating_desc
ON `travel-sample`.inventory.hotel(country, avg_rating DESC, name);

-- Re-verificar el plan
EXPLAIN
SELECT name, avg_rating, city
FROM `travel-sample`.inventory.hotel
WHERE country = "France"
ORDER BY avg_rating DESC
LIMIT 10;
```

3. Busca en el nuevo plan si el operador `Order` desaparece o si aparece una nota de **index order** — esto indica que Couchbase puede usar el orden del índice directamente.
4. Verifica también si el `LIMIT` se convierte en un **pushdown** (aparece antes del `Fetch` en el plan).

**Verificación:**

```sql
-- La combinación ORDER BY + LIMIT con índice ordenado debe mostrar
-- "limit": "10" aplicado directamente en el IndexScan (pushdown)
EXPLAIN
SELECT name, avg_rating, city
FROM `travel-sample`.inventory.hotel
WHERE country = "France"
ORDER BY avg_rating DESC
LIMIT 10;
-- Buscar: ausencia de "#operator": "Order" o presencia de "index_order": true
```

---

### Paso 5 — Anti-patrón: LIMIT sin pushdown

**Objetivo:** Entender cuándo `LIMIT` se aplica tardíamente (después de procesar todos los resultados) y cómo forzar su pushdown al nivel del índice.

#### 5.1 Consulta donde LIMIT no hace pushdown

```sql
-- CONSULTA PROBLEMÁTICA #5: LIMIT sin pushdown por función en SELECT
EXPLAIN
SELECT name, UPPER(city) AS city_upper, avg_rating
FROM `travel-sample`.inventory.hotel
WHERE country = "United Kingdom"
ORDER BY avg_rating DESC
LIMIT 5;
```

**Instrucciones:**

1. Ejecuta el `EXPLAIN`. Observa que `UPPER(city)` en el `SELECT` no impide el pushdown del `LIMIT` al índice si el índice cubre `country` y `avg_rating`.
2. Ahora prueba este caso donde el LIMIT sí se retrasa:

```sql
-- Caso donde agregación impide pushdown de LIMIT
EXPLAIN
SELECT country, COUNT(*) AS total, AVG(avg_rating) AS avg_score
FROM `travel-sample`.inventory.hotel
WHERE country IS NOT MISSING
GROUP BY country
ORDER BY total DESC
LIMIT 5;
```

3. Observa que el `LIMIT` aparece **después** del operador `Aggregate` — no puede hacerse pushdown porque el conteo requiere procesar todos los grupos primero.

#### 5.2 Optimización: índice cubriente para la agregación

```sql
-- Índice cubriente para la consulta de agregación
CREATE INDEX idx_hotel_country_agg
ON `travel-sample`.inventory.hotel(country, avg_rating)
WHERE country IS NOT MISSING;

-- Re-verificar plan
EXPLAIN
SELECT country, COUNT(*) AS total, AVG(avg_rating) AS avg_score
FROM `travel-sample`.inventory.hotel
WHERE country IS NOT MISSING
GROUP BY country
ORDER BY total DESC
LIMIT 5;
```

4. Con el índice cubriente, el plan debe mostrar `IndexScan3` sin `Fetch`, lo que significa que la agregación opera sobre datos del índice (más rápido), aunque el `LIMIT` sigue aplicándose después del `Aggregate`.
5. Registra la mejora de tiempo.

---

### Paso 6 — Anti-patrón: JOIN sin índice en campo de unión

**Objetivo:** Demostrar el impacto de un `JOIN` entre colecciones cuando el campo de unión no está indexado.

#### 6.1 Consulta de JOIN problemática

```sql
-- CONSULTA PROBLEMÁTICA #6: JOIN sin índice en campo de unión
EXPLAIN
SELECT r.sourceairport, r.destinationairport, r.distance,
       al.name AS airline_name
FROM `travel-sample`.inventory.route AS r
JOIN `travel-sample`.inventory.airline AS al
  ON r.airlineid = META(al).id
WHERE r.sourceairport = "SFO"
  AND r.distance > 1000
LIMIT 20;
```

**Instrucciones:**

1. Ejecuta el `EXPLAIN` y observa el tipo de join utilizado (`NestedLoopJoin` o `HashJoin`).
2. Identifica si hay `IndexScan3` en ambas ramas del join o si alguna usa `PrimaryScan3`.
3. Ejecuta la consulta y registra el tiempo.

#### 6.2 Crear índices para ambas ramas del JOIN

```sql
-- Índice en la tabla izquierda (route) para el predicado de filtro
CREATE INDEX idx_route_sourceairport_distance
ON `travel-sample`.inventory.route(sourceairport, distance, airlineid)
WHERE sourceairport IS NOT MISSING;

-- El índice primario de airline ya existe; verificar
SELECT name, state FROM system:indexes
WHERE keyspace_id = "airline" AND `using` = "#primary";

-- Re-verificar plan del JOIN
EXPLAIN
SELECT r.sourceairport, r.destinationairport, r.distance,
       al.name AS airline_name
FROM `travel-sample`.inventory.route AS r
JOIN `travel-sample`.inventory.airline AS al
  ON r.airlineid = META(al).id
WHERE r.sourceairport = "SFO"
  AND r.distance > 1000
LIMIT 20;
```

4. Confirma que el plan ahora muestra `IndexScan3` en la rama de `route` con el índice recién creado.
5. Registra la mejora de tiempo.

**Verificación:**

```sql
-- El plan optimizado debe mostrar IndexScan3 para route
-- con "index": "idx_route_sourceairport_distance"
-- y el LIMIT aplicado con pushdown
```

---

### Paso 7 — Anti-patrón: LIKE con wildcard inicial

**Objetivo:** Demostrar por qué `LIKE '%texto%'` no puede usar un índice de árbol B y explorar alternativas.

#### 7.1 Consulta con LIKE y wildcard inicial

```sql
-- CONSULTA PROBLEMÁTICA #7: LIKE '%prefix%' no puede usar índice B-Tree
EXPLAIN
SELECT name, city, country
FROM `travel-sample`.inventory.hotel
WHERE name LIKE "%Marriott%";
```

**Instrucciones:**

1. Ejecuta el `EXPLAIN`. Confirma que aunque exista un índice sobre `name`, el plan muestra `IndexScan3` o `PrimaryScan3` seguido de un `Filter` que aplica el `LIKE` sobre cada documento (no hay pushdown del predicado al índice).
2. Ejecuta la consulta y registra el tiempo.

#### 7.2 Alternativa A: LIKE con prefijo fijo (sin wildcard inicial)

```sql
-- Si el caso de uso lo permite, usar prefijo fijo es mucho más eficiente
CREATE INDEX idx_hotel_name_btree
ON `travel-sample`.inventory.hotel(name)
WHERE name IS NOT MISSING;

-- Esta consulta SÍ puede usar el índice con pushdown
EXPLAIN
SELECT name, city, country
FROM `travel-sample`.inventory.hotel
WHERE name LIKE "Marriott%";
-- El span del índice irá desde "Marriott" hasta "Marriott\u{FFFF}"
```

#### 7.3 Alternativa B: Full Text Search para búsqueda de subcadenas

```sql
-- Para búsquedas de subcadenas, FTS es la solución correcta
-- (referencia al Lab de FTS — aquí solo documentamos el anti-patrón)
-- La consulta LIKE "%Marriott%" requiere FTS o procesamiento externo
-- No hay optimización posible con índices B-Tree para wildcard inicial
```

3. Documenta en tu tabla comparativa que `LIKE '%texto%'` es un caso donde el **anti-patrón no tiene solución con índices B-Tree** y debe redirigirse a Full Text Search.

---

### Paso 8 — Agregación sin índice cubriente

**Objetivo:** Optimizar una consulta de agregación asegurando que todos los campos necesarios estén en el índice para eliminar el operador `Fetch`.

#### 8.1 Consulta de agregación problemática

```sql
-- CONSULTA PROBLEMÁTICA #8: Agregación con Fetch innecesario
EXPLAIN
SELECT country,
       COUNT(*) AS total_hotels,
       AVG(avg_rating) AS avg_score,
       MAX(avg_rating) AS max_score,
       MIN(avg_rating) AS min_score
FROM `travel-sample`.inventory.hotel
WHERE country IS NOT MISSING
  AND avg_rating IS NOT MISSING
GROUP BY country
HAVING COUNT(*) > 5
ORDER BY avg_score DESC;
```

**Instrucciones:**

1. Ejecuta el `EXPLAIN` y observa si hay un operador `Fetch` antes del `Aggregate`.
2. Ejecuta la consulta y registra el tiempo.

#### 8.2 Índice cubriente para la agregación

```sql
-- Índice cubriente: incluye todos los campos usados en SELECT, WHERE, GROUP BY y HAVING
CREATE INDEX idx_hotel_agg_covering
ON `travel-sample`.inventory.hotel(country, avg_rating)
WHERE country IS NOT MISSING AND avg_rating IS NOT MISSING;

-- Re-verificar plan
EXPLAIN
SELECT country,
       COUNT(*) AS total_hotels,
       AVG(avg_rating) AS avg_score,
       MAX(avg_rating) AS max_score,
       MIN(avg_rating) AS min_score
FROM `travel-sample`.inventory.hotel
WHERE country IS NOT MISSING
  AND avg_rating IS NOT MISSING
GROUP BY country
HAVING COUNT(*) > 5
ORDER BY avg_score DESC;
```

3. Confirma que el nuevo plan **no contiene `Fetch`** — la agregación opera directamente sobre los datos del índice.
4. Registra la mejora de tiempo.

**Verificación:**

```sql
-- Buscar en el plan: ausencia de "#operator": "Fetch"
-- Presencia de "covers" en el IndexScan3 que incluye country y avg_rating
```

---

### Paso 9 — Orden de campos en índice compuesto

**Objetivo:** Demostrar que el orden de los campos en un índice compuesto determina si el índice puede ser utilizado para un predicado dado.

#### 9.1 Índice con orden incorrecto de campos

```sql
-- Crear índice con orden subóptimo (campo de alta cardinalidad primero, pero predicado de igualdad segundo)
CREATE INDEX idx_hotel_wrong_order
ON `travel-sample`.inventory.hotel(avg_rating, country);

-- Consulta que busca por country (igualdad) y avg_rating (rango)
EXPLAIN
SELECT name, avg_rating
FROM `travel-sample`.inventory.hotel
WHERE country = "Spain"
  AND avg_rating BETWEEN 3.5 AND 5.0
ORDER BY avg_rating;
```

**Instrucciones:**

1. Ejecuta el `EXPLAIN` con `idx_hotel_wrong_order`. Observa que el índice puede ser utilizado pero de forma menos eficiente (el span cubre un rango amplio de `avg_rating` y luego filtra `country`).
2. Ahora crea el índice con el orden correcto:

```sql
-- Regla: campos de igualdad primero, campos de rango después
CREATE INDEX idx_hotel_correct_order
ON `travel-sample`.inventory.hotel(country, avg_rating);

-- Re-verificar plan
EXPLAIN
SELECT name, avg_rating
FROM `travel-sample`.inventory.hotel
WHERE country = "Spain"
  AND avg_rating BETWEEN 3.5 AND 5.0
ORDER BY avg_rating;
```

3. Compara los spans en ambos planes. Con el orden correcto, el span de `country` es de igualdad exacta y el span de `avg_rating` es el rango — esto es mucho más selectivo.
4. Ejecuta ambas versiones de la consulta y registra los tiempos.

**Verificación:**

```sql
-- Verificar qué índice elige el optimizador cuando ambos existen
EXPLAIN
SELECT name, avg_rating
FROM `travel-sample`.inventory.hotel
WHERE country = "Spain"
  AND avg_rating BETWEEN 3.5 AND 5.0
ORDER BY avg_rating;
-- El optimizador debe preferir idx_hotel_correct_order
-- Buscar: "index": "idx_hotel_correct_order" en el plan
```

---

### Paso 10 — Consulta compleja: combinando múltiples anti-patrones

**Objetivo:** Aplicar todas las técnicas aprendidas en una consulta compleja que combina varios anti-patrones simultáneamente.

#### 10.1 Consulta con múltiples anti-patrones

```sql
-- CONSULTA PROBLEMÁTICA #10: Múltiples anti-patrones combinados
EXPLAIN
SELECT *
FROM `travel-sample`.inventory.hotel AS h
WHERE LOWER(h.country) = "united states"
  AND h.avg_rating > 4.0
  AND h.free_breakfast = true
ORDER BY h.name
LIMIT 10;
```

**Instrucciones:**

1. Ejecuta el `EXPLAIN` e identifica **todos** los anti-patrones presentes:
   - `SELECT *` (fuerza `Fetch`)
   - `LOWER()` en predicado (impide uso eficiente del índice)
   - `ORDER BY` sin índice ordenado
   - Múltiples predicados sin índice compuesto óptimo

2. Reescribe la consulta corrigiendo todos los anti-patrones:

```sql
-- CONSULTA OPTIMIZADA #10: Todos los anti-patrones corregidos
-- Paso A: Crear índice cubriente con orden correcto
CREATE INDEX idx_hotel_us_optimized
ON `travel-sample`.inventory.hotel(country, avg_rating, free_breakfast, name)
WHERE country IS NOT MISSING;

-- Paso B: Reescribir la consulta
EXPLAIN
SELECT h.name, h.city, h.avg_rating, h.free_breakfast
FROM `travel-sample`.inventory.hotel AS h
WHERE h.country = "United States"
  AND h.avg_rating > 4.0
  AND h.free_breakfast = true
ORDER BY h.name
LIMIT 10;
```

3. Ejecuta ambas versiones (original y optimizada) sin `EXPLAIN` y registra los tiempos.
4. Verifica que el plan optimizado no contiene `Fetch`, usa `IndexScan3` con el nuevo índice, y que el `LIMIT` tiene pushdown.

---

## 7. Validación y Pruebas

### Tabla Comparativa de Resultados

Completa esta tabla con los valores registrados durante el laboratorio:

| # | Descripción del Anti-patrón | Tiempo Original (ms) | Operador Problemático | Optimización Aplicada | Tiempo Optimizado (ms) | Mejora % |
|---|---------------------------|---------------------|----------------------|----------------------|----------------------|---------|
| 1 | Sin índice secundario | ___ | PrimaryScan3 | Índice compuesto | ___ | ___ |
| 2 | LOWER() en WHERE | ___ | Filter residual | Índice funcional | ___ | ___ |
| 3 | SELECT * | ___ | Fetch innecesario | Índice cubriente | ___ | ___ |
| 4 | ORDER BY sin índice | ___ | Order en memoria | Índice con DESC | ___ | ___ |
| 5 | LIMIT sin pushdown | ___ | Limit tardío | Índice cubriente | ___ | ___ |
| 6 | JOIN sin índice | ___ | NestedLoopJoin caro | Índice en campo JOIN | ___ | ___ |
| 7 | LIKE '%texto%' | ___ | Filter full-scan | FTS (sin solución B-Tree) | N/A | N/A |
| 8 | Agregación sin cubriente | ___ | Fetch + Aggregate | Índice cubriente agg | ___ | ___ |
| 9 | Orden incorrecto índice | ___ | Span ineficiente | Orden correcto (=, rango) | ___ | ___ |
| 10 | Múltiples anti-patrones | ___ | Varios | Índice cubriente completo | ___ | ___ |

**Fórmula para calcular mejora porcentual:**

```
Mejora % = ((Tiempo Original - Tiempo Optimizado) / Tiempo Original) × 100
```

### Verificación final con system:completed_requests

```sql
-- Ver las últimas consultas ejecutadas con sus tiempos
SELECT statement,
       requestTime,
       elapsedTime,
       executionTime,
       resultCount,
       serviceLoad
FROM system:completed_requests
WHERE statement NOT LIKE "SELECT%system%"
  AND requestTime > NOW_STR()
ORDER BY requestTime DESC
LIMIT 20;
```

> **Nota:** `system:completed_requests` requiere que el servicio de Query tenga habilitado el tracking de consultas completadas. Si la tabla está vacía, verifica en **Settings → Query Settings → Completed Requests** que esté habilitado.

### Verificación del estado de índices creados

```sql
-- Listar todos los índices creados en este laboratorio
SELECT name, keyspace_id, state, `using`,
       index_key, condition
FROM system:indexes
WHERE name IN (
  "idx_hotel_rating_breakfast",
  "idx_hotel_name_lower",
  "idx_hotel_country_rating_covering",
  "idx_hotel_country_rating_desc",
  "idx_hotel_country_agg",
  "idx_hotel_agg_covering",
  "idx_hotel_wrong_order",
  "idx_hotel_correct_order",
  "idx_route_sourceairport_distance",
  "idx_hotel_us_optimized"
)
ORDER BY name;
-- Todos deben tener state = "online"
```

---

## 8. Resolución de Problemas

### Problema 1: EXPLAIN muestra PrimaryScan3 aunque existe un índice secundario

**Síntoma:** Después de crear un índice secundario, el `EXPLAIN` continúa mostrando `PrimaryScan3` o usa un índice diferente al esperado.

**Causa:** El índice puede no estar en estado `online` aún (está en `building` o `deferred`), los campos del índice no coinciden exactamente con los predicados de la consulta (diferencia de mayúsculas, campo anidado vs. plano), o el optimizador determina que el índice primario es más eficiente para conjuntos de resultados muy grandes (selectividad baja).

**Solución:**

```sql
-- 1. Verificar el estado del índice
SELECT name, state, keyspace_id
FROM system:indexes
WHERE name = "idx_hotel_rating_breakfast";
-- state debe ser "online", no "building" o "pending"

-- 2. Forzar el uso de un índice específico con USE INDEX hint
EXPLAIN
SELECT name, city, avg_rating
FROM `travel-sample`.inventory.hotel
  USE INDEX (idx_hotel_rating_breakfast USING GSI)
WHERE avg_rating > 4.5
  AND free_breakfast = true;

-- 3. Si el índice está en estado deferred, construirlo manualmente
BUILD INDEX ON `travel-sample`.inventory.hotel(idx_hotel_rating_breakfast) USING GSI;
```

---

### Problema 2: Las métricas de tiempo son inconsistentes entre ejecuciones

**Síntoma:** Al ejecutar la misma consulta varias veces, los tiempos varían significativamente (por ejemplo, 200ms, 50ms, 180ms, 45ms en ejecuciones consecutivas), dificultando la comparación antes/después.

**Causa:** Las primeras ejecuciones de una consulta realizan el calentamiento de caché del Data Service (KV store). Las ejecuciones subsiguientes se benefician del caché de datos en memoria. Además, la carga del sistema y el garbage collector del JVM de Couchbase pueden introducir variabilidad.

**Solución:**

```sql
-- 1. Ejecutar cada consulta al menos 3 veces y usar la mediana (no la primera ejecución)
-- Primera ejecución: puede ser lenta por cache miss
-- Segunda y tercera: más representativas del rendimiento real en producción con datos calientes

-- 2. Usar la métrica executionTime (no elapsedTime) para comparaciones
-- elapsedTime incluye tiempo de red y serialización
-- executionTime es el tiempo puro del motor de consultas

-- 3. Para comparaciones más precisas, revisar el plan en lugar del tiempo absoluto:
-- Contar el número de documentos escaneados (resultCount del IndexScan)
-- Un IndexScan que escanea 10 docs es objetivamente mejor que uno que escanea 900

-- 4. Verificar la métrica "usedMemory" para consultas con ORDER BY y GROUP BY
-- Si usedMemory es alto, el operador está spilling a disco (muy costoso)
SELECT statement, executionTime, resultCount, usedMemory
FROM system:completed_requests
ORDER BY requestTime DESC
LIMIT 5;
```

---

## 9. Limpieza del Entorno

Al finalizar el laboratorio, puedes optar por mantener los índices optimizados (recomendado) o limpiar los índices de práctica. Los índices consumen memoria del Index Service, por lo que es buena práctica eliminar los que no se usarán.

```sql
-- Eliminar índices subóptimos creados para demostración
DROP INDEX `travel-sample`.inventory.hotel.idx_hotel_wrong_order IF EXISTS;

-- Mantener los índices optimizados para futuros laboratorios
-- idx_hotel_rating_breakfast
-- idx_hotel_country_rating_covering
-- idx_hotel_correct_order
-- idx_route_sourceairport_distance
-- idx_hotel_agg_covering

-- Verificar índices restantes
SELECT name, keyspace_id, state
FROM system:indexes
WHERE bucket_id = "travel-sample"
   OR keyspace_id IN ("hotel","airline","airport","route","landmark")
ORDER BY keyspace_id, name;
```

> **Nota:** Los índices `idx_hotel_us_optimized`, `idx_hotel_country_rating_desc` y `idx_hotel_agg_covering` son útiles para los laboratorios posteriores. Se recomienda mantenerlos.

---

## 10. Resumen

### Conceptos Clave Aplicados

En este laboratorio aplicaste el ciclo diagnóstico completo de optimización de consultas SQL++ en Couchbase:

| Concepto | Lo que aprendiste |
|----------|-------------------|
| **EXPLAIN** | Leer el árbol de operadores JSON para identificar `PrimaryScan3`, `Fetch`, `Order` y otros operadores costosos |
| **Índice cubriente** | Eliminar el operador `Fetch` incluyendo todos los campos necesarios en el índice |
| **Índice funcional** | Resolver el anti-patrón de funciones en `WHERE` (e.g., `LOWER()`) creando un índice sobre la expresión |
| **Orden de campos** | Colocar campos de igualdad antes que campos de rango en índices compuestos |
| **Pushdown de LIMIT/ORDER BY** | Crear índices ordenados que permitan al motor aplicar `LIMIT` en el nivel del índice |
| **JOIN optimizado** | Asegurar índices en los campos de unión de ambas colecciones |
| **LIKE con wildcard inicial** | Reconocer que `LIKE '%texto%'` no tiene solución con índices B-Tree; usar Full Text Search |
| **Métricas de comparación** | Usar `executionTime`, `resultCount` y ausencia de `Fetch` como indicadores objetivos de mejora |

### Reglas de Oro del Query Tuning en Couchbase

1. **Si ves `PrimaryScan3`**, crea un índice secundario en los campos del `WHERE`.
2. **Si ves `Fetch` innecesario**, convierte el índice en cubriente añadiendo los campos del `SELECT`.
3. **Si ves `Order` en el plan**, considera un índice con el mismo orden que el `ORDER BY`.
4. **Nunca apliques funciones en predicados `WHERE`** sobre campos no indexados funcionalmente.
5. **En índices compuestos**, los campos de igualdad van primero, los de rango van al final.
6. **`SELECT *` siempre fuerza `Fetch`** — proyecta solo los campos que necesitas.
7. **`LIKE '%texto%`** requiere Full Text Search, no índices B-Tree.

### Recursos Adicionales

- [Documentación oficial: EXPLAIN en SQL++](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/explain.html)
- [Documentación oficial: Covering Indexes](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/covering-indexes.html)
- [Documentación oficial: Index Pushdowns](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/index-pushdowns.html)
- [Documentación oficial: Cost-Based Optimizer](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/cost-based-optimizer.html)
- [Documentación oficial: system:completed_requests](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/systeminformation.html)
- [Blog: Query Tuning Best Practices en Couchbase](https://www.couchbase.com/blog/n1ql-query-performance-tuning/)

---
*Lab 05-00-01 — Versión 1.0 — Couchbase Server 7.6.x — Dataset: travel-sample*
