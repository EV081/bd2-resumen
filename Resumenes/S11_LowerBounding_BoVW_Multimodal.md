# S11 — Lower Bounding, BoVW, Multimodal ⭐ FIJA

> Multimodal entra con **teoría fuerte**. Y el KNN con Lower Bounding salió como ejercicio en el Final 2025-1.

---

## 1. Filtrar-y-refinar

Idea: la distancia verdadera es cara. Usa una **distancia barata** como filtro para descartar objetos, y solo calcula la distancia verdadera con los candidatos.

```
      Q
      │
      ▼
  ┌────────┐   filtro   ┌────────────┐   refine  ┌──────────────┐
  │ INDEX  │──────────►│ candidatos  │──────────►│ mejores k    │
  └────────┘  (barato)  └────────────┘ (Dist real)└──────────────┘
```

Con múltiples filtros en cascada, se reduce progresivamente el número de candidatos antes de calcular la distancia real.

## 2. Lower Bounding Distance (LB) ⭐

**Propiedad clave**:
$$LB(x, y) \leq d_{\text{real}}(x, y)$$

Si `LB(Q, Ci) > best_so_far`, entonces `d_real(Q, Ci) > best_so_far` seguro → **puedo descartar Ci sin calcular d_real**.

Esto garantiza:
- **No hay falsos negativos** (el resultado sigue siendo correcto).
- Se calcula la distancia real muchas menos veces.

Si el filtro NO es cota inferior, pueden aparecer falsos negativos y la respuesta ya no está garantizada.

### Ejemplo — DTW (Dynamic Time Warping)

DTW compara series de tiempo alineándolas óptimamente:

$$DTW(Q, C) = \min_w \sum_{k=1}^{K} w_k$$

con recursión `M(i,j) = d(qi, cj) + min(M(i-1,j-1), M(i-1,j), M(i,j-1))`.

Complejidad: `O(N · D²)` — muy costoso.

**LB_Keogh** — cota inferior barata:

$$LB\_Keogh(Q, C) = \sum_{i=1}^{n} \begin{cases} (c_i - U_i)^2 & \text{si } c_i > U_i \\ (c_i - L_i)^2 & \text{si } c_i < L_i \\ 0 & \text{en otro caso} \end{cases}$$

donde `U, L` son la envolvente superior/inferior de Q. Complejidad: `O(N)`.

### Algoritmo — Sequential Scan sin LB (baseline, 1-NN)

```python
def Sequential_Scan(Q):
    best_so_far = infinity
    best_idx = None
    for i, Ci in enumerate(database):
        d = DTW(Ci, Q)                # CARO
        if d < best_so_far:
            best_so_far = d
            best_idx = i
    return best_idx
```

### Con Lower Bounding (1-NN)

```python
def Lower_Bounding_Sequential_Scan(Q):
    best_so_far = infinity
    best_idx = None
    for i, Ci in enumerate(database):
        lb = LB_Keogh(Ci, Q)          # BARATO
        if lb < best_so_far:
            d = DTW(Ci, Q)            # CARO — solo si LB no descarta
            if d < best_so_far:
                best_so_far = d
                best_idx = i
    return best_idx
```

## 3. KNN con Lower Bounding ⭐ (el que pidió el examen)

Extender de 1-NN a K-NN: mantener un **heap MAX de tamaño K** con los mejores candidatos vistos, y usar el **peor del top-K** como `best_so_far` para podar.

```python
import heapq

def Lower_Bounding_KNN(Q, K):
    # heap max simulado con negativos:  ( -dist_real, obj )
    heap = []                             # top-K candidatos

    for Ci in database:
        # Cota inferior barata
        lb = LB(Q, Ci)

        # PODA: si LB ya supera el peor del top-K, ni calculo la distancia real
        if len(heap) == K:
            worst_of_topk = -heap[0][0]   # heap[0] es el mayor negativo → menor real
            if lb >= worst_of_topk:
                continue                  # descartado

        # Cálculo real
        true_dist = Dist(Q, Ci)           # ej. DTW

        if len(heap) < K:
            heapq.heappush(heap, (-true_dist, Ci))
        else:
            worst_of_topk = -heap[0][0]
            if true_dist < worst_of_topk:
                heapq.heapreplace(heap, (-true_dist, Ci))

    # Ordenar el top-K por distancia ascendente
    return sorted([(-d, o) for d, o in heap])
```

