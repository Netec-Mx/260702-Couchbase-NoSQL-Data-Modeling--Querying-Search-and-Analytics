# Implementación de consultas de búsqueda simples y compuestas

## Metadatos

| Campo | Detalle |
|---|---|
| **Duración estimada** | 70 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Aplicar (Apply) |
| **Módulo** | 11 — Full Text Search: Consultas Simples y Compuestas |
| **Dataset requerido** | `travel-sample` (documentos `hotel` y `landmark`) |
| **Versión Couchbase** | 7.6.x (Community Edition o Enterprise Trial) |

---

## Descripción General

En este laboratorio implementarás el ciclo completo de consultas Full Text Search (FTS) en Couchbase, comenzando por la interfaz gráfica de la Web Console y avanzando hasta la integración con SQL++. Trabajarás exclusivamente sobre el bucket `travel-sample`, enfocándote en documentos de tipo `hotel` y `landmark` que contienen campos de texto libre como `description`, `name` y `reviews`. Explorarás los tipos de queries atómicas (match, term, phrase, prefix), construirás consultas compuestas con operadores booleanos, ejecutarás búsquedas vía REST API con `curl` y finalmente integrarás FTS dentro de SQL++ usando la función `SEARCH()`.

---

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Ejecutar consultas Full Text Search básicas desde la Web Console de Couchbase, identificando los componentes del panel Search y la estructura de los resultados (score, fragments, fields).
- [ ] Construir Query String Queries con operadores booleanos (`+`, `-`, `OR`), wildcards (`*`, `?`) y rangos numéricos sobre el bucket `travel-sample`.
- [ ] Implementar match queries, term queries, phrase queries y prefix queries, comprendiendo las diferencias semánticas entre cada tipo.
- [ ] Combinar condiciones de búsqueda usando consultas compuestas (conjunction, disjunction y boolean queries con `must`/`should`/`must_not`).
- [ ] Ejecutar consultas FTS mediante REST API con `curl` y Postman, e integrar FTS dentro de SQL++ usando la función `SEARCH()`.

---

## Prerrequisitos

### Conocimiento previo

- Haber creado previamente al menos un índice Full Text Search básico sobre el bucket `travel-sample` (cubierto en módulos anteriores del curso).
- Conocimiento básico de la estructura de documentos JSON en Couchbase (buckets, scopes, collections).
- Familiaridad con la navegación básica de la Web Console de Couchbase.
- Comprensión básica de sentencias SQL `SELECT` con cláusulas `WHERE`.
- `curl` instalado y funcional en el sistema (verificar con `curl --version`).

### Acceso requerido

- Acceso a la Web Console de Couchbase en `http://localhost:8091` con credenciales de administrador.
- El bucket `travel-sample` cargado y con documentos indexados.
- El servicio **Search** habilitado en el nodo Couchbase.
- Acceso a terminal/línea de comandos para ejecutar `curl`.

---

## Entorno de Laboratorio

### Requisitos de Hardware

| Recurso | Mínimo | Recomendado |
|---|---|---|
| **RAM** | 8 GB disponibles | 16 GB |
| **CPU** | 4 núcleos x86_64 | 8 núcleos |
| **Almacenamiento** | 20 GB libres (SSD) | 50 GB SSD |
| **Pantalla** | 1280×768 | 1280×800 o superior |
| **Red** | localhost funcional | Puertos 8091–8097 disponibles |

### Requisitos de Software

| Software | Versión requerida | Uso en este lab |
|---|---|---|
| Couchbase Server | 7.6.x | Servicio FTS, Web Console, Query |
| Navegador Web | Chrome 110+ / Firefox 110+ / Edge 110+ | Web Console |
| `curl` | 7.x o superior | REST API FTS |
| Postman o Bruno | Postman 10.x / Bruno 1.x | REST API FTS (alternativo) |
| VS Code u otro editor | Cualquier versión reciente | Editar JSON de queries |

### Verificación del Entorno

Antes de comenzar, ejecuta los siguientes comandos para confirmar que el entorno está operativo:

```bash
# Verificar que Couchbase responde
curl -s -u Administrator:password http://localhost:8091/pools | python3 -m json.tool | head -5

# Verificar que el servicio Search está activo
curl -s -u Administrator:password http://localhost:8091/pools/default/nodeServices \
  | python3 -m json.tool | grep -i "fts"

# Verificar que travel-sample está cargado
curl -s -u Administrator:password \
  http://localhost:8091/pools/default/buckets/travel-sample \
  | python3 -m json.tool | grep '"name"'
```

> **Nota:** Reemplaza `Administrator` y `password` con tus credenciales reales en todos los comandos de este laboratorio.

---

## Pasos del Laboratorio

---

### Parte 1 — Preparación: Verificar y Crear el Índice FTS Base

---

#### Paso 1.1 — Verificar el índice FTS existente en la Web Console

**Objetivo:** Confirmar que existe un índice FTS funcional sobre el bucket `travel-sample` antes de ejecutar cualquier consulta.

**Instrucciones:**

1. Abre tu navegador y navega a `http://localhost:8091`.
2. Inicia sesión con tus credenciales de administrador.
3. En el menú lateral izquierdo, haz clic en **Search**.
4. Observa la tabla de índices FTS disponibles. Verifica que exista un índice con las siguientes características:

| Campo | Valor esperado |
|---|---|
| **Nombre** | `travel-sample-fts-index` (o similar) |
| **Bucket** | `travel-sample` |
| **Estado** | `ready` |
| **Documentos indexados** | > 0 |

5. Si el índice **no existe**, crea uno básico siguiendo el sub-paso a continuación.

**Sub-paso — Crear índice FTS básico (solo si no existe):**

Si no tienes un índice FTS, créalo mediante REST API con el siguiente comando:

