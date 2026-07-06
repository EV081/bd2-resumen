# S10 — BD Vectoriales, Descriptores, KNN 🟠

---

## 1. BD Vectorial vs BD Relacional

Datos **no estructurados** (imágenes, audio, video, texto libre) → se extraen **vectores de características** (embeddings) que capturan el contenido.

```sql
-- Ejemplo conceptual
SELECT id, contenido
FROM articulos
WHERE MATCH(contenido_vector, "big data análisis");

SELECT id, rostro_path
FROM fotos_empleados
WHERE MATCH(rostro_vector, ExtractVector('/path/foto.jpg'));
```

Áreas: CBIR, digital libraries, video-on-demand, música, telemedicina, biometría, ChatGPT, Google.

## 2. BD Multimedia (MMDB)

Colección de datos con múltiples tipos: texto, imagen, audio, video.

### Arquitecturas

**Principio de autonomía**: cada tipo de medio con su propia estructura especializada. Rápida procesar consultas dentro de un medio, difícil hacer joins entre tipos.

**Principio de uniformidad**: una única abstracción para todos los medios (metadatos comunes). Más simple, pero es un **gran desafío** capturar todo con una sola estructura.

## 3. Recuperación basada en contenido (CBIR)

Busca por el **contenido de la imagen misma** (píxeles, colores, formas), no por metadatos (etiquetas manuales).

- Colores: histogramas, correlogramas.
- Descriptores locales: SIFT, SURF, ORB.
- Texturas: madera, piedra.
- Formas: contornos.

## 4. Vector de características / Embedding

- **Vector de características (descriptor)**: representación compacta del contenido calculada por reglas (histograma, gradientes).
- **Embedding**: representación **aprendida** por deep learning que captura semántica. Similitud geométrica ≈ similitud semántica.

## 5. Búsqueda por similitud

### Modelo

- Extracción de características → descriptor (vector).
- Función de similitud entre descriptores.
- La función debe **imitar** la similitud semántica humana.

$$f_s(P_1, P_3) > f_s(P_1, P_2) \iff P_1 \text{ es más similar a } P_3 \text{ que a } P_2$$

### Similitud vs distancia

Función de **distancia** mide **disimilitud**:
- A mayor distancia → más disímiles.
- `d(P, P) = 0`.

Se pueden convertir entre sí con cualquier función monótona decreciente:
$$d = 1 - s, \quad s = \frac{1}{1+d}, \dots$$

### Propiedades de una **métrica**

1. No-negativa: `d(x,y) ≥ 0`.
2. Idéntica: `d(x,x) = 0`.
3. Simétrica: `d(x,y) = d(y,x)`.
4. Desigualdad triangular: `d(x,z) ≤ d(x,y) + d(y,z)`.

## 6. Medidas de distancia

### Minkowski (parametrizada por p)

$$d_p(a, b) = \left( \sum_i |a_i - b_i|^p \right)^{1/p}$$

- **p=1**: Manhattan (`|a1-b1| + |a2-b2| + ...`).
- **p=2**: Euclidiana (`sqrt((a1-b1)² + ...)`).
- **p=∞**: Máximo (`max_i |a_i - b_i|`).
- Complejidad: `O(D)`.

### Similitud Coseno (angular)

$$\cos(\theta) = \frac{a \cdot b}{\|a\| \|b\|}$$

Buena para vectores donde importa la **dirección**, no la magnitud.

### Producto Punto

$$\text{dot}(a, b) = \sum_i a_i \cdot b_i$$

### Forma cuadrática

$$d_A(a, b) = \sqrt{(a-b)^T A (a-b)}$$

- `A`: matriz de similitud entre dimensiones.
- Ejemplos:
  - **Euclidiana** = A = identidad.
  - **Mahalanobis** = A = inversa de matriz de covarianza.
  - **SQFD** (Signature Quadratic Form Distance): permite comparar vectores de **distinta dimensión**.
- Complejidad: `O(D²)`.

### Cuándo cada una

