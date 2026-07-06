# S15 — Cassandra (BD No SQL 3) ⭐ FIJA

> **Anuncio explícito del profe**: estudiar (1) definición del modelo de datos, (2) arquitectura distribuida, (3) cómo se construye una tabla, (4) composición de la llave primaria, (5) consultas admisibles, (6) casos de aplicación.

---

## 1. Modelo de datos: Wide-Column Store

Cassandra es una **BD de columna ancha** inspirada en Google BigTable. Almacena datos en **familias de columnas** en lugar de filas de esquema rígido como los RDBMS.

### Jerarquía

```
Cluster
   └── Keyspace           (≈ schema en RDBMS)
         └── Table         (Column Family)
               └── Partition   (colección de filas con misma partition key)
                     └── Row   (identificada por PK)
                           └── Column  (nombre + valor + timestamp)
```

Representación interna simplificada:
```
Map< RowKey, SortedMap< ColumnKey, ColumnValue > >
```

### Diferencia con RDBMS (¡importante!)

- En RDBMS: agregar una columna nueva obliga a rellenar con NULL todas las filas.
- En Cassandra: **schema flexible** — se agregan columnas sin tocar las filas existentes. Cada fila puede tener columnas distintas.

### Comparación rápida

| Aspecto | RDBMS | MongoDB (Doc) | **Cassandra (Wide-Column)** |
|---|---|---|---|
| Modelo | Tablas rígidas | Documentos JSON | Familia de columnas |
| Schema | Fijo | Flexible | Flexible (por fila) |
| Escala | Vertical | Horizontal (sharding) | **Horizontal P2P** |
| Consistencia | ACID | CP | **AP (eventual)** |
| Failover | Single point | Elección primary | **Sin punto único** |

---

## 2. Arquitectura distribuida

### Peer-to-peer / masterless

- **No hay nodo maestro**. Todos los nodos son iguales.
- **Cualquier nodo** puede recibir lecturas y escrituras — el que recibe la petición se llama **coordinador** para esa operación.
- Mayor tolerancia a fallos: caída de un nodo ≠ pérdida de servicio.

### Ring + Consistent Hashing

Los nodos se organizan en un **anillo lógico**. A cada nodo se le asigna un rango de tokens. La partition key de una fila se hashea y su valor cae en algún punto del anillo → determina qué nodo la almacena.

```
        Nodo A (tokens 0..100)
             /               \
    Nodo D                    Nodo B
    (300..400)                (100..200)
             \               /
        Nodo C (tokens 200..300)
```

- **Virtual nodes (vnodes)**: cada nodo físico posee muchos rangos pequeños en vez de uno grande → mejor balance y rebalanceo tras caídas o joins.
- **Consistent hashing** significa que agregar/quitar un nodo solo mueve una fracción de los datos, no todos.

### Replicación

- **Replication Factor (RF)**: cuántas copias de cada dato hay. Ej. `RF=3`.
- Las réplicas van a los siguientes N-1 nodos del anillo (o según topología con Snitch).

```cql
CREATE KEYSPACE mi_ks WITH replication = {
  'class': 'SimpleStrategy',
  'replication_factor': 3
};
```

Estrategias:
- `SimpleStrategy`: para desarrollo (un solo DC).
- `NetworkTopologyStrategy`: producción multi-DC.

### Gossip protocol

Los nodos intercambian metadatos de estado periódicamente entre sí (P2P). Así se enteran de nodos caídos, cambios de topología, tokens, etc. **Sin coordinador central**.

### Snitch

Módulo que le dice al cluster qué nodo está en qué DC / rack. Se usa para colocar réplicas de forma segura (que no queden todas en el mismo rack, por ejemplo).

### Consistency Levels (CAP en la práctica)

Se configura por operación:

| Nivel | Lectura/Escritura | Uso |
|---|---|---|
| `ONE` | 1 réplica confirma | Máxima velocidad, menor consistencia |
| `QUORUM` | Mayoría `⌊RF/2⌋+1` | **Balance** — con RF=3 requiere 2 |
| `LOCAL_QUORUM` | Quorum solo en DC local | Multi-DC con baja latencia |
| `ALL` | Todas las réplicas | Máxima consistencia, menor disponibilidad |

Regla: `R + W > RF` garantiza **strong consistency** para una clave (aunque siga siendo eventual a nivel global). Ej. RF=3, R=QUORUM(2), W=QUORUM(2) → 2+2 > 3 ✅.

Cassandra es **AP** por defecto (siempre responde, eventual).

---

## 3. Cómo se construye una tabla (CQL)

```cql
CREATE TABLE sensor_data (
  Sensor    int,
  Date      date,
  Timestamp timestamp,
  Speed     float,
  Torque    float,
  Power     float,
  PRIMARY KEY ((Sensor, Date), Timestamp)
) WITH CLUSTERING ORDER BY (Timestamp ASC);
```

Tipos comunes: `int`, `bigint`, `float`, `decimal`, `text`, `varchar`, `date`, `timestamp`, `uuid`, `timeuuid`, `boolean`, `blob`.
Colecciones: `list<T>`, `set<T>`, `map<K,V>`.

---

## 4. Composición de la llave primaria ⭐

```
PRIMARY KEY ( ( Sensor, Date ),  Timestamp )
             |__________________|  |________|
                partition key       clustering key
```

### Partition key
- Puede ser **compuesta** (varias columnas entre paréntesis).
- Su hash determina **en qué nodo** vive la fila.
- Todas las filas con la misma partition key viven en la **misma partición** (mismo nodo).
- **Debe aparecer completa en el WHERE** de cualquier query eficiente.