```bash
curl -s -u Administrator:password \
  -X PUT \
  -H "Content-Type: application/json" \
  http://localhost:8094/api/index/travel-fts-lab \
  -d '{
    "type": "fulltext-index",
    "name": "travel-fts-lab",
    "sourceType": "gocbcore",
    "sourceName": "travel-sample",
    "planParams": {
      "maxPartitionsPerPIndex": 1024,
      "indexPartitions": 1
    },
    "params": {
      "doc_config": {
        "docid_prefix_delim": "",
        "docid_regexp": "",
        "mode": "type_field",
        "type_field": "type"
      },
      "mapping": {
        "default_analyzer": "standard",
        "default_datetime_parser": "dateTimeOptional",
        "default_field": "_all",
        "default_mapping": {
          "dynamic": true,
          "enabled": true
        },
        "default_type": "_default",
        "docvalues_dynamic": false,
        "index_dynamic": true,
        "store_dynamic": false,
        "type_field": "_type"
      },
      "store": {
        "indexType": "scorch",
        "segmentVersion": 15
      }
    },
    "sourceParams": {}
  }'
```

6. Espera entre 30 y 60 segundos y recarga la página de la Web Console. Verifica que el estado del índice sea `ready`.

**Salida esperada (REST API):**

```json
{"status":"ok"}
```

**Verificación:**

```bash
# Verificar estado del índice via REST
curl -s -u Administrator:password \
  http://localhost:8094/api/index/travel-fts-lab \
  | python3 -m json.tool | grep -E '"status"|"docCount"'
```

---

### Parte 2 — Consultas desde la Web Console

---

#### Paso 2.1 — Ejecutar una Query String Query simple

**Objetivo:** Familiarizarse con el panel Search de la Web Console ejecutando una búsqueda de texto libre sobre documentos `hotel`.

**Instrucciones:**

1. En la sección **Search** de la Web Console, localiza tu índice FTS y haz clic en el botón **Search** (lupa) de la fila correspondiente.
2. Se desplegará el panel de búsqueda debajo de la tabla de índices.
3. En el cuadro de texto de consulta, escribe:

```
hotel
```

4. Haz clic en el botón azul **Search**.
5. Observa los resultados. Identifica para cada resultado:
   - El campo **`id`** (clave del documento).
   - El campo **`score`** (puntuación de relevancia).
   - Los **fragmentos resaltados** (`fragments`) con el término marcado en negrita.
6. Anota el número total de resultados (*total hits*) que aparece en la parte superior del panel.

**Salida esperada:**

El panel mostrará una lista de documentos ordenados por score descendente. El número total de hits debería ser mayor a 100. Los fragmentos resaltarán la palabra "hotel" dentro del contexto del texto.

```
Total hits: 1XX
Result 1: id="hotel_10025", score=2.341, ...
Result 2: id="hotel_11288", score=2.198, ...
...
```

**Verificación:**

- Confirma que el campo `score` del primer resultado es mayor al del último resultado visible (orden descendente correcto).
- Confirma que los fragmentos contienen la palabra buscada resaltada.

---

#### Paso 2.2 — Usar operadores booleanos en Query String

**Objetivo:** Construir Query String Queries con operadores `+` (must), `-` (must not) y `OR` para refinar resultados.

**Instrucciones:**

1. En el mismo panel Search, escribe la siguiente query en el cuadro de texto:

```
+hotel +swimming pool
```

> **Explicación:** El operador `+` indica que el término es obligatorio. Esta query busca documentos que contengan "hotel" Y "swimming" Y opcionalmente "pool".

2. Haz clic en **Search** y anota el número de resultados.
3. Modifica la query para excluir documentos que mencionen "expensive":

```
+hotel +swimming -expensive
```

4. Haz clic en **Search** y compara el número de resultados con la query anterior.
5. Ahora prueba una query con `OR` explícito:

```
hotel OR landmark
```

6. Haz clic en **Search** y observa cómo aumenta el número de resultados al usar `OR`.
7. Prueba una búsqueda por campo específico (field-scoped):

```
name:hotel description:pool
```

> **Explicación:** La sintaxis `campo:valor` restringe la búsqueda a un campo específico del documento.

8. Haz clic en **Search** y observa los resultados.

**Salida esperada:**

| Query | Hits aproximados |
|---|---|
| `+hotel +swimming pool` | 30–80 |
| `+hotel +swimming -expensive` | Igual o menor que la anterior |
| `hotel OR landmark` | > 200 |
| `name:hotel description:pool` | 10–50 |

**Verificación:**

- La query con `-expensive` debe tener igual o menor cantidad de hits que la query sin ese operador.
- La query con `OR` debe tener más resultados que cualquiera de las queries con `AND` implícito.

---

#### Paso 2.3 — Wildcards y rangos en Query String

**Objetivo:** Usar caracteres comodín (`*`, `?`) y rangos numéricos en Query String Queries.

**Instrucciones:**

1. En el panel Search, escribe una query con wildcard de sufijo:

```
swim*
```

> **Explicación:** El asterisco `*` reemplaza cero o más caracteres. Esta query encontrará "swim", "swimming", "swimmer", etc.

2. Haz clic en **Search** y anota los resultados.
3. Prueba con wildcard de un carácter:

```
h?tel
```

> **Explicación:** El signo `?` reemplaza exactamente un carácter. Encontrará "hotel", "h0tel", etc.

4. Haz clic en **Search**.
5. Ahora prueba una búsqueda con rango numérico sobre el campo `reviews.ratings.Overall` (si existe en el índice):

```
reviews.ratings.Overall:>4
```

6. Haz clic en **Search** y observa los resultados.
7. Prueba un rango cerrado:

```
reviews.ratings.Overall:[3 TO 5]
```

8. Haz clic en **Search**.

**Salida esperada:**

- `swim*` debe retornar documentos que contengan variaciones de la palabra "swim".
- `h?tel` debe retornar principalmente documentos con la palabra "hotel".
- Los rangos numéricos deben filtrar documentos según el valor del campo especificado.

**Verificación:**

