# Creación de analyzers personalizados para búsqueda

## Metadatos

| Campo | Valor |
|---|---|
| **Duración estimada** | 60 minutos |
| **Complejidad** | Alta |
| **Nivel Bloom** | Crear (Create) |
| **Dataset requerido** | `travel-sample` |
| **Servicio Couchbase** | Search (FTS) |

---

## Descripción General

En este laboratorio construirás analyzers personalizados para Full Text Search en Couchbase, adaptados a contenido en español. Explorarás el pipeline de análisis completo (character filters → tokenizer → token filters) y crearás dos analyzers: `spanish_hotel_analyzer` para búsquedas en lenguaje natural en español, y `sku_analyzer` para coincidencia exacta de códigos. Validarás el comportamiento de cada analyzer usando el endpoint `/_analyzeDoc` y compararás los resultados frente al analyzer estándar para demostrar la mejora en relevancia.

---

## Objetivos de Aprendizaje

Al completar este laboratorio, serás capaz de:

- [ ] Identificar y describir las tres etapas del pipeline de análisis FTS: character filters, tokenizer y token filters
- [ ] Diseñar y registrar un analyzer personalizado (`spanish_hotel_analyzer`) que combine filtros de caracteres HTML, tokenizer unicode, lowercase, ASCII folding, stop words en español y stemmer
- [ ] Crear un segundo analyzer (`sku_analyzer`) con tokenizer `keyword` para búsqueda exacta de códigos
- [ ] Validar el comportamiento de los analyzers usando el endpoint REST `analyzeDoc` antes y después de indexar
- [ ] Comparar resultados de búsqueda entre el analyzer estándar y el analyzer personalizado para demostrar mejora en relevancia

---

## Prerrequisitos

### Conocimiento previo
- Haber completado la Práctica 12 o tener experiencia creando índices FTS con mappings personalizados
- Comprensión básica de la tokenización de texto (cubierta en la Lección 13.1)
- Familiaridad con la Web Console de Couchbase y el editor visual de índices FTS
- Conocimiento del formato JSON para definición de configuraciones

### Acceso y recursos
- Couchbase Server 7.6.x en ejecución (Community Edition o Enterprise Trial)
- Bucket `travel-sample` cargado y con el servicio Search habilitado
- Acceso a la Web Console en `http://localhost:8091`
- `curl` disponible en terminal (versión 7.x o superior)
- Credenciales de administrador (por defecto: `Administrator` / `password`)

---

## Entorno de Laboratorio

### Requisitos de hardware

| Recurso | Mínimo | Recomendado |
|---|---|---|
| RAM | 8 GB | 16 GB |
| CPU | 4 núcleos x86_64 | 8 núcleos |
| Almacenamiento | 20 GB libres (SSD) | 50 GB libres (SSD) |
| Red | localhost, puertos 8091–8097, 8094 disponibles | — |

### Requisitos de software

| Software | Versión |
|---|---|
| Couchbase Server | 7.6.x |
| Navegador web | Chrome 110+, Firefox 110+ o Edge 110+ |
| curl | 7.x o superior |
| Editor de texto | VS Code 1.80+ (o equivalente) |

### Verificación del entorno

Antes de comenzar, ejecuta los siguientes comandos para confirmar que el entorno está listo:

```bash
# 1. Verificar que Couchbase responde
curl -s -u Administrator:password http://localhost:8091/pools/default \
  | python3 -m json.tool | grep '"name"'

# 2. Verificar que el servicio Search (FTS) está disponible
curl -s -u Administrator:password http://localhost:8094/api/index \
  | python3 -m json.tool | head -5

# 3. Verificar que travel-sample está cargado
curl -s -u Administrator:password \
  http://localhost:8091/pools/default/buckets/travel-sample \
  | python3 -m json.tool | grep '"name"'
```

**Salida esperada de verificación:**

```
"name": "default"
...
"status": "ok"
...
"name": "travel-sample"
```

> **Nota:** Si el bucket `travel-sample` no está cargado, ve a **Settings → Sample Buckets** en la Web Console y selecciona `travel-sample`. La carga puede tardar 2–3 minutos.

---

## Procedimiento Paso a Paso

---

### Paso 1: Explorar el pipeline de análisis con el analyzer estándar

**Objetivo:** Comprender cómo el analyzer `standard` (inglés) tokeniza texto en español y por qué produce resultados subóptimos, usando el endpoint `analyzeDoc`.

#### Instrucciones

**1.1** Primero necesitamos un índice FTS de referencia para poder usar el endpoint `analyzeDoc`. Crea un índice básico temporal usando la Web Console:

1. Abre el navegador en `http://localhost:8091`
2. Ve a **Search** en el menú lateral izquierdo
3. Haz clic en **Add Index**
4. Configura los campos básicos:
   - **Index Name:** `hotels_analyzer_lab`
   - **Bucket:** `travel-sample`
   - **Scope:** `inventory`
   - **Collection:** `hotel`
5. Deja el resto de opciones por defecto y haz clic en **Create Index**

**1.2** Espera a que el índice termine de construirse (indicador verde en la lista de índices). Luego, prueba el analyzer `standard` con texto en español:

```bash
# Probar analyzer standard con texto en español
curl -s -u Administrator:password \
  -X POST \
  "http://localhost:8094/api/index/hotels_analyzer_lab/analyzeDoc" \
  -H "Content-Type: application/json" \
  -d '{
    "analyzer": "standard",
    "text": "Habitaciones con vista al mar, café incluido en el precio"
  }' | python3 -m json.tool
```

