# Creación y validación de índices en Couchbase

## 1. Metadatos

| Atributo | Valor |
|---|---|
| **Duración estimada** | 70 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (*Apply*) |
| **Dataset requerido** | `travel-sample` |
| **Servicio Couchbase** | Query Service + Index Service |

---

## 2. Descripción General

En este laboratorio los estudiantes crearán y compararán de forma progresiva los principales tipos de índices disponibles en Couchbase SQL++: primarios, secundarios simples, compuestos, parciales y de arreglos. A partir del dataset `travel-sample`, se explorarán técnicas avanzadas como el particionamiento de índices con `PARTITION BY HASH`, la modificación de propiedades con `ALTER INDEX` y la administración visual del servicio de índices desde la Web Console. Al finalizar, el estudiante contará con una tabla de referencia completa de todos los índices creados y comprenderá cómo cada tipo impacta el rendimiento de las consultas.

---

## 3. Objetivos de Aprendizaje

Al completar este laboratorio, el estudiante será capaz de:

- [ ] Crear índices primarios, secundarios, compuestos, parciales y de arreglos sobre colecciones del dataset `travel-sample` usando SQL++
- [ ] Implementar un índice particionado con `PARTITION BY HASH` y verificar la distribución de particiones mediante `system:indexes`
- [ ] Usar `ALTER INDEX` para modificar propiedades de índices existentes (número de réplicas, estado)
- [ ] Monitorear el estado, tamaño en disco, uso de memoria y número de solicitudes de cada índice desde el panel *Index Service* de la Web Console
- [ ] Comparar las configuraciones de almacenamiento de índices MOI (*Memory Optimized*) y Plasma, identificando sus diferencias y casos de uso

---

## 4. Prerrequisitos

### Conocimiento previo

- Haber completado el **Lab 03-00-01** o tener experiencia básica con consultas SQL++ (`SELECT`, `WHERE`, `FROM`)
- Comprensión de conceptos básicos de índices en bases de datos relacionales (B-tree, cobertura de índice, selectividad)
- Familiaridad con la Web Console de Couchbase (navegación básica entre servicios)

### Acceso y recursos

- Couchbase Server 7.6.x en ejecución (nodo único / single-node cluster)
- Dataset `travel-sample` cargado y disponible (verificar en **Buckets** → `travel-sample`)
- Acceso a la Web Console en `http://localhost:8091`
- Acceso a `cbq` (Couchbase Query Shell) o al editor de consultas de la Web Console
- Usuario administrador con permisos para crear y modificar índices

---

## 5. Entorno de Laboratorio

### Hardware mínimo recomendado

| Recurso | Mínimo | Recomendado |
|---|---|---|
| RAM | 8 GB | 16 GB |
| CPU | 4 núcleos x86_64 | 8 núcleos |
| Almacenamiento | 20 GB libres (SSD) | 50 GB libres (SSD) |
| Red | localhost habilitado | localhost habilitado |

> **⚠️ Nota sobre memoria del Index Service:** Este laboratorio requiere que el servicio Index tenga asignado **al menos 512 MB de RAM**. En sistemas con 8 GB, puede ser necesario reducir la memoria asignada al servicio Data. Verificar en **Settings → Cluster → Index Service Memory Quota**.

### Software requerido

| Software | Versión | Uso en este lab |
|---|---|---|
| Couchbase Server | 7.6.x | Motor principal |
| Navegador web | Chrome/Firefox/Edge 110+ | Web Console |
| `cbq` (Query Shell) | Incluido con Couchbase 7.6.x | Ejecución de SQL++ |
| `curl` | 7.x o superior | Verificaciones REST opcionales |

### Verificación del entorno antes de comenzar

Ejecutar los siguientes comandos para confirmar que el entorno está listo:

```bash
# 1. Verificar que Couchbase está en ejecución
curl -s -u Administrator:password http://localhost:8091/pools/default | python3 -m json.tool | grep '"name"'

# 2. Acceder a cbq (Query Shell)
cbq -u Administrator -p password -engine=http://localhost:8093

# 3. Dentro de cbq: verificar que travel-sample está disponible
\cbq> SELECT COUNT(*) FROM `travel-sample`.inventory.airline;
```

**Resultado esperado de la verificación:**

```json
{
    "results": [
        {
            "$1": 187
        }
    ]
}
```

Si el conteo devuelve 187, el dataset `travel-sample` está correctamente cargado y el laboratorio puede comenzar.

---

## 6. Pasos del Laboratorio

---

### Parte 1 — Revisión del estado inicial de índices

