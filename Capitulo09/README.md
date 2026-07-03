---LAB_START---
LAB_ID: 09-00-01
---MARKDOWN---
# Configuración de roles y pruebas de acceso en Couchbase

## Metadatos

| Campo        | Detalle                          |
|--------------|----------------------------------|
| **Duración** | 55 minutos                       |
| **Complejidad** | Media                         |
| **Nivel Bloom** | Aplicar (Apply)               |
| **Versión Couchbase** | 7.6.x                  |
| **Dataset requerido** | Ninguno (se crea en el lab) |

---

## Descripción General

En este laboratorio aplicarás los conceptos de Control de Acceso Basado en Roles (RBAC) de Couchbase creando un entorno de comercio electrónico con tres perfiles de usuario bien diferenciados: un usuario de solo lectura para reporting, un usuario de aplicación backend con permisos mixtos y un administrador de aplicación con control total del bucket. Verificarás el comportamiento de cada perfil ejecutando consultas SQL++ con credenciales específicas, comprobando tanto los accesos permitidos como los denegados. Al finalizar, habrás implementado el principio de mínimo privilegio en un escenario realista usando la Web Console, la REST API y `cbq`.

---

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Crear usuarios y grupos en Couchbase y asignarles roles predefinidos con alcance de bucket, scope y collection desde la Web Console y la REST API.
- [ ] Configurar roles específicos para el servicio Query (`Query Select`, `Query Insert`, `Query Update`) con granularidad de scope y collection.
- [ ] Verificar el comportamiento del control de acceso ejecutando consultas SQL++ con diferentes credenciales usando `cbq` y la Web Console.
- [ ] Implementar el principio de mínimo privilegio creando usuarios de aplicación con permisos estrictamente necesarios para sus operaciones.
- [ ] Utilizar grupos de usuarios para centralizar la asignación de roles y verificar la herencia de permisos.

---

## Prerrequisitos

### Conocimiento previo
- Haber completado el **Lab 02-00-01** (Couchbase Server instalado y operativo).
- Comprensión básica de autenticación vs. autorización y el concepto de roles.
- Familiaridad con SQL++ básico (`SELECT`, `INSERT`, `UPDATE`, `DELETE`).
- Conocimiento de la estructura bucket → scope → collection en Couchbase.

### Acceso requerido
- Acceso a Couchbase Web Console en `http://localhost:8091` con credenciales de administrador (`Administrator` / `password` o las configuradas en tu instalación).
- Terminal con acceso a `curl` y `cbq` (incluidos en Couchbase Server 7.6.x).
- Puertos 8091 y 11210 disponibles y sin bloqueo de firewall local.

---

## Entorno de Laboratorio

### Hardware mínimo recomendado

| Recurso | Mínimo | Recomendado |
|---------|--------|-------------|
| RAM | 8 GB | 16 GB |
| CPU | 4 núcleos x86_64 | 8 núcleos |
| Almacenamiento | 20 GB libres (SSD) | 50 GB SSD |
| Red | localhost funcional | localhost funcional |
| Pantalla | 1280×768 | 1280×800 o superior |

### Software requerido

| Software | Versión |
|----------|---------|
| Couchbase Server | 7.6.x (CE o Enterprise Trial) |
| Navegador web | Chrome 110+, Firefox 110+ o Edge 110+ |
| `curl` | 7.x o superior |
| `cbq` | Incluido con Couchbase Server 7.6.x |
| Editor de texto | VS Code 1.80+ o equivalente |

### Variables de entorno utilizadas en este laboratorio

Para simplificar los comandos, define las siguientes variables en tu terminal antes de comenzar:

```bash
# Credenciales del administrador de Couchbase
export CB_HOST="localhost"
export CB_ADMIN="Administrator"
export CB_PASS="password"       # Cambia esto por tu contraseña real
export CB_URL="http://${CB_HOST}:8091"
```

> **Nota:** Si usas Docker, reemplaza `localhost` por la IP del contenedor o usa `127.0.0.1`. Verifica que el contenedor esté corriendo con `docker ps`.

### Verificación del entorno inicial

Antes de comenzar, confirma que Couchbase está operativo:

```bash
curl -s -u ${CB_ADMIN}:${CB_PASS} ${CB_URL}/pools/default | python3 -m json.tool | grep '"name"'
```

**Salida esperada:**

```json
"name": "default",
```

Si no obtienes respuesta, verifica que el servicio Couchbase esté iniciado.

---

## Pasos del Laboratorio

---

### Paso 1: Crear el bucket `ecommerce` con scopes y collections

**Objetivo:** Preparar la estructura de datos sobre la que se aplicarán los controles de acceso: un bucket `ecommerce` con dos scopes (`catalog` y `orders`) y sus collections correspondientes.

#### Instrucciones

**1.1 Crear el bucket `ecommerce`**

Desde la terminal, ejecuta:

```bash
curl -s -X POST \
  -u ${CB_ADMIN}:${CB_PASS} \
  ${CB_URL}/pools/default/buckets \
  -d name=ecommerce \
  -d ramQuota=256 \
  -d bucketType=couchbase \
  -d replicaNumber=0
```