**1.3** Prueba ahora con el analyzer `es` (español built-in de Couchbase):

```bash
# Probar analyzer es (español) built-in
curl -s -u Administrator:password \
  -X POST \
  "http://localhost:8094/api/index/hotels_analyzer_lab/analyzeDoc" \
  -H "Content-Type: application/json" \
  -d '{
    "analyzer": "es",
    "text": "Habitaciones con vista al mar, café incluido en el precio"
  }' | python3 -m json.tool
```

#### Salida esperada

Para el analyzer `standard`, observarás tokens como:
```json
{
  "status": "ok",
  "analyzed_text": [
    {"term": "habitaciones", "position": 1, "start": 0, "end": 12},
    {"term": "con", "position": 2, "start": 13, "end": 16},
    {"term": "vista", "position": 3, "start": 17, "end": 22},
    {"term": "al", "position": 4, "start": 23, "end": 25},
    {"term": "mar", "position": 5, "start": 27, "end": 30},
    {"term": "café", "position": 6, "start": 32, "end": 36},
    {"term": "incluido", "position": 7, "start": 37, "end": 45},
    {"term": "en", "position": 8, "start": 46, "end": 48},
    {"term": "el", "position": 9, "start": 49, "end": 51},
    {"term": "precio", "position": 10, "start": 52, "end": 58}
  ]
}
```

> **Observación clave:** El analyzer `standard` convierte a minúsculas pero **no** elimina stop words en español (`con`, `al`, `en`, `el`), **no** hace stemming (`habitaciones` no se reduce a `habitar`) y **no** normaliza acentos (`café` permanece con acento). Esto significa que buscar `"habitacion"` (sin acento, sin plural) **no encontraría** este documento.

#### Verificación

- [ ] El endpoint devuelve `"status": "ok"`
- [ ] Los tokens del analyzer `standard` incluyen palabras vacías como `con`, `al`, `en`, `el`
- [ ] El token `café` conserva el acento en el analyzer `standard`
- [ ] El analyzer `es` produce menos tokens (elimina algunas stop words) y aplica stemming

---

### Paso 2: Diseñar el analyzer `spanish_hotel_analyzer`

**Objetivo:** Planificar y documentar la configuración del analyzer personalizado antes de implementarlo, entendiendo el rol de cada componente del pipeline.

#### Instrucciones

**2.1** Revisa la siguiente tabla de diseño del analyzer `spanish_hotel_analyzer`:

| Etapa | Tipo | Nombre | Propósito |
|---|---|---|---|
| Character Filter 1 | `html` | `html_strip` | Eliminar etiquetas HTML de descripciones copiadas de web |
| Tokenizer | `unicode` | — | Dividir correctamente texto multilingüe con acentos UTF-8 |
| Token Filter 1 | `to_lower` (lowercase) | `to_lower` | Normalizar mayúsculas/minúsculas |
| Token Filter 2 | `unicode_normalize` (ascii_folding) | `ascii_fold` | Normalizar acentos: `café → cafe`, `habitación → habitacion` |
| Token Filter 3 | `stop_tokens` | `spanish_stop` | Eliminar palabras vacías en español |
| Token Filter 4 | `stemmer` | `spanish_stem` | Reducir palabras a su raíz: `habitaciones → habitar` |

**2.2** Prepara la lista de stop words en español que usará el analyzer. Crea el archivo `spanish_stopwords.txt` en tu directorio de trabajo:

```bash
# Crear archivo con stop words en español (para referencia)
cat > /tmp/spanish_stopwords.txt << 'EOF'
de
la
el
en
con
por
para
un
una
los
las
del
al
se
su
es
que
y
a
o
no
si
EOF
```

**2.3** Construye el JSON de definición del analyzer. Guárdalo en `/tmp/spanish_hotel_analyzer.json` para usarlo en el siguiente paso:

```bash
cat > /tmp/spanish_hotel_analyzer.json << 'EOF'
{
  "char_filters": ["html"],
  "tokenizer": "unicode",
  "token_filters": [
    "to_lower",
    "ascii_folding",
    "stop_es",
    "stemmer_es"
  ]
}
EOF
```

> **Nota técnica:** En Couchbase FTS, los token filters `stop_es` y `stemmer_es` son identificadores que definiremos dentro de la configuración del índice. Los nombres `to_lower` y `ascii_folding` son token filters built-in de Couchbase.

#### Verificación

- [ ] Comprendes el rol de cada etapa del pipeline
- [ ] El character filter `html` se aplica **antes** de tokenizar
- [ ] El orden de los token filters importa: lowercase debe ir **antes** que stop words y stemming
- [ ] El archivo JSON de diseño está guardado en `/tmp/spanish_hotel_analyzer.json`

---

### Paso 3: Crear el índice FTS con analyzers personalizados vía Web Console

**Objetivo:** Registrar los analyzers `spanish_hotel_analyzer` y `sku_analyzer` dentro de un índice FTS usando el editor avanzado de la Web Console.

#### Instrucciones

**3.1** En la Web Console, ve a **Search → Add Index** y configura:

- **Index Name:** `hotels_custom_analyzers`
- **Bucket:** `travel-sample`
- **Scope:** `inventory`
- **Collection:** `hotel`

**3.2** Haz clic en **Advanced Settings** (o el botón equivalente según tu versión) para acceder al editor JSON del índice. Busca la sección **Custom Filters** o cambia al modo **JSON Editor** haciendo clic en el botón correspondiente en la parte superior del formulario.

**3.3** En el editor JSON, reemplaza el contenido completo con la siguiente definición de índice. Esta definición incluye ambos analyzers personalizados, los token filters custom y los mappings de campos:

```json
{
  "name": "hotels_custom_analyzers",
  "type": "fulltext-index",
  "params": {
    "doc_config": {
      "docid_prefix_delim": "",
      "docid_regexp": "",
      "mode": "scope.collection.type_field",
      "type_field": "type"
    },
    "mapping": {
      "analysis": {
        "char_filters": {
          "html_strip_filter": {
            "type": "html"
          }
        },
        "token_filters": {
          "spanish_stop_filter": {
            "type": "stop_tokens",
            "stop_token_map": "spanish_stops"
          },
          "spanish_stemmer_filter": {
            "type": "stemmer",
            "language": "es"
          }
        },
        "token_maps": {
          "spanish_stops": {
            "type": "custom",
            "tokens": [
              "de", "la", "el", "en", "con", "por", "para",
              "un", "una", "los", "las", "del", "al", "se",
              "su", "es", "que", "y", "a", "o", "no", "si",
              "lo", "le", "les", "me", "mi", "tu", "te",
              "nos", "hay", "ser", "fue", "son", "era"
            ]
          }
        },
        "analyzers": {
          "spanish_hotel_analyzer": {
            "type": "custom",
            "char_filters": ["html_strip_filter"],
            "tokenizer": "unicode",
            "token_filters": [
              "to_lower",
              "ascii_folding",
              "spanish_stop_filter",
              "spanish_stemmer_filter"
            ]
          },
          "sku_analyzer": {
            "type": "custom",
            "char_filters": [],
            "tokenizer": "keyword",
            "token_filters": [
              "to_upper"
            ]
          }
        }
      },
      "default_analyzer": "standard",
      "default_datetime_parser": "dateTimeOptional",
      "default_field": "_all",
      "default_mapping": {
        "dynamic": false,
        "enabled": false
      },
      "default_type": "_default",
      "docvalues_dynamic": false,
      "index_dynamic": false,
      "store_dynamic": false,
      "type_field": "_type",
      "types": {
        "inventory.hotel": {
          "dynamic": false,
          "enabled": true,
          "properties": {
            "name": {
              "enabled": true,
              "dynamic": false,
              "fields": [
                {
                  "name": "name",
                  "type": "text",
                  "analyzer": "spanish_hotel_analyzer",
                  "store": true,
                  "index": true,
                  "include_term_vectors": true,
                  "include_in_all": true,
                  "docvalues": false
                }
              ]
            },
            "description": {
              "enabled": true,
              "dynamic": false,
              "fields": [
                {
                  "name": "description",
                  "type": "text",
                  "analyzer": "spanish_hotel_analyzer",
                  "store": true,
                  "index": true,
                  "include_term_vectors": true,
                  "include_in_all": true,
                  "docvalues": false
                }
              ]
            },
            "sku": {
              "enabled": true,
              "dynamic": false,
              "fields": [
                {
                  "name": "sku",
                  "type": "text",
                  "analyzer": "sku_analyzer",
                  "store": true,
                  "index": true,
                  "include_term_vectors": false,
                  "include_in_all": false,
                  "docvalues": false
                }
              ]
            }
          }
        }
      }
    },
    "store": {
      "indexType": "scorch",
      "segmentVersion": 15
    }
  },
  "sourceType": "gocbcore",
  "sourceName": "travel-sample",
  "sourceParams": {},
  "planParams": {
    "maxPartitionsPerPIndex": 1024,
    "indexPartitions": 1,
    "numReplicas": 0
  }
}
```

**3.4** Haz clic en **Create Index** (o **Save**). Observa que el índice aparece en la lista con estado de construcción.

**3.5** Espera a que el índice termine de construirse. Puedes monitorear el progreso con:

```bash
# Monitorear el estado del índice
curl -s -u Administrator:password \
  "http://localhost:8094/api/index/hotels_custom_analyzers" \
  | python3 -m json.tool | grep -E '"status"|"docCount"'
```

#### Salida esperada

```json
"status": "Ready",
"docCount": 917
```

> **Nota:** El número exacto de documentos puede variar. El estado `Ready` indica que el índice está listo para consultas.

#### Verificación

- [ ] El índice `hotels_custom_analyzers` aparece en la lista de índices FTS
- [ ] El estado del índice es `Ready` (o equivalente)
- [ ] El conteo de documentos es mayor que 0
- [ ] No hay errores en la consola del índice

---

### Paso 4: Crear el índice vía REST API (método alternativo)

**Objetivo:** Registrar el mismo índice usando la API REST de Couchbase, demostrando el método programático para automatización y CI/CD.

> **Nota:** Este paso es **alternativo** al Paso 3. Si ya creaste el índice en el Paso 3, puedes saltar al Paso 5. Si deseas practicar el método REST, primero elimina el índice creado en el Paso 3.

#### Instrucciones

**4.1** Elimina el índice anterior si existe:

```bash
curl -s -u Administrator:password \
  -X DELETE \
  "http://localhost:8094/api/index/hotels_custom_analyzers"
```

**4.2** Guarda la definición completa del índice en un archivo JSON:

