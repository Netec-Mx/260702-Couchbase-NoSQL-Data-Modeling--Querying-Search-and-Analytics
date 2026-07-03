# Diseño de modelo de datos para aplicación distribuida

## Metadatos

| Campo | Detalle |
|---|---|
| **Duración estimada** | 60 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Crear (Create) |
| **Versión Couchbase** | 7.6.x (Community Edition o Enterprise Trial) |
| **Dataset requerido** | Ninguno (se crea desde cero en este laboratorio) |

---

## Descripción General

En este laboratorio aplicarás el proceso formal de modelado de datos —conceptual, lógico y físico— para diseñar el modelo de una aplicación de e-commerce simplificada en Couchbase. Partirás de los requerimientos de negocio y los patrones de acceso para tomar decisiones fundamentadas sobre embedding vs. referencing, desnormalización y estructura de keys. Al finalizar, habrás creado la estructura de buckets, scopes y collections en Couchbase, insertado documentos de prueba con SQL++ y verificado que los patrones de acceso más críticos funcionan correctamente.

---

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Aplicar el proceso de modelado formal (conceptual → lógico → físico) en el contexto de una base de datos documental distribuida
- [ ] Identificar y justificar las diferencias clave entre el modelado relacional y el modelado orientado a documentos en Couchbase
- [ ] Diseñar el modelo de datos de una aplicación de e-commerce considerando patrones de acceso, embedding vs. referencing y esquema de keys
- [ ] Evaluar trade-offs de diseño: desnormalización controlada vs. consistencia de datos, y su impacto en el rendimiento de lectura/escritura
- [ ] Crear la estructura física en Couchbase (bucket, scopes, collections) e insertar documentos de prueba verificando los patrones de acceso con SQL++

---

## Prerrequisitos

### Conocimiento previo

- Haber completado el **Lab 03-00-01** (SQL++ básico: SELECT, INSERT, UPDATE)
- Haber completado el **Lab 04-00-01** (Índices en Couchbase: GSI, creación y uso)
- Comprensión de diagramas Entidad-Relación (ER) y modelado relacional básico
- Familiaridad con estructuras JSON anidadas (objetos y arrays)
- Lectura de la lección **7.1: Explicar Data Modeling en Sistemas Distribuidos**

### Acceso requerido

- Couchbase Server 7.6.x en ejecución (nodo único / single-node cluster)
- Acceso a la Web Console: `http://localhost:8091`
- Acceso a `cbq` (Couchbase Query Shell) o al editor de consultas en la Web Console
- Credenciales de administrador (por defecto: `Administrator` / `password`)
- Editor de texto para registrar decisiones de diseño (VS Code recomendado)

---

## Entorno de Laboratorio

### Requisitos de Hardware

| Recurso | Mínimo | Recomendado |
|---|---|---|
| RAM | 8 GB | 16 GB |
| CPU | 4 núcleos x86_64 | 8 núcleos |
| Almacenamiento | 20 GB libres (SSD) | 50 GB libres (SSD) |
| Resolución de pantalla | 1280×768 | 1280×800 o superior |

### Requisitos de Software

| Software | Versión | Uso en este lab |
|---|---|---|
| Couchbase Server | 7.6.x | Base de datos principal |
| Navegador Web | Chrome/Firefox/Edge 110+ | Web Console |
| cbq | Incluido con Server 7.6.x | Ejecución de SQL++ |
| curl | 7.x o superior | Verificación REST (opcional) |
| VS Code (o editor) | 1.80+ | Documentar decisiones de diseño |

### Verificación del Entorno

Antes de comenzar, confirma que Couchbase está en ejecución y accesible:

```bash
# Verificar que el servicio responde
curl -s -u Administrator:password http://localhost:8091/pools/default | python3 -m json.tool | grep "clusterName"

# Abrir cbq (desde el directorio de instalación de Couchbase)
# En Linux/Mac:
/opt/couchbase/bin/cbq -u Administrator -p password -engine=http://localhost:8093/

# En Windows (PowerShell):
# "C:\Program Files\Couchbase\Server\bin\cbq.exe" -u Administrator -p password -engine=http://localhost:8093/
```

**Salida esperada de curl:**
```json
"clusterName": "My Cluster"
```

---

## Pasos del Laboratorio

---

### Parte 1 — Modelado Conceptual: Diagrama Entidad-Relación

**Objetivo:** Identificar las entidades, atributos y relaciones del sistema de e-commerce antes de tomar cualquier decisión de implementación.

#### Contexto de la Aplicación

Trabajarás con un sistema de gestión de pedidos de e-commerce simplificado. Los requerimientos de negocio son:

- Los **clientes** pueden realizar múltiples **pedidos**
- Cada **pedido** contiene uno o más **ítems de pedido** (OrderItem)
- Cada ítem referencia un **producto**
- Los **productos** pertenecen a una o más **categorías**
- Los clientes pueden escribir **reseñas** sobre productos

#### Instrucciones

1. En tu editor de texto (o en papel), crea un archivo llamado `ecommerce-data-model.md`. Este será tu documento de diseño durante todo el laboratorio.

2. Registra el siguiente diagrama ER en notación textual (o dibújalo en papel/herramienta de diagramas):

