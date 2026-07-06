# S12 — Fragmentación Horizontal ⭐ (con gráficos, como pidió el profe)

---

## 1. Diseño de BD Distribuidas

### Top-Down vs Bottom-Up

**Top-Down** (diseño desde cero):
```
[Diseño global del esquema] → [Fragmentar] → [Asignar a nodos]
```
Se aplica cuando: la BD se construye desde cero, similar al diseño centralizado tradicional.

**Bottom-Up** (integración de BDs existentes):
```
[BD local A]  ┐
[BD local B]  ├──► [Esquema exportado] → [Integración] → [Esquema global]
[BD local C]  ┘
```
Se aplica cuando:
- Ya existen **varias bases** en distintos sitios y hay que integrarlas.
- Sistemas **legacy / heredados**.
- **Fusiones o adquisiciones** de empresas.
- No hay problema de fragmentación (los datos ya están repartidos); el reto es la **integración de esquemas**.

### Buen sistema distribuido — propiedades

| Propiedad | Significa |
|---|---|
| **Transparencia** | Dar la impresión de un solo sistema (acceso, ubicación, heterogeneidad, concurrencia) |
| **Flexibilidad** | Agregar/quitar nodos sin downtime |
| **Confiabilidad** | Sigue trabajando con fallos (replicación, consenso) |
| **Desempeño** | Baja latencia, tiempo de respuesta |
| **Escalabilidad** | Crecer sin cuellos de botella cuadráticos |

**Regla clave**: minimizar el **costo de red** (transferencia entre nodos es lo más caro).

### Backup vs Replicación (pregunta clásica 1 pt)

| | **Backup** | **Replicación** |
|---|---|---|
| Qué es | Copia puntual e histórica | Copias vivas sincronizadas |
| Estado | Offline, no consultable | Online, consultable, activa |
| Objetivo | Recuperación, durabilidad | Disponibilidad, tolerancia a fallos |
| Frecuencia | Periódica (diaria, horaria) | Continua / casi tiempo real |
| Contra borrado lógico | **Protege** (rollback en el tiempo) | **NO protege** (replica el borrado) |

- **Backup** para: cumplimiento, ransomware, corrupción, borrado accidental, recuperación punto en el tiempo.
- **Replicación** para: alta disponibilidad 24/7, failover, escalar lecturas, baja latencia geográfica.

**Complementarios**. La replicación da disponibilidad; el backup da protección histórica.

---

## 2. Fragmentación — dos tipos

**Primaria**: usando predicados solo sobre R.
**Derivada**: usando predicados de relaciones foráneas — `F_i = R ⋉ S_i`.

Reglas obligatorias:
1. **Completitud**: `⋃ F_i = R` (nada se pierde).
2. **Desarticulación** (disyunción): `F_i ∩ F_j = ∅` para `i ≠ j` (nada se duplica).
3. **Reconstrucción**: R se puede recuperar por unión (H) o join (V).

---

## 3. Técnicas de fragmentación horizontal

### 3.1 Round-Robin

```
     R
     │
     ▼
┌────┬────┬────┐
│ N1 │ N2 │ N3 │  ← se distribuyen las tuplas 1,4,7,... a N1; 2,5,8,... a N2; etc.
└────┴────┴────┘
```
- ✅ Distribución **uniforme**.
- ✅ Bueno para **escanear** una relación completa.
- ❌ **Malo** para consultas puntuales o por rango (hay que ir a todos los nodos).

### 3.2 Hash

```
      R          f_hash(k) mod 3
      │
      ▼
┌──────────┬──────────┬──────────┐
│ N1: k=1  │ N2: k=2  │ N3: k=3  │
│    k=4   │    k=5   │    k=6   │
└──────────┴──────────┴──────────┘
```
- ✅ Distribución uniforme si la función hash es buena.
- ✅ Bueno para **consultas puntuales sobre la clave** y **joins por la clave**.
- ❌ Malo para consultas por rango.

**Uso típico**: id auto-incremental.

### 3.3 Range (por rango)

```
      R              vector [k0, k1]
      │
      ▼
┌──────────────┬─────────────────┬────────────────┐
│ N1: k < k0   │ N2: k0 ≤ k < k1 │ N3: k ≥ k1     │
└──────────────┴─────────────────┴────────────────┘
```
- ✅ Bueno para **consultas por rango** en el atributo de partición.
- ⚠️ Riesgo de sesgo si el vector `[k0, k1]` está mal elegido.