```bash
cat > /tmp/hotels_custom_analyzers_index.json << 'INDEXEOF'
{
  "name": "hotels_custom_analyzers",
  "type": "fulltext-index",
  "params": {
    "doc_config": {
      "docid_prefix_delim": "",
      "docid_regexp": "",
      "mode": "scope.collection.type_field",
      "type_field": "type"
    },
    "mapping": {
      "analysis": {
        "char_filters": {
          "html_strip_filter": {
            "type": "html"
          }
        },
        "token_filters": {
          "spanish_stop_filter": {
            "type": "stop_tokens",
            "stop_token_map": "spanish_stops"
          },
          "spanish_stemmer_filter": {
            "type": "stemmer",
            "language": "es"
          }
        },
        "token_maps": {
          "spanish_stops": {
            "type": "custom",
            "tokens": [
              "de", "la", "el", "en", "con", "por", "para",
              "un", "una", "los", "las", "del", "al", "se",
              "su", "es", "que", "y", "a", "o", "no", "si",
              "lo", "le", "les", "me", "mi", "tu", "te",
              "nos", "hay", "ser", "fue", "son", "era"
            ]
          }
        },
        "analyzers": {
          "spanish_hotel_analyzer": {
            "type": "custom",
            "char_filters": ["html_strip_filter"],
            "tokenizer": "unicode",
            "token_filters": [
              "to_lower",
              "ascii_folding",
              "spanish_stop_filter",
              "spanish_stemmer_filter"
            ]
          },
          "sku_analyzer": {
            "type": "custom",
            "char_filters": [],
            "tokenizer": "keyword",
            "token_filters": [
              "to_upper"
            ]
          }
        }
      },
      "default_analyzer": "standard",
      "default_datetime_parser": "dateTimeOptional",
      "default_field": "_all",
      "default_mapping": {
        "dynamic": false,
        "enabled": false
      },
      "default_type": "_default",
      "docvalues_dynamic": false,
      "index_dynamic": false,
      "store_dynamic": false,
      "type_field": "_type",
      "types": {
        "inventory.hotel": {
          "dynamic": false,
          "enabled": true,
          "properties": {
            "name": {
              "enabled": true,
              "dynamic": false,
              "fields": [
                {
                  "name": "name",
                  "type": "text",
                  "analyzer": "spanish_hotel_analyzer",
                  "store": true,
                  "index": true,
                  "include_term_vectors": true,
                  "include_in_all": true,
                  "docvalues": false
                }
              ]
            },
            "description": {
              "enabled": true,
              "dynamic": false,
              "fields": [
                {
                  "name": "description",
                  "type": "text",
                  "analyzer": "spanish_hotel_analyzer",
                  "store": true,
                  "index": true,
                  "include_term_vectors": true,
                  "include_in_all": true,
                  "docvalues": false
                }
              ]
            }
          }
        }
      }
    },
    "store": {
      "indexType": "scorch",
      "segmentVersion": 15
    }
  },
  "sourceType": "gocbcore",
  "sourceName": "travel-sample",
  "sourceParams": {},
  "planParams": {
    "maxPartitionsPerPIndex": 1024,
    "indexPartitions": 1,
    "numReplicas": 0
  }
}
INDEXEOF
```

**4.3** Crea el índice via REST API:

```bash
curl -s -u Administrator:password \
  -X PUT \
  "http://localhost:8094/api/index/hotels_custom_analyzers" \
  -H "Content-Type: application/json" \
  -d @/tmp/hotels_custom_analyzers_index.json \
  | python3 -m json.tool
```

#### Salida esperada

```json
{
  "status": "ok"
}
```

#### Verificación

- [ ] La respuesta de la API es `"status": "ok"`
- [ ] El índice aparece en `http://localhost:8094/api/index`

---

### Paso 5: Validar los analyzers con el endpoint `analyzeDoc`

**Objetivo:** Usar el endpoint REST `analyzeDoc` para confirmar que los analyzers personalizados transforman el texto exactamente como se diseñó en el Paso 2.

#### Instrucciones

**5.1** Valida el `spanish_hotel_analyzer` con texto que incluye HTML, acentos y palabras vacías:

```bash
# Test 1: Texto con HTML, acentos y stop words
curl -s -u Administrator:password \
  -X POST \
  "http://localhost:8094/api/index/hotels_custom_analyzers/analyzeDoc" \
  -H "Content-Type: application/json" \
  -d '{
    "analyzer": "spanish_hotel_analyzer",
    "text": "<p>Habitaciones con vista al mar, café incluido en el precio</p>"
  }' | python3 -m json.tool
```

**5.2** Valida el efecto del ASCII folding comparando con y sin acento:

```bash
# Test 2: Comparar "habitación" vs "habitacion"
# Primero con acento
curl -s -u Administrator:password \
  -X POST \
  "http://localhost:8094/api/index/hotels_custom_analyzers/analyzeDoc" \
  -H "Content-Type: application/json" \
  -d '{
    "analyzer": "spanish_hotel_analyzer",
    "text": "habitación"
  }' | python3 -m json.tool

# Luego sin acento
curl -s -u Administrator:password \
  -X POST \
  "http://localhost:8094/api/index/hotels_custom_analyzers/analyzeDoc" \
  -H "Content-Type: application/json" \
  -d '{
    "analyzer": "spanish_hotel_analyzer",
    "text": "habitacion"
  }' | python3 -m json.tool
```

**5.3** Verifica el efecto del stemming en español:

```bash
# Test 3: Stemming - distintas formas de la misma palabra
curl -s -u Administrator:password \
  -X POST \
  "http://localhost:8094/api/index/hotels_custom_analyzers/analyzeDoc" \
  -H "Content-Type: application/json" \
  -d '{
    "analyzer": "spanish_hotel_analyzer",
    "text": "habitaciones habitacion habitando habitado"
  }' | python3 -m json.tool
```

**5.4** Valida el `sku_analyzer` con un código de producto:

```bash
# Test 4: sku_analyzer - tokenizer keyword + uppercase
curl -s -u Administrator:password \
  -X POST \
  "http://localhost:8094/api/index/hotels_custom_analyzers/analyzeDoc" \
  -H "Content-Type: application/json" \
  -d '{
    "analyzer": "sku_analyzer",
    "text": "htl-madrid-001"
  }' | python3 -m json.tool
```

#### Salida esperada

**Test 1 (HTML + acentos + stop words):**
```json
{
  "status": "ok",
  "analyzed_text": [
    {"term": "habitacion", "position": 1},
    {"term": "vist", "position": 2},
    {"term": "mar", "position": 3},
    {"term": "cafe", "position": 4},
    {"term": "incluid", "position": 5},
    {"term": "preci", "position": 6}
  ]
}
```

> **Observa:** Las etiquetas `<p>` fueron eliminadas por el character filter `html`. Las stop words `con`, `al`, `en`, `el` fueron eliminadas. Los acentos fueron normalizados (`habitación → habitacion`, `café → cafe`). El stemmer redujo las palabras a sus raíces.

**Test 2 (ASCII folding):**
Ambas consultas (`habitación` y `habitacion`) deben producir **el mismo token raíz**, demostrando que el ASCII folding normaliza antes del stemming.

**Test 4 (sku_analyzer):**
```json
{
  "status": "ok",
  "analyzed_text": [
    {"term": "HTL-MADRID-001", "position": 1}
  ]
}
```

> **Observa:** El tokenizer `keyword` trata todo el texto como un único token, y el filter `to_upper` lo convierte a mayúsculas. Esto garantiza búsqueda exacta case-insensitive para códigos de producto.

#### Verificación

- [ ] El Test 1 muestra que las etiquetas HTML fueron eliminadas
- [ ] Las stop words en español no aparecen en los tokens resultantes
- [ ] Los tokens `habitación` y `habitacion` producen la misma raíz (Test 2)
- [ ] El `sku_analyzer` produce un único token en mayúsculas (Test 4)

---

### Paso 6: Ejecutar búsquedas comparativas

**Objetivo:** Demostrar la diferencia práctica entre el analyzer estándar y el `spanish_hotel_analyzer` ejecutando búsquedas reales sobre datos de hoteles.

#### Instrucciones

**6.1** Primero, crea un índice de referencia con el analyzer `standard` para comparación:

```bash
curl -s -u Administrator:password \
  -X PUT \
  "http://localhost:8094/api/index/hotels_standard_ref" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "hotels_standard_ref",
    "type": "fulltext-index",
    "params": {
      "doc_config": {
        "mode": "scope.collection.type_field",
        "type_field": "type"
      },
      "mapping": {
        "default_analyzer": "standard",
        "default_mapping": {"dynamic": false, "enabled": false},
        "types": {
          "inventory.hotel": {
            "dynamic": false,
            "enabled": true,
            "properties": {
              "name": {
                "enabled": true,
                "fields": [{"name": "name", "type": "text", "analyzer": "standard", "store": true, "index": true}]
              },
              "description": {
                "enabled": true,
                "fields": [{"name": "description", "type": "text", "analyzer": "standard", "store": true, "index": true}]
              }
            }
          }
        }
      },
      "store": {"indexType": "scorch", "segmentVersion": 15}
    },
    "sourceType": "gocbcore",
    "sourceName": "travel-sample",
    "planParams": {"maxPartitionsPerPIndex": 1024, "indexPartitions": 1, "numReplicas": 0}
  }' | python3 -m json.tool
```

**6.2** Espera a que ambos índices estén listos (30–60 segundos) y luego ejecuta la búsqueda comparativa. Busca `"habitacion"` (sin acento, sin plural) en el índice estándar:

```bash
# Búsqueda con analyzer standard: "habitacion" (sin acento)
curl -s -u Administrator:password \
  -X POST \
  "http://localhost:8094/api/index/hotels_standard_ref/query" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "query": "habitacion",
      "field": "description"
    },
    "size": 5,
    "fields": ["name", "description"]
  }' | python3 -m json.tool | grep -E '"name"|"total_hits"'
```

**6.3** Ejecuta la misma búsqueda en el índice con el analyzer personalizado:

```bash
# Búsqueda con spanish_hotel_analyzer: "habitacion" (sin acento)
curl -s -u Administrator:password \
  -X POST \
  "http://localhost:8094/api/index/hotels_custom_analyzers/query" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "query": "habitacion",
      "field": "description"
    },
    "size": 5,
    "fields": ["name", "description"]
  }' | python3 -m json.tool | grep -E '"name"|"total_hits"'
```

**6.4** Ejecuta una búsqueda con la forma plural y con acento para confirmar que el stemming funciona:

```bash
# Búsqueda con "habitaciones" (plural con acento)
# En índice custom - debería encontrar documentos con "habitación", "habitaciones", "habitacion"
curl -s -u Administrator:password \
  -X POST \
  "http://localhost:8094/api/index/hotels_custom_analyzers/query" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "query": "habitaciones",
      "field": "description"
    },
    "size": 10,
    "fields": ["name"]
  }' | python3 -m json.tool
```

**6.5** Prueba una búsqueda multi-término que aprovecha el stop words filter:

```bash
# Búsqueda multi-término: las stop words "en" y "con" serán ignoradas
curl -s -u Administrator:password \
  -X POST \
  "http://localhost:8094/api/index/hotels_custom_analyzers/query" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "query": "piscina con vista al mar",
      "field": "description"
    },
    "size": 5,
    "fields": ["name", "description"]
  }' | python3 -m json.tool
```

#### Salida esperada