- Para `swim*`, verifica en los fragmentos que aparezcan términos como "swimming", "swimmers" o "swim".
- Para el rango `[3 TO 5]`, verifica que los scores de los documentos retornados correspondan a documentos con ratings dentro del rango.

---

### Parte 3 — Tipos de Queries Atómicas (JSON Estructurado)

---

#### Paso 3.1 — Match Query

**Objetivo:** Comprender y ejecutar una Match Query, que aplica análisis de texto (tokenización, stemming) al término buscado.

**Instrucciones:**

1. En el panel Search de la Web Console, haz clic en el botón **Advanced** o cambia el modo a JSON (según la versión de la consola).
2. Reemplaza el contenido del cuadro de texto con el siguiente JSON:

```json
{
  "query": {
    "match": "swimming pool",
    "field": "description",
    "analyzer": "standard",
    "fuzziness": 0,
    "operator": "or"
  },
  "size": 10,
  "from": 0,
  "highlight": {
    "style": "html",
    "fields": ["description"]
  }
}
```

3. Haz clic en **Search**.
4. Observa los resultados. Los documentos deberían ser hoteles con descripciones que mencionen "swimming" o "pool" (o ambos, con mayor score los que tengan ambos términos).
5. Modifica el campo `"operator"` de `"or"` a `"and"` y vuelve a ejecutar:

```json
{
  "query": {
    "match": "swimming pool",
    "field": "description",
    "analyzer": "standard",
    "operator": "and"
  },
  "size": 10,
  "from": 0,
  "highlight": {
    "style": "html",
    "fields": ["description"]
  }
}
```

6. Compara el número de resultados entre `operator: "or"` y `operator: "and"`.

**Salida esperada:**

- Con `operator: "or"`: más resultados (documentos con "swimming" O "pool").
- Con `operator: "and"`: menos resultados (solo documentos con "swimming" Y "pool").

**Verificación:**

```bash
# Verificar vía REST API el mismo match query
curl -s -u Administrator:password \
  -X POST \
  -H "Content-Type: application/json" \
  http://localhost:8094/api/index/travel-fts-lab/query \
  -d '{
    "query": {
      "match": "swimming pool",
      "field": "description",
      "operator": "and"
    },
    "size": 5
  }' | python3 -m json.tool | grep -E '"total_hits"|"id"|"score"'
```

---

#### Paso 3.2 — Term Query

**Objetivo:** Ejecutar una Term Query, que busca el término exacto sin aplicar análisis de texto.

**Instrucciones:**

1. En el panel JSON del Search, ingresa la siguiente query:

```json
{
  "query": {
    "term": "Swimming",
    "field": "description"
  },
  "size": 10,
  "from": 0
}
```

> **Diferencia clave con Match Query:** La Term Query busca el término exacto tal como está escrito, incluyendo mayúsculas/minúsculas. No aplica tokenización ni stemming.

2. Haz clic en **Search** y anota el número de resultados.
3. Modifica el término a minúsculas:

```json
{
  "query": {
    "term": "swimming",
    "field": "description"
  },
  "size": 10,
  "from": 0
}
```

4. Haz clic en **Search** y compara los resultados con la búsqueda en mayúsculas.
5. Compara también con el resultado de la Match Query del paso anterior para el mismo término.

**Salida esperada:**

- `"term": "Swimming"` (mayúscula) probablemente retorne 0 resultados si el índice usa el analyzer `standard` (que convierte a minúsculas durante la indexación).
- `"term": "swimming"` (minúscula) retornará resultados.
- La Match Query típicamente retorna más resultados que la Term Query para el mismo término, porque aplica análisis.

**Verificación:**

Anota en tu cuaderno de laboratorio:

| Query Type | Término | Resultados |
|---|---|---|
| Term Query | "Swimming" | ___ |
| Term Query | "swimming" | ___ |
| Match Query | "swimming pool" (OR) | ___ |
| Match Query | "swimming pool" (AND) | ___ |

---

#### Paso 3.3 — Phrase Query

**Objetivo:** Ejecutar una Phrase Query para buscar una secuencia exacta de palabras en el orden especificado.

**Instrucciones:**

1. En el panel JSON del Search, ingresa:

```json
{
  "query": {
    "match_phrase": "bed and breakfast",
    "field": "description"
  },
  "size": 10,
  "from": 0,
  "highlight": {
    "style": "html",
    "fields": ["description"]
  }
}
```

2. Haz clic en **Search** y observa los resultados. Los documentos retornados deben contener la frase exacta "bed and breakfast" en el campo `description`.
3. Para comparar, ejecuta una Match Query con el mismo texto:

```json
{
  "query": {
    "match": "bed and breakfast",
    "field": "description",
    "operator": "or"
  },
  "size": 10,
  "from": 0
}
```

4. Compara el número de resultados entre la Phrase Query y la Match Query.

**Salida esperada:**

- La Phrase Query retornará menos resultados que la Match Query, pero con mayor precisión: solo documentos donde las palabras aparezcan juntas y en ese orden.
- Los fragmentos resaltados de la Phrase Query mostrarán la frase completa marcada.

**Verificación:**

```bash
curl -s -u Administrator:password \
  -X POST \
  -H "Content-Type: application/json" \
  http://localhost:8094/api/index/travel-fts-lab/query \
  -d '{
    "query": {
      "match_phrase": "bed and breakfast",
      "field": "description"
    },
    "size": 5,
    "highlight": {}
  }' | python3 -m json.tool | grep -E '"total_hits"|"id"'
```

---

#### Paso 3.4 — Prefix Query

**Objetivo:** Ejecutar una Prefix Query para encontrar documentos donde un campo comience con un prefijo específico.

**Instrucciones:**

1. En el panel JSON del Search, ingresa:

```json
{
  "query": {
    "prefix": "swim",
    "field": "description"
  },
  "size": 10,
  "from": 0,
  "highlight": {
    "style": "html",
    "fields": ["description"]
  }
}
```