```
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│  Customer   │ 1──N  │    Order    │ 1──N  │  OrderItem  │
│─────────────│       │─────────────│       │─────────────│
│ customer_id │       │ order_id    │       │ item_id     │
│ name        │       │ customer_id │       │ order_id    │
│ email       │       │ status      │       │ product_id  │
│ phone       │       │ total       │       │ quantity    │
│ created_at  │       │ created_at  │       │ unit_price  │
└─────────────┘       │ ship_addr   │       └──────┬──────┘
                      └─────────────┘              │ N
                                                   │
                      ┌─────────────┐       ┌──────┴──────┐
                      │  Category   │ N──N  │   Product   │
                      │─────────────│       │─────────────│
                      │ category_id │       │ product_id  │
                      │ name        │       │ name        │
                      │ slug        │       │ description │
                      └─────────────┘       │ price       │
                                            │ stock       │
                      ┌─────────────┐       └──────┬──────┘
                      │   Review    │ N──1          │ 1
                      │─────────────│               │
                      │ review_id   │───────────────┘
                      │ product_id  │ (Customer escribe Review)
                      │ customer_id │
                      │ rating      │
                      │ body        │
                      │ created_at  │
                      └─────────────┘
```

3. En tu documento de diseño, registra los **5 patrones de acceso más frecuentes** de la aplicación. Estos guiarán todas las decisiones de modelado posteriores:

```markdown
## Patrones de Acceso Identificados

| # | Patrón | Frecuencia | Tipo |
|---|--------|-----------|------|
| PA-1 | Obtener detalle completo de un pedido (ítems + datos cliente + dirección) | Muy alta | Lectura |
| PA-2 | Listar todos los pedidos de un cliente (resumen: id, fecha, total, status) | Alta | Lectura |
| PA-3 | Buscar productos por categoría con precio y stock | Alta | Lectura |
| PA-4 | Actualizar el estado de un pedido (pending → shipped → delivered) | Media | Escritura |
| PA-5 | Obtener reseñas de un producto con rating promedio | Media | Lectura |
```

**Resultado esperado:** Un documento `ecommerce-data-model.md` con el diagrama ER y los patrones de acceso documentados.

**Verificación:** Confirma que has identificado las 6 entidades (Customer, Product, Order, OrderItem, Category, Review) y las 5 relaciones entre ellas antes de continuar.

---

### Parte 2 — Modelado Lógico: Diseño de Documentos JSON

**Objetivo:** Transformar el modelo ER en documentos JSON, tomando decisiones explícitas y justificadas sobre qué relaciones se embeben y cuáles se referencian.

#### Instrucciones

1. Para cada entidad, analiza los patrones de acceso y decide: **¿embedding o referencing?** Registra tu decisión y justificación en `ecommerce-data-model.md`:

```markdown
## Decisiones de Modelado Lógico

### Decisión 1: OrderItem dentro de Order (EMBEDDING)
- **Decisión:** Embeber los ítems del pedido dentro del documento Order
- **Justificación:** PA-1 requiere leer todo el pedido de una vez. Los ítems no tienen
  existencia independiente fuera del pedido. Un pedido promedio tiene 1-10 ítems
  (tamaño manejable). Los ítems no son consultados de forma individual.
- **Trade-off aceptado:** Si necesitamos estadísticas globales de productos vendidos,
  debemos escanear todos los pedidos (costo de escritura vs. ganancia en lectura).

### Decisión 2: Datos de cliente en Order (EMBEDDING PARCIAL)
- **Decisión:** Embeber snapshot de nombre y email del cliente en el pedido
- **Justificación:** PA-1 requiere mostrar datos del cliente en el detalle del pedido.
  Si el cliente cambia su email, los pedidos históricos deben conservar el email
  original con el que se realizaron (requisito de auditoría).
- **Trade-off aceptado:** Duplicación de datos. Cambios en el nombre del cliente
  no se propagan automáticamente a pedidos históricos (comportamiento deseado).

### Decisión 3: Dirección de envío en Order (EMBEDDING)
- **Decisión:** Embeber la dirección de envío completa en el pedido
- **Justificación:** La dirección de envío es un snapshot del momento del pedido.
  Si el cliente cambia su dirección, los pedidos anteriores no deben verse afectados.

### Decisión 4: Product → Category (REFERENCING con embedding de nombre)
- **Decisión:** El producto almacena category_ids (array de referencias) más el
  nombre de la categoría para display
- **Justificación:** PA-3 requiere filtrar por categoría eficientemente. Las categorías
  son entidades estables con pocos cambios. Embeber el array de category_ids permite
  crear índices de array para búsquedas rápidas.
- **Trade-off aceptado:** Si el nombre de una categoría cambia, hay que actualizar
  todos los productos de esa categoría.

### Decisión 5: Review como documento separado (REFERENCING)
- **Decisión:** Las reseñas son documentos independientes que referencian product_id
  y customer_id
- **Justificación:** PA-5 requiere calcular rating promedio y listar reseñas paginadas.
  Un producto popular puede tener cientos de reseñas (el documento crecería sin límite
  si se embebieran). Las reseñas tienen su propio ciclo de vida (moderación, edición).
- **Trade-off aceptado:** PA-5 requiere una consulta separada (no un solo GET por key).
```

