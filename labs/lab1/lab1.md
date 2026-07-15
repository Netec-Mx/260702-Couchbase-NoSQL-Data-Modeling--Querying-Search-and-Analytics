---
layout: lab
title: "Práctica 1: Análisis de casos de uso NoSQL vs Relacional"
permalink: /lab1/lab1/
images_base: /labs/lab1/img
duration: "25 minutos"
objective:
  - Identificar limitaciones comunes de una base de datos relacional en aplicaciones modernas.
  - Proponer una estructura documental JSON como alternativa a un modelo altamente normalizado.
  - Diferenciar los tipos principales de NoSQL: clave-valor, documental, columnar y grafos.
  - Aplicar el Teorema CAP para decidir cuándo priorizar consistencia o disponibilidad.
prerequisites:
  - Conocimientos básicos de bases de datos relacionales.
  - Conocimientos básicos de SQL, especialmente SELECT, JOIN y WHERE.
  - Comprensión básica de estructuras JSON.
  - Nociones generales de arquitectura de aplicaciones.
introduction:
  - En esta práctica vas a analizar cuándo conviene usar una base de datos NoSQL en lugar de una base de datos relacional. Trabajarás con un caso de negocio sencillo, identificarás limitaciones del modelo relacional, propondrás una estructura JSON desnormalizada, clasificarás escenarios por tipo de NoSQL y aplicarás el Teorema CAP para tomar decisiones de diseño. Esta práctica es introductoria y no requiere instalación de software.
slug: lab1
lab_number: 1
final_result: >
  Al finalizar la práctica tendrás una tabla de problemas relacionales identificados, un documento JSON desnormalizado para representar un hotel, una clasificación de escenarios por tipo de NoSQL y una clasificación CAP básica aplicada a situaciones de negocio.
notes:
  - Esta práctica es de análisis guiado y no requiere Couchbase Server, Docker ni herramientas adicionales.
  - Puedes realizar las actividades en un editor de texto, cuaderno, documento digital o pizarra.
  - Las soluciones de referencia deben revisarse al final, después de completar tus respuestas.
references: []
prev: /
next: /lab2/lab2/
---

<!-- Aquí comienzan las instrucciones paso a paso de la práctica -->
---

## Distribución del tiempo

| Tarea | Actividad | Tiempo sugerido |
|---|---|---:|
| Tarea 1 | Analizar el problema relacional de ViajeYA | 7 min |
| Tarea 2 | Proponer un documento JSON desnormalizado | 6 min |
| Tarea 3 | Clasificar escenarios por tipo de NoSQL | 7 min |
| Tarea 4 | Aplicar decisiones CAP | 5 min |
| **Total** | &nbsp; | **25 min** |

## Entregables de la práctica

Al finalizar, deberás tener completos los siguientes entregables:

| Entregable | Tarea |
|---|---|
| Tabla de problemas relacionales identificados | Tarea 1 |
| Documento JSON desnormalizado para un hotel | Tarea 2 |
| Tabla de clasificación de 4 escenarios NoSQL | Tarea 3 |
| Clasificación CAP de 3 escenarios | Tarea 4 |

---

## Tarea 1. Analizar el problema relacional de ViajeYA

En esta tarea vas a revisar un caso de negocio donde una plataforma de reservas creció rápidamente y su modelo relacional comenzó a presentar problemas de rendimiento, flexibilidad y escalabilidad.

### Tarea 1.1. Comprender el contexto del problema

ViajeYA es una plataforma de reservas de viajes. Hace tres años procesaba cerca de **10 000 reservas diarias**. Actualmente, después de una campaña viral, procesa aproximadamente **2 millones de reservas diarias**.

El equipo de ingeniería reporta los siguientes síntomas:

- Las búsquedas de disponibilidad tardan entre 4 y 8 segundos en horas pico.
- Las migraciones de esquema para agregar nuevos tipos de alojamiento requieren ventanas de mantenimiento nocturnas.
- El servidor principal opera al 95 % de CPU durante los picos de demanda.
- Las réplicas de lectura muestran datos ligeramente desactualizados.

- {% include step_label.html %} Lee el contexto anterior y subraya los síntomas que están relacionados con rendimiento.
- {% include step_label.html %} Identifica los síntomas que están relacionados con cambios frecuentes en el modelo de datos.
- {% include step_label.html %} Identifica los síntomas que están relacionados con consistencia o datos desactualizados.