2. Haz clic en **Search**.
3. Verifica en los fragmentos que los términos encontrados comiencen con "swim" (swimming, swimmer, swimwear, etc.).
4. Prueba con el prefijo `"restaur"` sobre el campo `description`:

```json
{
  "query": {
    "prefix": "restaur",
    "field": "description"
  },
  "size": 10,
  "from": 0
}
```

5. Haz clic en **Search** y observa los resultados.

**Salida esperada:**

- Los resultados de `"prefix": "swim"` mostrarán documentos con palabras como "swimming", "swimmers", etc.
- Los resultados de `"prefix": "restaur"` mostrarán documentos con "restaurant", "restaurants", "restauration", etc.

**Verificación:**

- En los fragmentos resaltados, confirma visualmente que todos los términos marcados comienzan con el prefijo especificado.

---

### Parte 4 — Consultas Compuestas

---

#### Paso 4.1 — Conjunction Query (AND lógico)

**Objetivo:** Combinar múltiples condiciones de búsqueda con semántica AND usando una Conjunction Query.

**Instrucciones:**

1. En el panel JSON del Search, ingresa la siguiente Conjunction Query:

```json
{
  "query": {
    "conjuncts": [
      {
        "match": "hotel",
        "field": "type"
      },
      {
        "match": "swimming pool",
        "field": "description",
        "operator": "and"
      },
      {
        "match": "free wifi",
        "field": "description",
        "operator": "or"
      }
    ]
  },
  "size": 10,
  "from": 0,
  "highlight": {
    "style": "html",
    "fields": ["description", "name"]
  }
}
```

> **Explicación:** Una Conjunction Query requiere que **todos** los subqueries coincidan. Solo se retornarán documentos de tipo "hotel" que tengan "swimming pool" Y mencionen "free" o "wifi" en la descripción.

2. Haz clic en **Search** y anota el número de resultados.
3. Observa los fragmentos para confirmar que los documentos cumplen todas las condiciones.

**Salida esperada:**

Los resultados serán documentos que satisfagan simultáneamente las tres condiciones. El número de hits será menor que si se buscara cada condición por separado.

**Verificación:**

```bash
curl -s -u Administrator:password \
  -X POST \
  -H "Content-Type: application/json" \
  http://localhost:8094/api/index/travel-fts-lab/query \
  -d '{
    "query": {
      "conjuncts": [
        {"match": "hotel", "field": "type"},
        {"match": "swimming pool", "field": "description", "operator": "and"}
      ]
    },
    "size": 5
  }' | python3 -m json.tool | grep '"total_hits"'
```

---

#### Paso 4.2 — Disjunction Query (OR lógico)

**Objetivo:** Combinar condiciones con semántica OR usando una Disjunction Query, con control del mínimo de condiciones que deben cumplirse.

**Instrucciones:**

1. En el panel JSON del Search, ingresa:

```json
{
  "query": {
    "disjuncts": [
      {
        "match": "beach",
        "field": "description"
      },
      {
        "match": "mountain",
        "field": "description"
      },
      {
        "match": "city center",
        "field": "description",
        "operator": "and"
      }
    ],
    "min": 1
  },
  "size": 10,
  "from": 0,
  "highlight": {
    "style": "html",
    "fields": ["description"]
  }
}
```

> **Explicación:** Una Disjunction Query requiere que **al menos `min`** de los subqueries coincidan. Con `"min": 1`, basta con que una condición se cumpla. Con `"min": 2`, al menos dos deben cumplirse.

2. Haz clic en **Search** y anota el número de resultados.
3. Cambia `"min": 1` a `"min": 2` y vuelve a ejecutar. Observa cómo disminuye el número de resultados.
4. Cambia a `"min": 3` y ejecuta nuevamente.

**Salida esperada:**

| Valor de `min` | Hits esperados |
|---|---|
| 1 | Mayor número |
| 2 | Número intermedio |
| 3 | Menor número |

**Verificación:**

- Confirma que con `min: 2`, los fragmentos de cada resultado muestran al menos dos de los términos buscados (beach, mountain o city center).

---

#### Paso 4.3 — Boolean Query con must / should / must_not

**Objetivo:** Construir una Boolean Query completa usando las cláusulas `must`, `should` y `must_not` para control preciso de la relevancia.

**Instrucciones:**

1. En el panel JSON del Search, ingresa la siguiente Boolean Query:

```json
{
  "query": {
    "must": {
      "conjuncts": [
        {
          "match": "hotel",
          "field": "type"
        }
      ]
    },
    "should": {
      "disjuncts": [
        {
          "match": "pool",
          "field": "description"
        },
        {
          "match": "spa",
          "field": "description"
        },
        {
          "match": "gym",
          "field": "description"
        }
      ],
      "min": 1
    },
    "must_not": {
      "disjuncts": [
        {
          "match": "closed",
          "field": "description"
        }
      ]
    }
  },
  "size": 10,
  "from": 0,
  "highlight": {
    "style": "html",
    "fields": ["description", "name"]
  }
}
```

> **Semántica de las cláusulas:**
> - **`must`**: condiciones obligatorias. Los documentos que no las cumplan son excluidos.
> - **`should`**: condiciones que aumentan el score si se cumplen, pero no son obligatorias (a menos que `min` sea mayor que 0).
> - **`must_not`**: condiciones de exclusión. Los documentos que las cumplan son excluidos.

2. Haz clic en **Search** y analiza los resultados.
3. Verifica que:
   - Todos los resultados son de tipo "hotel" (condición `must`).
   - Los primeros resultados mencionan "pool", "spa" o "gym" (condición `should` eleva el score).
   - Ningún resultado menciona "closed" en la descripción (condición `must_not`).

**Salida esperada:**

Resultados de tipo hotel, ordenados por relevancia donde los que mencionan amenidades (pool, spa, gym) aparecen primero, y sin ningún documento que mencione "closed".

**Verificación:**

