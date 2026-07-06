# S14 — MongoDB (BD No SQL 2) ⭐ FIJA

> **Anuncio del profe**: interpretar consultas MongoDB — decir qué hace la aggregation.

---

## 1. Modelo Documento

- Almacena **documentos** en formato **BSON** (Binary JSON): extensión binaria de JSON con tipos adicionales (`Date`, `ObjectId`, binario, decimales).
- Documentos se agrupan en **colecciones** (≈ tabla).
- **Schema flexible**: cada documento puede tener campos distintos.

### Traducción de conceptos

| SQL | MongoDB |
|---|---|
| Database | Database |
| Table | **Collection** |
| Row | **Document** |
| Column | **Field** |
| JOIN | `$lookup` (aggregation) |
| Primary key | `_id` (auto `ObjectId` si no se especifica) |

### Ejemplo de documento

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "nombre": "Rubén",
  "edad": 32,
  "expertise": ["MongoDB", "NoSQL"],
  "direccion": {
    "calle": "Rutilo 11",
    "ciudad": "Madrid",
    "cp": "28041"
  }
}
```

Los subdocumentos y arrays son **nativos** — no necesitas normalizar.

---

## 2. CRUD

### Create

```javascript
db.productos.insertOne({ nombre: "Laptop", precio: 1500, categoria: "Electrónica" })

db.productos.insertMany([
  { nombre: "Mouse", precio: 20, categoria: "Accesorios" },
  { nombre: "Silla", precio: 300, categoria: "Hogar" }
])
```

### Read

```javascript
// Filtro simple
db.productos.find({ categoria: "Electrónica" })

// Proyección (equivalente SELECT columnas)
db.productos.find({}, { nombre: 1, precio: 1, _id: 0 })

// Operadores de comparación
db.productos.find({ precio: { $gt: 500 } })
db.productos.find({ precio: { $gte: 100, $lte: 1000 } })
db.productos.find({ categoria: { $in: ["Electrónica", "Hogar"] } })
db.productos.find({ stock: { $exists: true } })

// Lógicos
db.productos.find({
  $or: [
    { categoria: "Electrónica" },
    { precio: { $gt: 1000 } }
  ]
})
```

### Update / Delete

```javascript
db.productos.updateOne({ nombre: "Laptop" }, { $set: { precio: 1400 } })
db.productos.updateMany({ categoria: "Electrónica" }, { $inc: { precio: 100 } })

db.productos.deleteOne({ nombre: "Mouse" })
db.productos.deleteMany({ stock: 0 })
```

---

## 3. Aggregation Pipeline ⭐ (interpretar es exigencia del profe)

Sequence de **stages** que procesan los documentos como una tubería.

### Stages más comunes

| Stage | Equivalente SQL | Qué hace |
|---|---|---|
| `$match` | `WHERE` | Filtra documentos |
| `$group` | `GROUP BY` + agregados | Agrupa por `_id` y aplica sumas/counts |
| `$sort` | `ORDER BY` | Ordena |
| `$project` | `SELECT columnas` | Proyecta / renombra campos |
| `$limit` | `LIMIT N` | Corta el flujo |
| `$skip` | `OFFSET N` | Salta N |
| `$lookup` | `JOIN` | Une con otra colección |
| `$unwind` | (desnormalizar array) | Desdobla arrays en múltiples docs |
| `$count` | `COUNT(*)` | Solo cuenta |

Operadores en `$group`: `$sum`, `$avg`, `$min`, `$max`, `$first`, `$last`, `$push`, `$addToSet`.

### Ejemplo 1 — el que salió en Examen Final 2025-1

```javascript
db.usuarios.aggregate([
  { $match: { edad: { $gte: 18 } } },                    // (1) filtra adultos
  { $group: { _id: "$ciudad", total: { $sum: 1 } } },    // (2) cuenta por ciudad
  { $sort:  { total: -1 } }                              // (3) orden desc por total
])
```

**Interpretación** (esto es lo que el profe quiere que digas):

> Recupera las **ciudades ordenadas por cantidad de usuarios adultos** (edad ≥ 18), de la ciudad con más usuarios adultos a la que menos tiene.

**Equivalente SQL**:
```sql
SELECT ciudad, COUNT(*) AS total
FROM   usuarios
WHERE  edad >= 18
GROUP  BY ciudad
ORDER  BY total DESC;
```

### Ejemplo 2 — agregación con múltiples métricas

```javascript
db.ventas.aggregate([
  { $match: { fecha: { $gte: ISODate("2025-01-01") } } },
  { $group: {
      _id: "$categoria",
      total_ventas:   { $sum: "$monto" },
      promedio_venta: { $avg: "$monto" },
      cantidad:       { $sum: 1 }
    }
  },
  { $sort: { total_ventas: -1 } },
  { $limit: 5 }
])
```

**Interpretación**:
> Devuelve el **top-5 de categorías con más ventas totales en 2025**, mostrando por categoría: total facturado, promedio por venta y cantidad de transacciones.

### Ejemplo 3 — con `$lookup` (join)

```javascript
db.pedidos.aggregate([
  { $lookup: {
      from: "clientes",
      localField:   "id_cliente",
      foreignField: "_id",
      as: "cliente_info"
    }
  },
  { $unwind: "$cliente_info" },
  { $group: {
      _id: "$cliente_info.pais",
      total: { $sum: "$monto" }
    }
  }
])
```

**Interpretación**: total de pedidos agrupado por país del cliente (join implícito por id_cliente).

---

## 4. Indexación ⭐

MongoDB usa **B+ Tree**. Sin índice → collection scan `O(n)`.

### Tipos

```javascript
// Simple
db.users.createIndex({ score: 1 })       // 1 asc, -1 desc