La comparación debe mostrar una diferencia notable en `total_hits`:

| Búsqueda | Índice Standard | Índice Custom |
|---|---|---|
| `"habitacion"` (sin acento) | 0–5 resultados | 15–30 resultados |
| `"habitaciones"` (plural) | Pocos resultados | Más resultados (stemming) |

> **Interpretación:** El analyzer personalizado normaliza acentos y aplica stemming, por lo que `"habitacion"`, `"habitación"` y `"habitaciones"` convergen al mismo token raíz en el índice. El buscador estándar trata cada variante como un término completamente diferente.

#### Verificación

- [ ] El índice `hotels_standard_ref` devuelve menos resultados para `"habitacion"` sin acento
- [ ] El índice `hotels_custom_analyzers` devuelve más resultados para la misma búsqueda
- [ ] La búsqueda de `"habitaciones"` en el índice custom encuentra documentos con variantes de la palabra
- [ ] La búsqueda multi-término ignora las stop words correctamente

---

### Paso 7: Inspeccionar la definición del índice y los analyzers

**Objetivo:** Verificar que los analyzers personalizados están correctamente registrados en la definición del índice y aprender a inspeccionarlos programáticamente.

#### Instrucciones

**7.1** Recupera y examina la definición completa del índice para verificar los analyzers:

```bash
# Recuperar definición completa del índice
curl -s -u Administrator:password \
  "http://localhost:8094/api/index/hotels_custom_analyzers" \
  | python3 -m json.tool > /tmp/index_definition_retrieved.json

# Extraer solo la sección de analyzers
curl -s -u Administrator:password \
  "http://localhost:8094/api/index/hotels_custom_analyzers" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
analysis = data['indexDef']['params']['mapping'].get('analysis', {})
print(json.dumps(analysis, indent=2))
"
```

**7.2** Verifica los mappings de campos para confirmar que usan el analyzer correcto:

```bash
# Verificar mappings de campos
curl -s -u Administrator:password \
  "http://localhost:8094/api/index/hotels_custom_analyzers" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
types = data['indexDef']['params']['mapping'].get('types', {})
for type_name, type_def in types.items():
    print(f'Type: {type_name}')
    props = type_def.get('properties', {})
    for field_name, field_def in props.items():
        for f in field_def.get('fields', []):
            print(f'  Field: {field_name} -> analyzer: {f.get(\"analyzer\", \"(default)\")}')
"
```

**7.3** Consulta las estadísticas del índice para verificar que los documentos fueron indexados:

```bash
# Estadísticas del índice
curl -s -u Administrator:password \
  "http://localhost:8094/api/index/hotels_custom_analyzers/stats" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
stats = data.get('stats', {})
print(f'Documentos indexados: {stats.get(\"doc_count\", \"N/A\")}')
print(f'Tamaño del índice: {stats.get(\"num_bytes_used_disk\", \"N/A\")} bytes')
"
```

#### Salida esperada

```
Type: inventory.hotel
  Field: name -> analyzer: spanish_hotel_analyzer
  Field: description -> analyzer: spanish_hotel_analyzer
  Field: sku -> analyzer: sku_analyzer

Documentos indexados: 917
Tamaño del índice: XXXXX bytes
```

#### Verificación

- [ ] La sección `analysis` del índice contiene los analyzers `spanish_hotel_analyzer` y `sku_analyzer`
- [ ] Los campos `name` y `description` tienen asignado `spanish_hotel_analyzer`
- [ ] El campo `sku` tiene asignado `sku_analyzer`
- [ ] El conteo de documentos es mayor que 0

---

## Validación y Pruebas

### Prueba de validación integral

Ejecuta la siguiente secuencia de pruebas para confirmar que todo el laboratorio funciona correctamente:

```bash
#!/bin/bash
# Script de validación completa del laboratorio

echo "=== VALIDACIÓN LAB 13-00-01 ==="
echo ""

# Test 1: Verificar que el índice existe y está listo
echo "--- Test 1: Estado del índice ---"
STATUS=$(curl -s -u Administrator:password \
  "http://localhost:8094/api/index/hotels_custom_analyzers" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('status','ERROR'))")
echo "Estado: $STATUS"
echo ""

# Test 2: Verificar que el analyzer elimina HTML
echo "--- Test 2: Character filter HTML ---"
TOKENS=$(curl -s -u Administrator:password \
  -X POST "http://localhost:8094/api/index/hotels_custom_analyzers/analyzeDoc" \
  -H "Content-Type: application/json" \
  -d '{"analyzer": "spanish_hotel_analyzer", "text": "<b>Hotel</b>"}' \
  | python3 -c "
import json,sys
d=json.load(sys.stdin)
terms=[t['term'] for t in d.get('analyzed_text',[])]
print(terms)
")
echo "Tokens de '<b>Hotel</b>': $TOKENS"
echo "Esperado: ['hotel'] (sin etiquetas HTML)"
echo ""

# Test 3: Verificar ASCII folding
echo "--- Test 3: ASCII folding (acento) ---"
TOKEN_CON=$(curl -s -u Administrator:password \
  -X POST "http://localhost:8094/api/index/hotels_custom_analyzers/analyzeDoc" \
  -H "Content-Type: application/json" \
  -d '{"analyzer": "spanish_hotel_analyzer", "text": "café"}' \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print([t['term'] for t in d.get('analyzed_text',[])])")
echo "Tokens de 'café': $TOKEN_CON"
echo "Esperado: ['cafe'] (sin acento)"
echo ""

# Test 4: Verificar stop words
echo "--- Test 4: Stop words en español ---"
TOKENS_STOP=$(curl -s -u Administrator:password \
  -X POST "http://localhost:8094/api/index/hotels_custom_analyzers/analyzeDoc" \
  -H "Content-Type: application/json" \
  -d '{"analyzer": "spanish_hotel_analyzer", "text": "de la vista al mar"}' \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print([t['term'] for t in d.get('analyzed_text',[])])")
echo "Tokens de 'de la vista al mar': $TOKENS_STOP"
echo "Esperado: ['vist', 'mar'] (sin stop words de, la, al)"
echo ""

# Test 5: Verificar sku_analyzer
echo "--- Test 5: SKU analyzer (keyword + uppercase) ---"
TOKEN_SKU=$(curl -s -u Administrator:password \
  -X POST "http://localhost:8094/api/index/hotels_custom_analyzers/analyzeDoc" \
  -H "Content-Type: application/json" \
  -d '{"analyzer": "sku_analyzer", "text": "htl-bcn-042"}' \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print([t['term'] for t in d.get('analyzed_text',[])])")
echo "Tokens de 'htl-bcn-042': $TOKEN_SKU"
echo "Esperado: ['HTL-BCN-042'] (un solo token en mayúsculas)"
echo ""

echo "=== FIN DE VALIDACIÓN ==="
```

