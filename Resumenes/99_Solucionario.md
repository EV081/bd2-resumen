# Solucionario — Simulacro Final BD2

> Respuestas modelo. Consulta después de intentar en [99_Preguntas_Examen.md](99_Preguntas_Examen.md).

---

## Bloque A — Cortas

**A1.** Bottom-Up aplica cuando ya existen varias BDs en distintos sitios y hay que integrarlas. Ejemplos: (a) **integración de sistemas legacy** (empresas con BDs viejas por departamento); (b) **fusiones/adquisiciones** entre empresas, cada una con su BD y hay que unificarlas en un esquema global. No hay problema de fragmentación (los datos ya están repartidos) — el reto es la **integración de esquemas** (esquema local → esquema de exportación → integración → esquema unificado).

**A2.** Backup = copia puntual histórica offline; replicación = copias vivas sincronizadas online. El **backup protege contra DROP TABLE accidental** porque es un estado congelado en el tiempo del que se puede restaurar. La replicación NO protege — replica el DROP inmediatamente a todos los nodos.

**A3.**
```sql
CREATE INDEX idx_doc_fts ON documentos
USING GIN (to_tsvector('spanish', contenido));

SELECT d.id,
       ts_rank_cd(to_tsvector('spanish', d.contenido), q, 32) AS score
FROM   documentos d,
       to_tsquery('spanish', 'oro & plata & camion') AS q
WHERE  to_tsvector('spanish', d.contenido) @@ q
ORDER  BY score DESC
LIMIT 10;
```
Por qué aproxima coseno:
- `@@` con GIN filtra solo docs que comparten términos con la query (recorre posting lists → efecto de dot product en dimensiones no-cero).
- `ts_rank_cd` pondera por TF y proximidad de términos.
- El flag `32` normaliza dividiendo por la longitud del documento → imita división por `‖d‖` de coseno.

**A4.** **Similitudes**: NoSQL distribuido con escala horizontal, schema flexible. **Diferencias**: modelo (wide-column vs documento), arquitectura (P2P masterless vs Primary/Secondary), CAP (Cassandra AP, Mongo CP), queries (Cassandra restrictiva con PK obligatoria, Mongo query rica con aggregation).

**A5.** Devuelve las **ciudades ordenadas por cantidad de usuarios adultos** (edad ≥ 18), de la que más tiene a la que menos.
```sql
SELECT ciudad, COUNT(*) AS total
FROM   usuarios
WHERE  edad >= 18
GROUP  BY ciudad
ORDER  BY total DESC;
```

**A6.** Con índice `{cliente:1, fecha:-1}`:
1. `db.ventas.find({cliente: "Juan"}).sort({fecha: -1}).limit(20)` → usa el índice **completo** (filtro por prefijo + orden coincide).
2. `db.ventas.find({cliente: "Juan", fecha: {$gte: ISODate("2025-01-01")}})` → usa el índice para filtrar por prefijo (equality en `cliente`, range en `fecha`).

