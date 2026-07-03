# Instalación y exploración inicial de Couchbase Server

## Metadatos

| Campo            | Valor                                      |
|------------------|--------------------------------------------|
| **Duración**     | 50 minutos                                 |
| **Complejidad**  | Media                                      |
| **Nivel Bloom**  | Aplicar (*Apply*)                          |
| **Laboratorio**  | 02-00-01                                   |
| **Dependencias** | Lab 01-00-01 (o conocimientos previos NoSQL) |

---

## Descripción General

En este laboratorio instalarás y configurarás Couchbase Server 7.6.x en un nodo único, habilitando los servicios Data, Index, Query, Search y Eventing. Cargarás el dataset de muestra `travel-sample`, explorarás su estructura de documentos JSON y realizarás un recorrido guiado por la Web Console. Finalmente, ejecutarás tu primera consulta SQL++ y consultarás la REST API de administración con `curl`, validando el funcionamiento integral del clúster.

Este laboratorio conecta directamente con la historia de Couchbase estudiada en la lección 2.1: verás en acción la arquitectura de *vBuckets* heredada de Membase, el lenguaje SQL++ introducido en la versión 4.0, y las colecciones/*scopes* consolidadas en la versión 7.0.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:

- [ ] Instalar y configurar Couchbase Server 7.6.x en un nodo único con los servicios Data, Index, Query, Search y Eventing habilitados.
- [ ] Navegar la Web Console para explorar buckets, scopes, collections y el dashboard de métricas del servidor.
- [ ] Cargar el dataset `travel-sample` e identificar la estructura de documentos JSON en el Document Editor.
- [ ] Identificar los componentes arquitectónicos del nodo (vBuckets, Managed Cache, Storage Engine) y su función.
- [ ] Ejecutar una consulta SQL++ básica desde el Query Editor para validar el servicio Query.

---

## Prerrequisitos

### Conocimientos previos
- Conceptos básicos de bases de datos NoSQL (Lab 01-00-01 o equivalente).
- Familiaridad con la línea de comandos (terminal/PowerShell).
- Conocimiento básico de SQL (SELECT, FROM, WHERE).
- Haber leído la lección 2.1 sobre la historia de Couchbase Server.

### Acceso y software requerido
- Docker Desktop 4.x o superior **O** permisos de administrador en el sistema operativo.
- Puertos **8091–8097** y **11210** disponibles y no bloqueados por firewall local.
- Mínimo **4 GB de RAM** disponibles para el entorno de laboratorio.
- `curl` 7.x o superior instalado y accesible desde la terminal.
- Navegador web moderno: Chrome 110+, Firefox 110+ o Edge 110+.

---

## Entorno del Laboratorio

### Especificaciones de Hardware

| Recurso        | Mínimo              | Recomendado         |
|----------------|---------------------|---------------------|
| CPU            | 4 núcleos x86_64    | 8 núcleos           |
| RAM            | 4 GB disponibles    | 8 GB disponibles    |
| Almacenamiento | 20 GB libres (HDD)  | 20 GB libres (SSD)  |
| Red            | Acceso a localhost  | Acceso a localhost  |

### Especificaciones de Software

| Componente            | Versión requerida          |
|-----------------------|----------------------------|
| Couchbase Server      | 7.6.x (CE o Enterprise Trial) |
| Docker Desktop        | 4.x+ (si se usa Docker)    |
| Navegador Web         | Chrome/Firefox/Edge 110+   |
| curl                  | 7.x o superior             |

### Opción A — Instalación con Docker (recomendada para el laboratorio)

Verifica que Docker esté en ejecución antes de continuar:

```bash
docker --version
docker info | grep "Server Version"
```

### Opción B — Instalación nativa

Descarga el instalador desde: [https://www.couchbase.com/downloads/](https://www.couchbase.com/downloads/)  
Selecciona: **Couchbase Server → 7.6.x → Community Edition** (o Enterprise Trial).

---

## Pasos del Laboratorio

---

### Paso 1: Desplegar Couchbase Server

**Objetivo:** Tener una instancia de Couchbase Server 7.6.x en ejecución y accesible en `http://localhost:8091`.

#### Opción A — Docker (recomendada)

**Instrucciones:**

1. Abre una terminal y ejecuta el siguiente comando para descargar e iniciar el contenedor:

```bash
docker run -d \
  --name couchbase-lab \
  -p 8091-8097:8091-8097 \
  -p 11210:11210 \
  -p 18091-18097:18091-18097 \
  couchbase/server:7.6.2
```

2. Espera aproximadamente 30 segundos a que el contenedor inicialice. Verifica que esté corriendo:

```bash
docker ps --filter "name=couchbase-lab" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

3. Comprueba que el servicio REST responde:

```bash
curl -s http://localhost:8091/ui/index.html -o /dev/null -w "HTTP Status: %{http_code}\n"
```

**Salida esperada:**
```
HTTP Status: 200
```

#### Opción B — Instalación Nativa (Linux/macOS/Windows)

**Instrucciones:**

1. **Linux (Debian/Ubuntu):**
```bash
# Descargar el paquete (ajusta la versión si es necesario)
wget https://packages.couchbase.com/releases/7.6.2/couchbase-server-community_7.6.2-linux_amd64.deb

# Instalar
sudo dpkg -i couchbase-server-community_7.6.2-linux_amd64.deb

# Iniciar el servicio
sudo systemctl start couchbase-server
sudo systemctl enable couchbase-server
```

2. **macOS:** Ejecuta el instalador `.dmg` descargado y arrastra Couchbase Server a Aplicaciones. Ábrelo desde el Launchpad.

3. **Windows:** Ejecuta el instalador `.msi` con doble clic y sigue el asistente. El servicio se inicia automáticamente.

**Verificación (todas las plataformas):**

Abre el navegador y navega a `http://localhost:8091`. Deberías ver la pantalla de bienvenida de Couchbase Server.

> 📌 **Nota histórica:** El puerto 8091 es el puerto de administración REST que Couchbase heredó de Membase. La arquitectura de puertos múltiples (8091–8097) refleja la separación de servicios introducida con Multi-Dimensional Scaling (MDS) en la versión 4.0.

---

### Paso 2: Configurar el Clúster de Nodo Único

**Objetivo:** Completar el asistente de configuración inicial de Couchbase para crear un clúster de un solo nodo con todos los servicios necesarios habilitados.

**Instrucciones:**

1. Abre `http://localhost:8091` en el navegador. Haz clic en **"Setup New Cluster"**.

2. En la pantalla **"New Cluster"**, ingresa:
   - **Cluster Name:** `lab-cluster`
   - **Admin Username:** `Administrator`
   - **Password:** `Password123!`
   - **Confirm Password:** `Password123!`

   Haz clic en **"Next: Accept Terms"**.

3. Acepta los términos de uso y haz clic en **"Finish With Defaults"**.

   > ⚠️ **Importante:** NO uses "Finish With Defaults" si quieres controlar la asignación de memoria. En su lugar, haz clic en **"Configure Disk, Memory, Services"** para el paso siguiente.

4. Si seleccionaste **"Configure Disk, Memory, Services"**, configura lo siguiente:

   **Servicios a habilitar (marca todos):**
   - ✅ Data
   - ✅ Index
   - ✅ Query
   - ✅ Search
   - ✅ Eventing
   - ☐ Analytics *(opcional, requiere más RAM)*

   **Asignación de memoria (ajusta según tu RAM disponible):**

   | Servicio | RAM mínima | RAM recomendada (8 GB sistema) |
   |----------|-----------|-------------------------------|
   | Data     | 512 MB    | 1024 MB                        |
   | Index    | 256 MB    | 512 MB                         |
   | Search   | 256 MB    | 512 MB                         |
   | Eventing | 256 MB    | 256 MB                         |

   > 💡 **Tip de memoria:** En sistemas con 8 GB de RAM, asigna un total máximo de 3 GB a Couchbase para dejar recursos al sistema operativo y el navegador.

5. Haz clic en **"Save & Finish"**.

**Salida esperada:**
Serás redirigido al **Dashboard** principal de la Web Console. Deberías ver el clúster `lab-cluster` con el nodo activo y los servicios listados.

**Verificación:**

Confirma la configuración del clúster via REST API:

```bash
curl -s -u Administrator:Password123! \
  http://localhost:8091/pools/default \
  | python3 -m json.tool | grep -E '"name"|"memoryQuota"'
```

Salida esperada (valores aproximados):
```json
"name": "lab-cluster",
"memoryQuota": 1024,
```

---

### Paso 3: Cargar el Dataset `travel-sample`

**Objetivo:** Cargar el bucket de muestra `travel-sample` que contiene ~31,000 documentos distribuidos en 5 colecciones, replicando el tipo de datos con el que trabajarás en todos los laboratorios siguientes.

**Instrucciones:**

1. En la Web Console, navega al menú lateral: **Settings → Sample Buckets**.

2. En la lista de buckets disponibles, localiza **`travel-sample`** y marca su casilla.

3. Haz clic en **"Load Sample Data"**.

4. Espera a que el proceso de carga complete. Verás una barra de progreso. El proceso tarda aproximadamente 1–2 minutos.

5. Una vez completado, navega a **Buckets** en el menú lateral. Deberías ver `travel-sample` con el indicador de documentos cargados.

**Verificación:**

Confirma la carga via REST API:

```bash
curl -s -u Administrator:Password123! \
  http://localhost:8091/pools/default/buckets/travel-sample \
  | python3 -m json.tool | grep -E '"name"|"itemCount"'
```

Salida esperada:
```json
"name": "travel-sample",
"itemCount": 31591,
```

> 📌 **Conexión con la lección:** El dataset `travel-sample` utiliza el modelo de documentos JSON que Apache CouchDB popularizó. Cada documento es un objeto JSON autónomo sin esquema fijo, lo que refleja la filosofía "schema-flexible" que Couchbase heredó de CouchDB.

---

### Paso 4: Explorar la Estructura del Dataset en la Web Console

**Objetivo:** Navegar por los scopes y collections del bucket `travel-sample` y examinar documentos individuales con el Document Editor para comprender la estructura JSON del dataset.

**Instrucciones:**

1. En el menú lateral, haz clic en **Buckets**. Localiza `travel-sample` y haz clic en **"Scopes & Collections"** (ícono de tabla o enlace directo).

2. Observa la estructura de scopes y collections:

   ```
   travel-sample
   └── inventory (scope)
       ├── airline        (~187 documentos)
       ├── airport        (~1,968 documentos)
       ├── hotel          (~917 documentos)
       ├── landmark       (~4,495 documentos)
       └── route          (~24,024 documentos)
   ```

   > 📌 **Conexión con la lección:** Los *scopes* y *collections* fueron introducidos en Couchbase 6.5 y consolidados en la versión 7.0. Representan la evolución del modelo de organización de datos, acercándolo al concepto de bases de datos/tablas del mundo relacional.

3. Navega a **Documents** dentro de la collection `airline`:
   - Haz clic en `travel-sample` → `inventory` → `airline` → **Documents**.

4. Haz clic en el documento con ID `airline_10` para abrirlo en el **Document Editor**.

5. Examina la estructura JSON del documento:

   ```json
   {
     "id": 10,
     "type": "airline",
     "name": "40-Mile Air",
     "iata": "Q5",
     "icao": "MLA",
     "callsign": "MILE-AIR",
     "country": "United States"
   }
   ```

6. Ahora navega a la collection `hotel` y abre el documento `hotel_10025`. Observa cómo este documento tiene una estructura más compleja con campos anidados:

   ```json
   {
     "id": 10025,
     "type": "hotel",
     "name": "Medway Youth Hostel",
     "address": "Capstone Road, ME7 3JE",
     "city": "Medway",
     "country": "United Kingdom",
     "reviews": [ ... ],
     "geo": {
       "lat": 51.35785,
       "lon": 0.55818,
       "accuracy": "RANGE_INTERPOLATED"
     }
   }
   ```

7. Anota las diferencias estructurales entre los documentos de `airline` (estructura plana) y `hotel` (con arrays y objetos anidados). Esta diferencia es fundamental para el diseño de modelos de datos que estudiarás en laboratorios posteriores.

**Verificación:**

Confirma que puedes recuperar un documento específico via REST API:

```bash
curl -s -u Administrator:Password123! \
  "http://localhost:8091/pools/default/buckets/travel-sample/docs/airline_10" \
  | python3 -m json.tool
```

Salida esperada (fragmento):
```json
{
    "meta": {
        "id": "airline_10",
        "rev": "...",
        "expiration": 0,
        "flags": 0
    },
    "json": {
        "id": 10,
        "type": "airline",
        "name": "40-Mile Air",
        ...
    }
}
```

---

### Paso 5: Recorrido por la Web Console — Secciones Principales

**Objetivo:** Familiarizarse con cada sección de la Web Console de Couchbase para poder navegar con fluidez en laboratorios posteriores.

**Instrucciones:**

Realiza el siguiente recorrido guiado. Para cada sección, anota en tu cuaderno de laboratorio qué información muestra y para qué sirve.

#### 5.1 Dashboard (Inicio)

1. Haz clic en **Dashboard** en el menú lateral.
2. Observa las métricas en tiempo real:
   - **RAM Used / Total:** memoria utilizada por el servicio Data.
   - **Disk Used / Total:** espacio en disco utilizado.
   - **Items:** número total de documentos en todos los buckets.
   - **Ops/sec:** operaciones por segundo (lecturas + escrituras).

#### 5.2 Servers

1. Haz clic en **Servers** en el menú lateral.
2. Observa la tabla del nodo activo. Identifica:
   - **Node Name/IP:** dirección del nodo (debería ser `127.0.0.1` o el nombre del contenedor).
   - **Services:** lista de servicios habilitados (data, index, n1ql, fts, eventing).
   - **RAM:** memoria asignada y utilizada.
   - **Status:** debe mostrar `healthy` con ícono verde.

   > 📌 **Arquitectura:** Cada servicio listado aquí corresponde a un proceso independiente dentro del nodo. Esta separación de servicios es la base del **Multi-Dimensional Scaling (MDS)** introducido en Couchbase 4.0, que permite escalar cada servicio de forma independiente en clústeres multi-nodo.

#### 5.3 Buckets

1. Haz clic en **Buckets** en el menú lateral.
2. Observa el bucket `travel-sample`. Haz clic en **"..."** (menú de opciones) y selecciona **Edit** para ver:
   - **Memory Quota:** RAM asignada al bucket.
   - **Replicas:** número de réplicas (en nodo único, las réplicas están deshabilitadas o en 0).
   - **Bucket Type:** `Couchbase` (con persistencia) vs `Ephemeral` (solo memoria, herencia de Memcached).

   > 📌 **Conexión histórica:** El tipo de bucket `Ephemeral` es la evolución directa de Memcached: datos en memoria sin persistencia en disco. El tipo `Couchbase` representa la fusión con CouchDB: datos en memoria con persistencia automática.

#### 5.4 Indexes

1. Haz clic en **Indexes** en el menú lateral.
2. Observa los índices que se crearon automáticamente al cargar `travel-sample`. Identifica:
   - El índice **Primary Index** (si existe).
   - Los índices secundarios con sus nombres, estados y colecciones asociadas.

#### 5.5 Query

1. Haz clic en **Query** en el menú lateral.
2. Esta sección abre el **Query Editor**. La usarás en el Paso 6.

#### 5.6 Search

1. Haz clic en **Search** en el menú lateral.
2. Observa si hay índices FTS (Full Text Search) precargados con `travel-sample`. Esta sección será el foco de laboratorios posteriores.

#### 5.7 Security

1. Haz clic en **Security** en el menú lateral.
2. Navega a **Users**. Observa el usuario `Administrator` y sus roles.
3. Navega a **Roles**. Observa la lista de roles disponibles (RBAC introducido en Couchbase 5.0).

**Verificación:**

Confirma que todos los servicios están activos via REST API:

```bash
curl -s -u Administrator:Password123! \
  http://localhost:8091/pools/default/nodeServices \
  | python3 -m json.tool | grep -E '"services"|"n1ql"|"fts"|"eventing"|"index"'
```

---

### Paso 6: Primera Consulta SQL++ en el Query Editor

**Objetivo:** Ejecutar consultas SQL++ básicas desde el Query Editor de la Web Console para validar el funcionamiento del servicio Query y familiarizarse con la sintaxis del lenguaje.

**Instrucciones:**

1. Navega a **Query** en el menú lateral de la Web Console.

2. En el área de texto del Query Editor, escribe y ejecuta la siguiente consulta para listar las aerolíneas de Estados Unidos:

```sql
SELECT name, iata, country
FROM `travel-sample`.inventory.airline
WHERE country = "United States"
ORDER BY name
LIMIT 10;
```

3. Haz clic en **"Execute"** o presiona `Ctrl+Enter` / `Cmd+Enter`.

**Salida esperada (fragmento):**
```json
[
  { "country": "United States", "iata": "Q5", "name": "40-Mile Air" },
  { "country": "United States", "iata": "TQ", "name": "Atifly" },
  { "country": "United States", "iata": "MQ", "name": "Envoy Air" },
  ...
]
```

4. Ahora ejecuta una consulta de conteo para ver cuántas aerolíneas hay por país:

```sql
SELECT country, COUNT(*) AS total_airlines
FROM `travel-sample`.inventory.airline
GROUP BY country
ORDER BY total_airlines DESC
LIMIT 5;
```

**Salida esperada:**
```json
[
  { "country": "United States", "total_airlines": 116 },
  { "country": "United Kingdom", "total_airlines": 22 },
  { "country": "France", "total_airlines": 10 },
  ...
]
```

5. Ejecuta una consulta que acceda a campos anidados en la collection `hotel`:

```sql
SELECT name, city, geo.lat, geo.lon
FROM `travel-sample`.inventory.hotel
WHERE country = "United Kingdom"
  AND geo IS NOT NULL
LIMIT 5;
```

**Salida esperada (fragmento):**
```json
[
  {
    "city": "Medway",
    "lat": 51.35785,
    "lon": 0.55818,
    "name": "Medway Youth Hostel"
  },
  ...
]
```

6. Observa el panel **"Query Plan"** en la parte inferior del Query Editor. Haz clic en él para ver el plan de ejecución de la última consulta. Nota si usa un índice (`IndexScan`) o un escaneo completo (`PrimaryScan`).

> 📌 **Conexión con la lección:** La sintaxis `SELECT ... FROM ... WHERE` que acabas de usar es deliberadamente idéntica a SQL estándar. Esta fue una decisión de diseño clave cuando se introdujo N1QL en Couchbase 4.0 (2015): reducir la curva de aprendizaje para equipos con experiencia en bases de datos relacionales.

**Verificación:**

Ejecuta la misma consulta via REST API para confirmar que el servicio Query responde correctamente:

```bash
curl -s -u Administrator:Password123! \
  http://localhost:8093/query/service \
  -d 'statement=SELECT name, iata FROM `travel-sample`.inventory.airline WHERE country="United States" LIMIT 3' \
  | python3 -m json.tool | grep -E '"name"|"iata"|"status"'
```

Salida esperada (fragmento):
```json
"status": "success",
"name": "40-Mile Air",
"iata": "Q5",
```

---

### Paso 7: Explorar la Arquitectura del Nodo — vBuckets y Servicios

**Objetivo:** Identificar los componentes arquitectónicos del nodo Couchbase (vBuckets, Managed Cache, Storage Engine) y los puertos asociados a cada servicio.

**Instrucciones:**

1. Consulta la información de vBuckets del bucket `travel-sample` via REST API:

```bash
curl -s -u Administrator:Password123! \
  "http://localhost:8091/pools/default/buckets/travel-sample/nodes" \
  | python3 -m json.tool | grep -E '"hostname"|"status"|"services"'
```

2. Consulta el mapa de vBuckets para ver cómo se distribuyen las 1024 particiones virtuales:

```bash
curl -s -u Administrator:Password123! \
  "http://localhost:8091/pools/default/buckets/travel-sample/nodeLocator" \
  | python3 -m json.tool | head -20
```

3. Revisa la tabla de puertos de servicios de Couchbase. Confirma que cada puerto responde:

```bash
# Puerto de administración (heredado de Membase)
curl -s -o /dev/null -w "Admin REST (8091): %{http_code}\n" \
  http://localhost:8091/ui/index.html

# Puerto del servicio Query (N1QL/SQL++)
curl -s -o /dev/null -w "Query Service (8093): %{http_code}\n" \
  -u Administrator:Password123! \
  http://localhost:8093/query/service \
  -d 'statement=SELECT 1'

# Puerto del servicio Search (FTS)
curl -s -o /dev/null -w "Search Service (8094): %{http_code}\n" \
  -u Administrator:Password123! \
  http://localhost:8094/api/index
```

4. Consulta el resumen completo del clúster para ver todos los servicios y sus puertos:

```bash
curl -s -u Administrator:Password123! \
  http://localhost:8091/pools/default \
  | python3 -m json.tool | grep -E '"services"|"ports"' | head -20
```

**Tabla de referencia de puertos de Couchbase Server 7.6.x:**

| Puerto | Protocolo | Servicio                        |
|--------|-----------|----------------------------------|
| 8091   | HTTP      | Administración (Web Console, REST API) |
| 8092   | HTTP      | Views / MapReduce               |
| 8093   | HTTP      | Query Service (SQL++)           |
| 8094   | HTTP      | Search Service (FTS)            |
| 8095   | HTTP      | Analytics Service               |
| 8096   | HTTP      | Eventing Service                |
| 11210  | TCP       | Data Service (SDK / KV)         |
| 18091–18097 | HTTPS | Versiones TLS de los anteriores |

> 📌 **Conexión arquitectónica:** Los 1024 vBuckets que verás en la respuesta son la herencia directa de Membase. Esta cantidad fue elegida para permitir un rebalanceo eficiente del clúster cuando se agregan o eliminan nodos, distribuyendo los datos de forma uniforme sin necesidad de rehashing completo.

**Verificación:**

Confirma el número de vBuckets configurados:

```bash
curl -s -u Administrator:Password123! \
  "http://localhost:8091/pools/default/buckets/travel-sample" \
  | python3 -m json.tool | grep -E '"numVBuckets"'
```

Salida esperada:
```json
"numVBuckets": 1024,
```

---

### Paso 8: Consulta Avanzada a la REST API de Administración

**Objetivo:** Usar `curl` para interactuar con la REST API de Couchbase y obtener información detallada del clúster, consolidando el uso de la API que usarás en laboratorios posteriores.

**Instrucciones:**

1. Obtén información general del clúster:

```bash
curl -s -u Administrator:Password123! \
  http://localhost:8091/pools \
  | python3 -m json.tool
```

2. Obtén la lista de todos los buckets configurados:

```bash
curl -s -u Administrator:Password123! \
  http://localhost:8091/pools/default/buckets \
  | python3 -m json.tool | grep -E '"name"|"bucketType"|"quota"'
```

3. Obtén estadísticas de uso del bucket `travel-sample`:

```bash
curl -s -u Administrator:Password123! \
  "http://localhost:8091/pools/default/buckets/travel-sample/stats?zoom=minute" \
  | python3 -m json.tool | grep -E '"curr_items"|"mem_used"' | head -5
```

4. Obtén la lista de usuarios configurados en el clúster:

```bash
curl -s -u Administrator:Password123! \
  http://localhost:8091/settings/rbac/users \
  | python3 -m json.tool | grep -E '"id"|"name"|"roles"'
```

5. Obtén información sobre los índices activos:

```bash
curl -s -u Administrator:Password123! \
  "http://localhost:9102/api/v1/stats" \
  | python3 -m json.tool | head -30
```

> 💡 **Tip:** La REST API de Couchbase es el mecanismo subyacente que usa la Web Console. Todo lo que puedes hacer en la interfaz gráfica puede hacerse también via API, lo que permite automatizar la administración del clúster.

**Verificación:**

Confirma que el clúster reporta estado saludable:

```bash
curl -s -u Administrator:Password123! \
  http://localhost:8091/pools/default \
  | python3 -m json.tool | grep -E '"rebalanceStatus"|"balanced"'
```

Salida esperada:
```json
"rebalanceStatus": "none",
"balanced": true,
```

---

## Validación y Pruebas

Al completar todos los pasos, ejecuta la siguiente secuencia de verificación final para confirmar que el entorno está correctamente configurado:

```bash
#!/bin/bash
# Script de verificación final del Lab 02-00-01
echo "=== Verificación Final: Lab 02-00-01 ==="
echo ""

# 1. Verificar que Couchbase responde
echo "[1/5] Verificando Web Console..."
STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8091/ui/index.html)
[ "$STATUS" = "200" ] && echo "  ✅ Web Console accesible (HTTP $STATUS)" || echo "  ❌ Web Console NO accesible (HTTP $STATUS)"

# 2. Verificar autenticación
echo "[2/5] Verificando autenticación..."
AUTH=$(curl -s -u Administrator:Password123! -o /dev/null -w "%{http_code}" http://localhost:8091/pools)
[ "$AUTH" = "200" ] && echo "  ✅ Autenticación correcta" || echo "  ❌ Error de autenticación (HTTP $AUTH)"

# 3. Verificar bucket travel-sample
echo "[3/5] Verificando bucket travel-sample..."
ITEMS=$(curl -s -u Administrator:Password123! \
  "http://localhost:8091/pools/default/buckets/travel-sample" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('basicStats',{}).get('itemCount',0))" 2>/dev/null)
[ "$ITEMS" -gt "30000" ] 2>/dev/null && echo "  ✅ travel-sample cargado ($ITEMS documentos)" || echo "  ❌ travel-sample no encontrado o vacío"

# 4. Verificar servicio Query
echo "[4/5] Verificando servicio Query (SQL++)..."
QUERY_RESULT=$(curl -s -u Administrator:Password123! \
  http://localhost:8093/query/service \
  -d 'statement=SELECT COUNT(*) AS total FROM `travel-sample`.inventory.airline' \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('status','error'))" 2>/dev/null)
[ "$QUERY_RESULT" = "success" ] && echo "  ✅ Servicio Query operativo" || echo "  ❌ Servicio Query no responde"

# 5. Verificar servicio Search
echo "[5/5] Verificando servicio Search (FTS)..."
FTS=$(curl -s -u Administrator:Password123! -o /dev/null -w "%{http_code}" http://localhost:8094/api/index)
[ "$FTS" = "200" ] && echo "  ✅ Servicio Search accesible" || echo "  ❌ Servicio Search no accesible (HTTP $FTS)"

echo ""
echo "=== Verificación completada ==="
```

Copia y pega este script en tu terminal. Todos los ítems deben mostrar ✅ para considerar el laboratorio completado exitosamente.

---

## Resolución de Problemas

### Problema 1: La Web Console no carga en `http://localhost:8091`

**Síntomas:**
- El navegador muestra "No se puede acceder a este sitio" o "Connection refused".
- `curl http://localhost:8091` devuelve `curl: (7) Failed to connect to localhost port 8091`.

**Causa probable:**
El contenedor Docker no está en ejecución, o el proceso de Couchbase Server no ha terminado de inicializar. También puede deberse a un conflicto de puertos con otra aplicación.

**Solución:**

```bash
# Verificar si el contenedor está corriendo
docker ps -a --filter "name=couchbase-lab"

# Si el estado es "Exited", reiniciarlo
docker start couchbase-lab

# Esperar 30 segundos y verificar logs
docker logs couchbase-lab --tail 20

# Verificar conflictos de puertos
lsof -i :8091 2>/dev/null || netstat -tulnp 2>/dev/null | grep 8091

# Si hay conflicto, detener el proceso que usa el puerto o
# cambiar el mapeo de puertos al crear el contenedor:
docker run -d \
  --name couchbase-lab \
  -p 8191-8197:8091-8097 \
  -p 11310:11210 \
  couchbase/server:7.6.2
# (Luego acceder en http://localhost:8191)
```

---

### Problema 2: El servicio Query devuelve error al ejecutar consultas SQL++

**Síntomas:**
- El Query Editor muestra `"errors": [{"code": 4000, "msg": "..."}]`.
- El error menciona `"No index available"` o `"Keyspace not found"`.

**Causa probable:**
No existe un índice primario en la colección consultada, o el nombre del bucket/scope/collection está mal escrito (sensible a mayúsculas). También puede ocurrir si el dataset `travel-sample` no terminó de cargarse completamente.

**Solución:**

```bash
# Verificar que el dataset está completamente cargado
curl -s -u Administrator:Password123! \
  "http://localhost:8091/pools/default/buckets/travel-sample" \
  | python3 -m json.tool | grep '"itemCount"'
# Debe mostrar ~31591

# Si la colección no tiene índice, crear uno primario (solo para pruebas):
# En el Query Editor de la Web Console, ejecutar:
```

```sql
-- Crear índice primario en airline (solo si no existe)
CREATE PRIMARY INDEX ON `travel-sample`.inventory.airline
  USING GSI;

-- Verificar que el índice fue creado
SELECT * FROM system:indexes
WHERE keyspace_id = "airline"
  AND bucket_id = "travel-sample";
```

```bash
# Verificar la sintaxis del nombre del bucket (backticks obligatorios si contiene guiones)
# CORRECTO:
# FROM `travel-sample`.inventory.airline
# INCORRECTO:
# FROM travel-sample.inventory.airline
```

---

## Limpieza del Entorno

Al finalizar el laboratorio, **NO elimines el entorno** si continuarás con el Lab 03, ya que los laboratorios siguientes dependen del bucket `travel-sample` cargado.

Si necesitas liberar recursos temporalmente:

```bash
# Pausar el contenedor (conserva los datos)
docker stop couchbase-lab

# Para reanudar en la próxima sesión
docker start couchbase-lab
# Esperar 30 segundos antes de acceder a http://localhost:8091
```

Si deseas eliminar completamente el entorno (solo al finalizar el curso):

```bash
# Detener y eliminar el contenedor
docker stop couchbase-lab
docker rm couchbase-lab

# Eliminar la imagen (opcional, libera ~1 GB)
docker rmi couchbase/server:7.6.2
```

Para instalaciones nativas, el servicio puede detenerse sin eliminar datos:

```bash
# Linux
sudo systemctl stop couchbase-server

# macOS: usar el ícono de la barra de menú → "Stop Couchbase Server"

# Windows: Servicios → "CouchbaseServer" → Detener
```

---

## Resumen

En este laboratorio completaste la instalación y configuración inicial de Couchbase Server 7.6.x, estableciendo el entorno base que usarás en todos los laboratorios del curso. Los logros clave fueron:

| Actividad | Resultado |
|-----------|-----------|
| Instalación de Couchbase Server 7.6.x | Clúster `lab-cluster` de nodo único operativo |
| Configuración de servicios | Data, Index, Query, Search y Eventing habilitados |
| Carga de datos | ~31,591 documentos en `travel-sample` (5 colecciones) |
| Exploración de Web Console | 7 secciones revisadas: Dashboard, Servers, Buckets, Indexes, Query, Search, Security |
| Primera consulta SQL++ | 3 consultas ejecutadas exitosamente en el Query Editor |
| Uso de REST API | 8 endpoints consultados con `curl` |

### Conexiones con la Lección 2.1

A lo largo de este laboratorio observaste en acción los hitos históricos estudiados:

- **Herencia de Membase:** los 1024 vBuckets que distribuyen automáticamente los documentos del `travel-sample`.
- **Herencia de CouchDB:** los documentos JSON flexibles sin esquema fijo, como los documentos `hotel` con campos anidados y arrays.
- **Evolución a SQL++ (v4.0, 2015):** la sintaxis `SELECT ... FROM ... WHERE` que ejecutaste es directamente comparable a SQL relacional.
- **Colecciones y Scopes (v7.0, 2021):** la organización `travel-sample` → `inventory` → `airline/airport/hotel/landmark/route` que exploraste.

### Próximos Pasos

En el **Lab 03**, utilizarás este entorno para diseñar modelos de datos JSON para aplicaciones distribuidas, analizando los trade-offs entre documentos anidados y referencias entre colecciones. Asegúrate de que el bucket `travel-sample` permanezca cargado.

### Recursos Adicionales

- [Documentación oficial: Instalación de Couchbase Server](https://docs.couchbase.com/server/current/install/install-intro.html)
- [Documentación oficial: Sample Buckets](https://docs.couchbase.com/server/current/manage/manage-settings/install-sample-buckets.html)
- [Referencia de la REST API de Couchbase](https://docs.couchbase.com/server/current/rest-api/rest-intro.html)
- [Introducción a SQL++ en Couchbase](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/index.html)
- [Arquitectura de vBuckets](https://docs.couchbase.com/server/current/learn/buckets-memory-and-storage/vbuckets.html)
- [Notas de versión de Couchbase Server 7.6](https://docs.couchbase.com/server/current/release-notes/relnotes.html)

---