> **Nota:** `replicaNumber=0` es adecuado para un clúster de un solo nodo. En producción se usaría al menos 1.

**1.2 Esperar a que el bucket esté listo (5 segundos)**

```bash
sleep 5
```

**1.3 Crear el scope `catalog`**

```bash
curl -s -X POST \
  -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/pools/default/buckets/ecommerce/scopes" \
  -d name=catalog
```

**1.4 Crear el scope `orders`**

```bash
curl -s -X POST \
  -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/pools/default/buckets/ecommerce/scopes" \
  -d name=orders
```

**1.5 Crear la collection `products` dentro del scope `catalog`**

```bash
curl -s -X POST \
  -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/pools/default/buckets/ecommerce/scopes/catalog/collections" \
  -d name=products
```

**1.6 Crear la collection `categories` dentro del scope `catalog`**

```bash
curl -s -X POST \
  -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/pools/default/buckets/ecommerce/scopes/catalog/collections" \
  -d name=categories
```

**1.7 Crear la collection `purchases` dentro del scope `orders`**

```bash
curl -s -X POST \
  -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/pools/default/buckets/ecommerce/scopes/orders/collections" \
  -d name=purchases
```

**1.8 Crear la collection `customers` dentro del scope `orders`**

```bash
curl -s -X POST \
  -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/pools/default/buckets/ecommerce/scopes/orders/collections" \
  -d name=customers
```

**1.9 Verificar la estructura creada**

```bash
curl -s -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/pools/default/buckets/ecommerce/scopes" \
  | python3 -m json.tool | grep '"name"'
```

**Salida esperada:**

```
"name": "catalog",
"name": "products",
"name": "categories",
"name": "orders",
"name": "purchases",
"name": "customers",
```

**1.10 Insertar documentos de prueba usando `cbq`**

Abre `cbq` como administrador:

```bash
cbq -u ${CB_ADMIN} -p ${CB_PASS} -engine=http://${CB_HOST}:8093
```

Dentro de `cbq`, ejecuta los siguientes inserts:

```sql
-- Insertar productos en catalog.products
INSERT INTO ecommerce.catalog.products (KEY, VALUE) VALUES
  ("prod-001", {"id": "prod-001", "name": "Laptop Pro 15", "price": 1299.99, "category": "electronics", "stock": 50}),
  ("prod-002", {"id": "prod-002", "name": "Wireless Mouse", "price": 29.99, "category": "accessories", "stock": 200}),
  ("prod-003", {"id": "prod-003", "name": "USB-C Hub", "price": 49.99, "category": "accessories", "stock": 150});
```

```sql
-- Insertar categorías en catalog.categories
INSERT INTO ecommerce.catalog.categories (KEY, VALUE) VALUES
  ("cat-001", {"id": "cat-001", "name": "Electronics", "description": "Electronic devices and gadgets"}),
  ("cat-002", {"id": "cat-002", "name": "Accessories", "description": "Computer and device accessories"});
```

```sql
-- Insertar clientes en orders.customers
INSERT INTO ecommerce.orders.customers (KEY, VALUE) VALUES
  ("cust-001", {"id": "cust-001", "name": "Ana García", "email": "ana@example.com", "tier": "premium"}),
  ("cust-002", {"id": "cust-002", "name": "Carlos López", "email": "carlos@example.com", "tier": "standard"});
```

```sql
-- Insertar pedidos en orders.purchases
INSERT INTO ecommerce.orders.purchases (KEY, VALUE) VALUES
  ("order-001", {"id": "order-001", "customerId": "cust-001", "productId": "prod-001", "quantity": 1, "total": 1299.99, "status": "delivered"}),
  ("order-002", {"id": "order-002", "customerId": "cust-002", "productId": "prod-002", "quantity": 2, "total": 59.98, "status": "pending"});
```

Sal de `cbq`:

```sql
\quit
```

**Salida esperada de cada INSERT:**

```json
{
    "requestID": "...",
    "status": "success",
    "metrics": {
        "mutationCount": 3,
        ...
    }
}
```

#### Verificación

Desde `cbq` como administrador, confirma que los datos existen:

```bash
cbq -u ${CB_ADMIN} -p ${CB_PASS} -engine=http://${CB_HOST}:8093 \
  --script="SELECT COUNT(*) AS total FROM ecommerce.catalog.products;"
```

**Salida esperada:**

```json
{
    "results": [{"total": 3}],
    "status": "success"
}
```

---

### Paso 2: Crear el usuario `app-readonly` (Perfil 1 — Solo lectura para reporting)

**Objetivo:** Crear un usuario con acceso de solo lectura al scope `catalog`, implementando el principio de mínimo privilegio para el equipo de reporting.

#### Instrucciones

**2.1 Crear el usuario `app-readonly` vía REST API**

```bash
curl -s -X PUT \
  -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/settings/rbac/users/local/app-readonly" \
  -d "name=App+ReadOnly+Reporting" \
  -d "password=Readonly@2024!" \
  -d "roles=data_reader[ecommerce:catalog],query_select[ecommerce:catalog]"
```