> **NOTA:** En esta práctica no necesitas resolver el problema como DBA. Solo debes identificar por qué el modelo actual puede dejar de ser adecuado cuando aumentan el volumen, la velocidad y la variedad de los datos.
{: .lab-note .info .compact}

### Tarea 1.2. Revisar el modelo relacional actual

El modelo relacional actual separa la información de hoteles, habitaciones y disponibilidad en tres tablas.

```sql
CREATE TABLE hotels (
    id          INT PRIMARY KEY,
    name        VARCHAR(200),
    city        VARCHAR(100),
    country     VARCHAR(100),
    stars       INT,
    amenities   TEXT
);

CREATE TABLE rooms (
    id          INT PRIMARY KEY,
    hotel_id    INT REFERENCES hotels(id),
    type        VARCHAR(50),
    capacity    INT,
    price       DECIMAL(10,2)
);

CREATE TABLE availability (
    id           INT PRIMARY KEY,
    room_id      INT REFERENCES rooms(id),
    date         DATE,
    is_available BOOLEAN
);
```

- {% include step_label.html %} Revisa la tabla `hotels` e identifica qué información general guarda del hotel.
- {% include step_label.html %} Revisa la tabla `rooms` e identifica cómo se relacionan las habitaciones con los hoteles.
- {% include step_label.html %} Revisa la tabla `availability` e identifica por qué puede crecer muy rápido.
- {% include step_label.html %} Observa el campo `amenities` y responde por qué guardar JSON como `TEXT` puede limitar las consultas.

> **IMPORTANTE:** El campo `amenities` está guardado como texto plano. Eso significa que la base de datos no puede consultar fácilmente amenidades específicas como `wifi`, `pool` o `parking` sin aplicar procesamiento adicional.
{: .lab-note .important .compact}

### Tarea 1.3. Analizar la consulta problemática

La siguiente consulta busca disponibilidad en Barcelona para un rango de fechas:

```sql
EXPLAIN ANALYZE
SELECT h.name, r.type, r.price
FROM hotels h
JOIN rooms r ON h.id = r.hotel_id
JOIN availability a ON r.id = a.room_id
WHERE a.date BETWEEN '2024-12-20' AND '2024-12-27'
  AND h.city = 'Barcelona';
```

El resultado simplificado del plan de ejecución es el siguiente:

```text
Nested Loop
  -> Seq Scan on availability  (rows=45,000,000)
  -> Index Scan on rooms       (rows=3,000,000)
  -> Filter on hotels.city     (rows=500,000)

Execution Time: 6847.4 ms
```

- {% include step_label.html %} Localiza la operación `Seq Scan on availability`.
- {% include step_label.html %} Interpreta el número de filas revisadas: `45,000,000`.
- {% include step_label.html %} Explica por qué revisar una tabla tan grande puede provocar tiempos de respuesta de varios segundos.
- {% include step_label.html %} Relaciona el uso de varios `JOINs` con el aumento de costo de la consulta.

> **NOTA:** Un `Seq Scan` significa que la base de datos está revisando secuencialmente muchas filas. En cargas altas, esto puede consumir CPU, memoria y tiempo de respuesta.
{: .lab-note .info .compact}

### Tarea 1.4. Completar la tabla de análisis

Completa la siguiente tabla con tus propias palabras.

| Síntoma observado | Causa raíz probable | Categoría |
|---|---|---|
| La búsqueda tarda entre 4 y 8 segundos | &nbsp; | Escala / Velocidad / Variedad |
| Las migraciones requieren ventanas de mantenimiento | &nbsp; | Escala / Velocidad / Variedad |
| El servidor opera al 95 % de CPU | &nbsp; | Escala / Velocidad / Variedad |
| Las réplicas muestran datos desactualizados | &nbsp; | Escala / Velocidad / Variedad |

Después responde brevemente:

1. ¿Qué significa `Seq Scan on availability`?
2. ¿Por qué revisar 45 millones de filas puede afectar el tiempo de respuesta?
3. ¿Qué problema genera guardar `amenities` como texto plano?
4. ¿Qué pasaría si mañana se agrega un nuevo tipo de alojamiento llamado `glamping` con atributos propios?

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea 1 finalizada" content=r1 %}

---

## Tarea 2. Proponer un documento JSON desnormalizado

En esta tarea vas a diseñar una estructura JSON sencilla para representar un hotel, sus habitaciones y su disponibilidad dentro de un mismo documento.