2. Define los esquemas JSON finales para cada tipo de documento:

**Documento: Customer**
```json
{
  "type": "customer",
  "customer_id": "CUST-0001",
  "name": "María González",
  "email": "maria.gonzalez@ejemplo.com",
  "phone": "+52-55-1234-5678",
  "addresses": [
    {
      "label": "casa",
      "street": "Av. Insurgentes Sur 1500",
      "city": "Ciudad de México",
      "state": "CDMX",
      "zip": "03810",
      "country": "MX"
    }
  ],
  "created_at": "2024-01-15T09:00:00Z",
  "status": "active"
}
```

**Documento: Category**
```json
{
  "type": "category",
  "category_id": "CAT-001",
  "name": "Electrónica",
  "slug": "electronica",
  "parent_id": null,
  "description": "Dispositivos electrónicos y accesorios"
}
```

**Documento: Product**
```json
{
  "type": "product",
  "product_id": "PROD-001",
  "sku": "TEC-MEC-001",
  "name": "Teclado Mecánico RGB",
  "description": "Teclado mecánico con switches Cherry MX Blue y retroiluminación RGB",
  "price": 1200.00,
  "currency": "MXN",
  "stock": 45,
  "category_ids": ["CAT-001", "CAT-003"],
  "category_names": ["Electrónica", "Periféricos"],
  "images": ["img/tec-mec-001-front.jpg"],
  "attributes": {
    "brand": "KeyMaster",
    "switch_type": "Cherry MX Blue",
    "connectivity": "USB-C"
  },
  "created_at": "2024-01-10T08:00:00Z",
  "active": true
}
```

**Documento: Order (con embedding de ítems y snapshot de cliente)**
```json
{
  "type": "order",
  "order_id": "ORD-20240315-8821",
  "customer_id": "CUST-0001",
  "customer_snapshot": {
    "name": "María González",
    "email": "maria.gonzalez@ejemplo.com"
  },
  "shipping_address": {
    "street": "Av. Insurgentes Sur 1500",
    "city": "Ciudad de México",
    "state": "CDMX",
    "zip": "03810",
    "country": "MX"
  },
  "items": [
    {
      "product_id": "PROD-001",
      "sku": "TEC-MEC-001",
      "name": "Teclado Mecánico RGB",
      "quantity": 1,
      "unit_price": 1200.00,
      "subtotal": 1200.00
    },
    {
      "product_id": "PROD-002",
      "sku": "MOU-INL-002",
      "name": "Mouse Inalámbrico Ergonómico",
      "quantity": 2,
      "unit_price": 350.00,
      "subtotal": 700.00
    }
  ],
  "subtotal": 1900.00,
  "tax": 304.00,
  "total": 2204.00,
  "currency": "MXN",
  "status": "shipped",
  "payment_method": "credit_card",
  "created_at": "2024-03-15T10:22:00Z",
  "updated_at": "2024-03-15T14:35:00Z"
}
```

**Documento: Review**
```json
{
  "type": "review",
  "review_id": "REV-001",
  "product_id": "PROD-001",
  "customer_id": "CUST-0001",
  "customer_name": "María González",
  "rating": 5,
  "title": "Excelente teclado",
  "body": "Los switches son muy precisos y la retroiluminación es hermosa. Muy recomendado.",
  "verified_purchase": true,
  "created_at": "2024-03-20T16:00:00Z",
  "status": "approved"
}
```

**Resultado esperado:** Tu documento `ecommerce-data-model.md` contiene 5 decisiones de modelado justificadas y los 5 esquemas JSON definidos.

**Verificación:** Cada decisión de embedding/referencing debe estar vinculada a al menos uno de los patrones de acceso identificados en la Parte 1.

---

### Parte 3 — Modelado Físico: Estructura en Couchbase y Esquema de Keys

**Objetivo:** Definir la estructura física de buckets, scopes, collections, el esquema de keys para cada tipo de documento y los índices necesarios para los patrones de acceso.

#### Instrucciones

1. Registra la estructura física en tu documento de diseño:

```markdown
## Modelo Físico

### Estructura de Bucket / Scope / Collection

Bucket: ecommerce
└── Scope: store
    ├── Collection: customers
    ├── Collection: products
    ├── Collection: orders
    ├── Collection: categories
    └── Collection: reviews

### Justificación del diseño de Scope
Se usa un único scope "store" porque todas las colecciones pertenecen
al mismo dominio de negocio y con frecuencia participan en consultas JOIN.
Un scope por dominio facilita la gestión de permisos y el aislamiento lógico.
```

2. Define el esquema de keys para cada collection:

```markdown
### Esquema de Document Keys

| Collection | Patrón de Key | Ejemplo | Justificación |
|---|---|---|---|
| customers | `cust::{customer_id}` | `cust::CUST-0001` | Prefijo evita colisiones; customer_id legible facilita debugging |
| products | `prod::{product_id}` | `prod::PROD-001` | Acceso directo por ID de producto |
| orders | `ord::{order_id}` | `ord::ORD-20240315-8821` | Fecha en ID permite ordenamiento cronológico natural |
| categories | `cat::{category_id}` | `cat::CAT-001` | Catálogo estable; keys simples y legibles |
| reviews | `rev::{review_id}` | `rev::REV-001` | Independiente del producto para evitar keys muy largas |
```