**Objetivo:** Examinar los índices existentes en el clúster usando el catálogo del sistema `system:indexes` antes de crear nuevos índices.

#### Instrucciones

1. Abrir la Web Console en `http://localhost:8091` e iniciar sesión con las credenciales de administrador.

2. Navegar a **Query** en el menú lateral para abrir el editor de consultas SQL++.

3. Ejecutar la siguiente consulta para listar todos los índices existentes en el bucket `travel-sample`:

```sql
SELECT name, keyspace_id, namespace_id, scope_id, bucket_id,
       `using`, state, `index_key`, condition
FROM system:indexes
WHERE bucket_id = "travel-sample"
ORDER BY scope_id, keyspace_id, name;
```

4. Registrar en la **Tabla de Referencia de Índices** (ver Sección 6.7) los índices que ya existen antes de comenzar el laboratorio.

5. Ejecutar también la siguiente consulta para ver el resumen por colección:

```sql
SELECT keyspace_id AS coleccion,
       COUNT(*) AS total_indices,
       ARRAY_AGG(name) AS nombres
FROM system:indexes
WHERE bucket_id = "travel-sample"
GROUP BY keyspace_id
ORDER BY keyspace_id;
```

#### Resultado esperado

La consulta debe mostrar los índices precargados con el dataset `travel-sample`. Típicamente se encontrará al menos un índice primario en cada colección principal. El campo `state` debe mostrar `"online"` para todos los índices activos.

```
Ejemplo de fila en el resultado:
{
  "coleccion": "airline",
  "total_indices": 1,
  "nombres": ["def_primary"]
}
```

#### Verificación

✅ La consulta a `system:indexes` devuelve filas sin errores.
✅ Se identifican las colecciones disponibles: `airline`, `airport`, `hotel`, `landmark`, `route`.
✅ Se registra el número inicial de índices en la Tabla de Referencia.

---

### Parte 2 — Creación de índices: primario, secundario, compuesto, parcial y de arreglo

**Objetivo:** Crear cinco tipos diferentes de índices sobre las colecciones del dataset `travel-sample`, comprendiendo el propósito de cada uno.

#### 2.1 Índice Primario (solo para referencia y pruebas)

```sql
-- NOTA: Los índices primarios existen en el dataset por defecto.
-- Crear uno adicional con nombre explícito para observar su comportamiento.
CREATE PRIMARY INDEX idx_lab_primary_airline
ON `travel-sample`.inventory.airline
WITH {"defer_build": false};
```

> **⚠️ Advertencia:** Los índices primarios realizan *full scans* en producción. Este índice se crea únicamente con fines educativos y será eliminado en la sección de Cleanup.

**Verificar creación:**

```sql
SELECT name, state, `index_key`
FROM system:indexes
WHERE name = "idx_lab_primary_airline";
```

#### 2.2 Índice Secundario Simple

```sql
-- Índice secundario sobre el campo "country" en la colección airline
CREATE INDEX idx_airline_country
ON `travel-sample`.inventory.airline(country)
WITH {"defer_build": false};
```

**Probar el índice:**

```sql
-- Esta consulta debe usar idx_airline_country (verificar con EXPLAIN)
SELECT name, callsign, country
FROM `travel-sample`.inventory.airline
WHERE country = "United States"
LIMIT 5;
```

#### 2.3 Índice Compuesto (múltiples campos)

```sql
-- Índice compuesto sobre "country" y "callsign"
-- El campo de mayor selectividad (country) va primero
CREATE INDEX idx_airline_country_callsign
ON `travel-sample`.inventory.airline(country, callsign)
WITH {"defer_build": false};
```

**Probar el índice:**

```sql
-- Consulta que aprovecha ambos campos del índice compuesto
SELECT name, callsign
FROM `travel-sample`.inventory.airline
WHERE country = "United States"
  AND callsign IS NOT NULL
ORDER BY callsign
LIMIT 10;
```

#### 2.4 Índice Parcial (con cláusula WHERE)

```sql
-- Índice parcial: solo indexa aerolíneas (type = "airline")
-- Reduce el tamaño del índice al excluir otros tipos de documentos
CREATE INDEX idx_airline_name_partial
ON `travel-sample`.inventory.airline(name)
WHERE type = "airline"
WITH {"defer_build": false};
```

**Probar el índice:**

```sql
-- Esta consulta debe usar el índice parcial
SELECT name, country
FROM `travel-sample`.inventory.airline
WHERE type = "airline"
  AND name LIKE "A%"
ORDER BY name
LIMIT 10;
```

