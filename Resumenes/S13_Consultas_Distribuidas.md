# S13 — Consultas Distribuidas ⭐⭐ (con pseudocódigo estilo profe)

> Este es el bloque más pesado del examen. Siempre cae un ejercicio "empresa X con 3 esclavos + 1 coordinador".

---

## 1. Pasos del procesamiento de una consulta distribuida

```
    Consulta SQL de alto nivel
              │
              ▼
   ┌──────────────────────┐
   │  1. DESCOMPOSICIÓN   │  ← convertir a álgebra relacional
   │  (normalización,     │
   │   eliminar redund.,  │
   │   optimización sint.)│
   └──────────┬───────────┘
              ▼
   ┌──────────────────────┐
   │  2. LOCALIZACIÓN     │  ← reemplazar R por R1 ∪ R2 ∪ ...
   │  (aplicar reglas     │
   │   sobre fragmentos)  │
   └──────────┬───────────┘
              ▼
   ┌──────────────────────┐
   │  3. OPTIMIZACIÓN     │  ← minimizar tráfico de red
   │  (reparticionar,     │
   │   semi-join, orden)  │
   └──────────┬───────────┘
              ▼
    Plan de ejecución distribuido
```

---

## 2. Descomposición

### Normalización algebraica

Convertir SQL a álgebra:
- `SELECT ... FROM R WHERE p` → `σ_p(R)`
- `SELECT cols FROM R` → `π_cols(R)`
- `JOIN` → `⋈`.

### Eliminación de redundancias

- Condiciones falsas: `p ∧ ¬p ≡ false`.
- Condiciones redundantes: `p ∧ p ≡ p`.
- Sub-expresiones repetidas: factor común.

### Optimización sintáctica

**Empujar σ y π hacia abajo** — filtrar/proyectar lo más cerca posible de la fuente, para reducir datos antes de operar.

---

## 3. Localización (reglas)

### Regla 1 — Filtro sobre fragmentación horizontal

Si `R = R_1 ∪ R_2` con `R_1 = σ_{E<10}(R)` y consulta `σ_{E=3}(R)`:
- `R_2` no puede contener tuplas con E=3 → **se elimina del árbol**.
- Solo se ejecuta `σ_{E=3}(R_1)`.

### Regla 2 — Join con fragmentos horizontales

```
R ⋈ S = (R_1 ∪ R_2) ⋈ (S_1 ∪ S_2)
      = (R_1 ⋈ S_1) ∪ (R_1 ⋈ S_2) ∪ (R_2 ⋈ S_1) ∪ (R_2 ⋈ S_2)
```

Si hay **fragmentación derivada** con `S_i = S ⋉ R_i`, los cruces mixtos son vacíos:
- `R_1 ⋈ S_2 = ∅`
- `R_2 ⋈ S_1 = ∅`
- Solo quedan `R_1 ⋈ S_1` y `R_2 ⋈ S_2` → **JOIN LOCAL** en cada nodo. 🎯

### Regla 3 — Proyección sobre fragmentación vertical

Si `R = R_1(K,A,B) ⋈_K R_2(K,C,D)` y consulta `π_A(R)`:
- Solo `R_1` tiene `A` → no hace falta join.
- `π_A(R) = π_A(R_1)`.

---

## 4. Optimización — operaciones distribuidas

Todas siguen el mismo patrón: **si el atributo de la operación coincide con el atributo de fragmentación → local. Si no → reparticionar (fragmentación intermedia)**.

### 4.1 Ordenación distribuida

#### Caso A — R ya está fragmentado por el atributo de orden `k`

```
     S1: R_a  (k ∈ [0, k0))     S2: R_b (k ∈ [k0, k1))     S3: R_c (k ≥ k1)
        │                         │                          │
        └── ordenar local ─────── + ────── ordenar local ────┘
                            │
              concatenar resultados en orden → resultado final
```

Cada nodo ordena local, se concatenan en el coordinador. `O(N/P · log(N/P))` por nodo.

#### Caso B — R fragmentado por OTRO atributo, se pide `ORDER BY k`

**Hay que reparticionar por rango de `k` primero** — vector de partición estimado por el coordinador (cada nodo envía `min, max, #tuplas`).

**Pseudocódigo estilo profe** (ordenar `R.k` cuando R está en S1 y S2 pero se necesita distribuido por rango de `k`):

