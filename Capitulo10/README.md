# Ejecución de búsquedas básicas con Couchbase Search

## Metadatos

| Atributo         | Valor                          |
|------------------|-------------------------------|
| **Duración**     | 50 minutos                    |
| **Complejidad**  | Media                         |
| **Nivel Bloom**  | Aplicar (Apply)               |
| **Servicio**     | Couchbase Search (FTS)        |
| **Dataset**      | travel-sample / hotel         |

---

## Descripción General

En este laboratorio explorarás el servicio **Couchbase Search (Full Text Search)** comparándolo directamente con SQL++ para búsquedas de texto libre. Crearás un índice de búsqueda sobre la colección `hotel` del dataset `travel-sample`, configurarás field mappings con el analizador de inglés y ejecutarás cuatro tipos de consultas básicas: `match`, `match_phrase`, `prefix` y `boolean`. Finalmente, interpretarás los scores de relevancia y reproducirás una consulta FTS mediante la REST API usando `curl`.

---

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Identificar por qué Couchbase Search supera a `SQL++ LIKE` para búsquedas de texto libre, relevancia y análisis lingüístico
- [ ] Crear un Search Index con field mappings y el analizador `en` (English) usando el editor de la Web Console
- [ ] Ejecutar consultas FTS básicas (`match`, `match_phrase`, `prefix`, `boolean`) desde la Web Console y via REST API con `curl`
- [ ] Interpretar el score de relevancia de los resultados FTS y explicar por qué ciertos documentos rankean más alto
- [ ] Comparar cuantitativamente los resultados de `SQL++ LIKE` vs Couchbase Search para la misma intención de búsqueda

---

## Prerrequisitos

### Conocimiento previo
- Haber completado **Lab 02-00-01**: Couchbase Server instalado y configurado con el bucket `travel-sample` cargado
- Haber completado **Lab 03-00-01**: Consultas SQL++ básicas, incluyendo el operador `LIKE`
- Comprensión básica de documentos JSON y colecciones en Couchbase
- Familiaridad con la Web Console de Couchbase (navegación entre servicios)

### Acceso y servicios requeridos
- Couchbase Server 7.6.x en ejecución (local o Docker)
- Servicio **Search** habilitado en el nodo (verificar en `http://localhost:8091`)
- Servicio **Query** habilitado (para comparación con SQL++)
- Dataset `travel-sample` cargado con la colección `hotel` disponible
- `curl` instalado y accesible desde terminal
- Navegador web moderno (Chrome 110+, Firefox 110+ o Edge 110+)

---

## Entorno de Laboratorio

### Requisitos de hardware

| Recurso       | Mínimo              | Recomendado          |
|---------------|---------------------|----------------------|
| RAM           | 8 GB disponibles    | 16 GB                |
| CPU           | 4 núcleos x86_64    | 8 núcleos            |
| Almacenamiento| 20 GB libres (SSD)  | 50 GB SSD            |
| Red           | localhost funcional | localhost funcional  |
| Pantalla      | 1280×768            | 1920×1080            |

### Puertos requeridos

| Puerto | Servicio                         |
|--------|----------------------------------|
| 8091   | Couchbase Web Console            |
| 8093   | Query Service (SQL++)            |
| 8094   | Search Service (FTS)             |
| 11210  | Data Service (SDK/KV)            |

### Verificación del entorno

Antes de comenzar, ejecuta los siguientes comandos en tu terminal para confirmar que el entorno está operativo:

```bash
# Verificar que Couchbase responde
curl -s -u Administrator:password http://localhost:8091/pools \
  | python3 -m json.tool | grep '"name"'

# Verificar que el servicio Search está activo
curl -s -u Administrator:password http://localhost:8094/api/index \
  | python3 -m json.tool | head -5

# Verificar que el bucket travel-sample existe
curl -s -u Administrator:password \
  http://localhost:8091/pools/default/buckets/travel-sample \
  | python3 -m json.tool | grep '"name"'
```

> **Nota:** Sustituye `Administrator` y `password` por las credenciales configuradas en tu instalación de Couchbase. Si usas Docker, asegúrate de que el contenedor esté en ejecución con `docker ps`.

### Verificación del dataset hotel

```bash
# Contar documentos en la colección hotel via Query Service
curl -s -u Administrator:password \
  http://localhost:8093/query/service \
  -d 'statement=SELECT COUNT(*) as total FROM `travel-sample`.`inventory`.`hotel`' \
  | python3 -m json.tool
```

**Salida esperada:** El campo `total` debe mostrar aproximadamente `917` documentos.