#### 2.5 Índice de Arreglo (DISTINCT ARRAY)

```sql
-- Índice sobre los elementos del arreglo "schedule" en la colección route
-- Permite buscar rutas que operan en un día específico de la semana
CREATE INDEX idx_route_schedule_day
ON `travel-sample`.inventory.route
    (DISTINCT ARRAY s.day FOR s IN schedule END)
WITH {"defer_build": false};
```

**Probar el índice:**

```sql
-- Buscar rutas que operan el lunes (day = 1)
SELECT sourceairport, destinationairport, airline
FROM `travel-sample`.inventory.route
WHERE ANY s IN schedule SATISFIES s.day = 1 END
LIMIT 10;
```

#### 2.6 Índice Funcional (expresión LOWER)

```sql
-- Índice funcional para búsquedas case-insensitive por nombre de aerolínea
CREATE INDEX idx_airline_name_lower
ON `travel-sample`.inventory.airline(LOWER(name))
WITH {"defer_build": false};
```

**Probar el índice:**

```sql
-- Búsqueda sin distinción de mayúsculas/minúsculas
SELECT name, country, callsign
FROM `travel-sample`.inventory.airline
WHERE LOWER(name) = "delta air lines";
```

#### Resultado esperado (Parte 2)

Cada sentencia `CREATE INDEX` debe completarse sin errores. Al ejecutar:

```sql
SELECT name, state
FROM system:indexes
WHERE bucket_id = "travel-sample"
  AND name LIKE "idx_%"
ORDER BY name;
```

Se deben ver los 6 índices creados con `state = "online"`.

#### Verificación

✅ Los 6 índices aparecen en `system:indexes` con estado `"online"`.
✅ Cada consulta de prueba devuelve resultados (no errores de "no index available").
✅ Los índices se registran en la Tabla de Referencia con su propósito.

---

### Parte 3 — Índice Particionado con PARTITION BY HASH

**Objetivo:** Crear un índice particionado y verificar la distribución de particiones usando el catálogo del sistema.

#### Instrucciones

1. Crear un índice particionado sobre la colección `route`. El particionamiento distribuye el índice en múltiples particiones lógicas basadas en el valor hash de la clave indicada:

```sql
-- Índice particionado sobre sourceairport en la colección route
-- En un clúster single-node, todas las particiones residen en el mismo nodo,
-- pero la estructura interna sigue siendo particionada
CREATE INDEX idx_route_source_partitioned
ON `travel-sample`.inventory.route(sourceairport, destinationairport)
PARTITION BY HASH(sourceairport)
WITH {
    "defer_build": false,
    "num_partition": 8
};
```

2. Verificar la creación del índice particionado:

```sql
SELECT name, state, `index_key`, partition
FROM system:indexes
WHERE name = "idx_route_source_partitioned";
```

3. Consultar las particiones individuales del índice usando `system:index_partitions`:

```sql
SELECT indexName, partitionId, host, state, numDocs
FROM system:index_partitions
WHERE indexName = "idx_route_source_partitioned"
ORDER BY partitionId;
```

4. Probar que el índice particionado es utilizado por el planificador:

```sql
-- Consulta que debe aprovechar el índice particionado
SELECT sourceairport, destinationairport, airline, distance
FROM `travel-sample`.inventory.route
WHERE sourceairport = "SFO"
ORDER BY distance DESC
LIMIT 10;
```

#### Resultado esperado

La consulta a `system:index_partitions` debe mostrar 8 filas (una por partición), todas con `state = "online"`. El campo `numDocs` indicará cuántos documentos indexados hay en cada partición. En un nodo único, el campo `host` será el mismo para todas las particiones.

```
Ejemplo de salida parcial:
{ "indexName": "idx_route_source_partitioned", "partitionId": 0, "host": "127.0.0.1:8091", "state": "online", "numDocs": 1847 }
{ "indexName": "idx_route_source_partitioned", "partitionId": 1, "host": "127.0.0.1:8091", "state": "online", "numDocs": 1823 }
...
```

> **📝 Nota conceptual:** En un clúster multi-nodo con múltiples nodos de Index Service, las particiones se distribuirían entre los diferentes nodos, mejorando el paralelismo en la resolución de consultas y la capacidad de almacenamiento del índice.

#### Verificación

✅ El índice `idx_route_source_partitioned` aparece en `system:indexes` con `state = "online"`.
✅ `system:index_partitions` muestra exactamente 8 particiones.
✅ La consulta de prueba sobre `sourceairport = "SFO"` devuelve resultados.