- Histogramas de color con "rojo se parece más a rosado que a azul" → **forma cuadrática** (la euclidiana los ve igual de distintos).
- Embeddings normalizados → **coseno**.
- Series temporales alineables → **DTW** (Dynamic Time Warping).

## 7. Tipos de consulta

### Búsqueda por rango

$$R(q, r) = \{ o \in \mathbb{U} : d(q, o) \leq r \}$$

Devuelve entre 0 y **toda** la colección. Problema: ¿qué radio elegir?

### k-NN (K vecinos más cercanos)

Devuelve **exactamente K** elementos con menor distancia a q. Siempre retorna algo aunque estén lejos.

### Ranking incremental (give-me-more)

Cuando no sabes ni radio ni K. Se llama `getNext(k_i)` para traer los siguientes vecinos hasta que el usuario se satisfaga.

### Búsqueda secuencial (baseline)

```python
def RangeSearch(Q, r):
    result = []
    for Ci in collection:
        if Dist(Q, Ci) < r:
            result.append(Ci)
    return result
# O(N · D^p)

def KnnSearch(Q, k):
    result = []
    for Ci in collection:
        result.append((Ci, Dist(Q, Ci)))
    result.sort(key=lambda x: x[1])
    return result[:k]
# O(N · D^p + N log N)
```

Con `N` grande, esto no escala → necesitamos **índices**.

## 8. Eficiencia vs Efectividad

- **Eficiencia**: costo de búsqueda (tiempo de CPU + I/O).
- **Efectividad**: calidad de la respuesta (Precision, Recall).
  - Descriptores de mayor resolución no siempre implican mejor efectividad.
  - Trade-off: dimensionalidad alta → más semántica pero **maldición de dimensionalidad**.

### Precision & Recall

$$P = \frac{TP}{TP + FP}, \quad R = \frac{TP}{TP + FN}$$

Curva P-R típica: relación empírica inversa.

## 9. Índices multidimensionales

Los índices particionan el espacio para acelerar KNN/range.

- **R-Tree**: rectángulos jerárquicos (MBR).
- **R*-Tree**: R-Tree optimizado (minimiza solapamiento). Baja dimensión.
- **KD-Tree**: árbol binario que particiona por dimensión alternada. Baja/media dimensión.
- **Ball Tree**: hiperesferas. Mejor en alta dimensión.
- Todos son **dinámicos**: inserción/borrado en `O(log n)`.

## 10. Maldición de la dimensionalidad

En alta dimensión (~100+), **las distancias entre puntos aleatorios se concentran** — todos parecen igual de lejos. Los índices espaciales pierden efectividad.

Estrategias:
- Reducción de dimensionalidad (PCA, autoencoders).
- Búsqueda **aproximada** (ANN — próxima semana).
- Cuantización (PQ, IVF).

## 11. Ejemplos de aplicación

### Reconocimiento facial
- Detectar rostros → extraer embedding → búsqueda KNN por similitud.
- Librerías: `face-api.js`, `face-recognition`.

### Segmentación automática de videos
- Capturar audio + video + slides.
- Extraer texto de slides.
- Segmentar por cambio de slide.
- Índice textual + índice espacio-temporal.
- Búsqueda por palabra clave → salta al segmento.

### Búsqueda de audio musical
- Extraer MFCC (coeficientes cepstrales en frecuencias de Mel).
- Cuantizar → histograma de audio.
- Nota: no conserva el orden temporal.

### Series temporales
- Datos de video como series de gestos.
- Distancia: **DTW**.

## 12. Preguntas típicas

1. ¿Por qué la distancia euclidiana no siempre es una buena medida para histogramas de color?
2. Explica el trade-off entre eficiencia y efectividad.
3. ¿Qué es la maldición de la dimensionalidad y cómo afecta a los índices multidimensionales?
4. Define las 4 propiedades de una métrica.
5. Diferencias entre búsqueda por rango, k-NN y ranking incremental.
6. Con `N=10⁶` documentos y `D=128`, ¿cuál es la complejidad de un k-NN secuencial?
7. Explica qué es un descriptor vs un embedding.
