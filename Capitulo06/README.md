---LAB_START---
LAB_ID: 06-00-01
---MARKDOWN---
# Creación y despliegue de una Function de Eventing

## Metadatos

| Campo            | Detalle                                      |
|------------------|----------------------------------------------|
| **Duración**     | 60 minutos                                   |
| **Complejidad**  | Media                                        |
| **Nivel Bloom**  | Crear (*Create*)                             |
| **Servicio**     | Couchbase Eventing Service                   |
| **Dataset**      | `app-data` (creado durante el laboratorio)   |

---

## Descripción General

En las lecciones teóricas exploramos las limitaciones del *polling*, los *database triggers* y las colas de mensajes externas para reaccionar a cambios en datos: latencia variable, acoplamiento fuerte y el riesgo de la doble escritura. Couchbase Eventing resuelve estos problemas con **Functions** nativas que se ejecutan directamente en el clúster, sin infraestructura adicional ni responsabilidad en la aplicación.

En este laboratorio crearás tres Functions de Eventing que cubren los patrones más frecuentes en producción: **auditoría de cambios**, **enriquecimiento de documentos** y **validación con cuarentena**. Realizarás el ciclo completo de deploy, prueba, observación de logs y undeploy de cada Function.

---

## Objetivos de Aprendizaje

- [ ] Crear y configurar una Function de Eventing con bindings correctos (source collection, metadata collection y alias a colección destino)
- [ ] Implementar los handlers `OnUpdate` y `OnDelete` en JavaScript para los tres casos de uso: auditoría, enriquecimiento y validación
- [ ] Ejecutar el ciclo completo de deploy → prueba → verificación de logs → undeploy de una Function
- [ ] Configurar los parámetros avanzados de una Function: worker count, log level y checkpoint interval
- [ ] Distinguir cuándo usar `OnUpdate` vs `OnDelete` y aplicarlos en escenarios reales

---

## Prerrequisitos

### Conocimiento previo
- Haber completado el **Lab 02-00-01** (Couchbase Server instalado y operativo con Web Console accesible)
- Conocimientos básicos de JavaScript: funciones, objetos literales, condicionales `if/else` y operador ternario
- Comprensión de la estructura de documentos JSON en Couchbase (buckets, scopes, collections)
- Familiaridad con la Web Console de Couchbase (navegación por menús, creación de buckets)

### Acceso requerido
- Couchbase Server 7.6.x en ejecución (local o Docker)
- Web Console accesible en `http://localhost:8091`
- Credenciales de administrador (por defecto: `Administrator` / `password`)
- **Servicio Eventing habilitado** en el nodo (verificar en *Servers → Services*)

---

## Entorno de Laboratorio

### Requisitos de Hardware y Software

| Recurso        | Mínimo                          | Recomendado                     |
|----------------|---------------------------------|---------------------------------|
| RAM            | 8 GB (4 GB para Couchbase)      | 16 GB                           |
| CPU            | 4 núcleos x86_64                | 8 núcleos                       |
| Almacenamiento | 20 GB libres (SSD)              | 50 GB SSD                       |
| Navegador      | Chrome/Firefox/Edge 110+        | Chrome 120+                     |
| Couchbase      | Server 7.6.x                    | Enterprise Trial 7.6.x          |

> **Nota Docker:** Si ejecutas Couchbase en Docker, asigna al menos **2 CPU cores** y **4 GB de RAM** al contenedor para que el servicio Eventing funcione correctamente.

### Verificación del Servicio Eventing

Antes de comenzar, confirma que el servicio Eventing está activo:

```bash
# Verificar servicios activos en el nodo
curl -s -u Administrator:password \
  http://localhost:8091/pools/default/nodeServices \
  | python3 -m json.tool | grep eventing
```

Si el servicio no aparece, ve a **Couchbase Web Console → Servers → tu nodo → Edit** y habilita el servicio **Eventing** (requiere reinicio del nodo).

### Configuración Inicial del Entorno

Ejecuta los siguientes comandos para crear la estructura de buckets y colecciones que usarás en todo el laboratorio.

**Paso 0.1 — Crear el bucket `app-data`:**

```bash
curl -s -u Administrator:password \
  -X POST http://localhost:8091/pools/default/buckets \
  -d name=app-data \
  -d ramQuota=256 \
  -d bucketType=couchbase \
  -d replicaNumber=0
```

Espera 5 segundos para que el bucket se inicialice completamente:

```bash
sleep 5
```

**Paso 0.2 — Crear el scope `workshop`:**