3. Define los índices necesarios para los 5 patrones de acceso:

```markdown
### Índices Requeridos por Patrón de Acceso

| Patrón | Collection | Índice | Campos |
|---|---|---|---|
| PA-1 (detalle pedido) | orders | Acceso por key directa | N/A (GET por key) |
| PA-2 (pedidos por cliente) | orders | idx_orders_customer | customer_id, created_at DESC |
| PA-3 (productos por categoría) | products | idx_products_category | category_ids (ARRAY), price, stock |
| PA-4 (update status pedido) | orders | Acceso por key directa | N/A (UPDATE por key) |
| PA-5 (reseñas por producto) | reviews | idx_reviews_product | product_id, rating, created_at |
```

**Resultado esperado:** El modelo físico completo documentado con bucket/scope/collection, esquema de keys e índices.

**Verificación:** Cada patrón de acceso debe tener un índice correspondiente o justificación de por qué el acceso por key directa es suficiente.

---

### Parte 4 — Validación: Crear la Estructura en Couchbase

**Objetivo:** Materializar el modelo físico diseñado creando el bucket, scope y collections en Couchbase, e insertar documentos de prueba.

#### Paso 4.1 — Crear el Bucket `ecommerce`

**Instrucciones:**

1. Abre la Web Console en `http://localhost:8091` e inicia sesión con tus credenciales de administrador.

2. Navega a **Buckets** → **Add Bucket** y configura:
   - **Name:** `ecommerce`
   - **Memory Quota:** `256 MB` (suficiente para este laboratorio)
   - **Bucket Type:** Couchbase
   - Deja los demás valores por defecto

3. Haz clic en **Add Bucket**.

4. Alternativamente, usa la REST API:

```bash
curl -s -X POST http://localhost:8091/pools/default/buckets \
  -u Administrator:password \
  -d "name=ecommerce" \
  -d "bucketType=couchbase" \
  -d "ramQuota=256" \
  -d "replicaNumber=0"
```

**Salida esperada (REST API):** Sin output (HTTP 202 Accepted). Verifica con:

```bash
curl -s -u Administrator:password http://localhost:8091/pools/default/buckets/ecommerce | python3 -m json.tool | grep '"name"'
```

Debe mostrar: `"name": "ecommerce"`

#### Paso 4.2 — Crear Scope y Collections

**Instrucciones:**

1. En la Web Console, navega a **Buckets** → `ecommerce` → **Scopes & Collections** → **Add Scope**.
   - **Scope Name:** `store`

2. Dentro del scope `store`, crea las 5 collections haciendo clic en **Add Collection** para cada una:
   - `customers`
   - `products`
   - `orders`
   - `categories`
   - `reviews`

3. Alternativamente, usa SQL++ desde `cbq` o el Query Editor de la Web Console:

```sql
-- Crear scope
CREATE SCOPE ecommerce.store;

-- Crear collections
CREATE COLLECTION ecommerce.store.customers;
CREATE COLLECTION ecommerce.store.products;
CREATE COLLECTION ecommerce.store.orders;
CREATE COLLECTION ecommerce.store.categories;
CREATE COLLECTION ecommerce.store.reviews;
```

**Salida esperada:**
```
{
    "requestID": "...",
    "status": "success",
    ...
}
```

**Verificación:**
```sql
-- Verificar que las collections existen
SELECT RAW name FROM system:keyspaces
WHERE `bucket` = "ecommerce" AND `scope` = "store"
ORDER BY name;
```

Debe retornar: `["categories", "customers", "orders", "products", "reviews"]`

#### Paso 4.3 — Insertar Documentos de Prueba

**Instrucciones:**

Ejecuta los siguientes INSERT en el Query Editor de la Web Console o en `cbq`. Inserta los documentos en el orden indicado:

```sql
-- =============================================
-- CATEGORÍAS
-- =============================================
INSERT INTO ecommerce.store.categories (KEY, VALUE) VALUES
("cat::CAT-001", {
  "type": "category",
  "category_id": "CAT-001",
  "name": "Electrónica",
  "slug": "electronica",
  "parent_id": null,
  "description": "Dispositivos electrónicos y accesorios"
});

INSERT INTO ecommerce.store.categories (KEY, VALUE) VALUES
("cat::CAT-003", {
  "type": "category",
  "category_id": "CAT-003",
  "name": "Periféricos",
  "slug": "perifericos",
  "parent_id": "CAT-001",
  "description": "Teclados, ratones y otros periféricos"
});
```