```bash
curl -s -u Administrator:password \
  -X POST \
  -H "Content-Type: application/json" \
  http://localhost:8094/api/index/travel-fts-lab/query \
  -d '{
    "query": {
      "must": {
        "conjuncts": [{"match": "hotel", "field": "type"}]
      },
      "must_not": {
        "disjuncts": [{"match": "closed", "field": "description"}]
      }
    },
    "size": 3
  }' | python3 -m json.tool | grep -E '"id"|"score"'
```

---

### Parte 5 — Ejecución vía REST API con curl y Postman

---

#### Paso 5.1 — Consulta básica con curl

**Objetivo:** Ejecutar consultas FTS directamente contra el endpoint REST del servicio Search, interpretando la estructura completa de la respuesta JSON.

**Instrucciones:**

1. Abre una terminal y ejecuta la siguiente consulta básica:

```bash
curl -s -u Administrator:password \
  -X POST \
  -H "Content-Type: application/json" \
  http://localhost:8094/api/index/travel-fts-lab/query \
  -d '{
    "query": {
      "match": "beachfront hotel",
      "field": "description",
      "operator": "or"
    },
    "size": 5,
    "from": 0,
    "highlight": {
      "style": "html",
      "fields": ["description", "name"]
    },
    "fields": ["name", "description", "country", "type"]
  }' | python3 -m json.tool
```

2. Analiza la respuesta JSON. Identifica las secciones principales:

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
      "index": "travel-fts-lab",
      "id": "hotel_XXXXX",
      "score": 2.345,
      "locations": { ... },
      "fragments": {
        "description": ["...texto con <mark>hotel</mark>..."]
      },
      "fields": {
        "name": "Nombre del hotel",
        "country": "United Kingdom"
      }
    }
  ],
  "total_hits": 42,
  "max_score": 2.345,
  "took": 5000000,
  "facets": null
}
```

3. Anota los siguientes valores de la respuesta:
   - `total_hits`: número total de documentos que coinciden.
   - `max_score`: puntuación del resultado más relevante.
   - `took`: tiempo de ejecución en nanosegundos.

**Salida esperada:**

```
"total_hits": XX,
"max_score": X.XXX,
"took": XXXXXXX
```

**Verificación:**

```bash
# Verificar que la respuesta tiene el campo status.successful = 1
curl -s -u Administrator:password \
  -X POST \
  -H "Content-Type: application/json" \
  http://localhost:8094/api/index/travel-fts-lab/query \
  -d '{"query": {"match_all": {}}, "size": 1}' \
  | python3 -m json.tool | grep '"successful"'
```

---

#### Paso 5.2 — Paginación con from y size

**Objetivo:** Implementar paginación en las consultas FTS REST API usando los parámetros `from` y `size`.

**Instrucciones:**

1. Ejecuta la primera página de resultados (documentos 1–5):

```bash
curl -s -u Administrator:password \
  -X POST \
  -H "Content-Type: application/json" \
  http://localhost:8094/api/index/travel-fts-lab/query \
  -d '{
    "query": {
      "match": "hotel",
      "field": "type"
    },
    "size": 5,
    "from": 0,
    "fields": ["name", "country"]
  }' | python3 -m json.tool | grep -E '"id"|"name"'
```

2. Ejecuta la segunda página (documentos 6–10):

```bash
curl -s -u Administrator:password \
  -X POST \
  -H "Content-Type: application/json" \
  http://localhost:8094/api/index/travel-fts-lab/query \
  -d '{
    "query": {
      "match": "hotel",
      "field": "type"
    },
    "size": 5,
    "from": 5,
    "fields": ["name", "country"]
  }' | python3 -m json.tool | grep -E '"id"|"name"'
```

3. Verifica que los IDs de los resultados de la segunda página son diferentes a los de la primera.

**Salida esperada:**

Página 1 y Página 2 deben mostrar conjuntos de IDs completamente distintos, con el mismo `total_hits` en ambas respuestas.

**Verificación:**

- Guarda los IDs de ambas páginas y confirma que no hay duplicados entre ellas.

---

#### Paso 5.3 — Consulta con curl usando consulta compuesta

**Objetivo:** Ejecutar una Boolean Query compleja vía REST API y procesar la respuesta con `python3`.

**Instrucciones:**

1. Crea un archivo JSON con la consulta para mayor legibilidad:

```bash
cat > /tmp/fts_query.json << 'EOF'
{
  "query": {
    "must": {
      "conjuncts": [
        {
          "match": "hotel",
          "field": "type"
        },
        {
          "prefix": "restaur",
          "field": "description"
        }
      ]
    },
    "should": {
      "disjuncts": [
        {"match": "wifi", "field": "description"},
        {"match": "parking", "field": "description"}
      ],
      "min": 1
    }
  },
  "size": 10,
  "from": 0,
  "fields": ["name", "country", "description"],
  "highlight": {
    "style": "html",
    "fields": ["description"]
  }
}
EOF
```

2. Ejecuta la consulta referenciando el archivo:

```bash
curl -s -u Administrator:password \
  -X POST \
  -H "Content-Type: application/json" \
  http://localhost:8094/api/index/travel-fts-lab/query \
  -d @/tmp/fts_query.json \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
print(f'Total hits: {data[\"total_hits\"]}')
print(f'Max score: {data[\"max_score\"]}')
print(f'Tiempo (ms): {data[\"took\"] / 1_000_000:.2f}')
print('--- Top 3 resultados ---')
for hit in data['hits'][:3]:
    print(f'  ID: {hit[\"id\"]} | Score: {hit[\"score\"]:.4f} | Nombre: {hit.get(\"fields\", {}).get(\"name\", \"N/A\")}')
"
```

**Salida esperada:**

```
Total hits: XX
Max score: X.XXXX
Tiempo (ms): X.XX
--- Top 3 resultados ---
  ID: hotel_XXXXX | Score: X.XXXX | Nombre: Hotel XYZ
  ID: hotel_XXXXX | Score: X.XXXX | Nombre: Hotel ABC
  ID: hotel_XXXXX | Score: X.XXXX | Nombre: Hotel DEF