**Uso típico**: fechas, valores continuos ordenables.

### 3.4 List (por categoría)

```
      R              País ∈ {A, B, C}
      │
      ▼
┌────────────┬────────────┬────────────┐
│ N1: País=A │ N2: País=B │ N3: País=C │
└────────────┴────────────┴────────────┘
```
- ✅ Ideal para atributos categóricos con **pocos valores discretos**.
- ✅ Cuando el 90% de las queries filtran por ese atributo.

### Cómo elegir

| Patrón de consulta | Técnica |
|---|---|
| Por rango de fecha | **Range** |
| Por id auto-incremental / puntual | **Hash** |
| Por categoría de pocos valores | **List** |
| Scan completo, sin filtro | **Round-robin** |

---

## 4. Cuidado con fragmentaciones malas

**Ejemplo 1 — pérdida de tuplas** (viola completitud):
```
F1: salary < 10       F2: salary ≥ 20
                    ⊥ tuplas con 10 ≤ salary < 20 se PIERDEN
```

**Ejemplo 2 — duplicación** (viola desarticulación):
```
F1: salary < 10       F2: salary > 5
                    ⊥ tuplas con 5 < salary < 10 aparecen en AMBOS
```

Solución sistemática: **Predicados MinTerm**.

---

## 5. Algoritmo MinTerm (predicados de términos mínimos) ⭐

### Idea

Dado un conjunto de predicados simples `P = {p1, p2, ..., pn}` usados en las queries frecuentes, generar **todos los mintérminos**: conjunciones donde cada `pk` aparece como `pk` (verdadero) o `¬pk` (falso).

$$m_j = p_1^* \land p_2^* \land \dots \land p_n^*, \quad \text{con } p_k^* \in \{p_k, \neg p_k\}$$

Son `2^n` combinaciones.

### Pasos

1. **Considerar predicados simples** que aparecen en las queries.
2. **Generar los 2^n mintérminos**.
3. **Eliminar los inútiles**:
   - **Contradictorios**: ej. `monto > 100 ∧ monto ≤ 20` es imposible.
   - **Semánticamente vacíos**: por conocimiento del dominio.
4. **Simplificar** los que quedan.
5. Cada mintérmino sobreviviente = un fragmento `σ_{m_j}(R)`.

**Se garantiza completitud + desarticulación** por construcción (los mintérminos son mutuamente exclusivos y cubren el espacio).

### Ejemplo completo (basado en PC3)

Predicados: `p1: tipoPago=EFECTIVO`, `p2: tipoPago=ELECTRONICO`, `p3: monto>100`, `p4: monto>20`.

**Observaciones semánticas**:
- `p1` y `p2` son **mutuamente exclusivos** (un pago es uno u otro): `p1 ∧ p2` es vacío.
- `p3 ⇒ p4` (todo monto>100 cumple monto>20): `p3 ∧ ¬p4` es vacío.
- Al menos uno de {p1, p2} debe ser cierto (si el catálogo se cierra a estos dos): `¬p1 ∧ ¬p2` se puede omitir según semántica.

**Mintérminos que sobreviven** (usando `p1 xor p2` y niveles `¬p4`, `p4∧¬p3`, `p3`):

| # | Fragmento |
|---|---|
| F1 | `tipoPago=EFECTIVO ∧ monto > 100` |
| F2 | `tipoPago=EFECTIVO ∧ 20 < monto ≤ 100` |
| F3 | `tipoPago=EFECTIVO ∧ monto ≤ 20` |
| F4 | `tipoPago=ELECTRONICO ∧ monto > 100` |
| F5 | `tipoPago=ELECTRONICO ∧ 20 < monto ≤ 100` |
| F6 | `tipoPago=ELECTRONICO ∧ monto ≤ 20` |

Son **6 fragmentos finales** — completos y disjuntos.

---

## 6. Fragmentación Horizontal Derivada (semi-join)

Cuando fragmentamos `R` en base a cómo se fragmentó otra tabla `S` con la que se hace join.

### Definición formal

$$R_i = R \ltimes S_i, \quad i = 1, \dots, n$$

El semi-join `R ⋉ S` devuelve las tuplas de R que tienen algún match en S:

$$R \ltimes S = \{ t : t \in R \land \exists s \in S \text{ tal que } t \text{ hace match con } s \}$$