```
-- vector de partición: [k0, k1]  (calculado por coordinador)

-- Crear fragmentos intermedios por rango de k
create temp s1.r1
    merge(
      select * from s1.ra where k < k0 order by k,
      select * from s2.rb where k < k0 order by k
    )

create temp s2.r2
    merge(
      select * from s1.ra where k >= k0 and k < k1 order by k,
      select * from s2.rb where k >= k0 and k < k1 order by k
    )

create temp s3.r3
    merge(
      select * from s1.ra where k >= k1 order by k,
      select * from s2.rb where k >= k1 order by k
    )

-- Resultado global: concatenar en orden
result = s1.r1 || s2.r2 || s3.r3
```

- Cada `merge(...)` es un **merge de streams ordenados** en el nodo destino.
- Como los rangos son disjuntos por construcción, la concatenación final ya está ordenada globalmente **sin sort central**.

### 4.2 JOIN distribuido — mismo patrón

#### Caso A — R y S ya fragmentados por el mismo atributo con la misma función

```
     R1 en S1        R2 en S2        R3 en S3
     S1' en S1       S2' en S2       S3' en S3
        │               │               │
    R1 ⋈ S1'        R2 ⋈ S2'        R3 ⋈ S3'    ← LOCAL
        │               │               │
        └───────── UNION en coordinador ┘
```

#### Caso B — R fragmentado por `a`, S fragmentado por `b`, join sobre `k`

Hay que reparticionar **ambos** por `k`. Usa la misma función de partición `f(k)`.

```
-- Reparticionar R por hash(k) mod 3
create temp s1.r_k
    (select * from s1.r where hash(k) mod 3 = 0)
    ∪ (select * from s2.r where hash(k) mod 3 = 0)
    ∪ (select * from s3.r where hash(k) mod 3 = 0)

create temp s2.r_k
    (select * from s1.r where hash(k) mod 3 = 1)
    ∪ (select * from s2.r where hash(k) mod 3 = 1)
    ∪ (select * from s3.r where hash(k) mod 3 = 1)

-- Idem con S:
create temp s1.s_k = ⋃ (select * from s_i.s where hash(k) mod 3 = 0)
create temp s2.s_k = ⋃ (select * from s_i.s where hash(k) mod 3 = 1)
create temp s3.s_k = ⋃ (select * from s_i.s where hash(k) mod 3 = 2)

-- JOIN LOCAL en cada nodo
select * from s1.r_k join s1.s_k using(k)
select * from s2.r_k join s2.s_k using(k)
select * from s3.r_k join s3.s_k using(k)

-- Coordinador: union all
result = s1_result ∪ s2_result ∪ s3_result
```

### 4.3 Semi-join para reducir tráfico

Idea: si solo un pequeño subconjunto de S hace match con R, no envíes todo S a donde está R. Envía solo `π_a(R)` a S, calcula `S ⋉ R`, y **manda de vuelta** solo lo relevante.

```
-- Suponiendo R en S1, S en S2, join sobre atributo a

-- Paso 1: proyectar solo la columna de join en R
create temp s1.r_a  =  select distinct a from s1.r

-- Paso 2: enviar a S2 y semi-join
create temp s2.s_prime  =  select s.*  from s2.s
                           where a in (select a from s1.r_a)

-- Paso 3: enviar s2.s_prime a S1 y hacer JOIN local
create temp result @s1  =  select r.*, s_prime.*
                           from s1.r join s2.s_prime on r.a = s_prime.a
```

**Cuándo conviene**: cuando `|π_a(R)| << |S|` y `|S ⋉ R| << |S|`. El tráfico total es `|π_a(R)| + |S ⋉ R|` en vez de `|S|`.

### 4.4 Agregación distribuida

#### GROUP BY sobre atributo de fragmentación → local

Si R está fragmentado por `dept` y query = `GROUP BY dept`:

```
-- Cada nodo agrega localmente
select dept, sum(sal) from s1.r group by dept
select dept, sum(sal) from s2.r group by dept
select dept, sum(sal) from s3.r group by dept

-- Coordinador: solo concatenar (no hay overlap de deptos entre nodos)
result = union all
```

#### GROUP BY sobre otro atributo → parcial + reparticionar + agregar