---

## Desarrollo del Laboratorio

---

### Parte 1: Comparación SQL++ LIKE vs Couchbase Search

#### Paso 1.1 — Ejecutar búsqueda con SQL++ usando LIKE

**Objetivo:** Establecer una línea base ejecutando la búsqueda `luxury hotel near airport` con SQL++ para documentar sus limitaciones.

**Instrucciones:**

1. Abre la Web Console en `http://localhost:8091` e inicia sesión.
2. En el menú lateral, haz clic en **Query** para abrir el editor SQL++.
3. Ejecuta la siguiente consulta:

```sql
SELECT h.name,
       h.city,
       h.country,
       h.description
FROM `travel-sample`.`inventory`.`hotel` h
WHERE LOWER(h.description) LIKE "%luxury%"
   OR LOWER(h.description) LIKE "%airport%"
ORDER BY h.name ASC
LIMIT 10;
```

4. Registra el número de resultados retornados y el tiempo de ejecución (visible en la pestaña **Plan** o en la cabecera de resultados).

5. Ahora ejecuta una variante con un error tipográfico deliberado:

```sql
SELECT h.name, h.description
FROM `travel-sample`.`inventory`.`hotel` h
WHERE LOWER(h.description) LIKE "%luxuri%"
LIMIT 5;
```

**Salida esperada del paso 1.1:**
- La primera consulta retorna hoteles que contienen exactamente `luxury` o `airport` en la descripción.
- La segunda consulta retorna **0 resultados** (el error tipográfico `luxuri` no coincide con nada).
- El tiempo de ejecución puede ser elevado si no existe un índice GSI que cubra este patrón.

**Verificación:**
En la pestaña **Plan** de la Web Console, observa si la consulta usa un `PrimaryScan` o un `IndexScan`. Un `PrimaryScan` indica que se está escaneando toda la colección, lo cual es ineficiente a escala.

---

#### Paso 1.2 — Documentar las limitaciones observadas

**Objetivo:** Registrar formalmente las limitaciones de `LIKE` antes de contrastarlas con Search.

**Instrucciones:**

En tu cuaderno o editor de texto, anota las respuestas a las siguientes preguntas basándote en lo observado:

| Pregunta | Observación |
|----------|-------------|
| ¿Los resultados están ordenados por relevancia? | |
| ¿Se encontraron variantes lingüísticas (ej. "luxurious")? | |
| ¿El error tipográfico `luxuri` retornó resultados? | |
| ¿Qué tipo de scan usó el plan de ejecución? | |
| ¿Cuánto tiempo tomó la consulta? | |

> **Concepto clave:** Como aprendiste en la lección 10.1, `LIKE` con `%` al inicio del patrón no puede aprovechar índices secundarios (GSI), lo que provoca un escaneo completo de la colección. Además, no realiza análisis lingüístico ni calcula relevancia.

---

### Parte 2: Creación del Search Index

#### Paso 2.1 — Acceder al editor de Search Index

**Objetivo:** Navegar al servicio Search en la Web Console y preparar la creación de un nuevo índice.

**Instrucciones:**

1. En la Web Console (`http://localhost:8091`), haz clic en **Search** en el menú lateral izquierdo.

   > Si no ves la opción **Search** en el menú, el servicio no está habilitado. Ve a **Settings → Manage Server Nodes** y verifica que el servicio FTS esté activo en el nodo.

2. En la página de Search, haz clic en el botón **+ Add Index** (esquina superior derecha).

3. Se abrirá el editor de creación de índices con varias pestañas: **General Settings**, **Type Mappings**, **Analyzers**, **Date/Time Parsers** y **Advanced**.

**Salida esperada:** Visualizas el formulario de creación de índice FTS con el campo **Index Name** vacío listo para configurar.

---

#### Paso 2.2 — Configurar los parámetros generales del índice

**Objetivo:** Definir el nombre del índice, el bucket, el scope y la colección objetivo.

**Instrucciones:**

1. En el campo **Index Name**, escribe:
   ```
   hotel-search-index
   ```

2. En el campo **Bucket**, selecciona `travel-sample`.

3. Expande la sección **Scope and Collection**:
   - **Scope:** `inventory`
   - **Collection:** `hotel`

4. En el campo **Index Storage**, deja la opción por defecto: `scorch` (motor de almacenamiento optimizado para FTS).

5. Verifica que la sección **Language** muestre `English` o déjala en `Standard` por ahora (configuraremos el analizador por campo en el siguiente paso).

