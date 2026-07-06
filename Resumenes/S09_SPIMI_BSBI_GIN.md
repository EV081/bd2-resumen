# S09 — SPIMI / BSBI / GIN ⭐⭐ FIJA (el profe lo dijo)

> **SPIMI es fija**. Estudia el algoritmo paso a paso, con gráfico.

---

## 1. Problema: ¿por qué necesitamos algoritmos escalables?

Un índice invertido **en memoria** no escala:
- Colecciones tienen billones de tokens.
- La RAM no alcanza.
- Hay que usar **disco** eficientemente: transferir por **bloques** (secuencial) es mucho más rápido que muchos accesos pequeños (random).

Restricciones:
- Memoria principal limitada.
- Memoria secundaria (disco) barata pero lenta.
- Costo `disk << CPU` de acceso.

---

## 2. BSBI — Blocked Sort-Based Indexing

### Idea

1. Recorrer los documentos y generar entradas `(termID, docID)` de 8 bytes cada una.
2. Acumular en un bloque hasta llenar la RAM disponible.
3. **Ordenar el bloque por termID** (in-memory sort).
4. Escribir el bloque ordenado a disco.
5. Al final, **merge multi-way** de todos los bloques.

### Gráfico

```
Bloque 1 (RAM)                Bloque 2 (RAM)                Bloque 3 (RAM)
────────────────              ────────────────              ────────────────
(W2, d1)                      (W2, d4)                      (W1, d7)
(W1, d1)         sort         (W4, d6)         sort         (W3, d6)
(W2, d3)   ────────►          (W5, d5)   ───►               (W1, d8)
(W1, d3)                      (W2, d5)                      (W6, d8)
(W3, d4)                      (W5, d4)                      (W6, d6)
   │                                │                              │
   ▼                                ▼                              ▼
DISCO b1                        DISCO b2                        DISCO b3
(W1, d1,d3)                     (W2, d4,d5)                     (W1, d7,d8)
(W2, d1,d3)                     (W4, d6)                        (W3, d6)
(W3, d4)                        (W5, d4,d5)                     (W6, d6,d8)
                                     │
                                     ▼
                              MERGE MULTI-WAY
                                     ▼
                        Índice final ordenado en disco
```

### Merge multi-way

Con `k` bloques, si haces merges binarios el árbol tiene `log₂(k)` niveles → lecturas repetidas. **Mejor: leer todos los bloques a la vez**.

- Abrir **todos** los bloques simultáneamente con un buffer de lectura para cada uno.
- **Cola de prioridad** (heap) para seleccionar el termID mínimo entre las cabeceras.
- Fusionar todas las posting lists del mismo termID y escribir al buffer de salida.

### Complejidad

- Cada bloque se escribe **una vez** a disco después de ordenar.
- Cada bloque se lee **una vez más** durante el merge.
- Total I/O = `O(T)` donde T = total de entradas, si el merge es multi-way.

### Problema pendiente de BSBI

**Asume que el diccionario `term → termID` cabe en RAM**. Con colecciones enormes esto tampoco es realista.

---

## 3. SPIMI — Single-Pass In-Memory Indexing ⭐

### Idea clave

**No mantener un mapeo `term ↔ termID` global**. Cada bloque tiene su **propio diccionario local** (hash), independiente de los demás. Trabaja directamente con `term` (string).

### Algoritmo

```
SPIMI-INVERT(token_stream):
    dictionary = new_hash()
    while memoria disponible AND token_stream tiene elementos:
        (term, docID) = next(token_stream)
        if term not in dictionary:
            posting_list = new_list()
            dictionary[term] = posting_list
        else:
            posting_list = dictionary[term]
        if posting_list está llena:
            posting_list = duplicar_tamaño(posting_list)
        posting_list.append(docID)
    ordenar términos del dictionary        ← ordena solo al final
    escribir bloque a disco
    return archivo_del_bloque
```

Bucle exterior: llamar `SPIMI-INVERT` sucesivamente sobre el flujo hasta procesar todo. Luego **merge de bloques** (igual que BSBI).

### Ventajas sobre BSBI

| | BSBI | **SPIMI** |
|---|---|---|
| Necesita mapping `term↔termID` global | ✅ (cuesta RAM) | ❌ (no lo mantiene) |
| Ordena entradas | Durante el bloque | Solo al final del bloque |
| Cabe más data por bloque | Menos (hay que reservar espacio) | **Más** (más términos + posting lists por bloque) |
| Merge más rápido | Comparable | **Más rápido** (bloques más grandes → menos bloques) |
| Guarda TF/posiciones fácilmente | Complicado | Fácil (agregas al posting_list directamente) |

### Gráfico SPIMI