Guarda el script como `/tmp/validate_lab13.sh`, dale permisos de ejecución y ejecútalo:

```bash
chmod +x /tmp/validate_lab13.sh
bash /tmp/validate_lab13.sh
```

### Criterios de éxito

| Prueba | Criterio de éxito |
|---|---|
| Estado del índice | Responde `ok` o `Ready` |
| Character filter HTML | `<b>Hotel</b>` produce token `hotel` (sin etiquetas) |
| ASCII folding | `café` produce token `cafe` |
| Stop words | `de la vista al mar` produce solo `['vist', 'mar']` |
| SKU analyzer | `htl-bcn-042` produce `['HTL-BCN-042']` como único token |
| Búsqueda comparativa | El índice custom devuelve más resultados que el estándar para términos con acento |

---

## Resolución de Problemas

### Problema 1: El endpoint `analyzeDoc` devuelve error 404 o "index not found"

**Síntoma:** Al ejecutar el comando `curl` con el endpoint `analyzeDoc`, la respuesta es:
```json
{"status": "fail", "error": "index not found"}
```
o un código HTTP 404.

**Causa:** El índice FTS especificado en la URL no existe todavía, o el nombre en la URL no coincide exactamente con el nombre del índice creado (incluyendo mayúsculas/minúsculas). También puede ocurrir si el servicio Search no está habilitado en el nodo.

**Solución:**

```bash
# 1. Listar todos los índices FTS existentes
curl -s -u Administrator:password \
  "http://localhost:8094/api/index" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
indexes = data.get('indexDefs', {}).get('indexDefs', {})
print('Índices disponibles:')
for name in indexes.keys():
    print(f'  - {name}')
"

# 2. Verificar que el servicio Search está habilitado
curl -s -u Administrator:password \
  "http://localhost:8091/pools/default/nodeServices" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
for node in data.get('nodesExt', []):
    services = node.get('services', {})
    fts_port = services.get('fts', 'NO DISPONIBLE')
    print(f'FTS port: {fts_port}')
"

# 3. Si el índice no existe, crearlo nuevamente usando el archivo guardado
curl -s -u Administrator:password \
  -X PUT \
  "http://localhost:8094/api/index/hotels_custom_analyzers" \
  -H "Content-Type: application/json" \
  -d @/tmp/hotels_custom_analyzers_index.json
```

Si el servicio Search no está habilitado, ve a **Settings → Services** en la Web Console y activa el servicio **Search**, luego reinicia el nodo si es necesario.

---

### Problema 2: El analyzer personalizado no elimina stop words o no aplica stemming correctamente

**Síntoma:** Al usar `analyzeDoc` con el `spanish_hotel_analyzer`, las stop words en español (`de`, `la`, `el`) siguen apareciendo en los tokens, o las palabras no se reducen a su raíz (por ejemplo, `habitaciones` no produce el mismo token que `habitacion`).

**Causa:** Hay dos causas comunes: (a) el `token_map` `spanish_stops` no fue definido correctamente en la sección `analysis.token_maps` del índice, o (b) el orden de los token filters es incorrecto (el stemmer debe aplicarse **después** del ASCII folding y el lowercase para trabajar sobre texto ya normalizado).

**Solución:**

```bash
# 1. Verificar la definición actual del analyzer en el índice
curl -s -u Administrator:password \
  "http://localhost:8094/api/index/hotels_custom_analyzers" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
analysis = data['indexDef']['params']['mapping'].get('analysis', {})

# Verificar token_maps
token_maps = analysis.get('token_maps', {})
print('Token maps definidos:', list(token_maps.keys()))
if 'spanish_stops' in token_maps:
    tokens = token_maps['spanish_stops'].get('tokens', [])
    print(f'Stop words registradas: {len(tokens)} palabras')
    print(f'Primeras 5: {tokens[:5]}')
else:
    print('ERROR: token_map spanish_stops NO ENCONTRADO')

# Verificar orden de token filters en el analyzer
analyzers = analysis.get('analyzers', {})
if 'spanish_hotel_analyzer' in analyzers:
    filters = analyzers['spanish_hotel_analyzer'].get('token_filters', [])
    print(f'Orden de token filters: {filters}')
    print('Orden correcto esperado: [to_lower, ascii_folding, spanish_stop_filter, spanish_stemmer_filter]')
"

# 2. Si el orden es incorrecto, actualizar el índice con la definición corregida
# Editar /tmp/hotels_custom_analyzers_index.json para corregir el orden
# y luego:
curl -s -u Administrator:password \
  -X PUT \
  "http://localhost:8094/api/index/hotels_custom_analyzers" \
  -H "Content-Type: application/json" \
  -d @/tmp/hotels_custom_analyzers_index.json

# 3. Después de actualizar, esperar a que el índice se reconstruya
sleep 30

# 4. Probar nuevamente el analyzer
curl -s -u Administrator:password \
  -X POST \
  "http://localhost:8094/api/index/hotels_custom_analyzers/analyzeDoc" \
  -H "Content-Type: application/json" \
  -d '{
    "analyzer": "spanish_hotel_analyzer",
    "text": "de la habitación con vista"
  }' | python3 -m json.tool
```