**Salida esperada:** El formulario muestra `travel-sample` como bucket, `inventory` como scope y `hotel` como colección.

---

#### Paso 2.3 — Configurar Type Mappings y Field Mappings

**Objetivo:** Definir qué campos del documento `hotel` serán indexados por el Search Service y con qué analizador.

**Instrucciones:**

1. Haz clic en la pestaña **Type Mappings** (o desplázate hasta la sección correspondiente).

2. Haz clic en **+ Add Type Mapping**.

3. En el campo **Type**, escribe `hotel` (o deja el valor por defecto si el sistema ya detectó el tipo de la colección).

4. **Desactiva** la opción **dynamic** para controlar explícitamente qué campos se indexan (esto reduce el tamaño del índice).

5. Haz clic en **+ Insert Child Field** para agregar el primer campo. Configura:

   | Parámetro    | Valor        |
   |--------------|--------------|
   | Field        | `name`       |
   | Type         | `text`       |
   | Analyzer     | `standard`   |
   | Store        | ✅ (activado)|
   | Include in _all field | ✅ |

6. Repite el proceso para agregar los siguientes campos:

   **Campo `description`:**
   | Parámetro    | Valor        |
   |--------------|--------------|
   | Field        | `description`|
   | Type         | `text`       |
   | Analyzer     | `en`         |
   | Store        | ✅           |
   | Include in _all field | ✅ |

   > **Importante:** El analizador `en` (English) aplica stemming en inglés, eliminación de stopwords y normalización. Esto permite que una búsqueda de "pool" encuentre documentos que contengan "pools" y viceversa.

   **Campo `city`:**
   | Parámetro    | Valor        |
   |--------------|--------------|
   | Field        | `city`       |
   | Type         | `text`       |
   | Analyzer     | `keyword`    |
   | Store        | ✅           |
   | Include in _all field | ✅ |

   **Campo `country`:**
   | Parámetro    | Valor        |
   |--------------|--------------|
   | Field        | `country`    |
   | Type         | `text`       |
   | Analyzer     | `keyword`    |
   | Store        | ✅           |
   | Include in _all field | ✅ |

   **Campo `reviews.content` (campo anidado):**
   | Parámetro    | Valor           |
   |--------------|-----------------|
   | Field        | `reviews.content`|
   | Type         | `text`          |
   | Analyzer     | `en`            |
   | Store        | ✅              |
   | Include in _all field | ✅   |

7. Revisa que tengas los 5 campos configurados en el type mapping.

**Salida esperada:** La sección Type Mappings muestra un mapping con 5 campos: `name`, `description`, `city`, `country` y `reviews.content`, cada uno con su analizador correspondiente.

---

#### Paso 2.4 — Crear el índice

**Objetivo:** Guardar la configuración y disparar la construcción del índice.

**Instrucciones:**

1. Desplázate hasta la parte inferior del formulario y haz clic en **Create Index**.

2. Serás redirigido a la lista de índices. Verás `hotel-search-index` con un indicador de progreso de indexación.

3. Espera a que el porcentaje de indexación llegue al **100%**. Esto puede tomar entre 15 y 60 segundos dependiendo del hardware.

   > Puedes actualizar la página periódicamente. El campo **Docs Indexed** debe acercarse a 917.

4. Una vez completado, haz clic en el nombre del índice `hotel-search-index` para ver sus detalles.

**Salida esperada:**

```
Index Name:    hotel-search-index
Status:        Ready
Docs Indexed:  917 (aproximadamente)
```

**Verificación via REST API:**

```bash
curl -s -u Administrator:password \
  http://localhost:8094/api/index/hotel-search-index \
  | python3 -m json.tool | grep -E '"status"|"docCount"'
```

Debes ver algo como:
```json
"status": "Ready",
"docCount": 917
```

---

### Parte 3: Consultas FTS Básicas desde la Web Console

#### Paso 3.1 — Match Query: búsqueda multi-término en `description`

**Objetivo:** Ejecutar una `match query` que busque los términos `pool wifi breakfast` en el campo `description`, aprovechando el analizador `en`.

**Instrucciones:**

1. En la página del índice `hotel-search-index`, haz clic en el botón **Search** (o ve a **Search → hotel-search-index → Search**).

2. En el campo de búsqueda de la Web Console, haz clic en **Advanced** para cambiar al modo de consulta JSON.

3. Ingresa el siguiente JSON de consulta:

```json
{
  "query": {
    "match": "pool wifi breakfast",
    "field": "description",
    "analyzer": "en",
    "operator": "or"
  },
  "size": 10,
  "from": 0,
  "highlight": {
    "style": "html",
    "fields": ["description"]
  },
  "fields": ["name", "city", "country", "description"]
}
```