---

### Parte 4 — Modificación de índices con ALTER INDEX

**Objetivo:** Usar `ALTER INDEX` para modificar propiedades de índices existentes, incluyendo el número de réplicas y el estado del índice.

#### 4.1 Verificar el estado actual del índice a modificar

```sql
-- Ver propiedades actuales del índice antes de modificarlo
SELECT name, state, `using`, num_replica
FROM system:indexes
WHERE name = "idx_airline_country";
```

#### 4.2 Modificar el número de réplicas

> **⚠️ Nota:** En un clúster single-node, no es posible crear réplicas reales (requieren nodos adicionales de Index Service). El siguiente comando muestra la sintaxis correcta; en un nodo único, Couchbase puede registrar la configuración pero no materializará réplicas adicionales.

```sql
-- Modificar el número de réplicas del índice (sintaxis para clúster multi-nodo)
ALTER INDEX `travel-sample`.inventory.airline.idx_airline_country
WITH {"action": "replica_count", "num_replica": 1};
```

#### 4.3 Poner un índice en estado BUILD (construir índice diferido)

Primero, crear un índice con `defer_build: true` para demostrar el estado `deferred`:

```sql
-- Crear índice con construcción diferida
CREATE INDEX idx_airport_city_deferred
ON `travel-sample`.inventory.airport(city, country)
WITH {"defer_build": true};
```

Verificar que el índice está en estado `deferred`:

```sql
SELECT name, state
FROM system:indexes
WHERE name = "idx_airport_city_deferred";
-- Resultado esperado: state = "deferred"
```

Ahora construir el índice usando `BUILD INDEX`:

```sql
-- Iniciar la construcción del índice diferido
BUILD INDEX ON `travel-sample`.inventory.airport(idx_airport_city_deferred);
```

Verificar que el índice pasa a estado `building` y luego a `online`:

```sql
-- Ejecutar varias veces hasta ver state = "online"
SELECT name, state
FROM system:indexes
WHERE name = "idx_airport_city_deferred";
```

#### 4.4 Pausar y reanudar un índice (Enterprise Edition)

```sql
-- Pausar el índice (solo Enterprise Edition)
-- En Community Edition, este comando puede no estar disponible
ALTER INDEX `travel-sample`.inventory.airline.idx_airline_country_callsign
WITH {"action": "move", "nodes": ["127.0.0.1:8091"]};
```

> **📝 Nota:** `ALTER INDEX` soporta las acciones: `replica_count`, `move` y en versiones Enterprise, gestión avanzada de réplicas. Consultar la documentación oficial para el conjunto completo de acciones disponibles según la edición.

#### Resultado esperado

```sql
-- Verificación final del estado de todos los índices del lab
SELECT name, state, num_replica
FROM system:indexes
WHERE bucket_id = "travel-sample"
  AND name LIKE "idx_%"
ORDER BY name;
```

Todos los índices deben aparecer con `state = "online"` excepto si alguno fue dejado intencionalmente en otro estado para observación.

#### Verificación

✅ El índice `idx_airport_city_deferred` pasó de `deferred` → `building` → `online`.
✅ El comando `ALTER INDEX` se ejecutó sin errores de sintaxis.
✅ Los cambios de configuración son visibles en `system:indexes`.

---

### Parte 5 — Exploración del panel Index Service en la Web Console

**Objetivo:** Navegar por el panel de administración del Index Service para revisar métricas operativas de cada índice: estado, tamaño en disco, uso de memoria y número de solicitudes.

#### Instrucciones

1. En la Web Console (`http://localhost:8091`), navegar a **Indexes** en el menú lateral izquierdo.

2. Observar la vista general del panel. Identificar las siguientes columnas en la tabla de índices:
   - **Index Name**: nombre del índice
   - **Keyspace**: bucket/scope/colección donde reside
   - **Status**: estado actual (`Ready`, `Building`, etc.)
   - **# Docs Indexed**: número de documentos indexados
   - **Data Size**: tamaño de los datos del índice en disco
   - **Memory Used**: memoria RAM consumida por el índice
   - **# Requests**: número de solicitudes de consulta que han usado este índice
   - **# Items Scanned**: número de entradas del índice escaneadas en total

3. Localizar en la lista los índices creados en este laboratorio (prefijo `idx_`). Registrar los valores de las métricas en la Tabla de Referencia.

4. Hacer clic en el nombre de un índice (por ejemplo, `idx_route_schedule_day`) para ver su detalle. Observar:
   - La definición completa del índice
   - Las estadísticas de uso en tiempo real
   - El nodo del Index Service donde reside