```

**Verificación:**

- Confirma que `Total hits` > 0.
- Confirma que los nombres de los hoteles son coherentes con la búsqueda.

---

#### Paso 5.4 — Ejecutar consulta FTS desde Postman

**Objetivo:** Reproducir la misma consulta REST API usando Postman para familiarizarse con herramientas de cliente HTTP.

**Instrucciones:**

1. Abre Postman (o Bruno).
2. Crea una nueva petición con los siguientes parámetros:

| Campo | Valor |
|---|---|
| **Método** | `POST` |
| **URL** | `http://localhost:8094/api/index/travel-fts-lab/query` |
| **Auth Type** | Basic Auth |
| **Username** | `Administrator` |
| **Password** | `password` |
| **Body Type** | `raw` → `JSON` |

3. En el cuerpo de la petición, pega:

```json
{
  "query": {
    "match_phrase": "bed and breakfast",
    "field": "description"
  },
  "size": 5,
  "from": 0,
  "highlight": {
    "style": "html",
    "fields": ["description"]
  },
  "fields": ["name", "country", "type"]
}
```

4. Haz clic en **Send**.
5. En la respuesta, verifica:
   - El código HTTP es `200 OK`.
   - El campo `status.successful` es `1`.
   - El campo `total_hits` es mayor que 0.
   - Los fragmentos en `hits[0].fragments.description` contienen la frase "bed and breakfast" resaltada.

**Salida esperada:**

```json
{
  "status": {"total": 1, "failed": 0, "successful": 1},
  "total_hits": XX,
  "hits": [...]
}
```

**Verificación:**

- Código de respuesta HTTP: `200 OK`.
- `"successful": 1` en el campo `status`.

---

### Parte 6 — Integración FTS con SQL++

---

#### Paso 6.1 — Usar la función SEARCH() en SQL++

**Objetivo:** Integrar Full Text Search dentro de SQL++ usando la función `SEARCH()` para combinar búsqueda de texto con filtros relacionales.

**Instrucciones:**

1. Abre la Web Console y navega a la sección **Query** (Editor SQL++).
2. Ejecuta la siguiente consulta SQL++ con la función `SEARCH()`:

```sql
SELECT h.name, h.country, h.description, SEARCH_SCORE() AS relevance_score
FROM `travel-sample` AS h
WHERE h.type = 'hotel'
  AND SEARCH(h, {
    "query": {
      "match": "swimming pool",
      "field": "description",
      "operator": "and"
    }
  })
ORDER BY relevance_score DESC
LIMIT 10;
```

> **Nota:** La función `SEARCH()` requiere que exista un índice FTS activo sobre el bucket. `SEARCH_SCORE()` retorna el score FTS del documento para ordenar por relevancia.

3. Haz clic en **Execute** y observa los resultados.
4. Modifica la consulta para agregar un filtro SQL++ adicional (filtro relacional combinado con FTS):

```sql
SELECT h.name, h.country, h.city, SEARCH_SCORE() AS relevance_score
FROM `travel-sample` AS h
WHERE h.type = 'hotel'
  AND h.country = 'United Kingdom'
  AND SEARCH(h, {
    "query": {
      "match": "breakfast",
      "field": "description"
    }
  })
ORDER BY relevance_score DESC
LIMIT 5;
```

5. Ejecuta la consulta y verifica que todos los resultados son hoteles del Reino Unido.

**Salida esperada:**

```
[
  {
    "name": "Hotel XYZ",
    "country": "United Kingdom",
    "city": "London",
    "relevance_score": 2.341
  },
  ...
]
```

**Verificación:**

- Todos los documentos retornados deben tener `"country": "United Kingdom"`.
- Los scores deben estar en orden descendente.

---

#### Paso 6.2 — SEARCH() con consulta compuesta en SQL++

**Objetivo:** Usar una Boolean Query compleja dentro de `SEARCH()` en SQL++, combinando con agregaciones SQL++.

**Instrucciones:**

1. En el editor SQL++ de la Web Console, ejecuta:

```sql
SELECT h.country,
       COUNT(*) AS total_hoteles,
       AVG(SEARCH_SCORE()) AS avg_score
FROM `travel-sample` AS h
WHERE h.type = 'hotel'
  AND SEARCH(h, {
    "query": {
      "must": {
        "conjuncts": [
          {"match": "hotel", "field": "type"}
        ]
      },
      "should": {
        "disjuncts": [
          {"match": "pool", "field": "description"},
          {"match": "spa", "field": "description"},
          {"match": "gym", "field": "description"}
        ],
        "min": 1
      }
    }
  })
GROUP BY h.country
ORDER BY total_hoteles DESC
LIMIT 10;
```

2. Ejecuta la consulta y analiza los resultados. Deberías ver un conteo de hoteles con amenidades (pool, spa o gym) agrupados por país.

3. Ejecuta también una consulta con `SEARCH()` que use una Phrase Query:

```sql
SELECT h.name, h.city, h.country, SEARCH_SCORE() AS score
FROM `travel-sample` AS h
WHERE SEARCH(h, {
  "query": {
    "match_phrase": "free parking",
    "field": "description"
  }
})
  AND h.type = 'hotel'
ORDER BY score DESC
LIMIT 8;
```

4. Ejecuta y verifica que los resultados contienen la frase "free parking" en la descripción.

**Salida esperada (consulta de agregación):**

```json
[
  {"country": "United Kingdom", "total_hoteles": XX, "avg_score": X.XXX},
  {"country": "France", "total_hoteles": XX, "avg_score": X.XXX},
  ...
]
```

**Verificación:**

```bash
# Verificar via cbq (Couchbase Query Shell)
cbq -u Administrator -p password -s \
  "SELECT COUNT(*) AS total FROM \`travel-sample\` WHERE type='hotel' AND SEARCH(\`travel-sample\`, {\"query\":{\"match\":\"pool\",\"field\":\"description\"}}) LIMIT 1;"
```

---

## Validación y Pruebas Finales