4. Haz clic en **Search**.

5. Observa los resultados. Para cada documento retornado, anota:
   - El valor del campo `score`
   - El nombre del hotel (`name`)
   - El fragmento resaltado (`fragments`) en la descripción

**Salida esperada:**
- Se retornan entre 5 y 15 hoteles con menciones de `pool`, `wifi` o `breakfast` en su descripción.
- Los resultados están ordenados por score descendente (mayor relevancia primero).
- Los fragmentos muestran los términos encontrados resaltados con etiquetas `<mark>` o similares.

**Verificación:**
El hotel con el score más alto debe tener más de uno de los tres términos en su descripción. Verifica que el stemming funciona: si un hotel tiene "pools" en su descripción, también debe aparecer en los resultados.

---

#### Paso 3.2 — Match Phrase Query: búsqueda de frase exacta `free parking`

**Objetivo:** Ejecutar una `match_phrase query` que busque la frase exacta `free parking` preservando el orden de los términos.

**Instrucciones:**

1. En el editor de consultas JSON del índice, reemplaza el contenido con:

```json
{
  "query": {
    "match_phrase": "free parking",
    "field": "description"
  },
  "size": 10,
  "from": 0,
  "highlight": {
    "style": "html",
    "fields": ["description", "name"]
  },
  "fields": ["name", "city", "country", "description"]
}
```

2. Haz clic en **Search**.

3. Compara el número de resultados con la `match query` anterior.

4. Verifica que **todos** los documentos retornados contengan la frase exacta `free parking` (las palabras juntas y en ese orden) en la descripción.

**Salida esperada:**
- Se retornan entre 1 y 5 hoteles con la frase exacta `free parking`.
- La cantidad de resultados es menor que con `match query` porque la frase exacta es más restrictiva.
- Los fragmentos resaltados muestran siempre `free parking` como unidad.

> **Concepto clave:** A diferencia de `match`, que busca cualquiera de los términos, `match_phrase` requiere que los términos aparezcan juntos y en el mismo orden. Esto es equivalente a `LIKE "%free parking%"` en SQL++, pero con la ventaja de que puede aprovechar el índice FTS eficientemente.

---

#### Paso 3.3 — Prefix Query: buscar hoteles con descripción que inicia con `beachfront`

**Objetivo:** Ejecutar una `prefix query` para encontrar términos que comiencen con el prefijo `beachfront`.

**Instrucciones:**

1. En el editor de consultas JSON, ingresa:

```json
{
  "query": {
    "prefix": "beachfront",
    "field": "description"
  },
  "size": 10,
  "from": 0,
  "highlight": {
    "style": "html",
    "fields": ["description"]
  },
  "fields": ["name", "city", "country", "description"]
}
```

2. Haz clic en **Search**.

3. Observa si se retornan documentos con variantes como `beachfront`, `beachfronts` u otras extensiones del prefijo.

4. Ahora prueba con un prefijo más corto para ver más resultados:

```json
{
  "query": {
    "prefix": "beach",
    "field": "description"
  },
  "size": 10,
  "from": 0,
  "fields": ["name", "description"]
}
```

**Salida esperada:**
- La primera consulta retorna hoteles cuya descripción contiene términos que empiezan con `beachfront`.
- La segunda consulta retorna más resultados al ser un prefijo más corto.

> **Nota:** La `prefix query` opera sobre los tokens del índice, no sobre el texto original completo. El analizador `en` puede haber transformado algunos tokens, por lo que los resultados pueden variar ligeramente.

---

#### Paso 3.4 — Boolean Query: combinación de condiciones must/should/must_not

**Objetivo:** Ejecutar una `boolean query` que combine múltiples condiciones para simular la búsqueda `luxury hotel near airport`.

**Instrucciones:**

1. En el editor de consultas JSON, ingresa la siguiente consulta booleana compuesta:

```json
{
  "query": {
    "conjuncts": [
      {
        "disjuncts": [
          {
            "match": "luxury",
            "field": "description",
            "boost": 2.0
          },
          {
            "match": "luxury",
            "field": "name",
            "boost": 3.0
          }
        ]
      }
    ]
  },
  "size": 10,
  "from": 0,
  "highlight": {
    "style": "html",
    "fields": ["name", "description"]
  },
  "fields": ["name", "city", "country", "description"]
}
```

2. Haz clic en **Search** y observa los resultados.

3. Ahora ejecuta la consulta booleana completa que simula `luxury hotel near airport`:

```json
{
  "query": {
    "bool": {
      "must": {
        "conjuncts": [
          {
            "match": "hotel",
            "field": "description"
          }
        ]
      },
      "should": {
        "disjuncts": [
          {
            "match": "luxury",
            "field": "description",
            "boost": 2.0
          },
          {
            "match": "airport",
            "field": "description",
            "boost": 1.5
          },
          {
            "match_phrase": "near airport",
            "field": "description",
            "boost": 3.0
          }
        ]
      },
      "must_not": {
        "disjuncts": [
          {
            "match": "hostel",
            "field": "description"
          }
        ]
      }
    }
  },
  "size": 10,
  "from": 0,
  "highlight": {
    "style": "html",
    "fields": ["name", "description"]
  },
  "fields": ["name", "city", "country", "description"]
}
```

4. Haz clic en **Search**.

5. Analiza los resultados:
   - ¿Qué documentos tienen el score más alto?
   - ¿Aparecen documentos con `hostel` en los resultados? (No deberían, por el `must_not`)
   - ¿Los documentos con `near airport` en la descripción tienen mayor score que los que solo tienen `airport`? (Deberían, por el `boost: 3.0`)

**Salida esperada:**
- Se retornan hoteles que mencionan `hotel` (obligatorio), con preferencia por los que mencionan `luxury` y/o `airport`.
- Ningún documento con `hostel` en la descripción aparece en los resultados.
- Los documentos con la frase `near airport` tienen scores más altos que los que solo mencionan `airport`.

---

### Parte 4: Interpretación de Scores y REST API

#### Paso 4.1 — Analizar el score de relevancia

**Objetivo:** Comprender cómo el motor FTS calcula el score TF-IDF y por qué ciertos documentos rankean más alto.

**Instrucciones:**

1. Toma los resultados de la consulta booleana del Paso 3.4 y selecciona los **3 documentos con mayor score**.

2. Para cada uno, completa la siguiente tabla de análisis:

| # | Nombre del hotel | Score | Términos encontrados | ¿Tiene boost aplicado? |
|---|-----------------|-------|---------------------|----------------------|
| 1 | | | | |
| 2 | | | | |
| 3 | | | | |

3. Ejecuta la siguiente consulta para ver los scores con más detalle, habilitando `explain: true`:

```json
{
  "query": {
    "match": "luxury airport",
    "field": "description",
    "analyzer": "en",
    "operator": "or"
  },
  "size": 3,
  "from": 0,
  "explain": true,
  "fields": ["name", "description"]
}
```

4. Expande el campo `explanation` en los resultados JSON. Observa la estructura:
   - `value`: score total del documento
   - `description`: descripción del componente de scoring
   - `details`: desglose de los factores contribuyentes (TF, IDF, field norm)

**Salida esperada:**
La respuesta JSON incluye un campo `explanation` por documento con la estructura de scoring. Ejemplo parcial:

```json
{
  "id": "hotel_10025",
  "score": 2.8471,
  "explanation": {
    "value": 2.8471,
    "description": "sum of:",
    "details": [
      {
        "value": 1.9234,
        "description": "weight(description:luxury in doc 42), result of:",
        "details": [...]
      },
      {
        "value": 0.9237,
        "description": "weight(description:airport in doc 42), result of:",
        "details": [...]
      }
    ]
  }
}
```

> **Concepto clave (TF-IDF):** El score combina **Term Frequency** (cuántas veces aparece el término en el documento) e **Inverse Document Frequency** (qué tan raro es el término en toda la colección). Un término que aparece muchas veces en un documento pero pocas en la colección general tendrá un score alto. Los campos con `boost` multiplican su contribución al score final.

---

#### Paso 4.2 — Ejecutar la misma consulta via REST API con curl

**Objetivo:** Reproducir la `match query` del Paso 3.1 usando la REST API del servicio Search en el puerto 8094.

**Instrucciones:**

1. Abre una terminal y ejecuta el siguiente comando `curl`:

```bash
curl -s -u Administrator:password \
  -X POST \
  -H "Content-Type: application/json" \
  http://localhost:8094/api/index/hotel-search-index/query \
  -d '{
    "query": {
      "match": "pool wifi breakfast",
      "field": "description",
      "analyzer": "en",
      "operator": "or"
    },
    "size": 5,
    "from": 0,
    "highlight": {
      "style": "html",
      "fields": ["description"]
    },
    "fields": ["name", "city", "country"]
  }' | python3 -m json.tool
```