5. Ejecutar la siguiente consulta varias veces para generar tráfico y observar cómo aumentan las métricas `# Requests` e `# Items Scanned`:

```sql
-- Ejecutar 5-10 veces para generar actividad en el índice
SELECT name, country
FROM `travel-sample`.inventory.airline
WHERE country = "United States";
```

6. Regresar al panel **Indexes** y verificar que los contadores del índice `idx_airline_country` han aumentado.

7. Navegar a **Settings → Index** (o **Index Settings** según la versión) para revisar las opciones de configuración del servicio de índices:
   - **Storage Mode**: observar si está configurado como `forestdb`, `plasma` o `memory_optimized`
   - **Max Rollback Points**: número de puntos de rollback mantenidos
   - **Memory Quota**: cuota de memoria asignada al Index Service

#### Resultado esperado

El panel debe mostrar todos los índices con estado **Ready** (equivalente a `online`). Los contadores de `# Requests` deben incrementarse después de ejecutar las consultas de prueba. La sección de configuración debe mostrar el modo de almacenamiento actualmente configurado en el nodo.

> **📸 Punto de captura:** Se recomienda tomar una captura de pantalla del panel de índices mostrando las métricas de los índices creados en este laboratorio para incluirla en el reporte de la práctica.

#### Verificación

✅ Todos los índices del laboratorio aparecen con estado **Ready** en la Web Console.
✅ Los contadores de `# Requests` aumentan después de ejecutar consultas que usan esos índices.
✅ Se identificó el modo de almacenamiento configurado en el nodo (Plasma, ForestDB o MOI).

---

### Parte 6 — Comparación de configuración MOI vs Plasma

**Objetivo:** Comprender las diferencias entre los modos de almacenamiento de índices disponibles en Couchbase y sus implicaciones de configuración.

#### 6.1 Verificar el modo de almacenamiento actual

Ejecutar la siguiente consulta REST para obtener la configuración actual del Index Service:

```bash
# Consultar la configuración del Index Service via REST API
curl -s -u Administrator:password \
  http://localhost:8091/settings/indexes | python3 -m json.tool
```

Observar el campo `storageMode` en la respuesta. Los valores posibles son:

| Valor | Descripción |
|---|---|
| `plasma` | Motor de almacenamiento Plasma (predeterminado en Enterprise 7.x) |
| `memory_optimized` | Memory Optimized Indexes (MOI) — índice completamente en RAM |
| `forestdb` | Motor legado (versiones anteriores a 5.0, no recomendado) |

#### 6.2 Tabla comparativa MOI vs Plasma

Completar la siguiente tabla con base en la configuración observada y el conocimiento de la lección:

| Característica | Memory Optimized (MOI) | Plasma |
|---|---|---|
| **Ubicación de datos** | Completamente en RAM | Disco + caché en RAM |
| **Velocidad de lectura** | Muy alta (acceso directo en memoria) | Alta (con caché eficiente) |
| **Capacidad máxima** | Limitada por RAM disponible | Limitada por disco (TB) |
| **Persistencia** | Volátil (se reconstruye al reiniciar) | Persistente en disco |
| **Costo de reinicio** | Alto (reconstrucción completa) | Bajo (carga desde disco) |
| **Caso de uso ideal** | Índices pequeños, latencia crítica | Índices grandes, producción general |
| **Disponibilidad** | Community + Enterprise | Enterprise Edition |
| **Memoria mínima recomendada** | 512 MB por índice activo | 256 MB cuota mínima |

#### 6.3 Observar el impacto de la configuración de memoria

En la Web Console, navegar a **Settings → Service Memory Quotas** y verificar la memoria asignada al Index Service. Anotar el valor actual.

```bash
# Alternativa via REST: verificar cuotas de memoria de servicios
curl -s -u Administrator:password \
  http://localhost:8091/pools/default | python3 -m json.tool | grep -A2 "indexMemoryQuota"
```

#### 6.4 Consulta de diagnóstico de memoria de índices

```sql
-- Ver el uso de memoria por índice (disponible en versiones recientes)
SELECT name, keyspace_id,
       index_key,
       state
FROM system:indexes
WHERE bucket_id = "travel-sample"
  AND name LIKE "idx_%"
ORDER BY keyspace_id, name;
```

> **📝 Nota pedagógica:** En este laboratorio con un nodo único y Community Edition, el cambio del modo de almacenamiento entre MOI y Plasma requiere reiniciar el servicio de índices y reconstruir todos los índices existentes. **No se realizará este cambio en el laboratorio** para evitar interrupciones. El objetivo es comprender las diferencias conceptuales y de configuración.