```
Flujo de tokens: (t1, d1), (t2, d1), (t1, d3), (t3, d4), (t2, d3), ...

┌────────────────────────────────────────┐
│           BLOQUE i (RAM)               │
│                                        │
│   Local Dictionary (hash)              │
│   ┌────────────────────────────────┐   │
│   │ w1 → [d1, d3]                  │   │
│   │ w2 → [d1, d3, d5, ...]         │   │
│   │ w3 → [d4]                      │   │
│   └────────────────────────────────┘   │
│                                        │
│   Cada posting list arranca con        │
│   tamaño fijo; si se llena, se         │
│   DUPLICA (amortized O(1) append).     │
└────────────────────────────────────────┘
              │
              ▼ (RAM llena)
      Ordena keys del dict
              │
              ▼
      Escribe bloque a disco
              │
              ▼
     Nuevo bloque desde cero
```

### Merge final

Igual que BSBI: multi-way merge con cola de prioridad sobre los términos (que ahora son strings, no IDs).

---

## 4. Alternativas de estructura para el diccionario

### B+ Tree

```
              [w2, w4]
             /   |    \
       [w1]   [w3]   [w5, w6]
        │       │       │
        B1      B4      B7  ← Buckets con las posting lists
```

- **Ventaja**: soporta inserciones/eliminaciones eficientes → bueno cuando **se agregan documentos nuevos** al índice.

### Extendible Hashing

Similar pero con hashing dinámico. También apto para actualizaciones.

---

## 5. Implementación en PostgreSQL

### GIN (Generalized Inverted Index) ⭐

- Índice **invertido generalizado**.
- Ideal para tipos con muchos valores compuestos: full-text search, arrays, JSONB.

```sql
-- Crear un índice GIN sobre un campo de texto
CREATE INDEX content_idx ON News
USING GIN (to_tsvector('english', content));

-- Consulta con match
SELECT * FROM News
WHERE to_tsvector('english', content) @@ to_tsquery('english', 'keyword1 & keyword2');
```

- `to_tsvector`: convierte texto a vector de tokens preprocesados (stemming, stop words).
- `to_tsquery`: parsea la query con operadores lógicos.
- `@@`: operador de match.

### GiST (Generalized Search Tree)

- Índice basado en árbol equilibrado.
- Más flexible para **actualizaciones frecuentes**.

```sql
CREATE INDEX docs_gist_idx ON news
USING GIST (to_tsvector('spanish', content));
```

### GIN vs GiST ⭐

| | **GIN** | **GiST** |
|---|---|---|
| Estructura | Índice invertido | Árbol |
| Búsqueda | **Muy rápida** | Rápida |
| Construcción | **Lenta** | Rápida |
| Actualizaciones | **Costosas** | Baratas |
| Tamaño en disco | Grande | Menor |
| Uso ideal | **Motores de búsqueda**, gran volumen de lecturas | **CMS**, muchas escrituras |

**Regla mnemotécnica**: GIN es para **G**randes búsquedas + **IN**mutable; GiST es para **G**estión con **iS**crituras frecuen**T**es.

### Consulta que aproxima similitud coseno con GIN ⭐

Esta pregunta salió en el Final 2025-1:

```sql
CREATE INDEX idx_doc_fts ON documentos
USING GIN (to_tsvector('spanish', contenido));

SELECT d.id,
       ts_rank_cd(to_tsvector('spanish', d.contenido), q, 32) AS score
FROM   documentos d,
       to_tsquery('spanish', 'oro & plata & camion') AS q
WHERE  to_tsvector('spanish', d.contenido) @@ q      -- GIN filtra candidatos
ORDER  BY score DESC
LIMIT  10;
```

**Por qué aproxima coseno**:
- `@@` usa el GIN como índice invertido → trae solo documentos que comparten términos con la query (equivalente a recorrer posting lists).
- `ts_rank_cd(...)` pondera por TF y proximidad (efecto TF).
- El flag `32` de normalización divide por la **longitud del documento** → imita la normalización por `‖d‖` del coseno.

## 6. MongoDB Full-Text Search

```javascript
db.articulos.createIndex({ campoTexto: "text" })

db.articulos.find({
  $text: { $search: "big data análisis" }
})
```

Usa **BM25** (versión mejorada de TF-IDF) internamente. Ranking automático por relevancia.

## 7. Preguntas típicas (autotest) ⭐

1. **Explica el algoritmo SPIMI paso a paso**. ¿Cuál es su ventaja principal sobre BSBI?
2. Dibuja un gráfico del proceso de indexación con BSBI: bloques → merge multi-way.
3. ¿Por qué SPIMI no necesita mantener el mapeo `term → termID`?
4. ¿Qué estructura usarías para el diccionario global si la colección se actualiza constantemente con nuevos documentos: B+ Tree o Extendible Hashing? Justifica.
5. Compara GIN vs GiST. ¿Cuál elegirías para un CMS con miles de posts nuevos/día?
6. Escribe una consulta en PostgreSQL con GIN que aproxime similitud coseno.
7. ¿Por qué el merge multi-way es preferible al merge binario?
8. Explica el rol de la duplicación de tamaño en las posting lists de SPIMI (amortized cost).