2. Observa la estructura de la respuesta JSON. Identifica:
   - `status.total`: número total de resultados
   - `hits`: array con los documentos encontrados
   - `hits[0].score`: score del primer resultado
   - `hits[0].fields`: campos almacenados retornados
   - `hits[0].fragments`: fragmentos de texto resaltados

3. Ejecuta la boolean query del Paso 3.4 via REST API:

```bash
curl -s -u Administrator:password \
  -X POST \
  -H "Content-Type: application/json" \
  http://localhost:8094/api/index/hotel-search-index/query \
  -d '{
    "query": {
      "bool": {
        "must": {
          "conjuncts": [
            {"match": "hotel", "field": "description"}
          ]
        },
        "should": {
          "disjuncts": [
            {"match": "luxury", "field": "description", "boost": 2.0},
            {"match": "airport", "field": "description", "boost": 1.5},
            {"match_phrase": "near airport", "field": "description", "boost": 3.0}
          ]
        },
        "must_not": {
          "disjuncts": [
            {"match": "hostel", "field": "description"}
          ]
        }
      }
    },
    "size": 5,
    "from": 0,
    "fields": ["name", "city", "country"]
  }' | python3 -m json.tool
```

4. Compara los resultados con los obtenidos en la Web Console. Deben ser idénticos.

**Salida esperada:**

```json
{
  "status": {
    "total": 1,
    "failed": 0,
    "successful": 1
  },
  "request": { ... },
  "hits": [
    {
      "index": "hotel-search-index",
      "id": "hotel_XXXXX",
      "score": 2.XXXX,
      "locations": { ... },
      "fragments": {
        "description": ["...términos <mark>resaltados</mark>..."]
      },
      "fields": {
        "name": "Nombre del Hotel",
        "city": "Ciudad",
        "country": "País"
      }
    }
  ],
  "total_hits": X,
  "max_score": X.XXXX,
  "took": XXXXX,
  "facets": null
}
```

---

#### Paso 4.3 — Comparación final: SQL++ LIKE vs Couchbase Search

**Objetivo:** Documentar de forma estructurada las diferencias observadas entre ambos enfoques.

**Instrucciones:**

1. Ejecuta la siguiente consulta SQL++ equivalente a la búsqueda `luxury hotel near airport`:

```sql
SELECT h.name,
       h.city,
       h.country,
       h.description
FROM `travel-sample`.`inventory`.`hotel` h
WHERE LOWER(h.description) LIKE "%luxury%"
   OR LOWER(h.description) LIKE "%airport%"
ORDER BY h.name ASC
LIMIT 10;
```

2. Compara los resultados con los obtenidos en el Paso 3.4 (Boolean Query FTS). Completa la tabla comparativa:

| Criterio | SQL++ LIKE | Couchbase Search FTS |
|----------|-----------|----------------------|
| Número de resultados | | |
| ¿Ordenados por relevancia? | No (por nombre) | Sí (por score) |
| ¿Detectó variantes lingüísticas? | | |
| ¿Toleró errores tipográficos? | | |
| ¿Puede combinar boost por campo? | | |
| ¿Usa índice eficientemente? | | |
| Tiempo aproximado de ejecución | | |

3. Escribe una conclusión de 2-3 oraciones en tu cuaderno respondiendo: *¿En qué escenario de la aplicación travel-sample usarías Search en lugar de SQL++?*

---

## Validación y Pruebas

### Checklist de validación del laboratorio

Ejecuta los siguientes comandos para validar que completaste correctamente cada parte:

```bash
# Validación 1: El índice existe y está listo
curl -s -u Administrator:password \
  http://localhost:8094/api/index/hotel-search-index \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
status = data.get('status', '')
count = data.get('planPIndexes', {})
print(f'Índice status: {status}')
print('✅ Índice creado correctamente' if status == 'ok' else '❌ Índice no encontrado')
"

# Validación 2: El índice tiene documentos indexados
curl -s -u Administrator:password \
  "http://localhost:8094/api/index/hotel-search-index/count" \
  | python3 -m json.tool

# Validación 3: Una match query retorna resultados
curl -s -u Administrator:password \
  -X POST \
  -H "Content-Type: application/json" \
  http://localhost:8094/api/index/hotel-search-index/query \
  -d '{"query": {"match": "pool", "field": "description"}, "size": 1}' \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
total = data.get('total_hits', 0)
print(f'Total hits para \"pool\": {total}')
print('✅ Match query funciona correctamente' if total > 0 else '❌ No se retornaron resultados')
"

# Validación 4: La boolean query excluye documentos con must_not
curl -s -u Administrator:password \
  -X POST \
  -H "Content-Type: application/json" \
  http://localhost:8094/api/index/hotel-search-index/query \
  -d '{
    "query": {
      "bool": {
        "must_not": {
          "disjuncts": [{"match": "hostel", "field": "description"}]
        }
      }
    },
    "size": 5,
    "fields": ["name", "description"]
  }' \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
hits = data.get('hits', [])
hostel_found = any('hostel' in h.get('fields', {}).get('description', '').lower() for h in hits)
print('✅ must_not funciona: ningún resultado contiene hostel' if not hostel_found else '❌ must_not no está filtrando correctamente')
"
```