```
-- Paso 1: agregación PARCIAL local (ya reduce datos)
create temp s1.g_partial  =  select dept, sum(sal) as s, count(*) as c
                             from s1.r group by dept
create temp s2.g_partial  =  select dept, sum(sal) as s, count(*) as c
                             from s2.r group by dept
create temp s3.g_partial  =  select dept, sum(sal) as s, count(*) as c
                             from s3.r group by dept

-- Paso 2: reparticionar por hash(dept) para que cada dept termine en un nodo
create temp s1.g_final
    (select * from s1.g_partial where hash(dept) mod 3 = 0)
  ∪ (select * from s2.g_partial where hash(dept) mod 3 = 0)
  ∪ (select * from s3.g_partial where hash(dept) mod 3 = 0)

-- (idem s2.g_final, s3.g_final)

-- Paso 3: agregación FINAL local en cada nodo
select dept, sum(s) as sum_total, sum(c) as cnt from s1.g_final group by dept
-- ...

-- Coordinador: union all
```

### AVG distribuido (ojo — AVG no es asociativa)

**Truco**: nunca transmitas AVG parcial. Transmite `SUM` y `COUNT`, y calcula el AVG al final:

$$\text{AVG global} = \frac{\sum \text{SUM}_i}{\sum \text{COUNT}_i}$$

```
-- Local en cada nodo
select dept, sum(sal) as s, count(*) as c from s_i.r group by dept

-- Coordinador
select dept, sum(s)/sum(c) as avg from union group by dept
```

### 4.5 Eliminación de duplicados (SELECT DISTINCT)

Dos opciones:
1. **Reparticionar por hash del atributo distinct** → eliminar local en cada nodo.
2. **Ordenar primero** → eliminar duplicados adyacentes durante el merge.

### 4.6 Selección con índices distribuidos

- El **vector de partición** puede actuar como raíz de un **B+ Tree distribuido**.
- Los índices sobre atributos que **no son** de partición son complicados (los nodos hoja están en distintos sitios).
- **Alternativa**: replicar el índice completo si es pequeño.

---

## 5. Plantilla de respuesta para el problema grande ⭐

Los ejercicios de examen tienen esta estructura fija:

### Paso 1 — Fragmentar (1 pt)

Elige técnica por tabla según el patrón de consultas:

```
Tabla_A = A_1 ∪ A_2 ∪ A_3,   con A_i = σ_{predicado_i}(Tabla_A)  ← justificar
Tabla_B = B_1 ∪ B_2 ∪ B_3,   con B_i = ...
```

### Paso 2 — Algoritmo optimizado con gráfico (4 pts)

Piensa en **3 puntos** donde puedes reparticionar:

1. Para hacer el **JOIN local** → reparticionar por atributo de join.
2. Para hacer el **GROUP BY local** → puede coincidir con lo anterior.
3. Para hacer el **ORDER BY sin sort central** → reparticionar por RANGE del atributo de orden.

**Gráfico obligatorio**:

```
    S1: A1, B1_derivado            S2: A2, B2_derivado           S3: A3, B3_derivado
        │                              │                              │
        ▼ Repartir por Ciudad          ▼ Repartir por Ciudad          ▼ Repartir por Ciudad
    ┌────────────┐                 ┌────────────┐                 ┌────────────┐
    │ Ciudad=Lim │                 │ Ciudad=Arq │                 │ Ciudad=Tru │
    └─────┬──────┘                 └─────┬──────┘                 └─────┬──────┘
          │ JOIN local + GROUP BY Ciudad local
          ▼                              ▼                              ▼
    (Ciudad, TotalRep, MontoTotal)  (Ciudad, ...)                (Ciudad, ...)
          │
          ▼ Repartir por RANGE(MontoTotal)
    ┌──────────────┐             ┌──────────────┐              ┌──────────────┐
    │Monto rango 1 │             │Monto rango 2 │              │Monto rango 3 │
    └──────┬───────┘             └──────┬───────┘              └──────┬───────┘
           │ ordenar local            │ ordenar local              │ ordenar local
           └──────────────── concatenar ──────────────────────────┘
                             ▼
                       Coordinador
```

### Paso 3 — SQL derivada (3 pts)

Combinar el pseudocódigo con SQL real:

```sql
-- Fragmentación horizontal
CREATE TABLE Pedidos (...)      PARTITION BY RANGE (FechaPedido);
CREATE TABLE Repartidores (...) PARTITION BY HASH  (IdRepartidor);

-- Consulta (con reparticionamiento intermedio implícito)
SELECT   R.Ciudad,
         COUNT(DISTINCT R.IdRepartidor) AS TotalRepartidores,
         SUM(P.Monto)                   AS MontoTotal
FROM     Repartidores R
JOIN     Pedidos P ON R.Ciudad = P.Ciudad
GROUP BY R.Ciudad
ORDER BY MontoTotal DESC;
```

---

## 6. Casos trabajados de exámenes anteriores

### Caso RAPPI (Final 2025-1) — el más ilustrativo

**Tablas**:
- `Pedidos(IdPedido, IdCliente, FechaPedido, Monto, Ciudad, Estado)` → fragmentar por RANGE de `FechaPedido`.
- `Repartidores(IdRepartidor, Nombre, TipoVehiculo, Ciudad, Disponibilidad, Calificacion)` → fragmentar por HASH de `IdRepartidor`.

**Consulta**:
```sql
SELECT R.Ciudad,
       COUNT(DISTINCT R.IdRepartidor) AS TotalRepartidores,
       SUM(P.Monto)                   AS MontoTotal
FROM Repartidores R JOIN Pedidos P ON R.Ciudad = P.Ciudad
GROUP BY R.Ciudad
ORDER BY MontoTotal DESC;
```

**Problema**: el JOIN es por `Ciudad`, pero las tablas NO están fragmentadas por Ciudad → hay que **reparticionar**.

**Gráfico del flujo**:

```
    S1: P1, R1                S2: P2, R2                S3: P3, R3
        │                         │                         │
        ▼ reparticionar Rep       ▼ reparticionar Rep       ▼ reparticionar Rep
        ▼ reparticionar Ped       ▼ reparticionar Ped       ▼ reparticionar Ped
        (por Ciudad)              (por Ciudad)              (por Ciudad)
    ┌───────────┐             ┌───────────┐             ┌───────────┐
    │Ciudad grp1│             │Ciudad grp2│             │Ciudad grp3│
    └────┬──────┘             └────┬──────┘             └────┬──────┘
         │ JOIN + GROUP BY Ciudad local
         ▼                         ▼                         ▼
    (Ciudad,TotalRep,Monto)  (Ciudad,...)             (Ciudad,...)
         │
         ▼ reparticionar por RANGE(MontoTotal)
    ┌───────────┐             ┌───────────┐             ┌───────────┐
    │Monto rgo1 │             │Monto rgo2 │             │Monto rgo3 │
    └────┬──────┘             └────┬──────┘             └────┬──────┘
         │ ordenar local (ya viene ordenado por rango)
         └────────────── concatenar en orden ──────────────────┘
                                 ▼
                            Coordinador
```

**Pseudocódigo**:

```
-- PASO 1: reparticionar Repartidores por Ciudad (una ciudad por nodo)
create temp s1.rep_c
    (select * from s1.rep where hash(Ciudad) mod 3 = 0)
  ∪ (select * from s2.rep where hash(Ciudad) mod 3 = 0)
  ∪ (select * from s3.rep where hash(Ciudad) mod 3 = 0)
create temp s2.rep_c  = ⋃ ... (hash mod 3 = 1)
create temp s3.rep_c  = ⋃ ... (hash mod 3 = 2)

-- PASO 1 bis: reparticionar Pedidos por Ciudad
create temp s1.ped_c  = ⋃ (select * from s_i.ped where hash(Ciudad) mod 3 = 0)
create temp s2.ped_c  = ⋃ (select * from s_i.ped where hash(Ciudad) mod 3 = 1)
create temp s3.ped_c  = ⋃ (select * from s_i.ped where hash(Ciudad) mod 3 = 2)

-- PASO 2: JOIN + GROUP BY local en cada nodo
create temp s1.agg
    select Ciudad,
           count(distinct IdRepartidor) as TotalRep,
           sum(Monto)                   as MontoTotal
    from s1.rep_c join s1.ped_c using (Ciudad)
    group by Ciudad
create temp s2.agg = ... (idem en s2)
create temp s3.agg = ... (idem en s3)

-- PASO 3: reparticionar por RANGE(MontoTotal) para ORDER BY sin sort central
--         vector [m0, m1] estimado por coordinador
create temp s1.final
    merge(
      select * from s1.agg where MontoTotal >= m1 order by MontoTotal desc,
      select * from s2.agg where MontoTotal >= m1 order by MontoTotal desc,
      select * from s3.agg where MontoTotal >= m1 order by MontoTotal desc
    )
create temp s2.final
    merge(
      select * from s_i.agg where m0 <= MontoTotal < m1 order by MontoTotal desc
    )
create temp s3.final
    merge(
      select * from s_i.agg where MontoTotal < m0 order by MontoTotal desc
    )

-- PASO 4: coordinador concatena en orden
result = s1.final || s2.final || s3.final
```