```bash
curl -s -u Administrator:password \
  -X POST \
  "http://localhost:8091/pools/default/buckets/app-data/scopes" \
  -d name=workshop
```

**Paso 0.3 — Crear las colecciones necesarias:**

```bash
# Colección de pedidos (fuente de eventos)
curl -s -u Administrator:password \
  -X POST \
  "http://localhost:8091/pools/default/buckets/app-data/scopes/workshop/collections" \
  -d name=orders

# Colección de log de auditoría (destino Caso 1)
curl -s -u Administrator:password \
  -X POST \
  "http://localhost:8091/pools/default/buckets/app-data/scopes/workshop/collections" \
  -d name=audit-log

# Colección de cuarentena (destino Caso 3)
curl -s -u Administrator:password \
  -X POST \
  "http://localhost:8091/pools/default/buckets/app-data/scopes/workshop/collections" \
  -d name=quarantine
```

**Paso 0.4 — Crear el bucket `eventing-meta` (para metadata de Eventing):**

```bash
curl -s -u Administrator:password \
  -X POST http://localhost:8091/pools/default/buckets \
  -d name=eventing-meta \
  -d ramQuota=256 \
  -d bucketType=couchbase \
  -d replicaNumber=0
```

**Paso 0.5 — Verificar la estructura creada:**

```bash
curl -s -u Administrator:password \
  "http://localhost:8091/pools/default/buckets/app-data/scopes" \
  | python3 -m json.tool
```

Deberías ver el scope `workshop` con las colecciones `orders`, `audit-log` y `quarantine`.

---

## Pasos del Laboratorio

---

### Paso 1: Explorar el Editor de Eventing en la Web Console

**Objetivo:** Familiarizarse con la interfaz de Eventing antes de crear la primera Function.

**Instrucciones:**

1. Abre la Web Console en `http://localhost:8091` e inicia sesión.
2. En el menú lateral izquierdo, haz clic en **Eventing**.
3. Observa la pantalla principal: estará vacía con el botón **+ Add Function** visible.
4. Haz clic en **+ Add Function** para abrir el formulario de creación (aún no guardes nada).
5. Examina los campos disponibles:
   - **Function Name:** identificador único de la Function
   - **Source Bucket / Scope / Collection:** origen de los eventos
   - **Metadata Bucket / Scope / Collection:** almacenamiento interno de checkpoints
   - **Bindings:** referencias a otras colecciones o credenciales
   - **Settings:** worker count, log level, checkpoint interval
6. Cierra el formulario con **Cancel** — lo usarás en el siguiente paso.

**Salida esperada:** Comprensión del formulario de creación. No se crea ninguna Function todavía.

**Verificación:** Puedes navegar por el formulario sin errores y ver todos los campos mencionados.

---

### Paso 2: Caso 1 — Function de Auditoría de Cambios

**Objetivo:** Crear una Function que registre cada modificación a un documento de `orders` en la colección `audit-log`, capturando el ID del documento, timestamp, tipo de operación y usuario.

#### 2.1 — Crear la Function de Auditoría

**Instrucciones:**

1. En la Web Console, ve a **Eventing → + Add Function**.
2. Completa el formulario con los siguientes valores:

   | Campo                        | Valor                          |
   |------------------------------|--------------------------------|
   | Function Name                | `AuditOrderChanges`            |
   | Source Bucket                | `app-data`                     |
   | Source Scope                 | `workshop`                     |
   | Source Collection            | `orders`                       |
   | Metadata Bucket              | `eventing-meta`                |
   | Metadata Scope               | `_default`                     |
   | Metadata Collection          | `_default`                     |

3. En la sección **Bindings**, haz clic en **+ Add Binding** y configura:

   | Campo          | Valor         |
   |----------------|---------------|
   | Binding Type   | `Bucket`      |
   | Alias          | `auditLog`    |
   | Bucket         | `app-data`    |
   | Scope          | `workshop`    |
   | Collection     | `audit-log`   |
   | Access         | `Read/Write`  |

4. Haz clic en **Next: Add Code** para abrir el editor de JavaScript.
5. **Reemplaza todo el contenido** del editor con el siguiente código:

```javascript
// Function: AuditOrderChanges
// Propósito: Registrar en audit-log cada cambio en la colección orders
// Demuestra: OnUpdate + OnDelete con binding a colección destino

function OnUpdate(doc, meta) {
    // Construir el registro de auditoría
    var auditRecord = {
        type: "audit",
        sourceDocId: meta.id,
        operation: "UPDATE",
        timestamp: new Date().toISOString(),
        // Capturar el usuario si el documento lo incluye
        modifiedBy: doc.userId || "system",
        // Snapshot del estado actual del documento
        docSnapshot: {
            status: doc.status || null,
            totalAmount: doc.totalAmount || null,
            itemCount: doc.items ? doc.items.length : 0
        }
    };

    // Construir el ID del registro de auditoría con timestamp
    var auditKey = "audit::" + meta.id + "::" + Date.now();

    // Escribir en la colección audit-log usando el binding 'auditLog'
    auditLog[auditKey] = auditRecord;

    log("AuditOrderChanges", "Audit record created for doc: " + meta.id);
}

function OnDelete(meta, options) {
    // Registrar también las eliminaciones
    var auditRecord = {
        type: "audit",
        sourceDocId: meta.id,
        operation: "DELETE",
        timestamp: new Date().toISOString(),
        modifiedBy: "system",
        isExpiration: options.expired || false
    };

    var auditKey = "audit::" + meta.id + "::deleted::" + Date.now();
    auditLog[auditKey] = auditRecord;

    log("AuditOrderChanges", "Delete audit record created for doc: " + meta.id);
}
```

6. Antes de guardar, haz clic en **Settings** (pestaña en el editor) y configura:

   | Parámetro            | Valor     | Razón                                          |
   |----------------------|-----------|------------------------------------------------|
   | Worker Count         | `1`       | Suficiente para laboratorio; producción: 3-6   |
   | Log Level            | `INFO`    | Captura los mensajes `log()` del código        |
   | Checkpoint Interval  | `10000`   | Checkpoint cada 10 segundos (ms)               |

7. Haz clic en **Save and Return**.

**Salida esperada:** La Function `AuditOrderChanges` aparece en la lista con estado **Undeployed**.

#### 2.2 — Deploy y Prueba de la Function de Auditoría

**Instrucciones:**

1. En la lista de Functions, localiza `AuditOrderChanges` y haz clic en **Deploy**.
2. En el diálogo de confirmación, selecciona **Deploy Function** (opción: *From now* para procesar solo eventos nuevos).
3. Espera hasta que el estado cambie a **Deployed** (puede tomar 10-30 segundos).
4. Abre una nueva pestaña del navegador y ve a **Query** en la Web Console.
5. Inserta un documento de prueba en la colección `orders`:

```sql
INSERT INTO `app-data`.workshop.orders (KEY, VALUE)
VALUES (
  "order::1001",
  {
    "type": "order",
    "orderId": "1001",
    "userId": "user::ana.garcia",
    "status": "pending",
    "items": [
      {"productId": "prod::A1", "name": "Laptop", "qty": 1, "price": 1200.00},
      {"productId": "prod::B2", "name": "Mouse", "qty": 2, "price": 25.00}
    ],
    "totalAmount": 0,
    "createdAt": "2024-06-15T10:00:00Z"
  }
);
```

6. Espera 3-5 segundos y luego verifica si se creó el registro de auditoría:

```sql
SELECT META().id AS auditId, *
FROM `app-data`.workshop.`audit-log`
WHERE type = "audit"
AND sourceDocId = "order::1001"
ORDER BY timestamp DESC;
```

7. Modifica el documento para generar un segundo evento:

```sql
UPDATE `app-data`.workshop.orders
SET status = "processing"
WHERE META().id = "order::1001";
```

8. Espera 3-5 segundos y vuelve a ejecutar la consulta de auditoría — deberías ver dos registros.

**Salida esperada:**

```json
[
  {
    "auditId": "audit::order::1001::1718445612345",
    "audit-log": {
      "type": "audit",
      "sourceDocId": "order::1001",
      "operation": "UPDATE",
      "timestamp": "2024-06-15T10:00:12.345Z",
      "modifiedBy": "user::ana.garcia",
      "docSnapshot": {
        "status": "processing",
        "totalAmount": 0,
        "itemCount": 2
      }
    }
  }
]
```

**Verificación:**

```bash
# Contar registros de auditoría generados
curl -s -u Administrator:password \
  -X POST http://localhost:8093/query/service \
  -H "Content-Type: application/json" \
  -d '{"statement": "SELECT COUNT(*) AS total FROM `app-data`.workshop.`audit-log` WHERE type = \"audit\""}' \
  | python3 -m json.tool
```

El campo `total` debe ser **≥ 2**.

#### 2.3 — Revisar los Logs de la Function

**Instrucciones:**

1. Regresa a **Eventing** en la Web Console.
2. Localiza `AuditOrderChanges` y haz clic en el ícono de **Logs** (icono de documento con líneas).
3. Observa los mensajes generados por las llamadas a `log()` en el código.
4. Deberías ver entradas como:
   ```
   [INFO] AuditOrderChanges: Audit record created for doc: order::1001
   ```