### Prueba de regresión: comparar conteos

```bash
# Contar hoteles totales en la colección
echo "=== Total documentos en colección hotel ==="
curl -s -u Administrator:password \
  http://localhost:8093/query/service \
  -d 'statement=SELECT COUNT(*) as total FROM `travel-sample`.`inventory`.`hotel`' \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['results'][0]['total'])"

# Contar documentos indexados en FTS
echo "=== Documentos indexados en hotel-search-index ==="
curl -s -u Administrator:password \
  http://localhost:8094/api/index/hotel-search-index/count \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('count', 'N/A'))"
```

**Resultado esperado:** Ambos conteos deben ser aproximadamente iguales (~917). Una diferencia pequeña es normal si la indexación FTS aún está en progreso.

---

## Solución de Problemas

### Problema 1: El servicio Search no aparece en la Web Console

**Síntomas:**
- El menú lateral no muestra la opción **Search**
- Al acceder a `http://localhost:8094/api/index` se obtiene un error de conexión rechazada (`Connection refused`)
- El comando `curl -u Administrator:password http://localhost:8094/api/index` devuelve error

**Causa:**
El servicio **Full Text Search (FTS)** no fue habilitado durante la configuración inicial del nodo Couchbase, o fue deshabilitado posteriormente. En instalaciones con recursos limitados (8 GB RAM), algunos estudiantes pueden haber omitido este servicio para ahorrar memoria.

**Solución:**
1. Ve a `http://localhost:8091` → **Settings** → **Cluster** → **Server Nodes**.
2. Haz clic en el nodo actual y selecciona **Edit**.
3. En la sección de servicios, activa la casilla **Search**.
4. Haz clic en **Save** y espera a que el nodo se reinicie (~30-60 segundos).
5. Verifica que el servicio esté activo:
   ```bash
   curl -s -u Administrator:password \
     http://localhost:8091/pools/default \
     | python3 -c "
   import json, sys
   d = json.load(sys.stdin)
   nodes = d.get('nodes', [])
   for n in nodes:
       services = n.get('services', [])
       print('Servicios activos:', services)
   "
   ```
6. Si usas Docker, verifica que el contenedor fue iniciado con la variable de entorno correcta:
   ```bash
   docker inspect <container_name> | grep -A5 "SERVICES"
   ```
   Si es necesario, reinicia el contenedor con el flag de Search habilitado.

---

### Problema 2: El índice FTS se crea pero no retorna resultados en ninguna consulta

**Síntomas:**
- El índice `hotel-search-index` aparece en la Web Console con estado `Ready`
- El conteo de documentos indexados (`/api/index/hotel-search-index/count`) muestra `0` o un número muy pequeño
- Todas las consultas FTS retornan `total_hits: 0` independientemente del término buscado

**Causa:**
El type mapping fue configurado incorrectamente. Las causas más comunes son:
1. El **scope** o **collection** en el índice no coincide con `inventory.hotel`
2. El mapping tiene `dynamic: false` pero no se agregaron los field mappings correctamente (campos vacíos o mal escritos)
3. El índice fue creado apuntando al scope/collection equivocado (ej. `_default._default` en lugar de `inventory.hotel`)

**Solución:**
1. Verifica el scope y collection del índice:
   ```bash
   curl -s -u Administrator:password \
     http://localhost:8094/api/index/hotel-search-index \
     | python3 -c "
   import json, sys
   d = json.load(sys.stdin)
   params = d.get('indexDef', {}).get('params', {})
   mapping = json.loads(params) if isinstance(params, str) else params
   print('Scope/Collection:', json.dumps(mapping.get('mapping', {}).get('default_mapping', {}), indent=2)[:500])
   "
   ```
2. Si el scope/collection es incorrecto, **elimina el índice** y créalo nuevamente:
   ```bash
   curl -s -u Administrator:password \
     -X DELETE \
     http://localhost:8094/api/index/hotel-search-index
   ```