```sql
-- =============================================
-- PRODUCTOS
-- =============================================
INSERT INTO ecommerce.store.products (KEY, VALUE) VALUES
("prod::PROD-001", {
  "type": "product",
  "product_id": "PROD-001",
  "sku": "TEC-MEC-001",
  "name": "Teclado Mecánico RGB",
  "description": "Teclado mecánico con switches Cherry MX Blue y retroiluminación RGB",
  "price": 1200.00,
  "currency": "MXN",
  "stock": 45,
  "category_ids": ["CAT-001", "CAT-003"],
  "category_names": ["Electrónica", "Periféricos"],
  "attributes": {
    "brand": "KeyMaster",
    "switch_type": "Cherry MX Blue",
    "connectivity": "USB-C"
  },
  "created_at": "2024-01-10T08:00:00Z",
  "active": true
});

INSERT INTO ecommerce.store.products (KEY, VALUE) VALUES
("prod::PROD-002", {
  "type": "product",
  "product_id": "PROD-002",
  "sku": "MOU-INL-002",
  "name": "Mouse Inalámbrico Ergonómico",
  "description": "Mouse inalámbrico con diseño ergonómico y 6 botones programables",
  "price": 350.00,
  "currency": "MXN",
  "stock": 120,
  "category_ids": ["CAT-001", "CAT-003"],
  "category_names": ["Electrónica", "Periféricos"],
  "attributes": {
    "brand": "ErgoTech",
    "dpi": "800-3200",
    "connectivity": "USB-A receiver"
  },
  "created_at": "2024-01-12T09:00:00Z",
  "active": true
});

INSERT INTO ecommerce.store.products (KEY, VALUE) VALUES
("prod::PROD-003", {
  "type": "product",
  "product_id": "PROD-003",
  "sku": "MON-4K-003",
  "name": "Monitor 4K 27 pulgadas",
  "description": "Monitor UHD 4K con panel IPS y 144Hz de refresco",
  "price": 8500.00,
  "currency": "MXN",
  "stock": 15,
  "category_ids": ["CAT-001"],
  "category_names": ["Electrónica"],
  "attributes": {
    "brand": "VisionPro",
    "resolution": "3840x2160",
    "refresh_rate": "144Hz",
    "panel": "IPS"
  },
  "created_at": "2024-02-01T10:00:00Z",
  "active": true
});
```

```sql
-- =============================================
-- CLIENTES
-- =============================================
INSERT INTO ecommerce.store.customers (KEY, VALUE) VALUES
("cust::CUST-0001", {
  "type": "customer",
  "customer_id": "CUST-0001",
  "name": "María González",
  "email": "maria.gonzalez@ejemplo.com",
  "phone": "+52-55-1234-5678",
  "addresses": [
    {
      "label": "casa",
      "street": "Av. Insurgentes Sur 1500",
      "city": "Ciudad de México",
      "state": "CDMX",
      "zip": "03810",
      "country": "MX"
    }
  ],
  "created_at": "2024-01-15T09:00:00Z",
  "status": "active"
});

INSERT INTO ecommerce.store.customers (KEY, VALUE) VALUES
("cust::CUST-0002", {
  "type": "customer",
  "customer_id": "CUST-0002",
  "name": "Carlos Ramírez",
  "email": "carlos.ramirez@ejemplo.com",
  "phone": "+52-33-9876-5432",
  "addresses": [
    {
      "label": "oficina",
      "street": "Calle Morelos 250",
      "city": "Guadalajara",
      "state": "JAL",
      "zip": "44100",
      "country": "MX"
    }
  ],
  "created_at": "2024-02-10T11:00:00Z",
  "status": "active"
});
```

```sql
-- =============================================
-- PEDIDOS (con embedding de ítems y snapshot)
-- =============================================
INSERT INTO ecommerce.store.orders (KEY, VALUE) VALUES
("ord::ORD-20240315-8821", {
  "type": "order",
  "order_id": "ORD-20240315-8821",
  "customer_id": "CUST-0001",
  "customer_snapshot": {
    "name": "María González",
    "email": "maria.gonzalez@ejemplo.com"
  },
  "shipping_address": {
    "street": "Av. Insurgentes Sur 1500",
    "city": "Ciudad de México",
    "state": "CDMX",
    "zip": "03810",
    "country": "MX"
  },
  "items": [
    {
      "product_id": "PROD-001",
      "sku": "TEC-MEC-001",
      "name": "Teclado Mecánico RGB",
      "quantity": 1,
      "unit_price": 1200.00,
      "subtotal": 1200.00
    },
    {
      "product_id": "PROD-002",
      "sku": "MOU-INL-002",
      "name": "Mouse Inalámbrico Ergonómico",
      "quantity": 2,
      "unit_price": 350.00,
      "subtotal": 700.00
    }
  ],
  "subtotal": 1900.00,
  "tax": 304.00,
  "total": 2204.00,
  "currency": "MXN",
  "status": "shipped",
  "payment_method": "credit_card",
  "created_at": "2024-03-15T10:22:00Z",
  "updated_at": "2024-03-15T14:35:00Z"
});

INSERT INTO ecommerce.store.orders (KEY, VALUE) VALUES
("ord::ORD-20240320-9102", {
  "type": "order",
  "order_id": "ORD-20240320-9102",
  "customer_id": "CUST-0001",
  "customer_snapshot": {
    "name": "María González",
    "email": "maria.gonzalez@ejemplo.com"
  },
  "shipping_address": {
    "street": "Av. Insurgentes Sur 1500",
    "city": "Ciudad de México",
    "state": "CDMX",
    "zip": "03810",
    "country": "MX"
  },
  "items": [
    {
      "product_id": "PROD-003",
      "sku": "MON-4K-003",
      "name": "Monitor 4K 27 pulgadas",
      "quantity": 1,
      "unit_price": 8500.00,
      "subtotal": 8500.00
    }
  ],
  "subtotal": 8500.00,
  "tax": 1360.00,
  "total": 9860.00,
  "currency": "MXN",
  "status": "pending",
  "payment_method": "bank_transfer",
  "created_at": "2024-03-20T15:10:00Z",
  "updated_at": "2024-03-20T15:10:00Z"
});

INSERT INTO ecommerce.store.orders (KEY, VALUE) VALUES
("ord::ORD-20240318-8950", {
  "type": "order",
  "order_id": "ORD-20240318-8950",
  "customer_id": "CUST-0002",
  "customer_snapshot": {
    "name": "Carlos Ramírez",
    "email": "carlos.ramirez@ejemplo.com"
  },
  "shipping_address": {
    "street": "Calle Morelos 250",
    "city": "Guadalajara",
    "state": "JAL",
    "zip": "44100",
    "country": "MX"
  },
  "items": [
    {
      "product_id": "PROD-001",
      "sku": "TEC-MEC-001",
      "name": "Teclado Mecánico RGB",
      "quantity": 1,
      "unit_price": 1200.00,
      "subtotal": 1200.00
    }
  ],
  "subtotal": 1200.00,
  "tax": 192.00,
  "total": 1392.00,
  "currency": "MXN",
  "status": "delivered",
  "payment_method": "credit_card",
  "created_at": "2024-03-18T09:00:00Z",
  "updated_at": "2024-03-19T11:20:00Z"
});
```