> **Explicación de los roles asignados:**
> - `data_reader[ecommerce:catalog]` — Permite leer documentos en cualquier collection del scope `catalog` del bucket `ecommerce`.
> - `query_select[ecommerce:catalog]` — Permite ejecutar sentencias `SELECT` en el scope `catalog`.

**2.2 Verificar que el usuario fue creado correctamente**

```bash
curl -s -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/settings/rbac/users/local/app-readonly" \
  | python3 -m json.tool
```

**Salida esperada (fragmento):**

```json
{
    "id": "app-readonly",
    "name": "App ReadOnly Reporting",
    "domain": "local",
    "roles": [
        {
            "role": "data_reader",
            "bucket_name": "ecommerce",
            "scope_name": "catalog"
        },
        {
            "role": "query_select",
            "bucket_name": "ecommerce",
            "scope_name": "catalog"
        }
    ]
}
```

**2.3 Verificar acceso permitido: SELECT en `catalog.products`**

```bash
cbq -u app-readonly -p "Readonly@2024!" \
  -engine=http://${CB_HOST}:8093 \
  --script="SELECT name, price FROM ecommerce.catalog.products WHERE price < 100;"
```

**Salida esperada:**

```json
{
    "results": [
        {"name": "Wireless Mouse", "price": 29.99},
        {"name": "USB-C Hub", "price": 49.99}
    ],
    "status": "success"
}
```

**2.4 Verificar acceso denegado: INSERT en `catalog.products`**

```bash
cbq -u app-readonly -p "Readonly@2024!" \
  -engine=http://${CB_HOST}:8093 \
  --script="INSERT INTO ecommerce.catalog.products (KEY, VALUE) VALUES ('prod-test', {'name': 'Test'});"
```

**Salida esperada (error de autorización):**

```json
{
    "errors": [
        {
            "code": 13014,
            "msg": "User does not have credentials to run INSERT queries on the ecommerce:catalog:products. Add role Query_Insert[ecommerce:catalog:products] to allow the query to run."
        }
    ],
    "status": "fatal"
}
```

**2.5 Verificar acceso denegado: SELECT en scope `orders`**

```bash
cbq -u app-readonly -p "Readonly@2024!" \
  -engine=http://${CB_HOST}:8093 \
  --script="SELECT * FROM ecommerce.orders.customers LIMIT 1;"
```

**Salida esperada (error de autorización):**

```json
{
    "errors": [
        {
            "code": 13014,
            "msg": "User does not have credentials to run SELECT queries on the ecommerce:orders:customers. Add role Query_Select[ecommerce:orders:customers] to allow the query to run."
        }
    ],
    "status": "fatal"
}
```

#### Verificación

✅ El usuario `app-readonly` puede hacer `SELECT` en `catalog` pero NO puede hacer `INSERT` ni acceder a `orders`. El principio de mínimo privilegio está funcionando correctamente.

---

### Paso 3: Crear el usuario `app-backend` (Perfil 2 — Aplicación backend)

**Objetivo:** Crear un usuario de aplicación backend con permisos mixtos: lectura en ambos scopes y capacidad de escritura/modificación en el scope `orders`.

#### Instrucciones

**3.1 Crear el usuario `app-backend` vía REST API**

```bash
curl -s -X PUT \
  -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/settings/rbac/users/local/app-backend" \
  -d "name=App+Backend+Service" \
  -d "password=Backend@2024!" \
  -d "roles=data_reader[ecommerce:catalog],query_select[ecommerce:catalog],data_reader[ecommerce:orders],data_writer[ecommerce:orders],query_select[ecommerce:orders],query_insert[ecommerce:orders],query_update[ecommerce:orders]"
```

> **Explicación de los roles asignados:**
> - `data_reader[ecommerce:catalog]` + `query_select[ecommerce:catalog]` — Solo lectura en catalog.
> - `data_reader[ecommerce:orders]` + `data_writer[ecommerce:orders]` — Lectura y escritura KV en orders.
> - `query_select[ecommerce:orders]` + `query_insert[ecommerce:orders]` + `query_update[ecommerce:orders]` — Operaciones SQL++ en orders.

**3.2 Verificar la creación del usuario**

```bash
curl -s -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/settings/rbac/users/local/app-backend" \
  | python3 -m json.tool | grep '"role"'
```

**Salida esperada:**

```
"role": "data_reader",
"role": "query_select",
"role": "data_reader",
"role": "data_writer",
"role": "query_select",
"role": "query_insert",
"role": "query_update",
```

**3.3 Verificar acceso permitido: SELECT en ambos scopes**

```bash
cbq -u app-backend -p "Backend@2024!" \
  -engine=http://${CB_HOST}:8093 \
  --script="SELECT name, price FROM ecommerce.catalog.products;"
```

```bash
cbq -u app-backend -p "Backend@2024!" \
  -engine=http://${CB_HOST}:8093 \
  --script="SELECT id, status, total FROM ecommerce.orders.purchases;"
```

**Salida esperada del segundo comando:**

```json
{
    "results": [
        {"id": "order-001", "status": "delivered", "total": 1299.99},
        {"id": "order-002", "status": "pending", "total": 59.98}
    ],
    "status": "success"
}
```

**3.4 Verificar acceso permitido: INSERT en `orders.purchases`**