**Claves del algoritmo**:
- Heap MAX (no min): permite acceder al **peor** del top-K en `O(1)` para usar como `best_so_far`.
- Si el heap no tiene K elementos, siempre agregar (no podar aún).
- Cuando el heap está lleno, el reemplazo es en `O(log K)`.

**Complejidad**:
- Peor caso: `O(N · (D_LB + D_real))`.
- Práctica: `O(N · D_LB + K · D_real)` cuando la LB es buena.

## 4. Descriptores Locales — SIFT

- **SIFT (Scale Invariant Feature Transform)**: detecta puntos clave (keypoints) invariantes a **escala y rotación**.
- Cada imagen genera un **número variable** de descriptores (uno por keypoint).
- Cada descriptor es un vector de dimensión **128**.

```python
sift = cv2.SIFT_create()
keypoints, descriptors = sift.detectAndCompute(image, None)
# descriptors.shape = (n_puntos, 128)
```

Imagen A → `{A1, A2, ..., An}` vectores.
Imagen B → `{B1, B2, ..., Bm}` vectores.

¿Cómo comparar `SIFT(A)` con `SIFT(B)`? No son un solo vector.

### Ventajas
- Eficientes al enfocarse en regiones específicas.
- **Robustos a oclusión** (basta que unos puntos hagan match).
- Flexibles.

### Desventajas
- Alta dimensionalidad (128).
- Sensibles a ruido.
- Escalables con dificultad → **búsqueda aproximada** (ANN).

## 5. Indexación con descriptores locales

### Un solo índice para todos los descriptores

```
Obj_i tiene descriptores {P1, P2, P3, P4, P5, P6}

Cada Pj se inserta como (IdImagen, Pj) en un índice multidimensional:

              INDEX
               │
       ┌───────┼───────┐
     R1        R2      R3         ← MBRs del índice
      │        │        │
   (obj_i,   (obj_i,   (obj_i,
    P1)      P3)       P6)  ...
```

### Búsqueda por descriptores locales — con votación

```
Query image → {Q1, Q2, Q3, Q4, Q5, Q6}

Para cada Qi hacer KNN sobre el índice:

  Q1  →  obj4, obj2, obj6            ← 3-NN
  Q2  →  obj1, obj4, obj2
  Q3  →  obj4, obj8, obj2
  Q4  →  obj2, obj4, obj6
  Q5  →  obj4, obj3, obj2
  Q6  →  obj1, obj8, obj4

Votación:
  obj4 aparece 6 veces  → 6 votos
  obj2 aparece 5 veces
  obj6 aparece 2 veces
  ...

Ranking final por votos.
```

**Búsqueda aproximada (ANN)** — no da el resultado exacto pero es escalable.

### Qué tipo de búsqueda soporta

**CBIR por objeto / sub-imagen**: encuentra imágenes que contienen el mismo objeto/región, aunque:
- Aparezca **parcialmente** (oclusión).
- A **distinta escala** o resolución.
- **Rotado**.

Fundamento: **basta con que un subconjunto de descriptores haga match**. SIFT es invariante a escala → encaja perfecto con "imágenes de distinta resolución" del enunciado del examen.

## 6. Bag of Visual Words (BoVW) ⭐

### Construcción

```
Colección de imágenes
   │
   ▼
[Extraer descriptores locales de todas las imágenes]  ← millones de vectores
   │
   ▼
[K-Means clustering]  ← K centroides = "vocabulario visual" (palabras visuales)
   │
   ▼
[Cuantización] para cada imagen:
   cada descriptor Pj se asigna a la palabra visual más cercana
   │
   ▼
[Histograma BoVW] por imagen: cuántas veces aparece cada palabra visual
   │
   ▼
[Pesos TF-IDF sobre el histograma]
```

