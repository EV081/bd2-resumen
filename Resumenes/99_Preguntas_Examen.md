# Banco de Preguntas — Simulacro Final BD2

> Resuelve sin ver el [solucionario](99_Solucionario.md). Están numeradas para poder discutirlas.

---

## Bloque A — Preguntas cortas (1 pt c/u)

**A1.** ¿En qué escenarios se aplica el diseño de BD distribuidas Bottom-Up? Da al menos 2 ejemplos concretos.

**A2.** Diferencias funcionales entre backup y replicación. ¿Cuál protege contra un `DROP TABLE` accidental y por qué?

**A3.** Escribe una consulta en PostgreSQL usando GIN que aproxime la similitud de coseno para buscar documentos que contengan "oro & plata & camion". Explica por qué `ts_rank_cd` con el flag `32` aproxima coseno.

**A4.** Cita 2 similitudes y 2 diferencias entre Cassandra y MongoDB.

**A5.** Interpreta esta consulta (di qué recupera en lenguaje natural):
```javascript
db.usuarios.aggregate([
  { $match: { edad: { $gte: 18 } } },
  { $group: { _id: "$ciudad", total: { $sum: 1 } } },
  { $sort:  { total: -1 } }
])
```

**A6.** MongoDB: se ha creado el índice `db.ventas.createIndex({ cliente: 1, fecha: -1 })`. Propón **dos consultas distintas** (usando `find` y `sort`) que aprovechen eficientemente ese índice. Justifica con la regla ESR/left-prefix.

**A7.** ¿Cómo se calculan el TF y el IDF en el histograma de palabras visuales (BoVW)? ¿Qué representa cada uno en imágenes?

**A8.** Propón un escenario **distinto al visto en clase** en el que resulte beneficioso usar Redis. Justifica aplicando el patrón A-SIDE (dibuja el flujo).

**A9.** ¿Cuándo aplicar fragmentación vertical basada en Dependencias Funcionales vs Matriz de Afinidad?

**A10.** Explica el algoritmo SPIMI paso a paso. ¿Cuál es su ventaja principal sobre BSBI?

**A11.** ¿Por qué en Cassandra la partition key es obligatoria en el `WHERE` para queries eficientes? ¿Qué hace `ALLOW FILTERING` y por qué es peligroso?

**A12.** Cassandra: dado `PRIMARY KEY ((a, b), c, d)`, ¿cuáles de estos `WHERE` son válidos sin `ALLOW FILTERING`?
   - `WHERE a=1 AND b=2`
   - `WHERE a=1 AND b=2 AND c=3 AND d>5`
   - `WHERE a=1`
   - `WHERE c=3`

**A13.** ¿Por qué se usa similitud coseno en vez de distancia euclidiana en IR? Da el ejemplo del documento duplicado.

**A14.** GIN vs GiST: ¿cuál elegirías para un blog con miles de posts nuevos por día? ¿Y para un motor de búsqueda de un archivo histórico? Justifica.

**A15.** ¿Cuántos hash slots tiene Redis Cluster y cómo se determina el nodo de una clave?

---

## Bloque B — Búsqueda Multimedia (3-4 pts c/u)

**B1.** Se tiene una colección de imágenes con distintas resoluciones. Una técnica de extracción convierte cada imagen (o subregión) en vectores. Explica cómo hacer una **búsqueda eficiente basada en descriptores locales** con un gráfico. Indica qué tipo de búsqueda soporta y por qué (con ejemplo).

**B2.** Modifica el algoritmo `Lower_Bounding_KNN(Q, K)` — que originalmente retorna solo 1 vecino — para retornar **K vecinos** eficientemente usando Lower Bounding Distance. Escribe el pseudocódigo y explica la estructura de datos que usas para el `best_so_far`.

**B3.** Diseña un sistema de detección de copyright en una plataforma tipo TikTok (cada video tiene imagen + audio + transcripción). Describe:
   - Framework escalable para grandes volúmenes.
   - Proceso de construcción del índice.
   - Proceso de recuperación desde un video de consulta.
   - Por qué tu propuesta es mejor en desempeño.
   Incluye gráficos del flujo.

**B4.** Explica la construcción y consulta de Bag of Visual Words con TF-IDF y similitud coseno. Incluye por qué se usa K-Means.

---

## Bloque C — Fragmentación y Consulta Distribuida (7-8 pts progresivo)

### Caso C1 — Empresa de delivery (RAPPI)

Tablas:
```
Pedidos(IdPedido, IdCliente, FechaPedido, Monto, Ciudad, Estado)
Repartidores(IdRepartidor, Nombre, TipoVehiculo, Ciudad, Disponibilidad, Calificacion)
```

Condiciones:
- `Pedidos` recibe consultas frecuentes por rango de `FechaPedido`.
- `Repartidores` se consulta principalmente por `id_repartidor` (auto-incremental).
- 3 servidores esclavos + 1 coordinador.

Consulta:
```sql
SELECT R.Ciudad,
       COUNT(DISTINCT R.IdRepartidor) AS TotalRepartidores,
       SUM(P.Monto) AS MontoTotal
FROM Repartidores R JOIN Pedidos P ON R.Ciudad = P.Ciudad
GROUP BY R.Ciudad
ORDER BY MontoTotal DESC;
```