#### Resultado esperado

```json
{
  "storageMode": "plasma",
  "maxRollbackPoints": 2,
  "memorySnapshotInterval": 200,
  "stableSnapshotInterval": 5000,
  "indexerThreads": 0,
  "logLevel": "info"
}
```

*(Los valores exactos pueden variar según la configuración del nodo)*

#### Verificación

✅ Se identificó el modo de almacenamiento actual del Index Service.
✅ La tabla comparativa MOI vs Plasma está completada.
✅ Se verificó la cuota de memoria asignada al Index Service.

---

### Parte 7 — Tabla de Referencia de Índices del Laboratorio

**Objetivo:** Consolidar en una tabla todos los índices creados durante el laboratorio para referencia futura.

Completar la siguiente tabla con los datos de cada índice creado:

| # | Nombre del Índice | Colección | Tipo | Campos Indexados | Condición WHERE | Propósito |
|---|---|---|---|---|---|---|
| 1 | `idx_lab_primary_airline` | `airline` | Primary | `META().id` | — | Referencia educativa; full scan |
| 2 | `idx_airline_country` | `airline` | Secondary | `country` | — | Filtrar aerolíneas por país |
| 3 | `idx_airline_country_callsign` | `airline` | Composite | `country, callsign` | — | Filtrar por país Y callsign |
| 4 | `idx_airline_name_partial` | `airline` | Partial | `name` | `type = "airline"` | Solo documentos tipo airline |
| 5 | `idx_route_schedule_day` | `route` | Array | `DISTINCT ARRAY s.day FOR s IN schedule END` | — | Buscar rutas por día de operación |
| 6 | `idx_airline_name_lower` | `airline` | Functional | `LOWER(name)` | — | Búsqueda case-insensitive |
| 7 | `idx_route_source_partitioned` | `route` | Partitioned | `sourceairport, destinationairport` | — | Distribución por hash de origen |
| 8 | `idx_airport_city_deferred` | `airport` | Composite (deferred) | `city, country` | — | Demostración de build diferido |

---

## 7. Validación y Pruebas Finales

Ejecutar las siguientes consultas de validación para confirmar que todos los índices del laboratorio están operativos y son utilizados correctamente por el planificador de consultas.

### 7.1 Verificación global de estado

```sql
-- Todos los índices del laboratorio deben estar en estado "online"
SELECT name,
       keyspace_id AS coleccion,
       state,
       CASE
           WHEN `index_key` IS NULL THEN "primary"
           WHEN condition IS NOT NULL THEN "partial"
           ELSE "secondary"
       END AS tipo_estimado
FROM system:indexes
WHERE bucket_id = "travel-sample"
  AND (name LIKE "idx_%")
ORDER BY coleccion, name;
```

**Resultado esperado:** 8 filas, todas con `state = "online"`.

### 7.2 Validar uso de índices con EXPLAIN

```sql
-- Verificar que idx_airline_country es seleccionado por el planificador
EXPLAIN
SELECT name, callsign
FROM `travel-sample`.inventory.airline
WHERE country = "France";
```

En el plan de ejecución, buscar la sección `"index"` que debe mencionar `idx_airline_country` o `idx_airline_country_callsign`. El operador debe ser `IndexScan3` (no `PrimaryScan`).

```sql
-- Verificar que el índice de arreglo es usado para búsqueda ANY...IN
EXPLAIN
SELECT sourceairport, destinationairport
FROM `travel-sample`.inventory.route
WHERE ANY s IN schedule SATISFIES s.day = 3 END;
```

El plan debe mostrar `idx_route_schedule_day` como índice seleccionado.

### 7.3 Consulta de cobertura de índice (Index Covering)

```sql
-- Esta consulta está cubierta completamente por idx_airline_country_callsign
-- (todos los campos de SELECT y WHERE están en el índice)
-- No debe requerir acceso al servicio de datos (KeyValueFetch)
EXPLAIN
SELECT country, callsign
FROM `travel-sample`.inventory.airline
WHERE country = "United Kingdom"
  AND callsign IS NOT NULL;
```

Verificar en el plan que **no aparece** el operador `Fetch` (acceso al documento completo). Si el índice cubre todos los campos requeridos, el plan solo mostrará `IndexScan3` → `Projection`, lo que indica una consulta cubierta por el índice (*index covering query*).

### 7.4 Resumen estadístico final