### TF-IDF en imágenes ⭐ (salió en examen anterior)

$$p_i = \frac{tf_{i,p}}{|p|} \cdot \log \frac{N}{df_i}$$

o con log-normalización:
$$w_{i,p} = \log(1 + tf_{i,p}) \cdot \log \frac{N}{df_i}$$

**Qué representa cada término**:
- **TF (`tf_{i,p}`)**: frecuencia con la que la palabra visual `i` aparece en la imagen `p`. Refleja cuánto de esa característica visual tiene la imagen.
- **IDF (`log(N/df_i)`)**: rareza de la palabra visual `i` en toda la colección. Palabras visuales muy frecuentes (aparecen en todas las imágenes) son poco discriminantes.
- **`|p|`**: cantidad total de palabras visuales en la imagen (normalización por longitud).

### Búsqueda

- Extraer descriptores de la query → cuantizar → histograma BoVW.
- Aplicar TF-IDF.
- **Similitud coseno** entre el histograma de la query y los histogramas de la colección.
- Ranking por coseno decreciente.

## 7. Inverted File (IVF)

### Construcción

1. **Muestreo**: recopilar descriptores locales de todas las imágenes de entrenamiento.
2. **Clustering (K-Means)**: centroides → vocabulario visual.
3. **Asignación**: cada descriptor de cada imagen → palabra visual más cercana.
4. **Indexación**: índice invertido con las palabras visuales como claves y las **posting lists** conteniendo los IDs de las imágenes.

### Búsqueda

1. Extraer descriptores de la query y cuantizar.
2. **Filtrado**: recuperar solo las imágenes que comparten al menos una palabra visual.
3. **Similitud coseno** entre histogramas BoVW ponderados TF-IDF de la query vs candidatos.

## 8. Búsqueda Aproximada (ANN)

Sacrifica precisión por eficiencia — necesario en alta dimensión.

| | Búsqueda Lineal | **Búsqueda Aproximada** |
|---|---|---|
| Definición | Compara con todo | Encuentra resultados "suficientemente buenos" |
| Complejidad | O(N·D) | Sublineal |
| Exactitud | Exacta | Sacrifica precisión |
| Ideal para | Datasets pequeños | Grandes + alta dim |
| Ventaja | Precisión total | Maneja curse of dim |
| Desventaja | Muy lenta | Pequeña pérdida de precisión |

### Técnicas ANN

- **IVF Flat**: índice invertido con compresión plana. Rápido de construir, eficiente en memoria. Menos preciso que HNSW.
- **HNSW (Hierarchical Navigable Small World)**: grafo jerárquico. Lento de construir, más preciso y rápido en búsqueda. Más memoria.
- **LSH (Locality Sensitive Hashing)**: hashing que preserva vecindad.
- **PQ (Product Quantization)**: comprimir vectores para caber en RAM.

### Implementaciones

- **PostgreSQL** con `pgvector`: soporta IVF, HNSW.
- **Python**: `scikit-learn` (LSHForest), **`Faiss`** (IVF, HNSW, LSH, GPU).

### pgvector

```sql
CREATE EXTENSION vector;

CREATE TABLE documents (
  id SERIAL PRIMARY KEY,
  content TEXT,
  embedding vector(768)
);

-- Búsqueda por similitud
SELECT id, content
FROM documents
ORDER BY embedding <-> '[0.1, 0.2, ...]'::vector
LIMIT 10;

-- Índice IVFFLAT
CREATE INDEX ON documents
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- Índice HNSW
CREATE INDEX ON documents
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

Operadores: `<->` (L2), `<=>` (coseno), `<#>` (neg. dot product).

## 9. Indexación Multimodal ⭐ (teoría fuerte)