### Caso E-commerce (Final anterior) — con fragmentación derivada

**Tablas**:
- `Clientes(IDCliente, Nombre, Apellidos, Pais, FechaNac)` → fragmentar por LIST de `Pais ∈ {A,B,C}` — el 90% filtra por país.
- `Ventas(IDVenta, IDCliente, ..., MontoTotal, FechaVenta)` → **fragmentación derivada** por `IDCliente`.

```
Clientes = C_A ∪ C_B ∪ C_C,       C_i = σ_{Pais=i}(Clientes)
Ventas   = V_A ∪ V_B ∪ V_C,       V_i = Ventas ⋉_IDCliente C_i
```

**Consulta**:
```sql
SELECT C.Nombre, C.Apellidos, V.FechaVenta, V.MontoTotal
FROM Venta V INNER JOIN Clientes C ON V.IDCliente = C.IDCliente
WHERE C.Pais IN (A, C)
ORDER BY V.MontoTotal DESC, V.FechaVenta;
```

**Estrategia**:

- Nodo A trabaja con `C_A, V_A` (Pais=A) → JOIN LOCAL.
- Nodo C trabaja con `C_C, V_C` (Pais=C) → JOIN LOCAL.
- Nodo B queda **libre** — se le puede usar para el sort intermedio.

**Gráfico del flujo**:

```
    S_A: C_A, V_A            S_B: C_B, V_B            S_C: C_C, V_C
    (Pais=A)                 (Pais=B, no aplica)      (Pais=C)
        │                         │                         │
        │ WHERE Pais IN (A,C)     │ ⟵ descartado por        │
        │                         │   localización           │
        ▼                         ✗                         ▼
    JOIN local                                          JOIN local
    V_A ⋈_IDCliente C_A                                 V_C ⋈_IDCliente C_C
    (funciona porque V_A es                             (idem)
     derivado de C_A ⇒ mismo nodo)
        │                                                    │
        ▼ proyectar Nombre, Apellidos,                       ▼
        │  FechaVenta, MontoTotal                            │
        ▼ ORDER BY MontoTotal DESC, FechaVenta               ▼
    ┌───────────────┐                              ┌───────────────┐
    │s_A.res orden. │                              │s_C.res orden. │
    └───────┬───────┘                              └───────┬───────┘
            │                                              │
            └────────────► MERGE de streams ◄──────────────┘
                     (en S_B libre, o en coordinador)
                                 │
                                 ▼
                            Resultado final
```

**Pseudocódigo**:

```
-- Paso 1: en cada nodo relevante, JOIN local + proyectar + ordenar
create temp s_A.res
    select C.Nombre, C.Apellidos, V.FechaVenta, V.MontoTotal
    from s_A.V_A join s_A.C_A using(IDCliente)
    order by MontoTotal desc, FechaVenta

create temp s_C.res
    select C.Nombre, C.Apellidos, V.FechaVenta, V.MontoTotal
    from s_C.V_C join s_C.C_C using(IDCliente)
    order by MontoTotal desc, FechaVenta

-- Paso 2: MERGE de dos streams ordenados (no un sort completo)
-- Podemos usar s_B libre para el merge, o hacerlo en el coordinador
result = merge(s_A.res, s_C.res)
```

### Caso MINSA (PC3) — con reparticionamiento por HASH(DNI)

**Tablas**:
- `Medico(Codigo, DNI_Medico, ..., Especialidad)` → fragmentado horizontal por Especialidad (categórico 20 opciones agrupadas en 3 nodos).
- `Paciente(IdPaciente, DNI_Paciente, ..., FechaNac, Sexo)` → fragmentado por HASH del año de nacimiento (por qué HASH y no RANGE: la tabla se actualiza con recién nacidos, un RANGE dinámico se desbalancea).