**Verificación:** Al menos 2 entradas de log correspondientes a las 2 operaciones realizadas.

---

### Paso 3: Caso 2 — Function de Enriquecimiento de Documentos

**Objetivo:** Crear una Function que, al insertar un pedido nuevo, calcule automáticamente el `totalAmount` sumando `qty * price` de cada ítem y lo escriba de vuelta en el documento original.

#### 3.1 — Crear la Function de Enriquecimiento

**Instrucciones:**

1. Ve a **Eventing → + Add Function**.
2. Completa el formulario:

   | Campo                        | Valor                          |
   |------------------------------|--------------------------------|
   | Function Name                | `EnrichOrderTotal`             |
   | Source Bucket                | `app-data`                     |
   | Source Scope                 | `workshop`                     |
   | Source Collection            | `orders`                       |
   | Metadata Bucket              | `eventing-meta`                |
   | Metadata Scope               | `_default`                     |
   | Metadata Collection          | `_default`                     |

3. En **Bindings**, agrega un binding para escribir de vuelta en la colección `orders`:

   | Campo          | Valor         |
   |----------------|---------------|
   | Binding Type   | `Bucket`      |
   | Alias          | `ordersCol`   |
   | Bucket         | `app-data`    |
   | Scope          | `workshop`    |
   | Collection     | `orders`      |
   | Access         | `Read/Write`  |

   > **Importante:** El binding a la misma colección fuente es válido en Couchbase Eventing. La Function puede leer y escribir en `orders` sin causar un bucle infinito **siempre que la escritura no dispare la misma condición de negocio** (en este caso, el campo `totalAmount` ya calculado actúa como guardia).

4. Haz clic en **Next: Add Code** y reemplaza el contenido con:

```javascript
// Function: EnrichOrderTotal
// Propósito: Calcular el totalAmount de un pedido sumando items * precio
// Demuestra: Enriquecimiento de documento con escritura de vuelta al mismo collection

function OnUpdate(doc, meta) {
    // Guardia: evitar re-procesar si el total ya fue calculado correctamente
    // Solo procesar documentos de tipo 'order' que tengan items
    if (!doc.items || doc.items.length === 0) {
        log("EnrichOrderTotal", "Skipping doc without items: " + meta.id);
        return;
    }

    // Guardia adicional: si el campo 'enriched' ya está en true, no reprocesar
    if (doc.enriched === true) {
        return;
    }

    // Calcular el total sumando qty * price de cada ítem
    var calculatedTotal = 0;
    for (var i = 0; i < doc.items.length; i++) {
        var item = doc.items[i];
        var itemTotal = (item.qty || 0) * (item.price || 0);
        calculatedTotal += itemTotal;
    }

    // Redondear a 2 decimales para evitar errores de punto flotante
    calculatedTotal = Math.round(calculatedTotal * 100) / 100;

    // Solo actualizar si el total calculado difiere del almacenado
    if (doc.totalAmount !== calculatedTotal) {
        doc.totalAmount = calculatedTotal;
        doc.enriched = true;
        doc.enrichedAt = new Date().toISOString();

        // Escribir el documento enriquecido de vuelta en la colección
        ordersCol[meta.id] = doc;

        log("EnrichOrderTotal",
            "Enriched doc " + meta.id +
            " with totalAmount: " + calculatedTotal);
    }
}

function OnDelete(meta, options) {
    // No se requiere acción al eliminar un pedido
    log("EnrichOrderTotal", "Order deleted: " + meta.id);
}
```

5. En **Settings**, configura:

   | Parámetro            | Valor   |
   |----------------------|---------|
   | Worker Count         | `1`     |
   | Log Level            | `INFO`  |
   | Checkpoint Interval  | `10000` |

6. Haz clic en **Save and Return**.

#### 3.2 — Deploy y Prueba del Enriquecimiento

**Instrucciones:**

1. Haz **Deploy** de la Function `EnrichOrderTotal` (desde *now*).
2. Espera a que el estado sea **Deployed**.
3. En la pestaña **Query**, inserta un pedido nuevo sin `totalAmount` correcto:

```sql
INSERT INTO `app-data`.workshop.orders (KEY, VALUE)
VALUES (
  "order::2001",
  {
    "type": "order",
    "orderId": "2001",
    "userId": "user::carlos.mendez",
    "status": "pending",
    "items": [
      {"productId": "prod::C3", "name": "Monitor 4K", "qty": 2, "price": 450.00},
      {"productId": "prod::D4", "name": "Teclado Mecánico", "qty": 1, "price": 89.99},
      {"productId": "prod::E5", "name": "Cable HDMI", "qty": 3, "price": 12.50}
    ],
    "totalAmount": 0,
    "createdAt": "2024-06-15T11:00:00Z"
  }
);
```

4. Espera 3-5 segundos y luego consulta el documento enriquecido:

```sql
SELECT META().id, totalAmount, enriched, enrichedAt, items
FROM `app-data`.workshop.orders
WHERE META().id = "order::2001";
```

**Cálculo esperado:**
- Monitor 4K: 2 × 450.00 = 900.00
- Teclado Mecánico: 1 × 89.99 = 89.99
- Cable HDMI: 3 × 12.50 = 37.50
- **Total: 1027.49**

**Salida esperada:**

```json
[
  {
    "id": "order::2001",
    "totalAmount": 1027.49,
    "enriched": true,
    "enrichedAt": "2024-06-15T11:00:05.123Z",
    "items": [...]
  }
]
```

**Verificación:** El campo `totalAmount` debe ser `1027.49` y `enriched` debe ser `true`.

5. Prueba que la guardia funciona — modifica un campo que no sean los items:

```sql
UPDATE `app-data`.workshop.orders
SET status = "processing"
WHERE META().id = "order::2001";
```

6. Espera 5 segundos y verifica que `totalAmount` no cambió y que los logs muestran que el documento fue omitido (la guardia `enriched === true` lo bloquea).

---

### Paso 4: Caso 3 — Function de Validación con Cuarentena

**Objetivo:** Crear una Function que valide que el campo `status` de un pedido solo contenga valores permitidos. Si el valor es inválido, el documento se mueve a la colección `quarantine`.

#### 4.1 — Crear la Function de Validación

**Instrucciones:**

1. Ve a **Eventing → + Add Function**.
2. Completa el formulario:

   | Campo                        | Valor                          |
   |------------------------------|--------------------------------|
   | Function Name                | `ValidateOrderStatus`          |
   | Source Bucket                | `app-data`                     |
   | Source Scope                 | `workshop`                     |
   | Source Collection            | `orders`                       |
   | Metadata Bucket              | `eventing-meta`                |
   | Metadata Scope               | `_default`                     |
   | Metadata Collection          | `_default`                     |

3. En **Bindings**, agrega **dos** bindings:

   **Binding 1 — Colección de cuarentena:**

   | Campo          | Valor         |
   |----------------|---------------|
   | Binding Type   | `Bucket`      |
   | Alias          | `quarantineCol` |
   | Bucket         | `app-data`    |
   | Scope          | `workshop`    |
   | Collection     | `quarantine`  |
   | Access         | `Read/Write`  |

   **Binding 2 — Colección de pedidos (para eliminar el inválido):**

   | Campo          | Valor         |
   |----------------|---------------|
   | Binding Type   | `Bucket`      |
   | Alias          | `ordersCol`   |
   | Bucket         | `app-data`    |
   | Scope          | `workshop`    |
   | Collection     | `orders`      |
   | Access         | `Read/Write`  |

4. Haz clic en **Next: Add Code** y reemplaza el contenido con:

```javascript
// Function: ValidateOrderStatus
// Propósito: Validar el campo 'status' de pedidos
//            Si el valor es inválido, mover el documento a 'quarantine'
// Demuestra: Validación de datos + movimiento entre colecciones

// Valores de status permitidos en el sistema
var VALID_STATUSES = ["pending", "processing", "completed", "cancelled"];

function OnUpdate(doc, meta) {
    // Solo validar documentos de tipo 'order'
    if (doc.type !== "order") {
        return;
    }

    // Omitir documentos que ya están marcados como cuarentenados
    if (doc.quarantined === true) {
        return;
    }

    var status = doc.status;

    // Verificar si el status es válido
    var isValid = VALID_STATUSES.indexOf(status) !== -1;

    if (!isValid) {
        log("ValidateOrderStatus",
            "INVALID STATUS detected in doc " + meta.id +
            ". Status value: '" + status + "'");

        // Enriquecer el documento con información de cuarentena
        doc.quarantined = true;
        doc.quarantineReason = "Invalid status value: '" + status +
                               "'. Allowed values: " + VALID_STATUSES.join(", ");
        doc.quarantinedAt = new Date().toISOString();
        doc.originalDocId = meta.id;

        // Escribir en la colección quarantine
        var quarantineKey = "quarantine::" + meta.id + "::" + Date.now();
        quarantineCol[quarantineKey] = doc;

        log("ValidateOrderStatus",
            "Document " + meta.id + " moved to quarantine as " + quarantineKey);
    } else {
        log("ValidateOrderStatus",
            "Document " + meta.id + " passed validation. Status: " + status);
    }
}

function OnDelete(meta, options) {
    // Limpiar posibles registros de cuarentena asociados
    // En producción, aquí se podría hacer limpieza adicional
    log("ValidateOrderStatus", "Order deleted: " + meta.id);
}
```

