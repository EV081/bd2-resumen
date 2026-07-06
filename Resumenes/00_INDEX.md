# Índice — Estudio Final BD2 (Prof. Heider Sanchez)

Guía maestra construida a partir de las 10 PPTs del curso + los 3 exámenes pasados.

## Semana por semana

| Archivo | Tema | Prioridad |
|---|---|---|
| [S08_Recuperacion_Textual.md](S08_Recuperacion_Textual.md) | Bag of Words, TF-IDF, similitud coseno, índice invertido básico | 🟠 Alto (teoría fuerte) |
| [S09_SPIMI_BSBI_GIN.md](S09_SPIMI_BSBI_GIN.md) | BSBI, **SPIMI**, GIN vs GiST en PostgreSQL | 🔴 **FIJA** (anuncio del profe) |
| [S10_BD_Vectoriales_KNN.md](S10_BD_Vectoriales_KNN.md) | Descriptores, distancias, búsqueda por similitud, KNN, precision/recall | 🟠 Alto |
| [S11_LowerBounding_BoVW_Multimodal.md](S11_LowerBounding_BoVW_Multimodal.md) | LB, filtrar-y-refinar, descriptores locales, BoVW, **multimodal** (teoría fuerte), ANN | 🔴 **FIJA** (multimodal + LB) |
| [S12_Fragmentacion_Horizontal.md](S12_Fragmentacion_Horizontal.md) | Round-robin/Hash/Range/List, MinTerm, derivada, gráficos ASCII | 🔴 **FIJA** (siempre cae) |
| [S13_Fragmentacion_Vertical.md](S13_Fragmentacion_Vertical.md) | Dependencias funcionales, matriz de afinidad, BEA, particionamiento | 🟡 Medio |
| [S13_Consultas_Distribuidas.md](S13_Consultas_Distribuidas.md) | Descomposición, localización, optimización, **pseudocódigo estilo profe** | 🔴 **FIJA** (siempre cae en problema grande) |
| [S14_MongoDB.md](S14_MongoDB.md) | Modelo documento, índices compuestos (left-prefix), **interpretar aggregations** | 🔴 **FIJA** (interpretar aggregations) |
| [S14_Redis.md](S14_Redis.md) | Estructuras, cluster (16384 slots), persistencia, **A-SIDE** | 🔴 **FIJA** (anuncio del profe) |
| [S15_Cassandra.md](S15_Cassandra.md) | Wide-column, P2P, **partition + clustering key**, CQL, consistency levels | 🔴 **FIJA** (anuncio del profe) |
| [99_Preguntas_Examen.md](99_Preguntas_Examen.md) | Banco de preguntas por tema | Práctica |
| [99_Solucionario.md](99_Solucionario.md) | Respuestas modelo del banco | Autoevaluación |

## Anuncios explícitos del profe

1. **Cassandra** entra fija — modelo de datos, arquitectura distribuida, cómo se construye una tabla, composición de la PK, consultas admisibles, casos de aplicación.
2. **Redis** entra fija — características del modelo, arquitectura distribuida, casos de uso, patrón A-SIDE.
3. **SPIMI** entra fija — construcción escalable de índice invertido.
4. **Fragmentación** entra con **gráficos** de cómo funciona.
5. **Consultas MongoDB** — interpretar aggregations, describir qué hace la consulta.
6. **Consultas distribuidas** con pseudocódigo estilo `create temp / merge / select`.
7. **Recuperación textual** (TF-IDF, coseno) — pueden venir preguntas teóricas.
8. **Multimodal** — flujo teórico.

## Patrones históricos de examen

| Bloque | Peso típico | Formato |
|---|---|---|
| Preguntas cortas (5) | 5 pts | 1 pt cada una — cubren temas de NoSQL, backup, Bottom-Up, MongoDB, GIN |
| BD Multimedia | 3–7 pts | Pregunta conceptual + modificar algoritmo (KNN con LB) o diseñar (multimodal) |
| Fragmentación + consulta distribuida | 5–8 pts | Progresivo: (1) fragmentar, (2) algoritmo optimizado con gráfico, (3) SQL |
| Fragmentación vertical | 2 pts | Matriz numérica → BEA → R1, R2 |

## Cómo usar esta carpeta

1. **Lee un MD por sesión** en orden de prioridad (los 🔴 primero).
2. Cada MD termina con una sección "Preguntas típicas" — respóndelas mentalmente.
3. Al terminar los MDs, resuelve `99_Preguntas_Examen.md` **sin ver** el solucionario.
4. Compara contra `99_Solucionario.md` y anota los huecos.

## Errores frecuentes que hay que evitar

- **Cassandra**: WHERE sin partition key es error, no una consulta pobre.
- **Redis A-SIDE**: siempre invalida cache en el write; siempre TTL en el read.
- **Fragmentación distribuida**: si el atributo de JOIN ≠ atributo de fragmentación → hay que **reparticionar** (no funciona local mágicamente).
- **ORDER BY distribuido**: para evitar sort central, reparticionar por **RANGE** del atributo de orden, no por hash.
- **Semi-join derivado**: el atributo de join debe ser LLAVE en la relación dueña (si no, se rompe la desarticulación).
- **KNN con LB**: se usa heap MAX de tamaño K y `LB > peor_del_top_K` para podar.
- **TF-IDF en imágenes**: es sobre **palabras visuales** (centroides de K-means), no sobre píxeles.
- **Similitud coseno vs euclidiana**: la euclidiana penaliza documentos largos aunque sean semánticamente idénticos.