// Compuesto
db.users.createIndex({ userid: 1, score: -1 })

// Multikey (arrays / subdocs)
db.users.createIndex({ "addr.zip": 1 })

// Único
db.users.createIndex({ email: 1 }, { unique: true })

// Texto
db.articulos.createIndex({ contenido: "text" })

// TTL (auto-eliminar después de N segundos)
db.sesiones.createIndex({ creado: 1 }, { expireAfterSeconds: 3600 })
```

### Regla LEFT-PREFIX (para índices compuestos) ⭐

Un índice `{a:1, b:-1, c:1}` sirve para consultas que **empiezan** por los primeros campos:

Con `db.ventas.createIndex({ cliente: 1, fecha: -1 })`:

**SÍ aprovechan**:
```javascript
db.ventas.find({ cliente: "Juan" }).sort({ fecha: -1 })       // filtro+sort exacto
db.ventas.find({ cliente: "Juan", fecha: { $gte: X } })       // filtro por prefijo
db.ventas.find({ cliente: "Juan" })                            // solo prefijo
db.ventas.find({ cliente: "Juan" }).sort({ fecha: 1 })         // sort invertido — MongoDB puede recorrer al revés
```

**NO aprovechan**:
```javascript
db.ventas.find({ fecha: { $gte: X } })                          // se salta "cliente" → scan
db.ventas.find({}).sort({ fecha: -1, cliente: 1 })              // orden invertido de campos
db.ventas.find({ fecha: X, cliente: Y }).sort({ fecha: -1 })    // orden filtro incorrecto — cuidado, aunque motor puede reordenar predicados en algunos casos
```

**Regla de oro para diseñar índices compuestos**: **ESR** = Equality → Sort → Range.

Ejemplo: para `find({cliente:"X", fecha:{$gte:D}}).sort({monto:-1})` → índice `{cliente:1, monto:-1, fecha:1}` (equality, sort, range).

---

## 5. Sharding

MongoDB distribuye datos horizontalmente entre **shards** con una **shard key**.

```
             ┌───────────────┐
             │ Config Servers│  (metadata)
             └───────┬───────┘
                     │
             ┌───────┴────────┐
             │    mongos      │  (router)
             └───┬────┬────┬──┘
                 │    │    │
              ┌──▼─┐┌─▼──┐┌─▼──┐
              │Shd1││Shd2││Shd3│
              └────┘└────┘└────┘
              (cada uno un replica set)
```

Métodos:
- **Hashed sharding**: distribución uniforme, sin range queries eficientes.
- **Ranged sharding**: buena para range queries, riesgo de hot spots.

**Hot spot**: un shard recibe mucho más tráfico que otros (ej. sharding por fecha con timestamp creciente → todas las escrituras van al último shard).

---

## 6. ReplicaSet

```
    ┌─────────┐  oplog stream  ┌────────────┐
    │ PRIMARY │ ──────────────>│SECONDARY 1 │
    └────┬────┘                └────────────┘
         │       heartbeat     ┌────────────┐
         ├────────────────────>│SECONDARY 2 │
         │                     └────────────┘
         │                     ┌────────────┐
         └────────────────────>│ARBITER (opc)│  (solo vota)
                               └────────────┘
```

- **Primary**: reads y writes.
- **Secondaries**: réplican via **oplog** (log de operaciones). Pueden servir reads con `readPreference=secondary`.
- **Failover**: si el primary cae, secondaries eligen nuevo primary (Raft-like).
- **Arbiter**: solo vota (para quórum), no guarda datos.

---

## 7. Full-text search en MongoDB (bonus, salió en S09)

```javascript
db.articulos.createIndex({ contenido: "text" })

db.articulos.find({ $text: { $search: "big data análisis" } })

// Con relevancia (usa BM25 internamente)
db.articulos.find(
  { $text: { $search: "python" } },
  { score: { $meta: "textScore" } }
).sort({ score: { $meta: "textScore" } })
```

MongoDB usa **BM25** (evolución de TF-IDF, ver S08).

---

## 8. Cassandra vs MongoDB — resumen para pregunta de 1 pt

**Similitudes** (elegir 2):
- NoSQL distribuido, escala horizontal.
- Schema flexible.
- Consistencia ajustable.
- Replicación automática.

**Diferencias** (elegir 2):

| | Cassandra | MongoDB |
|---|---|---|
| Modelo | Wide-column | Documento JSON/BSON |
| Arquitectura | P2P masterless | Replica set (Primary/Secondary) |
| CAP default | AP | CP |
| Queries | CQL, restrictivo (PK obligatoria) | Query rica, aggregation pipeline |

---

## 9. Preguntas típicas (autotest)

1. Interpreta esta consulta (di qué recupera):
   ```javascript
   db.ordenes.aggregate([
     { $match: { estado: "completado" } },
     { $group: { _id: "$mes", total: { $sum: "$monto" } } },
     { $sort: { total: -1 } },
     { $limit: 3 }
   ])
   ```
2. Con el índice `{ producto:1, fecha:-1 }`, ¿cuáles de estas consultas lo aprovechan y por qué?
   - `find({ producto:"A", fecha:{$lt:X} })`
   - `find({ fecha:{$lt:X} }).sort({ producto:1 })`
   - `find({ producto:"A" }).sort({ fecha:1 })`
3. Explica la regla ESR para diseñar índices compuestos.
4. ¿Qué es un hot spot en sharding y cómo se evita?
5. Diferencia entre replica set y Redis Cluster.
6. Escribe la aggregation MongoDB equivalente a:
   ```sql
   SELECT categoria, AVG(precio) FROM productos
   WHERE stock > 0
   GROUP BY categoria HAVING AVG(precio) > 100
   ORDER BY AVG(precio) DESC;
   ```