5. En **Settings**, configura:

   | Parámetro            | Valor     |
   |----------------------|-----------|
   | Worker Count         | `1`       |
   | Log Level            | `INFO`    |
   | Checkpoint Interval  | `10000`   |

6. Haz clic en **Save and Return**.

#### 4.2 — Deploy y Prueba de la Validación

**Instrucciones:**

1. Haz **Deploy** de `ValidateOrderStatus` (desde *now*).
2. Espera a que el estado sea **Deployed**.
3. Inserta un documento con un `status` **válido** para confirmar que pasa la validación:

```sql
INSERT INTO `app-data`.workshop.orders (KEY, VALUE)
VALUES (
  "order::3001",
  {
    "type": "order",
    "orderId": "3001",
    "userId": "user::maria.lopez",
    "status": "pending",
    "items": [{"productId": "prod::F6", "name": "Webcam", "qty": 1, "price": 75.00}],
    "totalAmount": 0,
    "createdAt": "2024-06-15T12:00:00Z"
  }
);
```

4. Verifica que **no** se creó ningún registro en quarantine para este documento:

```sql
SELECT COUNT(*) AS inQuarantine
FROM `app-data`.workshop.quarantine
WHERE originalDocId = "order::3001";
```

El resultado debe ser `0`.

5. Ahora inserta un documento con un `status` **inválido**:

```sql
INSERT INTO `app-data`.workshop.orders (KEY, VALUE)
VALUES (
  "order::3002",
  {
    "type": "order",
    "orderId": "3002",
    "userId": "user::pedro.ramirez",
    "status": "shipped",
    "items": [{"productId": "prod::G7", "name": "SSD 1TB", "qty": 1, "price": 120.00}],
    "totalAmount": 0,
    "createdAt": "2024-06-15T12:05:00Z"
  }
);
```

6. Espera 5 segundos y verifica la colección de cuarentena:

```sql
SELECT META().id AS quarantineId,
       originalDocId,
       quarantineReason,
       quarantinedAt,
       status
FROM `app-data`.workshop.quarantine
WHERE originalDocId = "order::3002";
```

**Salida esperada:**

```json
[
  {
    "quarantineId": "quarantine::order::3002::1718448305000",
    "originalDocId": "order::3002",
    "quarantineReason": "Invalid status value: 'shipped'. Allowed values: pending, processing, completed, cancelled",
    "quarantinedAt": "2024-06-15T12:05:05.000Z",
    "status": "shipped"
  }
]
```

7. Prueba con otro valor inválido para confirmar la robustez:

```sql
INSERT INTO `app-data`.workshop.orders (KEY, VALUE)
VALUES (
  "order::3003",
  {
    "type": "order",
    "orderId": "3003",
    "userId": "user::test",
    "status": "INVALID_STATUS",
    "items": [],
    "totalAmount": 0,
    "createdAt": "2024-06-15T12:10:00Z"
  }
);
```

**Verificación final de cuarentena:**

```sql
SELECT COUNT(*) AS totalQuarantined
FROM `app-data`.workshop.quarantine
WHERE quarantined = true;
```

El resultado debe ser **≥ 2**.

---

### Paso 5: Explorar las Estadísticas de las Functions

**Objetivo:** Observar las métricas de ejecución de las Functions desplegadas.

**Instrucciones:**

1. En la Web Console, ve a **Eventing**.
2. Para cada Function desplegada, observa las estadísticas en la fila:
   - **Processed:** número de documentos procesados exitosamente
   - **Failed:** número de errores de ejecución
   - **Backlog:** documentos pendientes de procesar
3. Haz clic en el ícono de **estadísticas** (gráfico de barras) de `AuditOrderChanges`.
4. Observa los gráficos de:
   - *Mutations Remaining* (backlog)
   - *Processed* (eventos completados)
   - *Failures* (errores)
5. Ejecuta algunas inserciones adicionales para ver cómo aumentan los contadores:

```sql
INSERT INTO `app-data`.workshop.orders (KEY, VALUE)
VALUES ("order::4001", {"type":"order","userId":"user::bulk1","status":"pending","items":[{"productId":"p1","name":"Item A","qty":2,"price":50.00}],"totalAmount":0});

INSERT INTO `app-data`.workshop.orders (KEY, VALUE)
VALUES ("order::4002", {"type":"order","userId":"user::bulk2","status":"completed","items":[{"productId":"p2","name":"Item B","qty":1,"price":200.00}],"totalAmount":0});
```

6. Regresa a las estadísticas y observa el incremento en *Processed*.

**Verificación:** El contador *Processed* de `AuditOrderChanges` debe reflejar todas las operaciones realizadas durante el laboratorio.

---

## Validación y Pruebas Finales

Ejecuta las siguientes consultas de validación para confirmar que las tres Functions funcionaron correctamente:

**Resumen de auditoría generada:**

```sql
SELECT operation,
       COUNT(*) AS total,
       MIN(timestamp) AS firstEvent,
       MAX(timestamp) AS lastEvent
FROM `app-data`.workshop.`audit-log`
WHERE type = "audit"
GROUP BY operation
ORDER BY operation;
```

**Pedidos enriquecidos correctamente:**

```sql
SELECT META().id AS orderId,
       totalAmount,
       enriched,
       enrichedAt
FROM `app-data`.workshop.orders
WHERE enriched = true
ORDER BY enrichedAt;
```

**Documentos en cuarentena:**

```sql
SELECT META().id AS quarantineId,
       originalDocId,
       status AS invalidStatus,
       quarantineReason
FROM `app-data`.workshop.quarantine
WHERE quarantined = true;
```

**Verificación del estado de todas las Functions:**

```bash
curl -s -u Administrator:password \
  http://localhost:8096/api/v1/functions \
  | python3 -m json.tool | grep -E '"name"|"deployment_status"'
```

Las tres Functions deben aparecer con `"deployment_status": "deployed"`.

---

## Solución de Problemas

### Problema 1: La Function se despliega pero no procesa documentos (backlog siempre en 0, processed en 0)

**Síntomas:**
- La Function muestra estado **Deployed** pero el contador *Processed* permanece en 0 aunque se insertan documentos.
- No aparecen mensajes en los logs de la Function.
- Las colecciones destino (`audit-log`, `quarantine`) permanecen vacías.

**Causa probable:**
El servicio Eventing no tiene conectividad con la colección fuente, o los bindings están mal configurados (nombre de scope/collection incorrecto, o el bucket de metadata no existe). También puede ocurrir si la Function fue desplegada con la opción *From beginning* pero los documentos se insertaron antes de que el deploy completara.

**Solución:**
1. Ve a **Eventing → tu Function → Edit** y verifica que el Source Collection sea exactamente `orders` (no `Orders` ni `order`), el Source Scope sea `workshop` y el Source Bucket sea `app-data`.
2. Verifica que el bucket `eventing-meta` existe: `curl -s -u Administrator:password http://localhost:8091/pools/default/buckets/eventing-meta | python3 -m json.tool`
3. Haz **Undeploy** de la Function, luego **Deploy** nuevamente seleccionando *From beginning* para que procese todos los documentos existentes.
4. Si el problema persiste, verifica en **Servers** que el servicio Eventing está marcado como activo en el nodo.

---

### Problema 2: La Function de Enriquecimiento causa un bucle infinito (CPU al 100%, processed crece sin parar)

**Síntomas:**
- El contador *Processed* de `EnrichOrderTotal` crece continuamente sin que se inserten documentos nuevos.
- La CPU del nodo Couchbase se dispara al 100%.
- Los logs muestran el mismo `meta.id` procesándose repetidamente.

**Causa probable:**
La guardia `if (doc.enriched === true) return;` fue eliminada o tiene un error tipográfico. Sin esta guardia, cada vez que la Function escribe `ordersCol[meta.id] = doc`, genera un nuevo evento `OnUpdate` que vuelve a disparar la misma Function, creando un bucle infinito. Esto es el equivalente en Eventing al problema de la **doble escritura** que estudiamos en la lección teórica.

**Solución:**
1. **Inmediatamente:** haz **Pause** de la Function `EnrichOrderTotal` desde la Web Console para detener el bucle.
2. Haz clic en **Edit** y verifica que el código contiene exactamente:
   ```javascript
   if (doc.enriched === true) {
       return;
   }
   ```
   en las primeras líneas del `OnUpdate`, antes de cualquier escritura.