```sql
-- Resumen de índices creados por colección
SELECT keyspace_id AS coleccion,
       COUNT(*) AS total_indices,
       ARRAY_AGG(name ORDER BY name) AS lista_indices
FROM system:indexes
WHERE bucket_id = "travel-sample"
  AND name LIKE "idx_%"
GROUP BY keyspace_id
ORDER BY keyspace_id;
```

---

## 8. Solución de Problemas

### Problema 1: El índice queda en estado `deferred` y no pasa a `online`

**Síntoma:** Después de ejecutar `CREATE INDEX ... WITH {"defer_build": false}`, la consulta a `system:indexes` muestra `state = "deferred"` en lugar de `"online"`. Las consultas que deberían usar el índice devuelven el error: `"No index available on keyspace"` o usan el índice primario en su lugar.

**Causa:** El servicio de índices puede estar temporalmente saturado, o el índice fue creado con `defer_build: true` de forma explícita o por defecto en ciertas configuraciones. También puede ocurrir si la cuota de memoria del Index Service es insuficiente para iniciar la construcción.

**Solución:**
```sql
-- Paso 1: Verificar el estado del índice
SELECT name, state FROM system:indexes WHERE name = "nombre_del_indice";

-- Paso 2: Si está en "deferred", ejecutar BUILD INDEX explícitamente
BUILD INDEX ON `travel-sample`.inventory.COLECCION(nombre_del_indice);

-- Paso 3: Monitorear hasta que el estado sea "online" (puede tomar 1-3 minutos)
SELECT name, state FROM system:indexes WHERE name = "nombre_del_indice";

-- Paso 4: Si el problema persiste, verificar la memoria del Index Service
-- Web Console → Settings → Service Memory Quotas → Index Service
-- Asegurarse de que tenga al menos 512 MB asignados
```

Si la memoria es insuficiente, aumentar la cuota desde **Settings → Service Memory Quotas** o reducir la cuota del Data Service en 256 MB para redistribuir recursos.

---

### Problema 2: EXPLAIN muestra `PrimaryScan` en lugar del índice secundario esperado

**Síntoma:** Al ejecutar `EXPLAIN SELECT ... FROM coleccion WHERE campo = valor`, el plan de ejecución muestra el operador `PrimaryScan` (usando el índice primario) en lugar de `IndexScan3` con el índice secundario creado. La consulta funciona pero es lenta.

**Causa:** El planificador de consultas no puede usar el índice secundario porque: (a) la condición `WHERE` en la consulta no coincide exactamente con la definición del índice (por ejemplo, diferencia de mayúsculas en el nombre del campo o uso de una función no indexada), (b) el índice parcial tiene una condición `WHERE` que no está incluida en la consulta, o (c) el índice fue creado en un keyspace diferente al consultado (verificar bucket/scope/collection).

**Solución:**
```sql
-- Paso 1: Verificar que el índice existe en el keyspace correcto
SELECT name, keyspace_id, scope_id, bucket_id, `index_key`, condition, state
FROM system:indexes
WHERE name = "idx_airline_country";

-- Paso 2: Comparar la condición del índice parcial con la consulta
-- Si el índice tiene: condition = "(type = "airline")"
-- La consulta DEBE incluir: WHERE type = "airline" AND ...

-- Paso 3: Forzar el uso de un índice específico con USE INDEX (para diagnóstico)
SELECT name, country
FROM `travel-sample`.inventory.airline
USE INDEX (idx_airline_country USING GSI)
WHERE country = "France";

-- Paso 4: Si el índice funciona con USE INDEX pero no sin él,
-- revisar si hay otro índice más selectivo que el planificador prefiera
-- o si las estadísticas del índice están desactualizadas

-- Paso 5: Verificar que el índice está en estado "online"
SELECT name, state FROM system:indexes WHERE name = "idx_airline_country";
```

Si el índice existe, está `online` y la condición coincide pero el planificador sigue sin usarlo, considerar ejecutar `UPDATE STATISTICS` para actualizar las estadísticas del planificador:

```sql
UPDATE STATISTICS FOR `travel-sample`.inventory.airline(country);
```

---

## 9. Limpieza del Entorno

Ejecutar los siguientes comandos para eliminar los índices creados durante el laboratorio y restaurar el entorno al estado inicial.

> **⚠️ Importante:** Solo eliminar los índices con prefijo `idx_` creados en este laboratorio. No eliminar los índices predeterminados del dataset `travel-sample` (como `def_primary`, `def_fts_index`, etc.).