3. Vuelve al Paso 2.2 y verifica cuidadosamente que seleccionas `inventory` como scope y `hotel` como collection antes de crear el índice.
4. Si el problema persiste con `dynamic: false`, prueba temporalmente activando `dynamic: true` para verificar que los documentos sí están siendo indexados:
   ```json
   {
     "query": {"match_all": {}},
     "size": 5,
     "fields": ["*"]
   }
   ```
   Si con `dynamic: true` aparecen resultados, el problema estaba en los field mappings específicos.

---

## Limpieza del Entorno

Al finalizar el laboratorio, el índice FTS creado puede conservarse para los laboratorios posteriores. Sin embargo, si necesitas liberar recursos o rehacer el laboratorio desde cero, ejecuta los siguientes pasos:

### Eliminar el índice FTS (opcional)

**Opción A — Via Web Console:**
1. Ve a **Search** en el menú lateral.
2. Localiza `hotel-search-index` en la lista.
3. Haz clic en el ícono de papelera (🗑️) o en el botón **Delete**.
4. Confirma la eliminación.

**Opción B — Via REST API:**
```bash
curl -s -u Administrator:password \
  -X DELETE \
  http://localhost:8094/api/index/hotel-search-index \
  | python3 -m json.tool
```

**Salida esperada de la eliminación:**
```json
{
  "status": "ok"
}
```

### Verificar limpieza

```bash
# Verificar que el índice ya no existe
curl -s -u Administrator:password \
  http://localhost:8094/api/index \
  | python3 -c "
import json, sys
d = json.load(sys.stdin)
indexes = d.get('indexDefs', {}).get('indexDefs', {})
if 'hotel-search-index' not in indexes:
    print('✅ Índice eliminado correctamente')
else:
    print('❌ El índice aún existe')
"
```

> **Nota:** El bucket `travel-sample` y sus datos **no deben eliminarse**, ya que son necesarios para los laboratorios restantes del curso.

---

## Resumen

En este laboratorio realizaste un recorrido completo por las capacidades básicas del servicio **Couchbase Search (Full Text Search)**:

### Lo que aprendiste

| Actividad | Concepto clave |
|-----------|---------------|
| Comparación SQL++ LIKE vs FTS | `LIKE` no usa índices eficientemente, no calcula relevancia ni realiza análisis lingüístico |
| Creación del Search Index | Los field mappings con `dynamic: false` controlan qué se indexa; el analizador `en` aplica stemming en inglés |
| Match Query | Busca términos individuales con análisis lingüístico; `operator: "or"` amplía los resultados |
| Match Phrase Query | Busca frases exactas preservando el orden; más restrictiva que match |
| Prefix Query | Encuentra términos que comienzan con un prefijo dado |
| Boolean Query | Combina condiciones `must`, `should` y `must_not` con boost por campo |
| Scoring TF-IDF | Los documentos con términos más frecuentes y menos comunes en la colección reciben scores más altos |
| REST API (puerto 8094) | Toda consulta FTS puede ejecutarse via HTTP POST con JSON, habilitando integración desde cualquier lenguaje |

### Decisión de diseño: cuándo usar cada tecnología

```
¿La búsqueda es sobre texto libre o requiere relevancia?
    ├── SÍ → Usar Couchbase Search (FTS)
    └── NO → ¿Es un filtro exacto, rango o agregación?
                ├── SÍ → Usar SQL++
                └── AMBOS → Usar SEARCH() integrado en SQL++
```

### Recursos adicionales

- [Documentación oficial: Full Text Search Overview](https://docs.couchbase.com/server/current/fts/fts-introduction.html)
- [Tipos de queries FTS disponibles](https://docs.couchbase.com/server/current/fts/fts-supported-queries.html)
- [Field Mappings y Analyzers en Couchbase Search](https://docs.couchbase.com/server/current/fts/fts-creating-index-from-UI-classic-editor-analyzers.html)
- [REST API del servicio Search](https://docs.couchbase.com/server/current/rest-api/rest-fts.html)
- [Función SEARCH() en SQL++](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/searchfun.html)
- [Bleve: motor FTS subyacente de Couchbase](https://blevesearch.com/docs/Query-String-Query/)

---

> **Próximo laboratorio:** En el **Lab 10-00-02** profundizarás en la creación de índices FTS avanzados con mappings personalizados, analizadores custom, index aliases y la integración de `SEARCH()` dentro de consultas SQL++ complejas para combinar filtros estructurados con búsqueda de texto completo en una sola instrucción.