3. Verifica también que la línea `doc.enriched = true;` está presente justo antes de `ordersCol[meta.id] = doc;`.
4. Guarda los cambios y haz **Resume** (o Undeploy + Deploy).
5. Si el bucket de metadata acumuló demasiados checkpoints corruptos, puede ser necesario hacer Undeploy, esperar 30 segundos y hacer Deploy *From now*.

---

## Limpieza del Entorno

Una vez completado el laboratorio, realiza el undeploy de todas las Functions y, opcionalmente, elimina los recursos creados.

**Paso 1 — Undeploy de todas las Functions:**

```bash
# Undeploy AuditOrderChanges
curl -s -u Administrator:password \
  -X POST \
  "http://localhost:8096/api/v1/functions/AuditOrderChanges/undeploy"

# Undeploy EnrichOrderTotal
curl -s -u Administrator:password \
  -X POST \
  "http://localhost:8096/api/v1/functions/EnrichOrderTotal/undeploy"

# Undeploy ValidateOrderStatus
curl -s -u Administrator:password \
  -X POST \
  "http://localhost:8096/api/v1/functions/ValidateOrderStatus/undeploy"
```

Espera 15 segundos para que el undeploy complete, luego verifica:

```bash
curl -s -u Administrator:password \
  http://localhost:8096/api/v1/functions \
  | python3 -m json.tool | grep -E '"name"|"deployment_status"'
```

Todas deben mostrar `"deployment_status": "undeployed"`.

**Paso 2 — (Opcional) Eliminar las Functions:**

Si deseas limpiar completamente el entorno para el siguiente laboratorio:

```bash
curl -s -u Administrator:password \
  -X DELETE "http://localhost:8096/api/v1/functions/AuditOrderChanges"

curl -s -u Administrator:password \
  -X DELETE "http://localhost:8096/api/v1/functions/EnrichOrderTotal"

curl -s -u Administrator:password \
  -X DELETE "http://localhost:8096/api/v1/functions/ValidateOrderStatus"
```

**Paso 3 — (Opcional) Eliminar los buckets creados:**

> ⚠️ **Solo si no continuarás con laboratorios que dependan de estos datos.**

```bash
curl -s -u Administrator:password \
  -X DELETE http://localhost:8091/pools/default/buckets/app-data

curl -s -u Administrator:password \
  -X DELETE http://localhost:8091/pools/default/buckets/eventing-meta
```

---

## Resumen

En este laboratorio implementaste los tres patrones fundamentales del servicio Couchbase Eventing, resolviendo directamente las limitaciones de las formas tradicionales de escuchar eventos que estudiaste en la lección teórica:

| Patrón Implementado     | Function Creada          | Limitación Tradicional que Resuelve                                    |
|-------------------------|--------------------------|------------------------------------------------------------------------|
| **Auditoría de cambios**| `AuditOrderChanges`      | Elimina la necesidad de polling para detectar cambios                  |
| **Enriquecimiento**     | `EnrichOrderTotal`       | Reemplaza triggers acoplados al motor de BD con lógica JavaScript portable |
| **Validación/Cuarentena**| `ValidateOrderStatus`   | Evita la doble escritura de las colas externas: la lógica vive en la BD |

**Conceptos clave consolidados:**

- **`OnUpdate`** se dispara ante cualquier escritura (INSERT o UPDATE) en la colección fuente — es el handler más utilizado.
- **`OnDelete`** se dispara ante eliminaciones y expiraciones (TTL) — útil para limpieza y auditoría de bajas.
- Los **bindings** son la forma segura de referenciar otras colecciones desde el código JavaScript sin hardcodear credenciales.
- Las **guardias** (campos como `enriched: true`) son esenciales para evitar bucles infinitos cuando una Function escribe en su propia colección fuente.
- El **bucket de metadata** almacena los checkpoints de progreso de Eventing — debe ser un bucket dedicado, separado del bucket de datos.
- El ciclo **Deploy → Prueba → Logs → Undeploy** es el flujo operativo estándar para gestionar Functions en producción.

### Recursos Adicionales

- [Couchbase Eventing Service — Documentación oficial](https://docs.couchbase.com/server/current/eventing/eventing-overview.html)
- [Eventing Function Settings Reference](https://docs.couchbase.com/server/current/eventing/eventing-adding-function.html)
- [JavaScript API para Eventing (handlers, bindings, N1QL)](https://docs.couchbase.com/server/current/eventing/eventing-api.html)
- [Patrones de Eventing: ejemplos de la comunidad Couchbase](https://github.com/couchbase/eventing/tree/master/samples)
- [Eventing Best Practices — Couchbase Blog](https://www.couchbase.com/blog/couchbase-eventing-best-practices/)

---
LAB_END---