Se pide:
- **(1 pt)** Proponer la fragmentación horizontal para cada tabla (justificar técnica).
- **(4 pts)** Diseñar el algoritmo distribuido optimizado. Usa pseudocódigo estilo `create temp / merge / select ...` como el profe. Incluye el gráfico del flujo de datos.
- **(3 pts)** Escribe las sentencias SQL derivadas.

### Caso C2 — Predicados MinTerm (2 pts)

Predicados frecuentes: `tipoPago = EFECTIVO`, `tipoPago = ELECTRONICO`, `monto > 100`, `monto > 20`.
Aplica el algoritmo MinTerm para hallar todos los fragmentos horizontales válidos:
- Desplegar todos los posibles.
- Eliminar inútiles.
- Simplificar.

### Caso C3 — Optimización de Join distribuido (2 pts)

Tablas:
```
S en Servidor 1, tamaño 40
T en Servidor 2, tamaño 15
V en Servidor 3, tamaño 20
```
Consulta: `SELECT * FROM S ⋈ T ⋈ V`.

Encuentra los posibles planes de ejecución. Grafica el árbol con casos de poda. Explica por qué se poda cada rama.

### Caso C4 — Fragmentación Vertical (2 pts)

Dada la matriz de afinidad:

|   | A | B | C | D | E |
|---|---|---|---|---|---|
| A |105|29 | 0 |48 | 5 |
| B |29 |38 | 1 |55 | 2 |
| C | 0 | 1 |30 | 7 |75 |
| D |48 |55 | 7 |39 |10 |
| E | 5 | 2 |75 |10 |51 |

Aplica BEA. Construye la Matriz Agrupada. Encuentra la partición óptima. Da los fragmentos verticales resultantes.

### Caso C5 — E-commerce con lista categórica (5 pts)

Tablas:
```
Ventas(IDVenta, Tienda, IDCliente, MetodoPago, MontoTotal, FechaVenta)
Clientes(IDCliente, Nombre, Apellidos, Pais, FechaNac)
```
Condiciones:
- 90% de queries sobre `Clientes` filtran por `Pais ∈ {A, B, C}`.
- `MontoTotal` sigue distribución uniforme entre 0 y 2000 soles.
- 3 esclavos + 1 maestro.

Consulta:
```sql
SELECT C.Nombre, C.Apellidos, V.FechaVenta, V.MontoTotal
FROM Venta V INNER JOIN Clientes C ON V.IDCliente = C.IDCliente
WHERE C.Pais IN (A, C)
ORDER BY V.MontoTotal DESC, V.FechaVenta;
```

- **(1 pt)** Fragmentación horizontal de `Clientes` + derivada para `Ventas`. Notación formal.
- **(4 pts)** Diseño del algoritmo distribuido optimizado usando **los 3 esclavos**. Sé lo más detallado posible con pseudocódigo.

### Caso C6 — MINSA con join sobre atributo NO fragmentado (6 pts)

Tablas:
```
Medico(Codigo, DNI_Medico, Nombre, Apellidos, Especialidad)
Paciente(IdPaciente, DNI_Paciente, Nombre, Apellidos, FechaNac, Sexo)
```
Condiciones:
- `Medico` fragmentado horizontalmente por `Especialidad` (categórico 20 opciones).
- `Paciente` se actualiza constantemente con nuevos pacientes (RN incluidos) — propón técnica adecuada para fragmentar por `FechaNac`.
- 3 esclavos + 1 coordinador.

Consulta:
```sql
SELECT Especialidad, COUNT(*) AS Conteo
FROM Medico M, Paciente P
WHERE M.DNI_Medico = P.DNI_Paciente
GROUP BY M.Especialidad
ORDER BY COUNT(*) DESC;
```

- **(3 pts)** Grafica con particiones originales y las fragmentaciones intermedias.
- **(3 pts)** Sentencias SQL que reflejen el algoritmo.

---

## Bloque D — Preguntas conceptuales cortas

**D1.** ¿Cómo garantizamos completitud y desarticulación en una fragmentación horizontal **derivada**? Escribe `R ⋉_COD S` en SQL.

**D2.** Dada una relación `R(K, ...)` inicialmente vacía y se quiere fragmentar sobre `K` en 4 fragmentos equitativos, sabiendo que `K` es auto-incremental. ¿Qué técnica aplicarías y por qué? Muestra un ejemplo.

**D3.** Explica los 3 pasos del procesamiento de una consulta distribuida (descomposición, localización, optimización).

**D4.** ¿Por qué en agregación distribuida no se puede propagar directamente `AVG` y hay que enviar `SUM` y `COUNT`?

**D5.** Enumera los 3 niveles de consistencia más importantes de Cassandra y cuándo usarías cada uno.

---

## Recomendación

- Bloque A → 15 min total (1 min por pregunta).
- Bloque B → 25 min.
- Bloque C → 60 min (el más pesado).
- Bloque D → 10 min.

Total ≈ 110 min = tiempo real del examen.
