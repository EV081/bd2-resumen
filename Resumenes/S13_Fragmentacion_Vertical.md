# S13 — Fragmentación Vertical

---

## 1. Idea

Dividir una tabla `R(K, A1, A2, ..., An)` en varios fragmentos verticales `R1, R2, ...` cada uno con un **subconjunto de atributos**, siempre incluyendo la **llave `K`** para poder reconstruir por join.

```
R(K, A, B, C, D, E)

  ↓ fragmentación vertical

R1(K, A, B, D)         R2(K, C, E)
```

Se parece a la **normalización** (formas normales), pero con motivación de **rendimiento distribuido** en lugar de eliminar redundancia lógica.

---

## 2. Propiedades

### Completitud

Cada atributo de R debe estar en al menos un fragmento:
$$\bigcup \text{attrs}(R_i) = \text{attrs}(R)$$

### Desarticulación (NO es deseable como en la horizontal)

En vertical, la llave se **repite** en todos los fragmentos. Los atributos **no-llave** deberían aparecer en un solo fragmento.

### Reconstrucción sin pérdida (Lossless Join)

R se recupera por join natural sobre K:
$$R = R_1 \bowtie_K R_2 \bowtie_K \dots$$

Sin tuplas espurias ni pérdida.

**Dos formas de garantizarlo**:
1. **Dependencias funcionales** (formalismo de la normalización).
2. **Matriz de afinidad de atributos**.

---

## 3. Fragmentación por Dependencias Funcionales

### DF: X → Y

Dos tuplas con el mismo valor de X deben tener el mismo valor de Y.

Ejemplo: `{Profesor} → {Curso}`.

### Descomposición sin pérdida

Sea `R = R_1 ∪ R_2` (unión de atributos):

- Preservan **atributos** de R: `attrs(R) = attrs(R_1) ∪ attrs(R_2)`.
- Preservan **DFs**: cada DF de F queda en algún F_i.
- **Sin tuplas falsas**: `(R_1 ∩ R_2) → R_1` **o** `(R_1 ∩ R_2) → R_2` debe ser una DF válida en `F^+`.

### Ejemplo

```
R(A, B, C, D, E),   F = { AB→C, C→D, C→E }

Descomposición: R1(A,B,C,D), R2(C,E)
F1 = { AB→C, C→D },  F2 = { C→E }
R1 ∩ R2 = {C},  C → E ∈ F^+  ✅
```

### Normalización a 3FN / FNBC

- **3FN**: para toda DF `X → Y`, `X` es superclave **o** `Y` es atributo primo.
- **FNBC**: para toda DF `X → Y`, `X` es superclave (sin excepción).

**Algoritmo 3FN**:
1. Calcular F mínimo (canónico).
2. Cada DF `X → Y` → una relación `R_i(X ∪ Y)`.
3. Si no hay llave en ninguna relación, agregarla.
4. Eliminar relaciones redundantes.

---

## 4. Fragmentación por Matriz de Afinidad + BEA ⭐

**Cuándo usarla**: cuando no aplican bien las DFs pero conocemos las **queries frecuentes** y sus **frecuencias de acceso**.

Objetivo: agrupar atributos que **frecuentemente se consultan juntos** en el mismo fragmento.

### Paso 1 — Matriz de uso

Para cada query `q_k`, definir:
$$\text{use}(q_k, A_j) = \begin{cases} 1 & \text{si } q_k \text{ referencia a } A_j \\ 0 & \text{en otro caso} \end{cases}$$

Ejemplo con `R(A1, A2, A3, A4)` y queries `{q1, q2, q3, q4}`:

|     | A1 | A2 | A3 | A4 |
|-----|----|----|----|----|
| q1  | 1  | 0  | 1  | 0  |
| q2  | 0  | 1  | 1  | 0  |
| q3  | 0  | 1  | 0  | 1  |
| q4  | 0  | 0  | 1  | 1  |

### Paso 2 — Matriz de afinidad

$$\text{aff}(A_i, A_j) = \sum_{\substack{k \,:\, \text{use}(q_k, A_i) = 1 \\ \wedge\, \text{use}(q_k, A_j) = 1}}\ \sum_{l} \text{ref}_l(q_k) \cdot \text{acc}_l(q_k)$$

- `ref_l(q_k)`: número de referencias al par de atributos en el sitio `l`.
- `acc_l(q_k)`: frecuencia de ejecución de `q_k` en el sitio `l`.
- Suele asumirse `ref_l(q_k) = 1`.

Ejemplo (con frecuencias `acc_l(q1)={15,20,10}`, etc.):

```
         A1    A2    A3    A4
    A1   45    0    45     0
    A2    0   80     5    75
    A3   45    5    53     3
    A4    0   75     3    78
```

### Paso 3 — Bond Energy Algorithm (BEA)

Reordena filas y columnas para **maximizar la afinidad global**. Los atributos que suelen ir juntos quedan **contiguos**.

**Fórmula de contribución** (al insertar `A_x` entre `A_i` y `A_j`):