```bash
cbq -u app-backend -p "Backend@2024!" \
  -engine=http://${CB_HOST}:8093 \
  --script="INSERT INTO ecommerce.orders.purchases (KEY, VALUE) VALUES ('order-003', {'id': 'order-003', 'customerId': 'cust-001', 'productId': 'prod-002', 'quantity': 3, 'total': 89.97, 'status': 'processing'});"
```

**Salida esperada:**

```json
{
    "status": "success",
    "metrics": {
        "mutationCount": 1
    }
}
```

**3.5 Verificar acceso permitido: UPDATE en `orders.purchases`**

```bash
cbq -u app-backend -p "Backend@2024!" \
  -engine=http://${CB_HOST}:8093 \
  --script="UPDATE ecommerce.orders.purchases SET status = 'confirmed' WHERE id = 'order-003';"
```

**Salida esperada:**

```json
{
    "status": "success",
    "metrics": {
        "mutationCount": 1
    }
}
```

**3.6 Verificar acceso denegado: INSERT en `catalog.products`**

```bash
cbq -u app-backend -p "Backend@2024!" \
  -engine=http://${CB_HOST}:8093 \
  --script="INSERT INTO ecommerce.catalog.products (KEY, VALUE) VALUES ('prod-hack', {'name': 'Unauthorized Product'});"
```

**Salida esperada (error de autorización):**

```json
{
    "errors": [
        {
            "code": 13014,
            "msg": "User does not have credentials to run INSERT queries on the ecommerce:catalog:products..."
        }
    ],
    "status": "fatal"
}
```

**3.7 Verificar acceso denegado: DELETE en `orders`**

```bash
cbq -u app-backend -p "Backend@2024!" \
  -engine=http://${CB_HOST}:8093 \
  --script="DELETE FROM ecommerce.orders.purchases WHERE id = 'order-003';"
```

**Salida esperada (error de autorización):**

```json
{
    "errors": [
        {
            "code": 13014,
            "msg": "User does not have credentials to run DELETE queries on the ecommerce:orders:purchases..."
        }
    ],
    "status": "fatal"
}
```

#### Verificación

✅ El usuario `app-backend` puede leer catalog, leer/insertar/actualizar en orders, pero NO puede insertar en catalog ni eliminar registros en orders. Los permisos mixtos están correctamente configurados.

---

### Paso 4: Crear el usuario `app-admin` (Perfil 3 — Administrador de aplicación)

**Objetivo:** Crear un usuario administrador de la aplicación con control total sobre el bucket `ecommerce`, incluyendo la capacidad de crear índices y gestionar el bucket.

#### Instrucciones

**4.1 Crear el usuario `app-admin` vía REST API**

```bash
curl -s -X PUT \
  -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/settings/rbac/users/local/app-admin" \
  -d "name=App+Administrator" \
  -d "password=Admin@2024!" \
  -d "roles=bucket_admin[ecommerce],query_manage_index[ecommerce]"
```

> **Explicación de los roles asignados:**
> - `bucket_admin[ecommerce]` — Control total sobre el bucket: incluye gestión de datos, scopes, collections y configuración del bucket.
> - `query_manage_index[ecommerce]` — Capacidad de crear, modificar y eliminar índices GSI en el bucket.

**4.2 Verificar la creación del usuario**

```bash
curl -s -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/settings/rbac/users/local/app-admin" \
  | python3 -m json.tool | grep -E '"role"|"bucket_name"'
```

**Salida esperada:**

```
"role": "bucket_admin",
"bucket_name": "ecommerce",
"role": "query_manage_index",
"bucket_name": "ecommerce",
```

**4.3 Verificar acceso completo: SELECT en ambos scopes**

```bash
cbq -u app-admin -p "Admin@2024!" \
  -engine=http://${CB_HOST}:8093 \
  --script="SELECT * FROM ecommerce.catalog.products; SELECT * FROM ecommerce.orders.customers;"
```

**4.4 Verificar capacidad de crear un índice GSI**

```bash
cbq -u app-admin -p "Admin@2024!" \
  -engine=http://${CB_HOST}:8093 \
  --script="CREATE INDEX idx_product_category ON ecommerce.catalog.products(category);"
```

**Salida esperada:**

```json
{
    "results": [],
    "status": "success"
}
```

**4.5 Verificar que el índice fue creado**

```bash
cbq -u app-admin -p "Admin@2024!" \
  -engine=http://${CB_HOST}:8093 \
  --script="SELECT name, state FROM system:indexes WHERE keyspace_id = 'products' AND bucket_id = 'ecommerce';"
```

**Salida esperada:**

```json
{
    "results": [
        {"name": "idx_product_category", "state": "online"}
    ],
    "status": "success"
}
```

**4.6 Verificar que `app-readonly` NO puede crear índices**

```bash
cbq -u app-readonly -p "Readonly@2024!" \
  -engine=http://${CB_HOST}:8093 \
  --script="CREATE INDEX idx_test ON ecommerce.catalog.products(name);"
```

**Salida esperada (error de autorización):**

```json
{
    "errors": [
        {
            "code": 13014,
            "msg": "User does not have credentials to run CREATE INDEX..."
        }
    ],
    "status": "fatal"
}
```