Regla ESR: **E**quality (cliente) → **S**ort/**R**ange (fecha). Left-prefix se respeta.

**A7.**
- **TF** (`tf_{i,p}`) = frecuencia con la que la **palabra visual** `i` aparece en la imagen `p`. Refleja cuánto de esa característica visual contiene la imagen.
- **IDF** (`log(N/df_i)`) = rareza global de la palabra visual `i` en la colección. Palabras visuales que aparecen en todas las imágenes son poco discriminantes.

Se calcula sobre el histograma BoVW (después de K-means).

**A8.** Ejemplo distinto de clase: **reserva temporal de asientos en cine**.
- Al ver la sala, la app consulta Redis (`GET sala:123:seats`). Si miss → consulta BD y guarda con TTL 3 min.
- Al reservar temporalmente un asiento → `SET seat:sala:123:A5 usuario_x EX 300` (bloqueo optimista, 5 min).
- Al confirmar → `UPDATE` en BD + `DEL cache` para invalidar.
- Justificación A-SIDE: lecturas mucho más frecuentes que escrituras, staleness aceptable dentro del TTL, si Redis cae la BD sigue funcionando (sistema degradado pero operativo).

Otros válidos: rate-limiting por IP, leaderboards de juego, sessions de e-commerce, catálogo de productos, feed personalizado con TTL corto.

**A9.**
- **Por DFs** cuando conoces la estructura formal de los datos (las DFs son declarativas y estables).
- **Por matriz de afinidad** cuando no hay DFs claras pero conoces las **queries frecuentes** y sus **frecuencias de acceso** por sitio. Es un enfoque **empírico/estadístico** basado en el uso.

**A10.** SPIMI:
1. Por cada bloque disponible en RAM, crea un **diccionario local** (hash) `term → posting_list`.
2. Al procesar cada token, busca `term` en el diccionario; si no está, crea nueva posting list.
3. Los `posting_list` empiezan con tamaño fijo; si se llenan, **duplican** su tamaño (amortized O(1) append).
4. Cuando la RAM se llena: ordena los términos del diccionario, escribe el bloque a disco.
5. Al final, merge multi-way de todos los bloques.

**Ventaja principal**: NO necesita mantener un mapeo global `term ↔ termID`, así que cabe más data por bloque, hay menos bloques → merge más rápido. Además almacena TF/posiciones fácilmente.

**A11.** La partition key determina el nodo donde vive el dato (via consistent hashing). Sin ella, la query tendría que ir a **todos los nodos** y filtrar → destroza la escalabilidad. `ALLOW FILTERING` fuerza ese escaneo distribuido — es un antipatrón en producción; se usa solo para exploración.

**A12.** Con `PRIMARY KEY ((a, b), c, d)`:
- `WHERE a=1 AND b=2` → ✅ (partition key completa).
- `WHERE a=1 AND b=2 AND c=3 AND d>5` → ✅ (PK completa + rango en clustering key con prefijo).
- `WHERE a=1` → ❌ (falta parte de la partition key `b`).
- `WHERE c=3` → ❌ (falta partition key completa).

**A13.** Coseno mide el ángulo entre vectores, ignorando magnitud. **Ejemplo del documento duplicado**: si tomas `d` y lo concatenas consigo mismo obtienes `d' = d ⊕ d`, semánticamente idéntico. La distancia euclidiana `‖d - d'‖` es grande (porque `d'` es el doble en magnitud), pero el **ángulo es 0** → coseno = 1. Por eso coseno respeta la similitud semántica; la euclidiana penaliza injustamente documentos largos.

**A14.**
- **Blog con miles de posts/día** → **GiST** (actualizaciones frecuentes son baratas; búsqueda un poco más lenta pero aceptable).
- **Archivo histórico / motor de búsqueda** → **GIN** (colección casi inmutable, se optimiza para lecturas rápidas; el costo alto de construcción/actualización no importa).

**A15.** **16 384 hash slots**. Se calcula `slot = CRC16(key) mod 16384`. Cada nodo master tiene asignado un rango de slots; la clave va al nodo que posee ese slot.

---

## Bloque B — Multimedia

**B1.** Búsqueda por descriptores locales:

```
Imagen query
    │
    ▼
[Extraer descriptores locales {Q1, Q2, …, Qn}] ← SIFT (dim 128)
    │
    ▼
[Índice multidimensional] ── KNN por cada Qi ──► candidatos (IdImg, Pj)
    │                        (filtrar-y-refinar)
    ▼
[Votación / combinación de resultados parciales]
    │
    ▼
Ranking final (aproximado, ANN)
```

Indexación previa: cada `Pj` de cada `Obji` se inserta como `(IdImagen, Pj)` en el índice.

Tipo de búsqueda que soporta: **CBIR por objeto / sub-imagen**. Encuentra imágenes que contienen el mismo objeto/región aunque aparezca **parcialmente (oclusión), a distinta escala o rotado**. SIFT es **invariante a escala** → maneja bien colecciones de imágenes con distintas resoluciones (como pide el enunciado).

Ejemplo válido: buscar todas las fotos de un logo en un dataset de miles de fotos de calles urbanas → basta con que un subconjunto de descriptores del logo haga match en la imagen candidata.

**B2.** Ver `S11_LowerBounding_BoVW_Multimodal.md` sección 3. Estructura clave: **heap MAX de tamaño K** (implementado con `heapq` en Python usando distancias negativas). El "peor del top-K" (raíz del heap max) funciona como `best_so_far` para podar con la Lower Bounding Distance.

```python
def Lower_Bounding_KNN(Q, K):
    heap = []
    for Ci in database:
        lb = LB(Q, Ci)
        if len(heap) == K:
            worst = -heap[0][0]
            if lb >= worst:
                continue         # poda sin calcular la distancia real
        true_dist = Dist(Q, Ci)
        if len(heap) < K:
            heapq.heappush(heap, (-true_dist, Ci))
        elif true_dist < -heap[0][0]:
            heapq.heapreplace(heap, (-true_dist, Ci))
    return sorted([(-d, o) for d, o in heap])
```

**B3.** Sistema de detección de copyright tipo TikTok:

**Framework**: pipeline multimodal con **late fusion**.

```
CONSTRUCCIÓN DEL ÍNDICE
─────────────────────────
Por cada video de la colección:
  1. Extraer N frames representativos     → embedding CNN (ResNet, ViT)
  2. Extraer audio → segmentar → MFCC     → embedding audio
  3. Transcribir con ASR                  → embedding BERT/BoW

Indexar cada modalidad por separado en su índice ANN:
  ├── índice_img   (HNSW sobre pgvector)
  ├── índice_audio (HNSW)
  └── índice_texto (GIN sobre to_tsvector)

RECUPERACIÓN
────────────
Video query → mismo pipeline → 3 rankings parciales
             │
             ▼
   score = α·score_img + β·score_audio + γ·score_texto   (late fusion, pesos ajustables)
             │
             ▼
        Top-K final (candidatos a copyright)
```

**Por qué es mejor en desempeño**:
- **Escalable**: cada índice ANN es sublineal (O(log N) con HNSW), y los tres corren en paralelo.
- **Robusto**: si a un video pirata le quitaron el audio, aún se detecta por imagen y texto.
- **Ajustable**: cambiar pesos α/β/γ no requiere reindexar (a diferencia de early fusion).
- **Modular**: cada modalidad usa la estructura de índice adecuada.

**B4.** BoVW:

Construcción:
1. Extraer descriptores locales (SIFT) de todas las imágenes.
2. **K-Means** sobre esa nube → K centroides = **vocabulario visual** (palabras visuales).
3. Cuantización: cada descriptor de una imagen → palabra visual más cercana.
4. Histograma por imagen sobre las K palabras.
5. Pesos TF-IDF sobre los bins del histograma.

Consulta:
1. Extraer descriptores de la query, cuantizar → histograma BoVW.
2. Aplicar TF-IDF.
3. Similitud coseno con los histogramas de la colección.
4. Ranking por coseno decreciente.

Por qué K-Means: define un vocabulario finito que permite tratar la imagen como "bag of words" → habilita índice invertido, TF-IDF y coseno (todas las técnicas de IR textual).

---

## Bloque C — Fragmentación y consulta distribuida

**C1 — RAPPI**

**(1 pt) Fragmentación**:
- `Pedidos` por RANGE(`FechaPedido`) — 3 rangos, uno por nodo. Razón: 90% queries por rango de fecha; permite partition pruning.
- `Repartidores` por HASH(`IdRepartidor`) — reparto uniforme (id auto-incremental); búsquedas puntuales por id van a 1 solo nodo.

Notación:
```
Pedidos      = P₁ ∪ P₂ ∪ P₃,   Pᵢ = σ_{FechaPedido ∈ rangoᵢ}(Pedidos)
Repartidores = R₁ ∪ R₂ ∪ R₃,   Rᵢ = σ_{HASH(IdRepartidor) mod 3 = i}(Repartidores)
```

**(4 pts) Algoritmo distribuido**: el JOIN es por `Ciudad`, pero ninguna tabla está fragmentada por `Ciudad` → **reparticionar**.

Gráfico:
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

Pseudocódigo:
```
-- Fase 1: reparticionar por Ciudad
create temp s1.rep_c  = ⋃_i (select * from s_i.rep where hash(Ciudad) mod 3 = 0)
create temp s2.rep_c  = ⋃_i (select * from s_i.rep where hash(Ciudad) mod 3 = 1)
create temp s3.rep_c  = ⋃_i (select * from s_i.rep where hash(Ciudad) mod 3 = 2)
(idem s_i.ped_c)

-- Fase 2: JOIN + GROUP BY LOCAL en cada nodo
create temp s_i.agg
    select Ciudad,
           count(distinct IdRepartidor) as TotalRep,
           sum(Monto)                   as MontoTotal
    from s_i.rep_c join s_i.ped_c using(Ciudad)
    group by Ciudad

-- Fase 3: reparticionar por RANGE(MontoTotal) para ORDER BY sin sort central
--         vector [m0, m1] estimado en el coordinador (min, max, #tuplas)
create temp s1.final
    merge(
      select * from s1.agg where MontoTotal >= m1 order by MontoTotal desc,
      select * from s2.agg where MontoTotal >= m1 order by MontoTotal desc,
      select * from s3.agg where MontoTotal >= m1 order by MontoTotal desc
    )
create temp s2.final  = ... (m0 ≤ MontoTotal < m1)
create temp s3.final  = ... (MontoTotal < m0)

-- Fase 4: coordinador concatena
result = s1.final || s2.final || s3.final
```

**(3 pts) SQL derivado**:
```sql
CREATE TABLE Pedidos (
  IdPedido int, IdCliente int, FechaPedido date, Monto decimal,
  Ciudad text, Estado text
) PARTITION BY RANGE (FechaPedido);

CREATE TABLE Repartidores (
  IdRepartidor int, Nombre text, TipoVehiculo text,
  Ciudad text, Disponibilidad bool, Calificacion float
) PARTITION BY HASH (IdRepartidor);

-- Consulta (con reparticionamiento intermedio manejado por el motor distribuido)
SELECT   R.Ciudad,
         COUNT(DISTINCT R.IdRepartidor) AS TotalRepartidores,
         SUM(P.Monto)                   AS MontoTotal
FROM     Repartidores R
JOIN     Pedidos P ON R.Ciudad = P.Ciudad
GROUP BY R.Ciudad
ORDER BY MontoTotal DESC;
```

**C2 — MinTerm**

Predicados: `p1: tipoPago=EFECTIVO`, `p2: tipoPago=ELECTRONICO`, `p3: monto>100`, `p4: monto>20`.

- `p1 ∧ p2` = contradictorio (mutuamente exclusivos).
- `p3 ⇒ p4`, entonces `p3 ∧ ¬p4` es contradictorio.
- `¬p1 ∧ ¬p2` se elimina si el catálogo se cierra a {EFECTIVO, ELECTRONICO}.

Fragmentos finales (6):

| # | Predicado |
|---|---|
| F1 | `tipoPago=EFECTIVO ∧ monto > 100` |
| F2 | `tipoPago=EFECTIVO ∧ 20 < monto ≤ 100` |
| F3 | `tipoPago=EFECTIVO ∧ monto ≤ 20` |
| F4 | `tipoPago=ELECTRONICO ∧ monto > 100` |
| F5 | `tipoPago=ELECTRONICO ∧ 20 < monto ≤ 100` |
| F6 | `tipoPago=ELECTRONICO ∧ monto ≤ 20` |

**C3 — Búsqueda exhaustiva de planes de join**

Tamaños: S=40 (Srv1), T=15 (Srv2), V=20 (Srv3).

Planes posibles (join asociativo/conmutativo):
- `(S ⋈ T) ⋈ V`: primero llevar S y T al mismo sitio.
  - Enviar S(40) a Srv2 → costo transporte 40; `S⋈T` cabe en Srv2, luego llevarlo a Srv3 con V.
  - Enviar T(15) a Srv1 → costo 15 (mejor).
- `(S ⋈ V) ⋈ T`: enviar V(20) a Srv1 → costo 20.
- `(T ⋈ V) ⋈ S`: enviar T(15) o V(20) al otro; luego join con S(40) → enviar T al Srv3 (15), luego el intermedio a Srv1.

**Poda**:
- Se poda enviar la relación MÁS grande (S=40) — nunca es óptimo.
- `(T ⋈ V) ⋈ S` empieza fusionando las 2 más pequeñas → **plan ganador**: enviar T a Srv3 (costo 15), calcular `T⋈V` en Srv3, luego enviar el resultado al Srv1 y hacer join con S.

Árbol:
```
                     JOIN de 3
                   /     |      \
              (S⋈T)⋈V  (S⋈V)⋈T  (T⋈V)⋈S   ← candidatos
                 │        │         │
              subplan   subplan   subplan
             (enviar     (enviar   (enviar T
              T a S1)    V a S1)    o V, elegir)
                 │        │          │
              costo       costo      COSTO MENOR ← elegido
              alto        medio       (T=15)
```

Podas: se descartan ramas que envían S(40).

**C4 — Fragmentación vertical (matriz 5×5)**

Aplicar BEA sobre:
```
       A    B    C    D    E
A     105   29   0   48   5
B      29   38   1   55   2
C       0    1  30    7  75
D      48   55   7   39  10
E       5    2  75   10  51
```

Grupos por afinidad: `{A, B, D}` (valores 29, 48, 55 fuertes entre sí) y `{C, E}` (75 muy fuerte). Cruzadas muy bajas (0, 1, 5, 7, 10).

Orden BEA: `A, B, D | C, E`.

Matriz agrupada:
```
       A    B    D  |  C    E
A     105   29  48  |  0    5
B      29   38  55  |  1    2
D      48   55  39  |  7   10
       ─────────────────────
C       0    1   7  | 30   75
E       5    2  10  | 75   51
```

Cortando entre `D` y `C`:
- `C_TA = 105+29+29+38+48+55+55+39 = 398`
- `C_BA = 30+75+75+51 = 231`
- `C_OQ = 0+5+1+2+7+10+0+1+7+5+2+10 = 50`
- `z = 398·231 − 50² = 91 938 − 2 500 = 89 438`

Fragmentación resultante:
$$R_1 = [K, A, B, D], \quad R_2 = [K, C, E]$$

**C5 — E-commerce**

**(1 pt) Fragmentación**:
- `Clientes` por LIST(`Pais`): `C_A = σ_{Pais=A}(Clientes)`, `C_B = σ_{Pais=B}(Clientes)`, `C_C = σ_{Pais=C}(Clientes)`.
- `Ventas` fragmentación derivada: `V_i = Ventas ⋉_{IDCliente} C_i` (el IDCliente es llave en Clientes → desarticulación garantizada).

**(4 pts) Algoritmo**:

- Nodo A: procesa `C_A` y `V_A`.
- Nodo C: procesa `C_C` y `V_C`.
- Nodo B: queda libre → usarlo para el sort intermedio (aprovechar los 3 esclavos como pide el enunciado).

```
-- Fase 1: JOIN LOCAL + orden local en nodos A y C
create temp s_A.res
    select C.Nombre, C.Apellidos, V.FechaVenta, V.MontoTotal
    from s_A.V_A join s_A.C_A using(IDCliente)
    order by MontoTotal desc, FechaVenta

create temp s_C.res
    select C.Nombre, C.Apellidos, V.FechaVenta, V.MontoTotal
    from s_C.V_C join s_C.C_C using(IDCliente)
    order by MontoTotal desc, FechaVenta

-- Fase 2: enviar los dos streams ordenados al nodo B, hacer MERGE (no sort)
create temp s_B.final
    merge(s_A.res, s_C.res)     -- merge de dos streams ordenados por (MontoTotal desc, FechaVenta asc)

-- Fase 3: coordinador recibe de s_B
result = select * from s_B.final
```

Notas:
- `Nodo B libre` → sort centralizado ahí en vez de en el coordinador (menos carga en el maestro).
- Alternativa: reparticionar por RANGE(MontoTotal) entre los 3 esclavos para orden distribuido total; útil si el resultado es enorme.

**C6 — MINSA**

Fragmentación de `Paciente` por año de nacimiento — como se agregan RN constantemente y RANGE se desbalancearía, elegir **HASH(YEAR(FechaNac))** para mantener balance dinámico. (Alternativa: RANGE con rebalanceo periódico.)

El JOIN es por DNI. Ninguna tabla está fragmentada por DNI → reparticionar ambas por `HASH(DNI) mod 3`.

Gráfico:
```
    S1: MED_esp1, PAC_h1        S2: MED_esp2, PAC_h2        S3: MED_esp3, PAC_h3
        │                            │                            │
        ▼ reparticionar por HASH(DNI) mod 3
    ┌───────────┐               ┌───────────┐               ┌───────────┐
    │ MED_d, PAC_d (hash 0)     │ MED_d, PAC_d (hash 1)     │ MED_d, PAC_d (hash 2)
    └────┬──────┘               └────┬──────┘               └────┬──────┘
         │ JOIN LOCAL + GROUP BY Especialidad LOCAL (parcial)
         ▼                             ▼                             ▼
    (Esp, count_parcial)          (Esp, count_parcial)          (Esp, count_parcial)
         │                             │                             │
         └──────────── coordinador: sumar parciales por Esp ──────────┘
                                     │
                                     ▼
                              ORDER BY count DESC
```

SQL:
```sql
-- Fragmentación original
CREATE TABLE Medico (...) PARTITION BY LIST (Especialidad);
  -- 3 grupos de ~7 especialidades cada uno, uno por nodo

CREATE TABLE Paciente (...) PARTITION BY HASH (EXTRACT(YEAR FROM FechaNac));

-- Reparticionamiento intermedio + join
WITH med_d AS (
  SELECT * FROM Medico  -- ya con reparticionamiento por HASH(DNI_Medico)
),
pac_d AS (
  SELECT * FROM Paciente
)
SELECT   M.Especialidad, COUNT(*) AS Conteo
FROM     med_d M
JOIN     pac_d P ON M.DNI_Medico = P.DNI_Paciente
GROUP BY M.Especialidad
ORDER BY Conteo DESC;
```

Puntos clave:
1. Reparticionar por HASH(DNI) permite JOIN local.
2. Agregación **parcial** local + suma final en coordinador (Especialidad tiene ≤ 20 valores, no vale reparticionar).
3. ORDER BY count DESC es central sobre pocas filas.

---

## Bloque D — Cortas conceptuales

**D1.** Completitud + reconstrucción requieren **integridad referencial** (todo valor del atributo de join en R debe existir en S). Desarticulación requiere que el **atributo de join sea llave** en la relación dueña S (si no, una tupla de R haría match con dos S_i distintas → aparecería en dos fragmentos R_i, R_j).

```sql
-- R ⋉_COD S
SELECT R.*
FROM R
WHERE R.cod IN (SELECT S.cod FROM S);
-- o equivalente:
SELECT DISTINCT R.*
FROM R JOIN S ON R.cod = S.cod;
```

**D2.** Como no hay datos y K es auto-incremental → distribución uniforme del hash → **HASH(K) mod 4**. Es la única técnica que garantiza balance sin conocer la distribución de los datos.

Ejemplo:
- F1 = `σ_{HASH(K) mod 4 = 0}(R)` → K=4, 8, 12, ...
- F2 = `σ_{HASH(K) mod 4 = 1}(R)` → K=1, 5, 9, ...
- F3 = `σ_{HASH(K) mod 4 = 2}(R)` → K=2, 6, 10, ...
- F4 = `σ_{HASH(K) mod 4 = 3}(R)` → K=3, 7, 11, ...

RANGE no serviría porque no conocemos el max/min de K.

**D3.**
1. **Descomposición**: SQL → álgebra relacional normalizada; eliminar redundancias; empujar σ y π hacia abajo.
2. **Localización**: reemplazar relaciones por unión de sus fragmentos; aplicar reglas por tipo de fragmentación (H primaria, H derivada, V) para eliminar sub-árboles vacíos.
3. **Optimización**: elegir el plan que minimiza **transporte de red**. Considera reparticionamiento, semi-joins, joins locales, agregación parcial.

**D4.** `AVG` **no es asociativa**: `AVG(AVG(A), AVG(B)) ≠ AVG(A ∪ B)` cuando A y B tienen tamaños distintos. Cada nodo debe enviar `SUM` y `COUNT` por separado; el coordinador calcula `SUM_total / COUNT_total`.

**D5.**
- **ONE**: prototipos, logs, contadores tolerantes a inconsistencia — máxima velocidad.
- **QUORUM**: transacciones de negocio típicas (perfil de usuario, pedidos). Balance entre disponibilidad y consistencia. Con RF=3 requiere 2 confirmaciones.
- **ALL**: datos críticos que exigen strong consistency (movimientos financieros, contadores de stock crítico). Requiere todas las réplicas; sensible a caídas.