$$\text{Cont}(A_i, A_x, A_j) = 2 \cdot \text{bond}(A_i, A_x) + 2 \cdot \text{bond}(A_x, A_j) - 2 \cdot \text{bond}(A_i, A_j)$$

donde `bond(A_a, A_b) = Σ_z aff(A_z, A_a) · aff(A_z, A_b)`.

**Condición de contorno**:
$$\text{aff}(A_0, A_j) = \text{aff}(A_i, A_0) = \text{aff}(A_{n+1}, A_j) = \text{aff}(A_i, A_{n+1}) = 0$$

**Proceso simplificado** (el que aparece en las slides):
1. Fija dos atributos.
2. Para cada atributo restante `A_x`, prueba insertarlo en cada posición y calcula `Cont(...)`.
3. Elige la posición con **mayor** contribución.
4. Repite hasta ubicar todos.
5. **Reordena las filas en el mismo orden que las columnas**.

Ejemplo del profe:
- Fijas `A1, A2`. Analizas dónde va `A3`:
  - `Cont(A0, A3, A1) = 8820`
  - `Cont(A1, A3, A2) = 10150`  ← máximo
  - `Cont(A2, A3, A5) = 1780`
  - Orden: `[A1, A3, A2]`.
- Analizas dónde va `A4`:
  - `Cont(A2, A4, A5) = ...`  (máximo)
  - Orden final: `[A1, A3, A2, A4]`.

**Matriz reordenada** (afinidad agrupada):

```
         A1    A3    A2    A4
    A1   45    45     0     0
    A3   45    53     5     3
    A2    0     5    80    75
    A4    0     3    75    78
```

Ya se ve visualmente que `{A1, A3}` están fuertemente relacionados y `{A2, A4}` también, mientras que las esquinas cruzadas son bajas.

### Paso 4 — Partitioning (encontrar el corte) ⭐

Encontrar puntos de división a lo largo de la diagonal que **maximicen accesos a un solo fragmento** y **minimicen accesos cruzados**.

Función objetivo:
$$z = C_{TA} \cdot C_{BA} - C_{OQ}^2$$

- `C_TA`: suma de afinidad en el cuadrante Top-Above del corte (fragmento 1).
- `C_BA`: suma de afinidad en el cuadrante Bottom-Below (fragmento 2).
- `C_OQ`: suma de afinidad en los cuadrantes cruzados (Top-Right + Bottom-Left).

Se **maximiza z** probando cada punto de corte posible.

Para el ejemplo, el corte entre `A3` y `A2`:

```
         A1    A3  |  A2    A4
    A1   45    45  |   0     0
    A3   45    53  |   5     3
         ─────────────────────
    A2    0     5  |  80    75
    A4    0     3  |  75    78

  Fragmento 1 (arriba): TA
  Fragmento 2 (abajo): BA
  Cruzados (OQ): esquinas cruzadas

  C_TA = 45+45+45+53 = 188
  C_BA = 80+75+75+78 = 308
  C_OQ = 0+0+5+3+0+5+0+3 = 16
  z = 188 · 308 - 16² = 57 704 - 256 = 57 448
```

Resultado:

$$R_1 = [K, A_1, A_3], \quad R_2 = [K, A_2, A_4]$$

La llave `K` se **repite** en ambos para el join de reconstrucción.

---

## 5. Ejemplo del examen anterior (matriz 5×5)

```
       A    B    C    D    E
A     105   29   0   48   5
B      29   38   1   55   2
C       0    1  30    7  75
D      48   55   7   39  10
E       5    2  75   10  51
```

**Grupos visibles**:
- `{A, B, D}` tienen afinidad alta entre sí: `aff(A,B)=29`, `aff(A,D)=48`, `aff(B,D)=55`.
- `{C, E}` tienen afinidad alta: `aff(C,E)=75`.
- Cruzadas bajas: `aff(A,C)=0`, `aff(A,E)=5`, `aff(B,C)=1`, `aff(B,E)=2`.

**Aplicando BEA** el orden queda: `A, B, D | C, E`. Corte tras `D`.

**Matriz agrupada**:
```
       A    B    D  |  C    E
   A  105   29  48  |  0    5
   B   29   38  55  |  1    2
   D   48   55  39  |  7   10
       ───────────────────────
   C    0    1   7  | 30   75
   E    5    2  10  | 75   51
```

**Fragmentación resultante**:
$$R_1 = [K, A, B, D], \quad R_2 = [K, C, E]$$

---

## 6. Preguntas típicas (autotest)

1. ¿En qué caso usar fragmentación vertical por DFs vs por matriz de afinidad?
2. Dada `R(A,B,C,D,E)` con `F = {A→B, A→C, C→D, B→E}`, aplica normalización a 3FN.
3. Dada una matriz de afinidad, aplica BEA paso a paso para determinar el orden y luego encuentra el punto de corte.
4. ¿Por qué se repite la llave en todos los fragmentos verticales?
5. ¿Qué mide la "desarticulación" en fragmentación vertical? ¿Es igual que en horizontal?
6. Explica qué representa `C_TA · C_BA - C_OQ²` en la fórmula de partitioning.