```sql
-- =============================================
-- RESEÑAS
-- =============================================
INSERT INTO ecommerce.store.reviews (KEY, VALUE) VALUES
("rev::REV-001", {
  "type": "review",
  "review_id": "REV-001",
  "product_id": "PROD-001",
  "customer_id": "CUST-0001",
  "customer_name": "María González",
  "rating": 5,
  "title": "Excelente teclado",
  "body": "Los switches son muy precisos y la retroiluminación es hermosa. Muy recomendado.",
  "verified_purchase": true,
  "created_at": "2024-03-20T16:00:00Z",
  "status": "approved"
});

INSERT INTO ecommerce.store.reviews (KEY, VALUE) VALUES
("rev::REV-002", {
  "type": "review",
  "review_id": "REV-002",
  "product_id": "PROD-001",
  "customer_id": "CUST-0002",
  "customer_name": "Carlos Ramírez",
  "rating": 4,
  "title": "Buen teclado, envío rápido",
  "body": "El teclado funciona perfectamente. El ruido de los switches es un poco alto pero es lo esperado.",
  "verified_purchase": true,
  "created_at": "2024-03-21T10:30:00Z",
  "status": "approved"
});
```

**Salida esperada de cada INSERT:**
```json
{
    "requestID": "...",
    "status": "success",
    "metrics": {
        "mutationCount": 1,
        ...
    }
}
```

**Verificación rápida del conteo:**
```sql
SELECT
  (SELECT RAW COUNT(*) FROM ecommerce.store.customers)[0] AS total_customers,
  (SELECT RAW COUNT(*) FROM ecommerce.store.products)[0] AS total_products,
  (SELECT RAW COUNT(*) FROM ecommerce.store.orders)[0] AS total_orders,
  (SELECT RAW COUNT(*) FROM ecommerce.store.categories)[0] AS total_categories,
  (SELECT RAW COUNT(*) FROM ecommerce.store.reviews)[0] AS total_reviews;
```

**Resultado esperado:**
```json
[
  {
    "total_customers": 2,
    "total_products": 3,
    "total_orders": 3,
    "total_categories": 2,
    "total_reviews": 2
  }
]
```

#### Paso 4.4 — Crear Índices para los Patrones de Acceso

**Instrucciones:**

Crea el índice primario (para desarrollo) y los índices secundarios definidos en el modelo físico:

```sql
-- Índice primario en cada collection (útil durante desarrollo)
CREATE PRIMARY INDEX ON ecommerce.store.customers;
CREATE PRIMARY INDEX ON ecommerce.store.products;
CREATE PRIMARY INDEX ON ecommerce.store.orders;
CREATE PRIMARY INDEX ON ecommerce.store.categories;
CREATE PRIMARY INDEX ON ecommerce.store.reviews;

-- PA-2: Listar pedidos por cliente ordenados por fecha
CREATE INDEX idx_orders_customer
ON ecommerce.store.orders(customer_id, created_at DESC);

-- PA-3: Buscar productos por categoría
-- ARRAY index para buscar dentro del array category_ids
CREATE INDEX idx_products_category
ON ecommerce.store.products(ALL ARRAY cat FOR cat IN category_ids END, price, stock)
WHERE active = true;

-- PA-5: Obtener reseñas de un producto
CREATE INDEX idx_reviews_product
ON ecommerce.store.reviews(product_id, rating, created_at DESC)
WHERE status = "approved";
```

**Salida esperada de cada CREATE INDEX:**
```json
{
    "requestID": "...",
    "status": "success",
    ...
}
```