### En SQL

```sql
-- R ⋉_cod S  (semi-join)
SELECT R.*
FROM R
WHERE R.cod IN (SELECT S.cod FROM S);

-- O equivalente:
SELECT DISTINCT R.*
FROM R JOIN S ON R.cod = S.cod;
```

### Notación de fragmento derivado

Si `S` está fragmentado como `S = S_1 ∪ S_2 ∪ ... ∪ S_n`, la fragmentación derivada de R es:

```
R_i = R ⋉_cod S_i
```

`R` es **miembro**, `S` es **dueño**.

### Diagrama

```
         S (dueño)                    R (miembro)
   ┌────┬────┬────┐              ┌────┬────┬────┐
   │ S1 │ S2 │ S3 │              │ R1 │ R2 │ R3 │
   └─┬──┴──┬─┴─┬──┘              └────┴────┴────┘
     │     │   │                    ↑    ↑    ↑
     └─────┼───┼────── R ⋉ S1 ──────┘    │    │
           └───┼────── R ⋉ S2 ───────────┘    │
               └────── R ⋉ S3 ────────────────┘
```

### Garantizar propiedades

**Completitud + reconstrucción** → **integridad referencial**:
> Cada valor del atributo de join en R debe existir en la relación dueña S.

**Desarticulación** → **el atributo de join debe ser LLAVE en la relación dueña**.
- Si el atributo NO es llave en S, una tupla de R puede hacer match con dos tuplas de S diferentes (en S1 y en S2) → aparecería en R1 y en R2 → viola desarticulación.

**Si esto falla**, aparecen:
- **Tuplas perdidas**: R tiene una fila sin match en ningún S_i (pierde reconstrucción).
- **Tuplas falsas / duplicadas**: al reconstruir por join se producen filas que no existían.

---

## 7. Ejercicio clásico

```
Libro(Cod, Titulo, Precio, ...)
Almacen(CodAlm, CodPostal, ...)
Existencias(Cod, CodAlm, TotalExistencias)
```

- Fragmentar `Libro` por rango de `Precio` en `[20, 50, 100]`:
  ```
  L1 = σ_{Precio < 20}(Libro)
  L2 = σ_{20 ≤ Precio < 50}(Libro)
  L3 = σ_{50 ≤ Precio < 100}(Libro)
  L4 = σ_{Precio ≥ 100}(Libro)
  ```
- Fragmentar `Almacen` por `CodPostal` en `[3500, 70000]`:
  ```
  A1 = σ_{CodPostal < 3500}(Almacen)
  A2 = σ_{3500 ≤ CodPostal < 70000}(Almacen)
  A3 = σ_{CodPostal ≥ 70000}(Almacen)
  ```
- `Existencias` fragmentado **derivadamente** respecto a `Almacen` (el `CodAlm` es llave en `Almacen`):
  ```
  E_i = Existencias ⋉_CodAlm A_i
  ```

Consulta:
```sql
SELECT Codigo, TotalExistencias
FROM Libro
WHERE Precio > 15 AND Precio < 55;
```
- Toca los fragmentos `L1` (Precio<20, subrango 15<x<20), `L2` (20≤x<50) y `L3` (50≤x<55, subrango).
- Cada nodo ejecuta su subconsulta filtrada; el coordinador une con `UNION ALL`.

---

## 8. Preguntas típicas (autotest)

1. ¿En qué escenarios se aplica el diseño Bottom-Up?
2. Diferencias funcionales entre backup y replicación. ¿Cuál protege contra borrado lógico?
3. Tienes una tabla `Pedidos(id, fecha, monto, ciudad)` con 3 nodos. Elige técnica de fragmentación horizontal si:
   - a) 90% de queries filtra por rango de fecha.
   - b) id es auto-incremental y las queries son puntuales.
   - c) Todas las queries filtran por ciudad ∈ {Lima, Arequipa, Trujillo}.
4. Aplica MinTerm a los predicados `{sexo='M', sexo='F', edad>=18, edad>=65}`. Da los fragmentos finales.
5. ¿Por qué el atributo de join debe ser llave en la relación dueña para una fragmentación derivada?
6. Escribe en SQL `Existencias ⋉_CodAlm Almacen`.
7. Dibuja gráficamente cómo se fragmenta round-robin vs hash vs range para 3 nodos.
