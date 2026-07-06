# S14 — Redis (BD No SQL 1) ⭐ FIJA

> **Anuncio explícito del profe**: características del modelo, arquitectura distribuida, casos de uso, patrón A-SIDE.

---

## 1. Modelo Clave-Valor en memoria

- **REDIS** = **RE**mote **DI**ctionary **S**erver.
- BD **in-memory** (RAM) → latencia sub-milisegundo.
- Almacén clave-valor **con estructuras** ricas: no solo `k→v`, sino colecciones especializadas.
- Open-source, escrito en C.

### Estructuras de datos

| Tipo | Comandos clave | Uso |
|---|---|---|
| **String** | `SET k v EX 60`, `GET k`, `INCR ctr` | Cache, contadores, flags |
| **List** (linked list) | `LPUSH q x`, `RPOP q`, `LRANGE q 0 9` | Colas FIFO/LIFO, jobs |
| **Hash** (map de campos) | `HSET user:1 name "Juan" edad 30`, `HGET user:1 name` | Objetos serializados por campos |
| **Set** (conjunto) | `SADD tags redis nosql`, `SINTER a b` | Tags, únicos, intersecciones |
| **Sorted Set** (ZSet) | `ZADD lb 1500 alice`, `ZREVRANGE lb 0 9 WITHSCORES` | **Leaderboards**, rankings por score |
| **Stream** | `XADD logs * k v`, `XREAD ...` | Event sourcing, Kafka-lite |
| **Pub/Sub** | `PUBLISH ch msg`, `SUBSCRIBE ch` | Mensajería real-time |
| **HyperLogLog** | `PFADD hll v`, `PFCOUNT hll` | Cardinalidad aproximada |
| **Bitmap** | `SETBIT users 1234 1` | Presencia, feature flags |
| **Geo** | `GEOADD locs -77 -12 lima`, `GEORADIUS ...` | Geolocalización |

---

## 2. Arquitectura distribuida

### Modo 1: Master-Replica (replicación simple)

```
     +-------+     replica     +--------+
     |Master |  ------------>  |Replica1|
     +-------+                 +--------+
        |    \                 +--------+
        |     ---replica---->  |Replica2|
        |                      +--------+
    (writes)              (reads escalados)
```

- **Master**: acepta reads y writes.
- **Replicas**: solo reads (por defecto).
- Replicación **asíncrona** → posible pérdida de writes recientes si el master cae.
- **Redis Sentinel**: monitoriza masters y hace **failover automático** promoviendo una replica.

### Modo 2: Redis Cluster (sharding automático)

```
┌───────────────────────── 16 384 hash slots ─────────────────────────┐
│                                                                     │
│  Nodo A          Nodo B            Nodo C                           │
│  slots 0-5460    slots 5461-10922  slots 10923-16383                │
│   │                │                 │                              │
│   └─replica A'     └─replica B'      └─replica C'                   │
└─────────────────────────────────────────────────────────────────────┘
```

- Espacio dividido en **16 384 hash slots** (no configurable).
- Cada clave se mapea a un slot: `slot = CRC16(key) mod 16384`.
- Cada nodo master posee un rango de slots + tiene replicas.
- Failover automático a nivel de cluster.
- **Mínimo recomendado**: 3 masters + 3 replicas.
- **Sharding transparente** al cliente (los clientes soportan protocolo cluster).

**Hash tags** para forzar co-localización:
```
SET {user:42}:profile ...    # solo "user:42" entra al hash
SET {user:42}:posts ...      # → mismo slot que el anterior
```

### Persistencia

Redis es in-memory pero **soporta persistencia opcional**:

| Método | Cómo | Ventajas | Desventajas |
|---|---|---|---|
| **RDB** (snapshot) | `BGSAVE` cada cierto tiempo → `dump.rdb` | Compacto, recuperación rápida | Puede perder datos entre snapshots |
| **AOF** (append-only file) | Log de cada operación (`appendonly yes`) | Máxima durabilidad | Archivo grande, más I/O |
| **RDB + AOF** | Ambos combinados | Balance | Más disco |

Configuración típica: `save 900 1` (RDB cada 15 min si hay ≥1 cambio) + `appendfsync everysec` (AOF flush cada segundo).

---

## 3. Patrón Cache-Aside (A-SIDE) ⭐

Es el patrón más común de cache. **Casi seguro cae en preguntas cortas**.