**Verificación:**
```sql
-- Verificar que los índices fueron creados
SELECT name, keyspace_id, `using`, state
FROM system:indexes
WHERE keyspace_id LIKE "ecommerce%"
ORDER BY name;
```

---

### Parte 5 — Validación de Patrones de Acceso

**Objetivo:** Confirmar que el modelo diseñado soporta eficientemente los 5 patrones de acceso identificados.

#### Instrucciones

Ejecuta las siguientes consultas y verifica que retornan los resultados correctos:

**PA-1: Detalle completo de un pedido (acceso por key directa)**
```sql
-- Acceso O(1) por document key - el más eficiente posible
SELECT *
FROM ecommerce.store.orders
USE KEYS "ord::ORD-20240315-8821";
```

**Resultado esperado:** El documento completo del pedido con sus ítems embebidos, snapshot del cliente y dirección de envío en una sola lectura.

---

**PA-2: Todos los pedidos de un cliente, ordenados por fecha**
```sql
SELECT order_id, status, total, currency, created_at,
       ARRAY_LENGTH(items) AS item_count
FROM ecommerce.store.orders
WHERE customer_id = "CUST-0001"
ORDER BY created_at DESC;
```

**Resultado esperado:**
```json
[
  {
    "order_id": "ORD-20240320-9102",
    "status": "pending",
    "total": 9860,
    "currency": "MXN",
    "created_at": "2024-03-20T15:10:00Z",
    "item_count": 1
  },
  {
    "order_id": "ORD-20240315-8821",
    "status": "shipped",
    "total": 2204,
    "currency": "MXN",
    "created_at": "2024-03-15T10:22:00Z",
    "item_count": 2
  }
]
```

---

**PA-3: Productos de una categoría con precio y stock**
```sql
SELECT product_id, name, price, stock, category_names
FROM ecommerce.store.products
WHERE "CAT-003" IN category_ids
  AND active = true
ORDER BY price ASC;
```

**Resultado esperado:** Los productos PROD-001 y PROD-002 que pertenecen a la categoría CAT-003 (Periféricos).

---

**PA-4: Actualizar el estado de un pedido**
```sql
UPDATE ecommerce.store.orders
USE KEYS "ord::ORD-20240320-9102"
SET status = "processing",
    updated_at = "2024-03-20T16:00:00Z"
RETURNING order_id, status, updated_at;
```

**Resultado esperado:**
```json
[
  {
    "order_id": "ORD-20240320-9102",
    "status": "processing",
    "updated_at": "2024-03-20T16:00:00Z"
  }
]
```

---

**PA-5: Reseñas de un producto con rating promedio**
```sql
SELECT
  r.review_id,
  r.customer_name,
  r.rating,
  r.title,
  r.body,
  r.created_at,
  (SELECT RAW AVG(rv.rating)
   FROM ecommerce.store.reviews rv
   WHERE rv.product_id = "PROD-001" AND rv.status = "approved")[0] AS avg_rating
FROM ecommerce.store.reviews r
WHERE r.product_id = "PROD-001"
  AND r.status = "approved"
ORDER BY r.created_at DESC;
```

**Resultado esperado:** Las 2 reseñas del PROD-001 con el rating promedio calculado (4.5).

---

## Validación y Pruebas Finales

Ejecuta esta consulta de validación integral que verifica la integridad del modelo:

```sql
-- Verificación integral: JOIN entre orders y products para confirmar
-- que las referencias son válidas
SELECT
  o.order_id,
  o.customer_snapshot.name AS customer,
  o.status,
  o.total,
  ARRAY item.name FOR item IN o.items END AS product_names,
  ARRAY p.stock FOR p IN
    (SELECT RAW stock FROM ecommerce.store.products
     WHERE product_id IN (ARRAY item.product_id FOR item IN o.items END))
  END AS current_stock_levels
FROM ecommerce.store.orders o
ORDER BY o.created_at DESC;
```

**Resultado esperado:** Los 3 pedidos con nombres de productos y niveles de stock actuales, demostrando que las referencias entre documentos son funcionales.

```sql
-- Verificación de consistencia: contar documentos por tipo
SELECT type, COUNT(*) AS total
FROM (
  SELECT type FROM ecommerce.store.customers
  UNION ALL
  SELECT type FROM ecommerce.store.products
  UNION ALL
  SELECT type FROM ecommerce.store.orders
  UNION ALL
  SELECT type FROM ecommerce.store.categories
  UNION ALL
  SELECT type FROM ecommerce.store.reviews
) AS all_docs
GROUP BY type
ORDER BY type;
```

**Resultado esperado:**
```json
[
  { "type": "category", "total": 2 },
  { "type": "customer", "total": 2 },
  { "type": "order",    "total": 3 },
  { "type": "product",  "total": 3 },
  { "type": "review",   "total": 2 }
]
```

---

## Resolución de Problemas

### Problema 1: Error "Keyspace not found" al ejecutar INSERT o SELECT

**Síntomas:**
```
"msg": "Keyspace not found in CB datastore: ecommerce:store.customers"
```
O bien la consulta retorna 0 resultados inesperadamente.

**Causa:**
El bucket `ecommerce`, el scope `store` o alguna de las collections no fue creada correctamente, o el nombre tiene un error tipográfico. También puede ocurrir si el bucket fue creado pero el servicio de Query aún no lo ha registrado (demora de propagación).