#### Verificación

✅ El usuario `app-admin` tiene control total sobre el bucket `ecommerce`, incluyendo la creación de índices. Los usuarios con roles más restrictivos no pueden realizar estas operaciones.

---

### Paso 5: Gestión de usuarios desde la Web Console

**Objetivo:** Familiarizarse con la interfaz gráfica de administración de seguridad en Couchbase Web Console, verificando los usuarios creados y explorando la asignación visual de roles.

#### Instrucciones

**5.1 Acceder al Security Manager**

1. Abre tu navegador y navega a `http://localhost:8091`.
2. Inicia sesión con las credenciales de administrador.
3. En el menú lateral izquierdo, haz clic en **Security**.
4. Selecciona la pestaña **Users**.

**5.2 Verificar los tres usuarios creados**

Deberías ver en la lista los usuarios:
- `app-readonly`
- `app-backend`
- `app-admin`

Haz clic en cada usuario para ver los roles asignados y confirmar que coinciden con lo configurado en los pasos anteriores.

**5.3 Explorar los roles disponibles (vista informativa)**

1. En la sección **Security**, selecciona la pestaña **Roles** (si está disponible en tu versión) o navega a la documentación integrada.
2. Alternativamente, lista todos los roles disponibles en el clúster vía REST:

```bash
curl -s -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/settings/rbac/roles" \
  | python3 -m json.tool | grep '"role"' | sort | uniq | head -20
```

**Salida esperada (fragmento):**

```
"role": "admin",
"role": "analytics_admin",
"role": "analytics_manager",
"role": "analytics_reader",
"role": "analytics_select",
"role": "bucket_admin",
"role": "bucket_full_access",
"role": "cluster_admin",
"role": "data_backup",
"role": "data_dcp_reader",
"role": "data_monitoring",
"role": "data_reader",
"role": "data_writer",
"role": "eventing_admin",
"role": "fts_admin",
"role": "fts_searcher",
"role": "mobile_sync_gateway",
"role": "query_delete",
"role": "query_insert",
"role": "query_manage_functions",
```

**5.4 Editar un usuario desde la Web Console**

1. En la lista de usuarios, haz clic en el usuario `app-backend`.
2. Selecciona **Edit**.
3. Observa la interfaz de asignación de roles con sus niveles jerárquicos (bucket → scope → collection).
4. **No guardes cambios** — solo es una exploración visual.
5. Haz clic en **Cancel**.

#### Verificación

✅ Los tres usuarios son visibles en la Web Console con sus roles correctamente asignados. La interfaz gráfica permite visualizar y modificar roles de forma intuitiva.

---

### Paso 6: Crear el grupo `reporting-team` y agregar usuarios

**Objetivo:** Implementar la gestión centralizada de permisos mediante grupos de usuarios. Crear el grupo `reporting-team` con el perfil de solo lectura y agregar dos usuarios nuevos que hereden esos permisos.

#### Instrucciones

**6.1 Crear el grupo `reporting-team` vía REST API**

```bash
curl -s -X PUT \
  -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/settings/rbac/groups/reporting-team" \
  -d "description=Equipo+de+Reporting+con+acceso+de+solo+lectura" \
  -d "roles=data_reader[ecommerce:catalog],query_select[ecommerce:catalog]"
```

**6.2 Verificar la creación del grupo**

```bash
curl -s -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/settings/rbac/groups/reporting-team" \
  | python3 -m json.tool
```

**Salida esperada:**

```json
{
    "id": "reporting-team",
    "description": "Equipo de Reporting con acceso de solo lectura",
    "roles": [
        {
            "role": "data_reader",
            "bucket_name": "ecommerce",
            "scope_name": "catalog"
        },
        {
            "role": "query_select",
            "bucket_name": "ecommerce",
            "scope_name": "catalog"
        }
    ]
}
```

**6.3 Crear el usuario `reporter-maria` y asignarlo al grupo**

```bash
curl -s -X PUT \
  -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/settings/rbac/users/local/reporter-maria" \
  -d "name=Maria+Fernandez" \
  -d "password=Reporter@2024!" \
  -d "groups=reporting-team"
```

**6.4 Crear el usuario `reporter-jose` y asignarlo al grupo**

```bash
curl -s -X PUT \
  -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/settings/rbac/users/local/reporter-jose" \
  -d "name=Jose+Martinez" \
  -d "password=Reporter@2024!" \
  -d "groups=reporting-team"
```

**6.5 Verificar que `reporter-maria` hereda los roles del grupo**

```bash
curl -s -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/settings/rbac/users/local/reporter-maria" \
  | python3 -m json.tool
```

**Salida esperada:**

```json
{
    "id": "reporter-maria",
    "name": "Maria Fernandez",
    "domain": "local",
    "roles": [],
    "groups": ["reporting-team"],
    "all_roles": [
        {
            "role": "data_reader",
            "bucket_name": "ecommerce",
            "scope_name": "catalog",
            "origins": [{"type": "group", "name": "reporting-team"}]
        },
        {
            "role": "query_select",
            "bucket_name": "ecommerce",
            "scope_name": "catalog",
            "origins": [{"type": "group", "name": "reporting-team"}]
        }
    ]
}
```