### Idea

Combinar múltiples tipos de datos (imagen + audio + texto) para búsquedas más ricas.

Ejemplo del examen anterior: **sistema de detección de copyright en TikTok**.

### Componentes de un video

```
Video ─┬─► Frames (imagen)     ──► Embedding CNN / descriptores locales
       ├─► Audio               ──► MFCC / embedding de audio (bag-of-audio-words)
       └─► Transcripción       ──► BoW / embedding BERT del texto
```

### Fusión — Early vs Late

**Early fusion**: concatenar los vectores de las 3 modalidades **antes** de indexar.

```
[emb_img | emb_audio | emb_texto]  →  ÚNICO ÍNDICE
```

- ❌ Distintas dimensiones y escalas → hay que normalizar.
- ❌ Si falta una modalidad, todo el vector queda mal.
- ❌ No escala si las modalidades tienen dinámicas muy distintas.

**Late fusion** (recomendada):

```
Query video
    │
    ├─► [emb_img]      ──► KNN sobre índice imagen   → ranking parcial R_img
    ├─► [emb_audio]    ──► KNN sobre índice audio    → ranking parcial R_aud
    └─► [emb_texto]    ──► KNN sobre índice texto    → ranking parcial R_txt

           ▼
   score_final = α · score_img + β · score_aud + γ · score_txt   (weighted sum)

           ▼
      Top-K final
```

- ✅ Cada modalidad se indexa con la estructura adecuada.
- ✅ Escalable: cada índice puede crecer/optimizarse independientemente.
- ✅ Robusta a modalidades faltantes (peso 0).

### Framework escalable

```
CONSTRUCCIÓN DEL ÍNDICE
────────────────────────
1. Para cada video de la colección:
     - Extraer N frames representativos.
     - Extraer audio → segmentar en ventanas → MFCC.
     - Extraer transcripción (ASR).
2. Calcular embeddings por modalidad.
3. Indexar cada modalidad en su índice ANN (HNSW o IVF).

RECUPERACIÓN
────────────
1. Del video query, extraer las 3 modalidades.
2. Calcular embeddings.
3. KNN en cada índice → rankings parciales.
4. Combinar scores (late fusion).
5. Top-K final.

Por qué escalable:
- ANN por modalidad (sublineal).
- Índices independientes (paralelizable).
- Late fusion no requiere reindexar si cambia el peso.
```

### Comparación teórica

| Aspecto | Early | **Late** |
|---|---|---|
| Cuándo se combina | Antes de indexar | Después de recuperar |
| Complejidad de indexación | Un índice grande | Varios índices pequeños |
| Robustez a modalidad faltante | Baja | **Alta** |
| Facilidad de ajustar pesos | Requiere reindexar | **Solo cambiar α, β, γ** |
| Escalabilidad | Baja | **Alta** |

## 10. Preguntas típicas (autotest)

1. **Modifica el algoritmo de 1-NN con LB para retornar K vecinos**. Explica la estructura de datos que usarías.
2. Explica qué representa TF y qué representa IDF cuando se calculan sobre un histograma **de palabras visuales** (BoVW).
3. Diseña un sistema de detección de copyright para TikTok. Explica el proceso de indexación y de consulta. ¿Por qué es escalable?
4. Compara early fusion vs late fusion multimodal. Ventajas y desventajas.
5. ¿Qué es Lower Bounding Distance? ¿Por qué garantiza que no haya falsos negativos?
6. ¿Cómo soporta la búsqueda por descriptores locales imágenes de distinta resolución del enunciado?
7. Explica la construcción de un índice IVF con K-Means paso a paso.
8. Diferencia entre HNSW y IVF Flat en términos de precisión, memoria y velocidad de construcción.
9. ¿Por qué la búsqueda por descriptores locales se llama "aproximada" (ANN)?
10. Diseña una tabla en PostgreSQL usando `pgvector` para almacenar embeddings de 512 dim con búsqueda por coseno.