### Clustering key
- Determina el **orden físico** dentro de la partición.
- Permite consultas por rango (`>=`, `<=`, `BETWEEN`).
- `CLUSTERING ORDER BY` define ASC/DESC.

### Ejemplo mental

Con `PRIMARY KEY ((Sensor, Date), Timestamp)`:

```
Nodo X:
  Partition (Sensor=1, Date=2025-01-01):
    Timestamp 08:00:00 → Speed, Torque, Power
    Timestamp 08:00:10 → Speed, Torque, Power
    Timestamp 08:00:20 → Speed, Torque, Power
    ...

Nodo Y:
  Partition (Sensor=2, Date=2025-01-01):
    Timestamp 08:00:00 → ...
```

Si haces `SELECT ... WHERE Sensor=1 AND Date='2025-01-01' AND Timestamp BETWEEN '08:00:00' AND '09:00:00'` → una sola partición, un solo nodo, orden secuencial. Perfecto.

---

## 5. Consultas admisibles ⭐

### ✅ Válidas

```cql
-- Partition key completa
SELECT * FROM sensor_data
WHERE Sensor = 1 AND Date = '2025-01-01';

-- Partition key + rango en clustering
SELECT * FROM sensor_data
WHERE Sensor = 1 AND Date = '2025-01-01'
  AND Timestamp >= '2025-01-01 08:00:00'
  AND Timestamp <  '2025-01-01 09:00:00';

-- Partition key + orden explícito (invertido)
SELECT * FROM sensor_data
WHERE Sensor = 1 AND Date = '2025-01-01'
ORDER BY Timestamp DESC;
```

### ❌ No válidas (o requieren `ALLOW FILTERING`)

```cql
-- Falta parte de la partition key
SELECT * FROM sensor_data WHERE Sensor = 1;   -- ERROR: falta Date

-- Filtro por columna que no es PK
SELECT * FROM sensor_data
WHERE Sensor = 1 AND Date = '2025-01-01' AND Speed > 5.0;  -- OK si Speed está en PK, si no requiere ALLOW FILTERING

-- Solo clustering key
SELECT * FROM sensor_data WHERE Timestamp > '...';  -- ERROR (recorrería todos los nodos)
```

### ALLOW FILTERING

Fuerza a Cassandra a escanear múltiples particiones. **Muy costoso** — solo para exploración o datasets pequeños. En producción, **modelar la tabla en función de las consultas** (query-driven modeling).

### INSERT / UPDATE

```cql
INSERT INTO sensor_data (Sensor, Date, Timestamp, Speed, Torque, Power)
VALUES (1, '2025-01-01', toTimestamp(now()), 45.2, 12.1, 200.0);

UPDATE sensor_data
SET Speed = 46.0
WHERE Sensor = 1 AND Date = '2025-01-01' AND Timestamp = '...';
```

**No hay JOINs**. Se **desnormaliza** duplicando datos en múltiples tablas, una por patrón de consulta.

---

## 6. Casos de aplicación

### ✅ Ideal para

| Caso | Por qué |
|---|---|
| **Time-series / IoT** | Millones de sensores escribiendo simultáneamente. PK = (sensor, día) + timestamp |
| **Logs de auditoría** | Alta escritura, consultas por rango de fecha por usuario/entidad |
| **Mensajería / feeds** | PK = usuario, clustering = timestamp DESC (últimos mensajes primero) |
| **Detección de fraude** | PK = id_usuario, clustering = timestamp DESC, consultas últimas 24h por usuario |
| **Métricas de aplicación** | Ingesta continua de eventos, dashboards por rango de tiempo |
| **Real-time analytics** | Escritura brutalmente rápida, lecturas por partition key |

### ❌ Mal encaje

- Transacciones ACID complejas (aunque hay `LWT` con Paxos, es caro).
- Muchos JOINs.
- Queries ad-hoc que no conocen la partition key.
- Datos altamente relacionados con integridad referencial estricta.

---

## 7. Ejemplo trabajado: detección de fraude en pagos

```cql
CREATE TABLE transacciones (
  id_usuario   uuid,
  timestamp    timestamp,
  monto        decimal,
  ubicacion    text,
  metodo_pago  text,
  PRIMARY KEY (id_usuario, timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);

-- Consulta: transacciones del usuario en las últimas 24h
SELECT * FROM transacciones
WHERE id_usuario = <UUID>
  AND timestamp >= '<hace 24h>';
```

- Todas las transacciones de un usuario viven juntas (misma partición) → escala a millones de usuarios (distribuidas entre nodos).
- Clustering DESC → las más recientes primero.
- Range query eficiente.

---

## 8. Preguntas típicas (autotest)

1. ¿Cuál es la diferencia entre partition key y clustering key? ¿Cómo afecta cada una al almacenamiento?
2. Dado `PRIMARY KEY ((a, b), c, d)`, ¿qué consultas son válidas sin `ALLOW FILTERING`?
3. ¿Qué es el gossip protocol y por qué Cassandra puede prescindir de un master?
4. Explica `R + W > RF` con RF=3 y da un ejemplo.
5. Cita 2 similitudes y 2 diferencias entre Cassandra y MongoDB.
6. Diseña una tabla para almacenar posts de una red social donde se consulta "los últimos 20 posts de un usuario".
7. ¿Por qué Cassandra no soporta JOIN y qué se hace en su lugar?
8. Un cliente reporta que su query `WHERE ciudad='Lima'` es muy lenta. La tabla tiene `PRIMARY KEY ((user_id), timestamp)`. ¿Qué está mal?
