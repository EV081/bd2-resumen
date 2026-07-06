# S08 — Recuperación de Información Textual (TF-IDF, coseno) 🟠 TEORÍA FUERTE

> El profe avisó: TF-IDF y coseno pueden venir en **preguntas teóricas**, no solo prácticas.

---

## 1. ¿Por qué no basta con `LIKE`?

```sql
SELECT * FROM News WHERE content ILIKE '%X%' AND content ILIKE '%Y%';
```

Problemas:
1. **No considera relación léxica** entre palabras (plural/singular, stemming, sinónimos).
2. **Sin ranking** — devuelve todo o nada, el usuario se ahoga.
3. **O(N·n)** — muy lento para grandes colecciones. Índices Hash o B+ Tree no ayudan sobre subcadenas.

## 2. Information Retrieval (IR)

- Ciencia de buscar información **no estructurada** que satisfaga una necesidad, en grandes colecciones.
- Diferente de BD relacional: acierto **aproximado** (no exacto), modelo **heurístico/probabilístico**, consulta en **lenguaje natural**, especificación **imprecisa**, insensible a errores en la respuesta.
- Input: colección + query de texto → Output: **ranked list** de documentos relevantes.

## 3. Bag of Words (BoW)

Transforma un texto en un vector numérico. Preprocesamiento:

1. **Tokenización**: cortar en palabras.
2. **Normalización**: `ONU = O.N.U.`, minúsculas.
3. **Stemming**: `amigas, amigos → amigo`.
4. **Stop words**: eliminar palabras muy comunes (`de, el, los`).
5. **Vectorización**: cada palabra → dimensión. Peso: incidencia (0/1), conteo o TF-IDF.

### Selección de keywords

Palabras de **frecuencia media** son las más informativas. Frecuencias extremas (muy raras o stopwords) aportan poco.

## 4. Modelo Booleano

Matriz de incidencia término × documento:
```
              d1  d2  d3
  antony      1   1   0
  brutus      1   1   1
```
Query = `brutus AND antony AND NOT calpurnia` → operaciones bit a bit sobre las filas.

**Problemas**:
- Requiere usuario experto.
- Resultado binario: 0 o miles de resultados sin ranking ("festival o hambruna").

## 5. Ranked Retrieval — asignar un `score(q, d) ∈ [0, 1]`

Idea: en vez de match booleano, ordenar los documentos por relevancia.

### Term Frequency (TF)

- `tf_{t,d}` = # de veces que el término `t` aparece en el documento `d`.
- **La relevancia NO crece linealmente** con `tf` — un documento con 10 apariciones no es 10× más relevante que uno con 1.
- Se **amortigua con log**:

$$w_{t,d} = \begin{cases} 1 + \log_{10}(tf_{t,d}) & \text{si } tf > 0 \\ 0 & \text{si no} \end{cases}$$

- `0 → 0`, `1 → 1`, `2 → 1.3`, `10 → 2`, `1000 → 4`.

**Qué representa el TF**: la **importancia local** del término dentro del documento. Un término repetido varias veces es más "sobre" ese tema.

### Document Frequency (DF) e IDF

- `df_t` = # de documentos que contienen el término `t`.
- Términos **raros** son más informativos que los frecuentes:
  - `"the"` aparece en todos los docs → aporta 0.
  - `"aracnocéntrico"` aparece en 1 doc → si sale en query, ese doc es casi seguro relevante.

$$idf_t = \log_{10}\left(\frac{N}{df_t}\right)$$

- N = tamaño de la colección.
- Se usa `log` para amortiguar diferencias enormes: con `N=10^6`, un término que aparece en 1 doc tiene idf=6; uno que aparece en `10^6` tiene idf=0.

**Qué representa el IDF**: la **rareza global** / distintividad del término. Es una medida **inversa de informatividad** de `df`.

### TF-IDF

$$w_{t,d} = (1 + \log_{10} tf_{t,d}) \cdot \log_{10}\left(\frac{N}{df_t}\right)$$

- Aumenta con la frecuencia local (t importante en d).
- Aumenta con la rareza global (t es distintivo).

## 6. Similitud Coseno ⭐ (teoría)

Cada documento y la query son vectores en el espacio de términos.

### ¿Por qué NO usar distancia euclidiana?