### Tarea 2.1. Identificar qué información se puede agrupar

En el modelo relacional anterior, la información está distribuida en tres tablas:

- `hotels`
- `rooms`
- `availability`

En un modelo documental puedes guardar información relacionada dentro de un mismo documento JSON.

- {% include step_label.html %} Identifica qué datos de la tabla `hotels` pertenecen al hotel.
- {% include step_label.html %} Identifica qué datos de la tabla `rooms` pertenecen a cada habitación.
- {% include step_label.html %} Identifica qué datos de la tabla `availability` pueden quedar integrados como fechas disponibles.
- {% include step_label.html %} Decide qué campos deberían ser objetos o arreglos dentro del documento JSON.

> **IMPORTANTE:** Desnormalizar no significa copiar datos sin control. Significa organizar los datos de acuerdo con la forma en que la aplicación los consulta con mayor frecuencia.
{: .lab-note .important .compact}

### Tarea 2.2. Crear la estructura base del hotel

Crea un documento JSON con los datos generales del hotel.

```json
{
  "type": "hotel",
  "id": "hotel_001",
  "name": "Hotel Mediterráneo",
  "location": {
    "city": "Barcelona",
    "country": "España"
  },
  "stars": 4
}
```

- {% include step_label.html %} Copia la estructura base en tu editor de texto o cuaderno.
- {% include step_label.html %} Cambia el nombre del hotel si lo deseas.
- {% include step_label.html %} Conserva el objeto `location`, porque permite agrupar ciudad y país de forma clara.

### Tarea 2.3. Convertir amenidades en objeto JSON

Agrega las amenidades como un objeto JSON consultable.

```json
"amenities": {
  "wifi": true,
  "pool": true,
  "parking": false,
  "breakfast_included": true
}
```

- {% include step_label.html %} Agrega el objeto `amenities` dentro del documento del hotel.
- {% include step_label.html %} Usa valores booleanos para indicar si una amenidad existe o no.
- {% include step_label.html %} Agrega al menos 4 amenidades.

> **NOTA:** Modelar `amenities` como objeto permite consultar campos específicos. Por ejemplo, podrías buscar hoteles donde `wifi = true` y `parking = true`.
{: .lab-note .info .compact}

### Tarea 2.4. Agregar habitaciones y disponibilidad

Agrega un arreglo llamado `rooms` con al menos dos habitaciones. Cada habitación debe incluir sus fechas disponibles.

```json
"rooms": [
  {
    "room_id": "room_001",
    "type": "standard",
    "capacity": 2,
    "price_per_night": 120.00,
    "available_dates": [
      "2024-12-20",
      "2024-12-21",
      "2024-12-22"
    ]
  },
  {
    "room_id": "room_002",
    "type": "glamping",
    "capacity": 4,
    "price_per_night": 180.00,
    "tent_material": "canvas",
    "max_wind_speed": 70,
    "eco_certification": true,
    "available_dates": [
      "2024-12-20",
      "2024-12-23",
      "2024-12-24"
    ]
  }
]
```

- {% include step_label.html %} Agrega el arreglo `rooms` dentro del documento del hotel.
- {% include step_label.html %} Incluye al menos una habitación estándar.
- {% include step_label.html %} Incluye una habitación tipo `glamping` con campos adicionales.
- {% include step_label.html %} Agrega `available_dates` como arreglo de fechas en cada habitación.
- {% include step_label.html %} Verifica que el documento JSON completo conserve llaves, comas y corchetes correctamente.

### Tarea 2.5. Reflexionar sobre ventajas y riesgos

Responde brevemente:

1. ¿Qué `JOINs` se eliminan al guardar habitaciones dentro del documento del hotel?
2. ¿Por qué `amenities` como objeto JSON es mejor que `amenities` como texto?
3. ¿Qué ventaja tiene permitir campos adicionales en una habitación tipo `glamping`?
4. ¿Qué posible desventaja tendría guardar demasiada información dentro de un solo documento?

Marca cada punto cuando lo hayas completado:

| Criterio | Completado |
|---|---|
| El JSON tiene sintaxis válida | <input type="checkbox" disabled> |
| El hotel tiene datos generales | <input type="checkbox" disabled> |
| `amenities` está modelado como objeto JSON | <input type="checkbox" disabled> |
| Las habitaciones están dentro de un arreglo | <input type="checkbox" disabled> |
| La disponibilidad está integrada en cada habitación | <input type="checkbox" disabled> |
| Existe al menos un tipo de alojamiento con atributos propios | <input type="checkbox" disabled> |