> **Observación clave:** El campo `roles` está vacío (sin roles directos), pero `all_roles` muestra los roles heredados del grupo `reporting-team` con `"origins": [{"type": "group", ...}]`. Esto confirma la herencia de permisos.

**6.6 Verificar acceso heredado: SELECT en `catalog.products` con `reporter-maria`**

```bash
cbq -u reporter-maria -p "Reporter@2024!" \
  -engine=http://${CB_HOST}:8093 \
  --script="SELECT name, price FROM ecommerce.catalog.products ORDER BY price;"
```

**Salida esperada:**

```json
{
    "results": [
        {"name": "Wireless Mouse", "price": 29.99},
        {"name": "USB-C Hub", "price": 49.99},
        {"name": "Laptop Pro 15", "price": 1299.99}
    ],
    "status": "success"
}
```

**6.7 Verificar acceso denegado: SELECT en `orders` con `reporter-jose`**

```bash
cbq -u reporter-jose -p "Reporter@2024!" \
  -engine=http://${CB_HOST}:8093 \
  --script="SELECT * FROM ecommerce.orders.purchases LIMIT 1;"
```

**Salida esperada (error de autorización):**

```json
{
    "errors": [
        {
            "code": 13014,
            "msg": "User does not have credentials to run SELECT queries on the ecommerce:orders:purchases..."
        }
    ],
    "status": "fatal"
}
```

#### Verificación

✅ Los usuarios `reporter-maria` y `reporter-jose` heredan correctamente los permisos del grupo `reporting-team`. Pueden acceder al scope `catalog` pero no al scope `orders`, sin necesidad de asignar roles individuales.

---

### Paso 7: Verificación avanzada desde la Web Console — Query Editor

**Objetivo:** Usar el Query Editor de la Web Console para verificar el comportamiento de control de acceso de forma interactiva, simulando el flujo de trabajo de un desarrollador.

#### Instrucciones

**7.1 Acceder al Query Editor como `app-readonly`**

1. Abre una ventana de navegador en modo incógnito/privado.
2. Navega a `http://localhost:8091`.
3. Inicia sesión con usuario `app-readonly` y contraseña `Readonly@2024!`.
4. En el menú lateral, haz clic en **Query**.

**7.2 Ejecutar una consulta permitida**

En el editor de consultas, escribe y ejecuta:

```sql
SELECT name, price, category
FROM ecommerce.catalog.products
WHERE category = "accessories"
ORDER BY price DESC;
```

**Resultado esperado:** Los productos de la categoría "accessories" aparecen correctamente.

**7.3 Intentar una consulta no permitida**

En el mismo editor, escribe y ejecuta:

```sql
SELECT *
FROM ecommerce.orders.purchases
LIMIT 5;
```

**Resultado esperado:** Error de autorización — el usuario no tiene acceso al scope `orders`.

**7.4 Verificar permisos del usuario actual con una consulta de sistema**

```sql
SELECT * FROM system:user_info;
```

**Resultado esperado:** Muestra la información del usuario `app-readonly` con sus roles asignados.

**7.5 Regresar a la sesión de administrador**

Cierra la ventana en modo incógnito y regresa a tu sesión principal de administrador.

#### Verificación

✅ La Web Console respeta el RBAC configurado. Los usuarios con roles restringidos no pueden ejecutar operaciones fuera de su alcance, incluso desde la interfaz gráfica.

---

## Validación y Pruebas Finales

Una vez completados todos los pasos, ejecuta el siguiente script de validación completo para confirmar que todos los permisos están configurados correctamente:

```bash
#!/bin/bash
# Script de validación de RBAC - Lab 09-00-01
CB_HOST="localhost"
CB_PORT="8093"

echo "=================================================="
echo "VALIDACIÓN DE RBAC - Lab 09-00-01"
echo "=================================================="

# Función para probar una consulta
test_query() {
    local user=$1
    local pass=$2
    local query=$3
    local expected_result=$4  # "success" o "error"
    local description=$5

    result=$(cbq -u "$user" -p "$pass" \
      -engine="http://${CB_HOST}:${CB_PORT}" \
      --script="$query" 2>&1)

    if echo "$result" | grep -q "\"status\": \"success\""; then
        actual="success"
    else
        actual="error"
    fi

    if [ "$actual" = "$expected_result" ]; then
        echo "  ✅ PASS: $description"
    else
        echo "  ❌ FAIL: $description (esperado: $expected_result, obtenido: $actual)"
    fi
}

echo ""
echo "--- Perfil 1: app-readonly ---"
test_query "app-readonly" "Readonly@2024!" \
  "SELECT name FROM ecommerce.catalog.products LIMIT 1;" \
  "success" "SELECT en catalog.products"

test_query "app-readonly" "Readonly@2024!" \
  "INSERT INTO ecommerce.catalog.products (KEY, VALUE) VALUES ('x', {'name': 'x'});" \
  "error" "INSERT en catalog.products (debe fallar)"

test_query "app-readonly" "Readonly@2024!" \
  "SELECT * FROM ecommerce.orders.customers LIMIT 1;" \
  "error" "SELECT en orders.customers (debe fallar)"

echo ""
echo "--- Perfil 2: app-backend ---"
test_query "app-backend" "Backend@2024!" \
  "SELECT name FROM ecommerce.catalog.products LIMIT 1;" \
  "success" "SELECT en catalog.products"

test_query "app-backend" "Backend@2024!" \
  "SELECT id FROM ecommerce.orders.purchases LIMIT 1;" \
  "success" "SELECT en orders.purchases"

test_query "app-backend" "Backend@2024!" \
  "UPDATE ecommerce.orders.purchases SET status = 'verified' WHERE id = 'order-001';" \
  "success" "UPDATE en orders.purchases"

test_query "app-backend" "Backend@2024!" \
  "INSERT INTO ecommerce.catalog.products (KEY, VALUE) VALUES ('x', {'name': 'x'});" \
  "error" "INSERT en catalog.products (debe fallar)"

test_query "app-backend" "Backend@2024!" \
  "DELETE FROM ecommerce.orders.purchases WHERE id = 'order-001';" \
  "error" "DELETE en orders.purchases (debe fallar)"

echo ""
echo "--- Perfil 3: app-admin ---"
test_query "app-admin" "Admin@2024!" \
  "SELECT * FROM ecommerce.catalog.products LIMIT 1;" \
  "success" "SELECT en catalog.products"

test_query "app-admin" "Admin@2024!" \
  "SELECT * FROM ecommerce.orders.purchases LIMIT 1;" \
  "success" "SELECT en orders.purchases"

echo ""
echo "--- Grupo reporting-team (reporter-maria) ---"
test_query "reporter-maria" "Reporter@2024!" \
  "SELECT name FROM ecommerce.catalog.categories LIMIT 1;" \
  "success" "SELECT en catalog.categories (heredado del grupo)"

test_query "reporter-maria" "Reporter@2024!" \
  "SELECT * FROM ecommerce.orders.customers LIMIT 1;" \
  "error" "SELECT en orders.customers (debe fallar)"

echo ""
echo "=================================================="
echo "Validación completada."
echo "=================================================="
```

Guarda este script como `validate_rbac.sh`, dale permisos de ejecución y ejecútalo:

```bash
chmod +x validate_rbac.sh
./validate_rbac.sh
```

**Salida esperada completa:**

```
==================================================
VALIDACIÓN DE RBAC - Lab 09-00-01
==================================================

--- Perfil 1: app-readonly ---
  ✅ PASS: SELECT en catalog.products
  ✅ PASS: INSERT en catalog.products (debe fallar)
  ✅ PASS: SELECT en orders.customers (debe fallar)

--- Perfil 2: app-backend ---
  ✅ PASS: SELECT en catalog.products
  ✅ PASS: SELECT en orders.purchases
  ✅ PASS: UPDATE en orders.purchases
  ✅ PASS: INSERT en catalog.products (debe fallar)
  ✅ PASS: DELETE en orders.purchases (debe fallar)

--- Perfil 3: app-admin ---
  ✅ PASS: SELECT en catalog.products
  ✅ PASS: SELECT en orders.purchases

--- Grupo reporting-team (reporter-maria) ---
  ✅ PASS: SELECT en catalog.categories (heredado del grupo)
  ✅ PASS: SELECT en orders.customers (debe fallar)

==================================================
Validación completada.
==================================================
```

---

## Resolución de Problemas

### Problema 1: El usuario creado recibe error 403 al intentar conectarse a `cbq`

**Síntoma:**

```
Error connecting to cbq: 403 Forbidden
```

o bien:

```
cbq: ERROR 401 - {"errors":[{"code":10000,"msg":"Unauthenticated"}]}
```

**Causa:** La contraseña del usuario no cumple con la política de contraseñas de Couchbase (por defecto requiere al menos 6 caracteres; en algunas configuraciones puede exigir mayúsculas, minúsculas y caracteres especiales). Alternativamente, el usuario fue creado con un error tipográfico en el nombre o la contraseña.

**Solución:**

1. Verifica que el usuario existe:
```bash
curl -s -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/settings/rbac/users/local" \
  | python3 -m json.tool | grep '"id"'
```

2. Si el usuario existe pero la contraseña falla, actualiza la contraseña:
```bash
curl -s -X PATCH \
  -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/settings/rbac/users/local/app-readonly" \
  -d "password=Readonly@2024!"
```

3. Verifica la política de contraseñas del clúster:
```bash
curl -s -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/settings/passwordPolicy" \
  | python3 -m json.tool
```

4. Intenta la conexión nuevamente con `cbq`:
```bash
cbq -u app-readonly -p "Readonly@2024!" -engine=http://${CB_HOST}:8093
```

---

### Problema 2: Las consultas SQL++ fallan con error de índice faltante (`PRIMARY KEY scan`) aunque el usuario tiene permisos correctos

**Síntoma:**

```json
{
    "errors": [
        {
            "code": 4000,
            "msg": "No index available on keyspace ecommerce.catalog.products that matches your query..."
        }
    ],
    "status": "fatal"
}
```