### Idea

La aplicación **maneja explícitamente** el cache:
- En **lectura**: primero busca en cache, si no, consulta la BD y guarda en cache.
- En **escritura**: actualiza la BD e **invalida** (o actualiza) la entrada en cache.

### Diagrama

```
       ┌──────────────┐
       │ Application  │
       └───┬──────┬───┘
     read  │      │  write
           ▼      ▼
   ┌────────┐    ┌─────────┐
   │ Redis  │    │  BD SQL │
   │(cache) │    │(fuente  │
   └────┬───┘    │ verdad) │
        │        └─────────┘
        │        ▲
      miss       │ SELECT ...
        │        │
        └────────┘
```

### Pseudocódigo lectura

```python
def get_user(uid):
    k = f"cache:user:{uid}"
    v = redis.get(k)
    if v is not None:
        return json.loads(v)                     # HIT
    # MISS
    u = db.query("SELECT * FROM users WHERE id=%s", uid)
    redis.setex(k, 3600, json.dumps(u))          # repopular con TTL
    return u
```

### Pseudocódigo escritura

```python
def update_user(uid, new_data):
    db.execute("UPDATE users SET ... WHERE id=%s", uid)
    redis.delete(f"cache:user:{uid}")            # invalidar (más común)
    # o: redis.setex(k, 3600, json.dumps(new_data))  # write-through desde app
```

### Cuándo justificar A-SIDE

- Fuente de verdad = BD relacional (autoridad).
- Lecturas **mucho más frecuentes** que escrituras.
- Es tolerable un poco de staleness dentro del TTL.
- Cache falla → el sistema sigue (degradado pero funcional).

### Ejemplos de escenarios (útil para pregunta "propon uno distinto al de clase")

1. **Reserva de tickets/asientos** en cine: catálogo de funciones y precios en Redis con TTL de 5 min; cada compra invalida el cache del asiento específico.
2. **Sesiones de usuario** en apps web (`session:<token>`).
3. **Rate limiting** por IP: `INCR ratelimit:<ip>` con `EXPIRE 60` — bloquear si supera N/min.
4. **Catálogo de productos e-commerce**: precios y stock frecuentemente leídos, actualizados por administradores.
5. **Feed de noticias personalizadas** con TTL corto.
6. **Leaderboards en juegos**: `ZADD` sobre sorted set, `ZREVRANGE` para top-10 al instante.

### Otros patrones

| Patrón | Flujo | Cuándo |
|---|---|---|
| **Read-through** | Cache consulta BD por ti (transparente) | Menos control, más simple |
| **Write-through** | Write: app → cache → BD (sincrónico) | Consistencia fuerte, más lento |
| **Write-behind** | Write: app → cache (async → BD) | Muy rápido, riesgo si cache falla |

---

## 4. Redis vs Memcached (por si aparece)

| | Redis | Memcached |
|---|---|---|
| Estructuras | Ricas (list, hash, set, zset...) | Solo strings |
| Persistencia | RDB + AOF | Ninguna |
| Pub/Sub | Sí | No |
| Cluster | Redis Cluster (16384 slots) | Solo client-side sharding |
| Réplicas | Master-replica | No nativo |

---

## 5. CAP y consistency

- Redis en modo master-replica: **CP** (si hay partición, se prioriza consistencia — el nodo aislado no sirve como master).
- Redis en modo cluster con replication async: **AP** en la práctica (writes al master aceptados incluso si replicas caídas).

---

## 6. Preguntas típicas (autotest)

1. Menciona 3 estructuras de datos de Redis y un caso de uso para cada una.
2. ¿Cómo funciona el sharding en Redis Cluster? ¿Cuántos slots hay y por qué ese número?
3. Explica el patrón Cache-Aside con un diagrama y pseudocódigo. ¿Por qué no basta con "cachear todo"?
4. Diferencia entre RDB y AOF. ¿Cuál elegirías para un sistema bancario?
5. Diseña con Redis un rate limiter de 100 peticiones/min por IP. Escribe los comandos.
6. Propón un caso de uso de Redis distinto al de clase y justifícalo con A-SIDE.
7. ¿Qué pasa si el master de un Redis Cluster cae? ¿Y en modo master-replica sin Sentinel?
8. ¿Por qué usar `EXPIRE`/`SETEX` en las claves de cache?