Una vez completados todos los pasos, ejecuta las siguientes validaciones para confirmar que el laboratorio fue completado correctamente:

### Validación 1 — Verificar que el índice FTS está operativo

```bash
curl -s -u Administrator:password \
  http://localhost:8094/api/index/travel-fts-lab \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
status = data.get('status', 'unknown')
doc_count = data.get('indexDef', {}).get('params', {})
print(f'Estado del índice: {status}')
"
```

**Resultado esperado:** `Estado del índice: ok`

### Validación 2 — Confirmar que las queries retornan resultados

```bash
# Test match query
curl -s -u Administrator:password \
  -X POST -H "Content-Type: application/json" \
  http://localhost:8094/api/index/travel-fts-lab/query \
  -d '{"query":{"match":"hotel","field":"type"},"size":1}' \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print('Match Query OK - hits:', d['total_hits'])"

# Test phrase query
curl -s -u Administrator:password \
  -X POST -H "Content-Type: application/json" \
  http://localhost:8094/api/index/travel-fts-lab/query \
  -d '{"query":{"match_phrase":"bed and breakfast","field":"description"},"size":1}' \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print('Phrase Query OK - hits:', d['total_hits'])"

# Test boolean query
curl -s -u Administrator:password \
  -X POST -H "Content-Type: application/json" \
  http://localhost:8094/api/index/travel-fts-lab/query \
  -d '{"query":{"conjuncts":[{"match":"hotel","field":"type"},{"match":"pool","field":"description"}]},"size":1}' \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print('Boolean Query OK - hits:', d['total_hits'])"
```

**Resultado esperado:**

```
Match Query OK - hits: XXX
Phrase Query OK - hits: XX
Boolean Query OK - hits: XX
```

Todos deben mostrar `hits > 0`.

### Validación 3 — Confirmar integración SQL++ con SEARCH()

En el editor SQL++ de la Web Console, ejecuta:

```sql
SELECT COUNT(*) AS fts_sql_count
FROM `travel-sample` AS h
WHERE h.type = 'hotel'
  AND SEARCH(h, {"query": {"match": "pool", "field": "description"}});
```

**Resultado esperado:** Un número mayor que 0 en el campo `fts_sql_count`.

### Lista de verificación de completitud

| Tarea | Estado |
|---|---|
| Índice FTS creado y en estado `ready` | ☐ |
| Query String Query ejecutada en Web Console | ☐ |
| Operadores booleanos (`+`, `-`, `OR`) usados | ☐ |
| Match Query ejecutada con `operator: and` y `operator: or` | ☐ |
| Term Query ejecutada y comparada con Match Query | ☐ |
| Phrase Query ejecutada sobre `description` | ☐ |
| Prefix Query ejecutada con al menos 2 prefijos | ☐ |
| Conjunction Query ejecutada con 3 condiciones | ☐ |
| Disjunction Query ejecutada con variación de `min` | ☐ |
| Boolean Query con `must`/`should`/`must_not` ejecutada | ☐ |
| Consulta REST API básica con `curl` ejecutada | ☐ |
| Paginación con `from`/`size` implementada | ☐ |
| Consulta ejecutada desde Postman | ☐ |
| `SEARCH()` en SQL++ con filtro relacional ejecutado | ☐ |
| Consulta SQL++ con agregación y `SEARCH()` ejecutada | ☐ |

---

## Resolución de Problemas

### Problema 1 — Error "index not found" al ejecutar consulta REST API

**Síntoma:**

Al ejecutar `curl` contra el endpoint FTS, se recibe la siguiente respuesta con código HTTP `400` o `404`:

```json
{
  "error": "rest_get_index: no such index",
  "status": "fail"
}
```

O en la Web Console, el panel Search muestra "No indexes found" o el índice aparece en estado `indexing` en lugar de `ready`.

**Causa:**

1. El nombre del índice en la URL no coincide exactamente con el nombre registrado en Couchbase (sensible a mayúsculas/minúsculas).
2. El índice fue creado pero aún está en proceso de indexación inicial y no ha alcanzado el estado `ready`.
3. El servicio Search no está habilitado en el nodo al que se está conectando, o el puerto `8094` está bloqueado.

**Solución:**

```bash
# Paso 1: Listar todos los índices FTS disponibles
curl -s -u Administrator:password \
  http://localhost:8094/api/index \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
indexes = data.get('indexDefs', {}).get('indexDefs', {})
for name, info in indexes.items():
    print(f'Índice: {name} | Tipo: {info.get(\"type\", \"?\")}')
"

# Paso 2: Verificar el estado del índice específico
curl -s -u Administrator:password \
  http://localhost:8094/api/index/travel-fts-lab/count \
  | python3 -m json.tool

# Paso 3: Si el índice no existe, verificar que el servicio FTS esté activo
curl -s -u Administrator:password \
  http://localhost:8091/pools/default/nodeServices \
  | python3 -m json.tool | grep -i "fts\|search"
```

- Si el índice existe pero está en estado `indexing`, espera 30–60 segundos adicionales y vuelve a intentar.
- Si el nombre no coincide, usa el nombre exacto que aparece en la lista del Paso 1.
- Si el servicio FTS no aparece en `nodeServices`, habilítalo desde la sección **Servers** de la Web Console asignando el servicio Search al nodo.

---

### Problema 2 — La función SEARCH() en SQL++ no retorna resultados (0 hits)

**Síntoma:**

La consulta SQL++ con `SEARCH()` se ejecuta sin errores pero retorna 0 documentos, aunque se sabe que existen documentos que deberían coincidir con los criterios de búsqueda:

```sql
-- Esta query retorna 0 resultados inesperadamente
SELECT name FROM `travel-sample`
WHERE type = 'hotel'
  AND SEARCH(`travel-sample`, {"query": {"match": "pool", "field": "description"}});
```

**Causa:**