**Experimento mental**: toma un documento `d` y duplícalo → `d' = d ⊕ d`. Semánticamente son iguales.
- Distancia euclidiana `‖d - d'‖` es **grande** (magnitudes distintas).
- Ángulo entre `d` y `d'` es **0** (misma dirección).

La distancia euclidiana penaliza documentos largos aunque tengan la **misma distribución** de términos. Mal.

### Solución: usar el ángulo

$$\cos(q, d) = \frac{q \cdot d}{\|q\| \|d\|} = \sum_i \frac{q_i \cdot d_i}{\|q\| \|d\|}$$

- Normaliza por longitud → documentos largos y cortos comparables.
- El ranking por coseno decreciente = ranking por ángulo creciente.

### Efecto de la normalización

$$\hat{x} = \frac{x}{\|x\|_2}, \quad \|x\|_2 = \sqrt{\sum_i x_i^2}$$

Después de normalizar, el producto punto = coseno.

### Ejemplo (novelas de Austen y Brontë)

| Término | SS | OP | CB |
|---|---|---|---|
| afecto | 0.789 | 0.832 | 0.524 |
| celoso | 0.515 | 0.555 | 0.465 |
| chisme | 0.335 | 0 | 0.405 |
| borrascoso | 0 | 0 | 0.588 |

- `cos(SS, OP) ≈ 0.94` → muy similares (ambas de Austen).
- `cos(SS, CB) ≈ 0.79`.
- `cos(OP, CB) ≈ 0.69` → distintos autores, menos similares.

## 7. Índice invertido

Reemplaza matriz densa por listas de posting:

```
Term       Posting List (docID: tf)
─────────────────────────────────────
casa       [1:1, 2:1, 3:2, 4:1]
grande     [1:1, 3:2]
gato       [2:1]
```

- Cada término → lista de documentos donde aparece, con tf.
- Ordenada por docID → merge lineal.

### Consulta AND — merge lineal

```
p1 → Bruto:  [2, 4, 8, 16, 32, 64, 128]
p2 → Cesar:  [1, 2, 3, 5, 8, 13, 21, 34]

recorrer ambos avanzando el menor:
  Bruto=2, Cesar=1 → avanzar Cesar
  Bruto=2, Cesar=2 → MATCH, ambos avanzan
  Bruto=4, Cesar=3 → avanzar Cesar
  ...

resultado: [2, 8]     O(n + m)
```

### Consulta OR

Union con merge similar, añade ambos si son distintos.

### Consulta NOT (t1 AND NOT t2)

Recorrer p1 y saltarse los docs que aparezcan en p2. También `O(n+m)`.

## 8. Precision & Recall

$$\text{Precision} = \frac{TP}{TP + FP}, \quad \text{Recall} = \frac{TP}{TP + FN}$$

- **Precision**: de los recuperados, ¿cuántos son relevantes?
- **Recall**: de los relevantes que existen, ¿cuántos recuperé?
- Trade-off: bajar el umbral → +recall, -precision.

Colecciones de referencia: TREC, MIR Flickr, CoPhIR.

## 9. Ejemplo TF-IDF numérico

Doc: "seguro de carro, seguro de auto"
Query: "el mejor seguro de carro"

Con N desconocido pero grande, df de `seguro`=1000, `carro`=10000, `mejor`=50000, `auto`=5000.

- `idf(seguro) = log(N/1000) = 3.0`
- `idf(carro) = 2.0`
- `idf(mejor) = 1.3`
- `idf(auto) = 2.3`

Score(q,d) = producto punto normalizado. En ejemplo del profe:

$$\text{Score} = 0 + 0 + 0.27 + 0.53 = 0.8$$

## 10. Preguntas típicas (autotest)

1. ¿Por qué se usa similitud coseno y no distancia euclidiana en IR? Da un ejemplo.
2. Explica qué representa el TF y qué representa el IDF. ¿Por qué log?
3. Si un término tiene `idf = 0`, ¿qué significa?
4. Diferencia entre búsqueda booleana y ranked retrieval. Dilema "festival o hambruna".
5. Recorre el algoritmo de merge lineal para `AND` sobre dos posting lists.
6. Precision vs Recall — ¿en qué se sacrifica cada uno?
7. ¿Por qué el índice invertido es preferible a la matriz densa? Cálculo de espacio.