{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea 2 finalizada" content=r2 %}

---

## Tarea 3. Clasificar escenarios por tipo de NoSQL

En esta tarea vas a seleccionar el tipo de base de datos NoSQL más adecuado según el patrón de acceso y el problema de negocio.

### Tarea 3.1. Revisar los tipos principales de NoSQL

Antes de resolver los escenarios, revisa esta tabla de referencia.

| Tipo de NoSQL | Modelo de datos | Fortaleza principal | Caso típico |
|---|---|---|---|
| **Clave-Valor** | Par `key → value` | Acceso muy rápido por clave | Sesiones, caché, carritos |
| **Documental** | Documentos JSON/BSON | Flexibilidad de esquema y consultas por campos | Catálogos, perfiles, contenido |
| **Columnar** | Familias de columnas | Escrituras masivas y consultas por rango | IoT, logs, series temporales |
| **Grafos** | Nodos y relaciones | Recorrido de relaciones complejas | Redes sociales, fraude, recomendaciones |

- {% include step_label.html %} Lee la descripción de cada tipo de NoSQL.
- {% include step_label.html %} Observa que clave-valor y documental pueden consultar por ID, pero no resuelven el mismo problema.
- {% include step_label.html %} Distingue si el escenario necesita buscar por clave, consultar campos internos, escribir eventos masivos o recorrer relaciones.

> **NOTA:** No respondas solo “porque es rápido”. Explica qué característica técnica del modelo de datos ayuda al caso.
{: .lab-note .info .compact}

### Tarea 3.2. Analizar los escenarios

Lee los siguientes escenarios y selecciona el tipo principal de NoSQL para cada uno.

**Escenario A: Carrito de compras**

Una tienda en línea necesita guardar el carrito de compras de cada usuario. El carrito se actualiza con cada clic, expira después de 24 horas y debe recuperarse en menos de 10 ms usando el ID del usuario.

**Escenario B: Catálogo de productos**

Una tienda vende electrónicos, ropa, alimentos y muebles. Cada categoría tiene atributos diferentes. Por ejemplo, un televisor tiene resolución y HDR, mientras que una camiseta tiene talla y material.

**Escenario C: Telemetría IoT**

Una fábrica tiene 10 000 sensores que envían temperatura, presión y vibración cada 5 segundos. Las consultas principales son por sensor y rango de tiempo.

**Escenario D: Red social profesional**

Una red social necesita calcular “personas que quizá conozcas”. Para hacerlo, debe recorrer amigos de amigos, conexiones por empresa, industria y ciudad.

- {% include step_label.html %} Para cada escenario, identifica primero el patrón de acceso principal.
- {% include step_label.html %} Selecciona un tipo de NoSQL.
- {% include step_label.html %} Escribe una justificación breve de 1 o 2 oraciones.
- {% include step_label.html %} Evita mezclar todos los escenarios con el mismo tipo de base de datos.

### Tarea 3.3. Completar la tabla de clasificación

Completa la siguiente tabla:

| Escenario | Tipo NoSQL elegido | Justificación |
|---|---|---|
| A. Carrito de compras | &nbsp; | &nbsp; |
| B. Catálogo de productos | &nbsp; | &nbsp; |
| C. Telemetría IoT | &nbsp; | &nbsp; |
| D. Red social profesional | &nbsp; | &nbsp; |

Marca cada punto cuando lo hayas completado:

| Criterio | Completado |
|---|---|
| Clasifiqué los 4 escenarios | <input type="checkbox" disabled> |
| Justifiqué cada selección con una razón técnica | <input type="checkbox" disabled> |
| Diferencié acceso por clave de consulta por campos | <input type="checkbox" disabled> |
| Identifiqué cuándo las relaciones complejas requieren grafos | <input type="checkbox" disabled> |

{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea 3 finalizada" content=r3 %}

---

## Tarea 4. Aplicar decisiones CAP

En esta tarea vas a aplicar el Teorema CAP para decidir si un sistema distribuido debe priorizar consistencia o disponibilidad.

### Tarea 4.1. Revisar el marco de decisión CAP

El Teorema CAP indica que, en un sistema distribuido, no siempre puedes garantizar al mismo tiempo las siguientes propiedades:

| Propiedad | Significado |
|---|---|
| **C - Consistencia** | Todos los nodos ven el dato más reciente |
| **A - Disponibilidad** | El sistema responde a las solicitudes |
| **P - Tolerancia a particiones** | El sistema sigue operando aunque haya fallas de red |

En sistemas distribuidos reales, la tolerancia a particiones suele ser obligatoria. Por eso, la decisión práctica suele ser entre **CP** y **AP**.

| Elección | Significado |
|---|---|
| **CP** | Prefieres consistencia, aunque algunas solicitudes puedan rechazarse temporalmente |
| **AP** | Prefieres disponibilidad, aunque algunos datos puedan estar ligeramente desactualizados |

- {% include step_label.html %} Lee la diferencia entre CP y AP.
- {% include step_label.html %} Piensa en el costo de mostrar, guardar o confirmar datos incorrectos.
- {% include step_label.html %} Recuerda que CP puede sacrificar disponibilidad durante una falla de red.
- {% include step_label.html %} Recuerda que AP puede responder más rápido, pero aceptar consistencia eventual.

> **IMPORTANTE:** La pregunta clave no es “¿cuál es mejor?”, sino “¿cuál es el costo de una inconsistencia temporal para el negocio?”.
{: .lab-note .important .compact}

### Tarea 4.2. Clasificar escenarios CAP

Lee cada escenario y decide si conviene priorizar **CP** o **AP**.

**Escenario CAP-1: Transferencia bancaria**

Un banco procesa transferencias entre cuentas. Dos cajeros automáticos consultan la misma cuenta al mismo tiempo.

**Pregunta:** ¿Sería aceptable mostrar un saldo desactualizado o permitir un doble débito?

**Escenario CAP-2: Feed de red social**

Un usuario publica una foto y sus seguidores la verán en su feed.

**Pregunta:** ¿Es crítico que todos los usuarios vean la publicación exactamente al mismo tiempo?

**Escenario CAP-3: Últimos asientos de un vuelo**

Una aerolínea vende los últimos 3 asientos de un vuelo. Dos agentes intentan reservar el mismo asiento al mismo tiempo.

**Pregunta:** ¿Sería aceptable vender el mismo asiento dos veces?

- {% include step_label.html %} Lee el primer escenario y clasifícalo como CP o AP.
- {% include step_label.html %} Escribe una justificación de una oración.
- {% include step_label.html %} Repite el análisis para los otros dos escenarios.
- {% include step_label.html %} Usa el costo de inconsistencia como criterio principal.

### Tarea 4.3. Completar la tabla CAP

Completa la siguiente tabla:

| Escenario | Clasificación CP/AP | Justificación |
|---|---|---|
| Transferencia bancaria | &nbsp; | &nbsp; |
| Feed de red social | &nbsp; | &nbsp; |
| Últimos asientos de un vuelo | &nbsp; | &nbsp; |

Después responde brevemente:

> ¿Por qué no todos los sistemas deberían elegir siempre CP?

Pista: piensa en aplicaciones donde responder rápido es más importante que mostrar el dato perfectamente actualizado.

Marca cada punto cuando lo hayas completado:

| Criterio | Completado |
|---|---|
| Clasifiqué los 3 escenarios como CP o AP | <input type="checkbox" disabled> |
| Justifiqué cada decisión con base en el costo de la inconsistencia | <input type="checkbox" disabled> |
| Entendí que CP puede reducir disponibilidad | <input type="checkbox" disabled> |
| Entendí que AP acepta consistencia eventual | <input type="checkbox" disabled> |

{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea 4 finalizada" content=r4 %}

---

## Validación final de la práctica

Antes de finalizar, confirma que tienes completos los siguientes entregables:

| Entregable | Completado |
|---|---|
| Tabla de análisis de problemas relacionales de ViajeYA | <input type="checkbox" disabled> |
| Documento JSON desnormalizado de hotel | <input type="checkbox" disabled> |
| Tabla de clasificación de 4 escenarios NoSQL | <input type="checkbox" disabled> |
| Tabla de clasificación CAP de 3 escenarios | <input type="checkbox" disabled> |

## Criterios de evaluación

| Criterio | Puntos |
|---|---:|
| Identifica correctamente el cuello de botella relacional | 25 |
| Propone un documento JSON coherente y válido | 25 |
| Clasifica correctamente los tipos de NoSQL | 25 |
| Aplica correctamente CP/AP en escenarios de negocio | 25 |
| **Total** | **100** |