1. **Índice FTS no asociado correctamente:** El índice FTS fue creado sobre el bucket completo pero la función `SEARCH()` está referenciando un alias o nombre de colección incorrecto.
2. **Campo no indexado:** El campo `description` no está siendo indexado por el índice FTS (el índice tiene `index_dynamic: false` y no incluye ese campo en el mapping).
3. **Analyzer incompatible:** El analyzer configurado en el índice transformó los tokens de una manera que no coincide con la query (por ejemplo, un analyzer que no hace stemming sobre un término en plural).
4. **Índice creado sobre una colección específica (scope/collection) pero la query no especifica la colección correcta.**

**Solución:**

```bash
# Paso 1: Verificar que el índice indexa el campo 'description'
# Revisar la definición del índice
curl -s -u Administrator:password \
  http://localhost:8094/api/index/travel-fts-lab \
  | python3 -m json.tool | grep -A 5 '"default_mapping"'

# Paso 2: Probar la misma query directamente contra la REST API de FTS
# Si la REST API retorna resultados pero SQL++ no, el problema es de referencia al índice
curl -s -u Administrator:password \
  -X POST -H "Content-Type: application/json" \
  http://localhost:8094/api/index/travel-fts-lab/query \
  -d '{"query":{"match":"pool","field":"description"},"size":5}' \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print('FTS directo - hits:', d['total_hits'])"

# Paso 3: En SQL++, asegurarse de usar el nombre correcto del índice
# La función SEARCH() puede recibir el nombre del índice como tercer parámetro
```

En el editor SQL++ de la Web Console, usa la sintaxis con nombre de índice explícito:

```sql
-- Forma correcta con nombre de índice explícito
SELECT h.name, SEARCH_SCORE() AS score
FROM `travel-sample` AS h
USE INDEX (USING FTS)
WHERE SEARCH(h, {
  "query": {"match": "pool", "field": "description"},
  "ctl": {"timeout": 10000}
}, {"index": "travel-fts-lab"})
LIMIT 5;
```

Si el problema persiste, verifica que el índice tiene `index_dynamic: true` para indexar todos los campos dinámicamente, o agrega el campo `description` explícitamente al mapping del índice desde la Web Console (sección **Search** → **Edit Index** → **Add Child Field**).

---

## Limpieza del Entorno

Una vez completado el laboratorio, realiza las siguientes acciones de limpieza opcionales para mantener el entorno ordenado:

### Eliminar archivos temporales

```bash
# Eliminar el archivo de query temporal creado durante el lab
rm -f /tmp/fts_query.json

# Verificar que el archivo fue eliminado
ls -la /tmp/fts_query.json 2>/dev/null || echo "Archivo eliminado correctamente"
```

### Conservar o eliminar el índice FTS del laboratorio

> **Decisión:** Si planeas continuar con laboratorios posteriores que usen FTS, **conserva** el índice. Si deseas limpiar completamente, elimínalo con el siguiente comando:

```bash
# OPCIONAL: Eliminar el índice FTS creado en este lab
# Solo ejecutar si no se necesita para labs posteriores
curl -s -u Administrator:password \
  -X DELETE \
  http://localhost:8094/api/index/travel-fts-lab \
  | python3 -m json.tool
```

**Salida esperada si se elimina:**

```json
{"status": "ok"}
```

### Verificación del estado final del entorno

```bash
# Verificar que Couchbase sigue operativo
curl -s -u Administrator:password \
  http://localhost:8091/pools/default \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print('Clúster OK -', d.get('name', 'unknown'))"

# Verificar que travel-sample sigue disponible
curl -s -u Administrator:password \
  http://localhost:8091/pools/default/buckets/travel-sample \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print('Bucket OK:', d.get('name','?'))"
```

---

## Resumen

En este laboratorio implementaste el ciclo completo de consultas Full Text Search en Couchbase, desde la interfaz gráfica hasta la integración con SQL++:

| Sección | Habilidades adquiridas |
|---|---|
| **Web Console** | Navegación al panel Search, ejecución de queries simples, interpretación de scores y fragments |
| **Query String Queries** | Operadores `+`/`-`/`OR`, wildcards `*`/`?`, rangos `[X TO Y]`, búsqueda por campo |
| **Queries atómicas** | Match Query (con analyzer), Term Query (exacta), Phrase Query (secuencia), Prefix Query |
| **Queries compuestas** | Conjunction (AND), Disjunction (OR con `min`), Boolean (`must`/`should`/`must_not`) |
| **REST API** | Peticiones POST con `curl`, estructura de respuesta JSON, paginación con `from`/`size` |
| **Postman** | Configuración de autenticación Basic Auth, envío de peticiones JSON |
| **SQL++ + FTS** | Función `SEARCH()`, `SEARCH_SCORE()`, combinación con filtros relacionales y agregaciones |

### Diferencias clave entre tipos de queries

| Tipo | Aplica analyzer | Busca exacto | Secuencia | Prefijo |
|---|---|---|---|---|
| **Match Query** | ✅ Sí | ❌ No | ❌ No | ❌ No |
| **Term Query** | ❌ No | ✅ Sí | ❌ No | ❌ No |
| **Phrase Query** | ✅ Sí | ❌ No | ✅ Sí | ❌ No |
| **Prefix Query** | ❌ No | Parcial | ❌ No | ✅ Sí |

### Recursos de Referencia

- [Documentación oficial — Full Text Search Query Types](https://docs.couchbase.com/server/current/fts/fts-query-types.html)
- [Documentación oficial — FTS REST API](https://docs.couchbase.com/server/current/fts/fts-rest-api.html)
- [Documentación oficial — SEARCH() function en SQL++](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/searchfun.html)
- [Documentación oficial — Query String Query syntax](https://docs.couchbase.com/server/current/fts/fts-query-string-query.html)
- [Tutorial — Full Text Search con travel-sample](https://developer.couchbase.com/tutorial-full-text-search-travel-sample)
- [Documentación oficial — Searching from the UI](https://docs.couchbase.com/server/current/fts/fts-searching-from-the-UI.html)

---