**Consulta**:
```sql
SELECT Especialidad, count(*) as Conteo
FROM Medico M, Paciente P
WHERE M.DNI_Medico = P.DNI_Paciente
GROUP BY M.Especialidad
ORDER BY count(*) DESC;
```

**Problema**: JOIN por DNI, pero nadie está fragmentado por DNI.

**Estrategia**:

1. Reparticionar `Medico` y `Paciente` por `HASH(DNI)`.
2. JOIN local en cada nodo.
3. GROUP BY Especialidad **parcial** local.
4. Agregación final (por hash de Especialidad o centralizada — Especialidad tiene ≤20 valores, no vale la pena reparticionar).
5. ORDER BY count DESC → centralizado sobre pocas filas.

**Gráfico del flujo**:

```
    S1: Med(Esp₁) Pac(año h=0)   S2: Med(Esp₂) Pac(año h=1)   S3: Med(Esp₃) Pac(año h=2)
        │                              │                              │
        ▼ reparticionar Med por HASH(DNI_Medico)                       ▼
        ▼ reparticionar Pac por HASH(DNI_Paciente)                     ▼
    ┌────────────────┐          ┌────────────────┐          ┌────────────────┐
    │ h(DNI) mod 3=0 │          │ h(DNI) mod 3=1 │          │ h(DNI) mod 3=2 │
    └────────┬───────┘          └────────┬───────┘          └────────┬───────┘
             │ JOIN local Med ⋈ Pac (DNI)
             │ + GROUP BY Especialidad PARCIAL local
             ▼                          ▼                            ▼
    (Especialidad, c)           (Especialidad, c)              (Especialidad, c)
             │                          │                            │
             └────────────►  UNION en coordinador  ◄─────────────────┘
                          + SUM(c) GROUP BY Especialidad
                          + ORDER BY Conteo DESC
                                     │
                                     ▼
                                Resultado final
```

```
-- Reparticionar por HASH(DNI)
create temp s1.med_d  = ⋃_i (select * from s_i.medico     where hash(DNI_Medico)    mod 3 = 0)
create temp s2.med_d  = ⋃_i (select * from s_i.medico     where hash(DNI_Medico)    mod 3 = 1)
create temp s3.med_d  = ⋃_i (select * from s_i.medico     where hash(DNI_Medico)    mod 3 = 2)

create temp s1.pac_d  = ⋃_i (select * from s_i.paciente   where hash(DNI_Paciente)  mod 3 = 0)
create temp s2.pac_d  = ⋃_i (select * from s_i.paciente   where hash(DNI_Paciente)  mod 3 = 1)
create temp s3.pac_d  = ⋃_i (select * from s_i.paciente   where hash(DNI_Paciente)  mod 3 = 2)

-- JOIN local + agregación parcial por Especialidad
create temp s1.agg
    select Especialidad, count(*) as c
    from s1.med_d join s1.pac_d on DNI_Medico = DNI_Paciente
    group by Especialidad

-- (idem s2.agg, s3.agg)

-- Coordinador: sumar los parciales y ordenar
result = select Especialidad, sum(c) as Conteo
         from (s1.agg ∪ s2.agg ∪ s3.agg)
         group by Especialidad
         order by Conteo desc
```

---

## 7. Preguntas típicas (autotest)

1. Explica los 3 pasos del procesamiento distribuido (descomposición, localización, optimización).
2. Diseña el algoritmo distribuido (con pseudocódigo estilo `create temp / merge`) para ordenar una tabla R por atributo `k` cuando R está fragmentada por otro atributo.
3. ¿Cuándo el JOIN se puede resolver localmente sin reparticionar?
4. Escribe la estrategia semi-join para reducir tráfico en un JOIN entre nodos remotos.
5. ¿Por qué `AVG` no se puede propagar directamente y hay que usar SUM+COUNT?
6. Diseña el plan para la consulta:
   ```sql
   SELECT Ciudad, SUM(Monto)
   FROM Pedidos
   WHERE Estado = 'ENTREGADO'
   GROUP BY Ciudad
   ORDER BY SUM(Monto) DESC;
   ```
   sabiendo que `Pedidos` está fragmentada por rango de fecha en 3 nodos.