```sql
-- Eliminar índices de la colección airline
DROP INDEX `travel-sample`.inventory.airline.idx_lab_primary_airline;
DROP INDEX `travel-sample`.inventory.airline.idx_airline_country;
DROP INDEX `travel-sample`.inventory.airline.idx_airline_country_callsign;
DROP INDEX `travel-sample`.inventory.airline.idx_airline_name_partial;
DROP INDEX `travel-sample`.inventory.airline.idx_airline_name_lower;

-- Eliminar índices de la colección route
DROP INDEX `travel-sample`.inventory.route.idx_route_schedule_day;
DROP INDEX `travel-sample`.inventory.route.idx_route_source_partitioned;

-- Eliminar índices de la colección airport
DROP INDEX `travel-sample`.inventory.airport.idx_airport_city_deferred;
```

**Verificar que la limpieza fue exitosa:**

```sql
-- No debe devolver filas con prefijo "idx_"
SELECT name, keyspace_id
FROM system:indexes
WHERE bucket_id = "travel-sample"
  AND name LIKE "idx_%"
ORDER BY name;
```

**Resultado esperado:** La consulta debe devolver 0 filas (conjunto vacío).

```sql
-- Verificar que los índices predeterminados del dataset siguen intactos
SELECT name, state
FROM system:indexes
WHERE bucket_id = "travel-sample"
  AND name NOT LIKE "idx_%"
ORDER BY name;
```

Los índices predeterminados del dataset `travel-sample` deben seguir presentes con `state = "online"`.

---

## 10. Resumen

### Conceptos aplicados en este laboratorio

En este laboratorio se aplicaron de forma práctica los conceptos fundamentales de indexación en Couchbase SQL++:

| Concepto | Comando / Técnica | Beneficio |
|---|---|---|
| Índice primario | `CREATE PRIMARY INDEX` | Exploración completa; útil en desarrollo |
| Índice secundario | `CREATE INDEX ... ON coleccion(campo)` | Acceso eficiente por campo específico |
| Índice compuesto | `CREATE INDEX ... ON coleccion(campo1, campo2)` | Cubre múltiples predicados en una sola operación |
| Índice parcial | `CREATE INDEX ... WHERE condicion` | Reduce tamaño del índice; más rápido para subconjuntos |
| Índice de arreglo | `DISTINCT ARRAY ... FOR ... IN ... END` | Búsqueda dentro de listas JSON anidadas |
| Índice funcional | `CREATE INDEX ... ON coleccion(LOWER(campo))` | Soporta expresiones en predicados |
| Índice particionado | `PARTITION BY HASH(campo)` | Distribuye carga en clústeres multi-nodo |
| Build diferido | `WITH {"defer_build": true}` + `BUILD INDEX` | Crea el índice sin bloquear; construye después |
| Modificación | `ALTER INDEX ... WITH {"action": ...}` | Cambia réplicas y configuración sin recrear |
| Monitoreo | Web Console → Indexes | Métricas en tiempo real por índice |
| Diagnóstico | `EXPLAIN SELECT ...` | Verifica qué índice usa el planificador |
| Catálogo | `system:indexes`, `system:index_partitions` | Introspección programática del estado |

### Puntos clave para recordar

1. **El índice primario es un antipatrón en producción**: siempre preferir índices secundarios específicos para las consultas de producción.
2. **El orden de campos en índices compuestos importa**: colocar primero el campo de mayor selectividad (el que filtra más documentos).
3. **Los índices parciales son una optimización de espacio**: cuando siempre se filtra por un valor fijo, incluirlo en la definición del índice reduce su tamaño.
4. **`EXPLAIN` es la herramienta diagnóstica fundamental**: siempre verificar que el planificador usa el índice esperado.
5. **MOI vs Plasma**: MOI es más rápido en lecturas pero limitado por RAM y volátil; Plasma es el estándar de producción en Enterprise Edition.

### Recursos adicionales

- [Documentación oficial: CREATE INDEX](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/createindex.html)
- [Documentación oficial: ALTER INDEX](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/alterindex.html)
- [Documentación oficial: Index Partitioning](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/index-partitioning.html)
- [Documentación oficial: Array Indexing](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/indexing-arrays.html)
- [Documentación oficial: Memory Optimized Indexes](https://docs.couchbase.com/server/current/learn/services-and-indexes/indexes/storage-modes.html)
- [Documentación oficial: system:indexes](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/systeminformation.html)
- [Blog: Mejores prácticas de indexación en SQL++](https://www.couchbase.com/blog/n1ql-index-best-practices/)

---