**Solución:**
1. Verifica que el bucket existe:
```bash
curl -s -u Administrator:password http://localhost:8091/pools/default/buckets | python3 -m json.tool | grep '"name"'
```
2. Verifica las collections desde SQL++:
```sql
SELECT RAW name FROM system:keyspaces
WHERE `bucket` = "ecommerce"
ORDER BY name;
```
3. Si el bucket existe pero la consulta falla, espera 10-15 segundos para que el servicio de Query propague los metadatos y vuelve a intentarlo.
4. Si la collection no aparece, recréala:
```sql
CREATE COLLECTION ecommerce.store.customers;
```

---

### Problema 2: El índice de array `idx_products_category` no se usa en la consulta PA-3

**Síntomas:**
La consulta de PA-3 (`WHERE "CAT-003" IN category_ids`) es lenta o el `EXPLAIN` muestra un `PrimaryScan` en lugar de un `IndexScan`.

**Causa:**
La sintaxis de la consulta no coincide con la definición del índice de array (`ALL ARRAY cat FOR cat IN category_ids END`). La cláusula `WHERE active = true` en el índice es una condición parcial que debe estar presente en la consulta. Si se omite `active = true` en el WHERE, el optimizador no puede usar el índice parcial.

**Solución:**
1. Verifica que la consulta incluye `AND active = true`:
```sql
-- CORRECTO: incluye la condición del índice parcial
SELECT product_id, name, price
FROM ecommerce.store.products
WHERE "CAT-003" IN category_ids
  AND active = true;
```
2. Verifica el plan de ejecución:
```sql
EXPLAIN
SELECT product_id, name, price
FROM ecommerce.store.products
WHERE "CAT-003" IN category_ids
  AND active = true;
```
El plan debe mostrar `"index": "idx_products_category"` en el operador de scan.

3. Si el índice aún no está listo, verifica su estado:
```sql
SELECT name, state FROM system:indexes
WHERE name = "idx_products_category";
```
El campo `state` debe ser `"online"`. Si es `"building"`, espera a que termine la construcción.

---

## Limpieza del Entorno

> **Nota:** Ejecuta la limpieza **solo si el instructor lo indica** o si necesitas liberar recursos. El bucket `ecommerce` será utilizado en laboratorios posteriores de este capítulo.

```sql
-- Opción 1: Eliminar solo los documentos de prueba (conserva la estructura)
DELETE FROM ecommerce.store.reviews;
DELETE FROM ecommerce.store.orders;
DELETE FROM ecommerce.store.customers;
DELETE FROM ecommerce.store.products;
DELETE FROM ecommerce.store.categories;
```

```bash
# Opción 2: Eliminar el bucket completo (elimina todo: datos, índices, collections)
curl -s -X DELETE http://localhost:8091/pools/default/buckets/ecommerce \
  -u Administrator:password
```

---

## Resumen

En este laboratorio aplicaste el proceso completo de modelado formal de datos para una aplicación de e-commerce en Couchbase:

| Etapa | Actividad realizada | Resultado |
|---|---|---|
| **Modelado Conceptual** | Diagrama ER con 6 entidades y 5 patrones de acceso | Comprensión del dominio del negocio |
| **Modelado Lógico** | 5 decisiones documentadas de embedding vs. referencing | Esquemas JSON optimizados para los patrones de acceso |
| **Modelado Físico** | Bucket/Scope/Collections + esquema de keys + índices | Estructura lista para implementar |
| **Validación** | INSERT de documentos de prueba + consultas de verificación | Confirmación de que el modelo funciona |

### Conceptos Clave Aprendidos

- **El modelado guiado por patrones de acceso** es el principio central en bases de datos documentales: primero define cómo se leerá la información, luego decide cómo almacenarla.
- **Embedding** es preferible cuando los datos relacionados siempre se leen juntos, tienen un tamaño acotado y no tienen existencia independiente (ej: OrderItem dentro de Order).
- **Referencing** es preferible cuando los datos relacionados crecen sin límite, tienen su propio ciclo de vida o son consultados de forma independiente (ej: Review separado de Product).
- **El snapshot pattern** (embeber un snapshot de datos de otra entidad en el momento de la transacción) resuelve el trade-off entre consistencia histórica y eficiencia de lectura.
- **Los índices de array** (`ALL ARRAY ... FOR ... IN ... END`) son esenciales para buscar eficientemente dentro de arrays en documentos JSON.

### Recursos Adicionales

- [Couchbase Docs: Data Modeling Fundamentals](https://docs.couchbase.com/server/current/learn/data/document-data-model.html)
- [Couchbase Docs: Scopes and Collections](https://docs.couchbase.com/server/current/learn/data/scopes-and-collections.html)
- [Couchbase Docs: Array Indexing](https://docs.couchbase.com/server/current/n1ql/n1ql-language-reference/indexing-arrays.html)
- [Couchbase Blog: JSON Document Modeling Best Practices](https://www.couchbase.com/blog/json-document-modeling-best-practices/)
- Kleppmann, M. — *Designing Data-Intensive Applications*, O'Reilly (Capítulos 2 y 3)

---