> **Regla de oro:** El orden correcto de token filters siempre debe ser: `to_lower` → `ascii_folding` → `stop_tokens` → `stemmer`. Si el stemmer va antes del lowercase, puede no reconocer correctamente las raíces de palabras en mayúsculas.

---

## Limpieza del Entorno

Una vez completado el laboratorio, elimina los índices creados para liberar recursos:

```bash
# Eliminar índice principal del laboratorio
curl -s -u Administrator:password \
  -X DELETE \
  "http://localhost:8094/api/index/hotels_custom_analyzers" \
  | python3 -m json.tool

# Eliminar índice de referencia estándar
curl -s -u Administrator:password \
  -X DELETE \
  "http://localhost:8094/api/index/hotels_standard_ref" \
  | python3 -m json.tool

# Eliminar índice temporal del Paso 1
curl -s -u Administrator:password \
  -X DELETE \
  "http://localhost:8094/api/index/hotels_analyzer_lab" \
  | python3 -m json.tool

# Limpiar archivos temporales
rm -f /tmp/spanish_stopwords.txt \
      /tmp/spanish_hotel_analyzer.json \
      /tmp/hotels_custom_analyzers_index.json \
      /tmp/index_definition_retrieved.json \
      /tmp/validate_lab13.sh

echo "Limpieza completada."
```

**Verificación de limpieza:**

```bash
# Confirmar que los índices fueron eliminados
curl -s -u Administrator:password \
  "http://localhost:8094/api/index" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
indexes = data.get('indexDefs', {}).get('indexDefs', {})
lab_indexes = [n for n in indexes.keys() if 'hotel' in n]
if lab_indexes:
    print(f'ATENCIÓN: Aún existen índices del lab: {lab_indexes}')
else:
    print('OK: Todos los índices del laboratorio fueron eliminados.')
"
```

---

## Resumen

### Conceptos clave aprendidos

En este laboratorio has construido analyzers personalizados de FTS desde cero, aplicando los siguientes conceptos del pipeline de análisis:

| Componente | Función | Ejemplo aplicado |
|---|---|---|
| **Character Filter `html`** | Elimina etiquetas HTML antes de tokenizar | `<p>Hotel</p>` → `Hotel` |
| **Tokenizer `unicode`** | Divide texto respetando límites Unicode (acentos, UTF-8) | `café-hotel` → `["café", "hotel"]` |
| **Tokenizer `keyword`** | Trata el campo completo como un único token | `htl-001` → `["htl-001"]` |
| **Token Filter `to_lower`** | Normaliza mayúsculas | `HOTEL` → `hotel` |
| **Token Filter `ascii_folding`** | Elimina diacríticos | `café` → `cafe` |
| **Token Filter `stop_tokens`** | Elimina palabras vacías del idioma | `de la vista` → `vista` |
| **Token Filter `stemmer` (es)** | Reduce palabras a su raíz morfológica | `habitaciones` → raíz común con `habitacion` |

### Lecciones aprendidas

1. **El orden importa:** Los token filters se aplican en secuencia. `lowercase` debe preceder a `stop_tokens` y `stemmer` para garantizar coincidencias correctas.

2. **Coherencia indexación-consulta:** El mismo analyzer debe usarse tanto al indexar como al consultar. Couchbase aplica automáticamente el analyzer del campo al procesar la consulta si se especifica el campo en la query.

3. **`analyzeDoc` es tu mejor aliado:** Antes de crear un índice en producción, siempre valida el comportamiento del analyzer con el endpoint `analyzeDoc`. Esto evita reindexaciones costosas.

4. **Especialización por campo:** No todos los campos necesitan el mismo analyzer. Los campos de texto libre en lenguaje natural se benefician de `spanish_hotel_analyzer`, mientras que los identificadores exactos requieren `sku_analyzer` con tokenizer `keyword`.

5. **ASCII folding y stemming son complementarios:** El ASCII folding normaliza variantes tipográficas (`café/cafe`), mientras que el stemming normaliza variantes morfológicas (`habitaciones/habitacion`). Juntos maximizan el recall de la búsqueda.

### Recursos adicionales

- [Documentación oficial: Custom Analyzers en Couchbase FTS](https://docs.couchbase.com/server/current/fts/fts-analyzers.html)
- [Referencia de Tokenizers disponibles](https://docs.couchbase.com/server/current/fts/fts-tokenizers.html)
- [Referencia de Token Filters disponibles](https://docs.couchbase.com/server/current/fts/fts-token-filters.html)
- [API REST de Couchbase Search](https://docs.couchbase.com/server/current/rest-api/rest-fts.html)
- [Stemmers disponibles por idioma (Snowball)](https://snowballstem.org/algorithms/)

---