**Causa:** Las collections `catalog.products`, `catalog.categories`, `orders.purchases` y `orders.customers` no tienen un índice primario (`PRIMARY INDEX`) creado. Sin este índice, las consultas `SELECT` con `FROM` directo no pueden ejecutarse aunque el usuario tenga el rol `query_select`. El error de índice ocurre antes de la validación de permisos, por lo que puede confundirse con un problema de RBAC.

**Solución:**

Crea índices primarios en todas las collections usando el usuario administrador:

```bash
cbq -u ${CB_ADMIN} -p ${CB_PASS} -engine=http://${CB_HOST}:8093 \
  --script="CREATE PRIMARY INDEX ON ecommerce.catalog.products;"

cbq -u ${CB_ADMIN} -p ${CB_PASS} -engine=http://${CB_HOST}:8093 \
  --script="CREATE PRIMARY INDEX ON ecommerce.catalog.categories;"

cbq -u ${CB_ADMIN} -p ${CB_PASS} -engine=http://${CB_HOST}:8093 \
  --script="CREATE PRIMARY INDEX ON ecommerce.orders.purchases;"

cbq -u ${CB_ADMIN} -p ${CB_PASS} -engine=http://${CB_HOST}:8093 \
  --script="CREATE PRIMARY INDEX ON ecommerce.orders.customers;"
```

> **Nota:** Los índices primarios son útiles para desarrollo y pruebas, pero en producción se deben usar índices secundarios (GSI) específicos para las consultas más frecuentes. Vuelve a ejecutar las consultas de verificación de permisos después de crear los índices.

---

## Limpieza del Entorno

Una vez completado el laboratorio, puedes limpiar los recursos creados para liberar espacio y evitar conflictos con futuros laboratorios.

> ⚠️ **Advertencia:** Ejecuta los comandos de limpieza solo si has completado todas las verificaciones y no necesitas el entorno para revisiones posteriores.

**Eliminar usuarios creados:**

```bash
for user in app-readonly app-backend app-admin reporter-maria reporter-jose; do
  curl -s -X DELETE \
    -u ${CB_ADMIN}:${CB_PASS} \
    "${CB_URL}/settings/rbac/users/local/${user}"
  echo "Usuario ${user} eliminado."
done
```

**Eliminar el grupo `reporting-team`:**

```bash
curl -s -X DELETE \
  -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/settings/rbac/groups/reporting-team"
echo "Grupo reporting-team eliminado."
```

**Eliminar el bucket `ecommerce` (elimina todos los scopes, collections y datos):**

```bash
curl -s -X DELETE \
  -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/pools/default/buckets/ecommerce"
echo "Bucket ecommerce eliminado."
```

**Verificar que el bucket fue eliminado:**

```bash
curl -s -u ${CB_ADMIN}:${CB_PASS} \
  "${CB_URL}/pools/default/buckets" \
  | python3 -m json.tool | grep '"name"'
```

El bucket `ecommerce` no debe aparecer en la lista.

---

## Resumen

En este laboratorio implementaste un sistema completo de Control de Acceso Basado en Roles (RBAC) en Couchbase Server 7.6.x, aplicando los principios aprendidos en la Lección 9.1:

| Concepto aplicado | Implementación realizada |
|-------------------|--------------------------|
| **Principio de mínimo privilegio** | Cada usuario recibió solo los roles necesarios para su función específica |
| **Granularidad de roles** | Roles asignados a nivel de scope (`catalog`, `orders`) y bucket (`ecommerce`) |
| **Separación de responsabilidades** | Tres perfiles diferenciados: reporting (solo lectura), backend (lectura/escritura mixta) y admin (control total) |
| **Gestión centralizada con grupos** | Grupo `reporting-team` con herencia de permisos para múltiples usuarios |
| **Verificación de control de acceso** | Pruebas positivas y negativas con `cbq` y la Web Console |
| **Múltiples interfaces de gestión** | REST API, Web Console y `cbq` para crear y verificar usuarios/roles |

### Roles de Couchbase utilizados

| Rol | Descripción | Alcance usado |
|-----|-------------|---------------|
| `data_reader` | Lectura de documentos vía KV | Scope |
| `data_writer` | Escritura de documentos vía KV | Scope |
| `query_select` | Sentencias SELECT en SQL++ | Scope |
| `query_insert` | Sentencias INSERT en SQL++ | Scope |
| `query_update` | Sentencias UPDATE en SQL++ | Scope |
| `bucket_admin` | Administración completa del bucket | Bucket |
| `query_manage_index` | Gestión de índices GSI | Bucket |

### Recursos adicionales

- [Documentación oficial de RBAC en Couchbase Server 7.6](https://docs.couchbase.com/server/current/learn/security/roles.html)
- [Referencia de roles predefinidos de Couchbase](https://docs.couchbase.com/server/current/learn/security/roles.html#roles-reference)
- [Gestión de usuarios y grupos vía REST API](https://docs.couchbase.com/server/current/rest-api/rbac.html)
- [Mejores prácticas de seguridad en Couchbase](https://docs.couchbase.com/server/current/learn/security/security-best-practices.html)
- [Integración con LDAP para usuarios externos](https://docs.couchbase.com/server/current/manage/manage-security/configure-ldap.html)

---
LAB_END---
