# Análisis de sesiones y bloqueos

## 1. Metadatos

| Campo            | Detalle                                      |
|------------------|----------------------------------------------|
| **Duración**     | 40 minutos                                   |
| **Complejidad**  | Media                                        |
| **Nivel Bloom**  | Aplicar (Apply)                              |
| **Módulo**       | 2 — Concurrencia y Bloqueos en Aurora PostgreSQL |
| **Lab ID**       | 02-00-01                                     |

---

## 2. Descripción General

En este laboratorio aplicarás las técnicas de diagnóstico de bloqueos y concurrencia estudiadas en la lección 2.1 sobre un clúster Aurora PostgreSQL real. Crearás escenarios controlados de bloqueo con dos sesiones `psql` simultáneas, consultarás las vistas del sistema `pg_locks`, `pg_stat_activity` y `pg_blocking_pids()` para identificar la cadena de bloqueos, y configurarás parámetros de timeout en el Aurora Cluster Parameter Group para mitigar esperas prolongadas. Finalmente, revisarás el comportamiento del autovacuum y correlacionarás métricas de bloqueo en CloudWatch con los eventos observados en la base de datos.

---

## 3. Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Consultar `pg_locks`, `pg_stat_activity` y `pg_blocking_pids()` para identificar qué sesiones están bloqueando a otras en Aurora PostgreSQL.
- [ ] Crear y observar escenarios controlados de bloqueo de fila (`RowExclusiveLock`) usando `SELECT FOR UPDATE`, `NOWAIT` y `SKIP LOCKED`.
- [ ] Configurar `lock_timeout` y `statement_timeout` en un Aurora Cluster Parameter Group y verificar su efecto en sesiones bloqueadas.
- [ ] Analizar el impacto del autovacuum en la concurrencia usando `pg_stat_user_tables` y detectar tablas con alto conteo de dead tuples.
- [ ] Correlacionar métricas de bloqueo en CloudWatch (`BlockedTransactions`, `DatabaseConnections`) con eventos observados directamente en la base de datos.

---

## 4. Prerrequisitos

### Conocimiento previo

- Haber completado el Laboratorio 01-00-01 o contar con una instancia Aurora PostgreSQL con datos de prueba cargados.
- Comprensión básica de transacciones SQL: `BEGIN`, `COMMIT`, `ROLLBACK`.
- Familiaridad conceptual con bloqueos optimistas y pesimistas.
- Conocimiento básico de AWS CLI y CloudWatch.

### Acceso y recursos

- Clúster Aurora PostgreSQL activo (instancia escritora accesible vía `psql`).
- AWS CLI 2.x configurado con permisos para describir y modificar parameter groups de RDS.
- Dos terminales independientes (o dos pestañas de terminal) para simular sesiones concurrentes.
- Permisos IAM: `rds:DescribeDBClusterParameterGroups`, `rds:ModifyDBClusterParameterGroup`, `cloudwatch:GetMetricData`.

---

## 5. Entorno de Laboratorio

### Hardware recomendado

| Recurso               | Mínimo requerido                          |
|-----------------------|-------------------------------------------|
| RAM local             | 8 GB                                      |
| Almacenamiento libre  | 500 MB (scripts y logs)                   |
| Núcleos CPU           | 4 (para terminales concurrentes)          |
| Conexión a Internet   | 10 Mbps estable                           |

### Software requerido

| Herramienta           | Versión mínima     | Uso en este lab                         |
|-----------------------|--------------------|-----------------------------------------|
| `psql`                | 14.x               | Sesiones concurrentes de base de datos  |
| AWS CLI               | 2.x                | Modificar parameter groups              |
| `jq`                  | 1.6                | Parsear respuestas JSON de AWS CLI      |
| Navegador web         | Última versión     | Consola AWS / CloudWatch                |

### Variables de entorno — configuración inicial

Antes de iniciar los pasos, abre una terminal y define las siguientes variables. Sustitúyelas con los valores reales de tu entorno:

```bash
# ── Ajusta estos valores según tu entorno ──────────────────────────────────
export PGHOST="aurora-lab-cluster.cluster-xxxxxxxxxxxx.us-east-1.rds.amazonaws.com"
export PGPORT="5432"
export PGUSER="labuser"
export PGPASSWORD="LabPassword123!"   # En producción usa AWS Secrets Manager
export PGDATABASE="labdb"
export CLUSTER_ID="aurora-lab-cluster"
export PARAM_GROUP="aurora-lab-pg15-params"
export AWS_REGION="us-east-1"

# Verificar conectividad
psql -c "SELECT version();"
```

**Salida esperada:**

```
                                                 version
---------------------------------------------------------------------------------------------------------
 PostgreSQL 15.x (Aurora PostgreSQL) on aarch64-unknown-linux-gnu, compiled by gcc ...
(1 row)
```

---

## 6. Instrucciones Paso a Paso

---

### Paso 1 — Preparar el esquema y datos de prueba

**Objetivo:** Crear las tablas `pedidos` y `trabajos` que se usarán en los escenarios de bloqueo, y poblarlas con datos suficientes para observar contención realista.

#### Instrucciones

**1.1.** Conéctate a la base de datos y crea el esquema de prueba:

```sql
-- Ejecutar en la Sesión A (Terminal 1)
\c labdb

-- Crear tabla de pedidos
CREATE TABLE IF NOT EXISTS pedidos (
    id          SERIAL PRIMARY KEY,
    cliente_id  INTEGER NOT NULL,
    estado      VARCHAR(20) NOT NULL DEFAULT 'NUEVO',
    monto       NUMERIC(10,2),
    creado_en   TIMESTAMPTZ DEFAULT now()
);

-- Crear tabla de trabajos (para patrón SKIP LOCKED)
CREATE TABLE IF NOT EXISTS trabajos (
    id          SERIAL PRIMARY KEY,
    estado      VARCHAR(20) NOT NULL DEFAULT 'PENDIENTE',
    payload     TEXT,
    creado_en   TIMESTAMPTZ DEFAULT now()
);

-- Crear índice para acelerar búsquedas por estado
CREATE INDEX IF NOT EXISTS idx_pedidos_estado  ON pedidos(estado);
CREATE INDEX IF NOT EXISTS idx_trabajos_estado ON trabajos(estado);
```

**1.2.** Insertar datos de prueba:

```sql
-- Insertar 1000 pedidos
INSERT INTO pedidos (cliente_id, estado, monto)
SELECT
    (random() * 500 + 1)::int,
    (ARRAY['NUEVO','EN_PROCESO','COMPLETADO'])[ceil(random()*3)::int],
    round((random() * 1000 + 10)::numeric, 2)
FROM generate_series(1, 1000);

-- Insertar 500 trabajos en estado PENDIENTE
INSERT INTO trabajos (estado, payload)
SELECT
    'PENDIENTE',
    'payload_' || gs
FROM generate_series(1, 500) gs;

COMMIT;

-- Verificar datos
SELECT estado, count(*) FROM pedidos GROUP BY estado;
SELECT estado, count(*) FROM trabajos GROUP BY estado;
```

**Salida esperada:**

```
   estado    | count
-------------+-------
 COMPLETADO  |   334
 EN_PROCESO  |   333
 NUEVO       |   333
(3 rows)

  estado   | count
-----------+-------
 PENDIENTE |   500
(1 row)
```

#### Verificación

```sql
SELECT count(*) AS total_pedidos FROM pedidos;
SELECT count(*) AS total_trabajos FROM trabajos;
-- Ambas consultas deben retornar valores > 0
```

---

### Paso 2 — Observar el estado base de sesiones y bloqueos

**Objetivo:** Familiarizarse con las vistas `pg_stat_activity` y `pg_locks` en un estado sin contención, estableciendo una línea base antes de crear bloqueos.

#### Instrucciones

**2.1.** Consultar sesiones activas actuales:

```sql
-- Sesión A — Terminal 1
SELECT
    pid,
    usename,
    application_name,
    state,
    wait_event_type,
    wait_event,
    left(query, 60) AS query_truncada
FROM pg_stat_activity
WHERE datname = 'labdb'
  AND pid <> pg_backend_pid()
ORDER BY state;
```

**Salida esperada (línea base sin carga):**

```
 pid  | usename  | application_name | state  | wait_event_type | wait_event | query_truncada
------+----------+------------------+--------+-----------------+------------+----------------
(0 rows)   -- o solo conexiones de autovacuum/background
```

**2.2.** Consultar bloqueos activos actuales:

```sql
SELECT
    locktype,
    relation::regclass AS tabla,
    mode,
    granted,
    pid
FROM pg_locks
WHERE database = (SELECT oid FROM pg_database WHERE datname = current_database())
  AND relation IS NOT NULL
ORDER BY relation, mode;
```

**2.3.** Verificar que no hay sesiones "idle in transaction":

```sql
SELECT pid, usename, state, now() - xact_start AS duracion_tx
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY duracion_tx DESC;
-- Debe retornar 0 filas en un entorno limpio
```

#### Verificación

Confirma que no hay bloqueos no concedidos (`granted = false`) ni sesiones en estado `idle in transaction`. Esta es tu línea base.

---

### Paso 3 — Crear un escenario controlado de bloqueo de fila

**Objetivo:** Simular una situación real donde una transacción retiene un bloqueo de fila y bloquea a una segunda sesión, luego diagnosticar el bloqueo usando las vistas del sistema.

> ⚠️ **Importante:** Para este paso necesitas **dos terminales abiertas** conectadas a la misma base de datos. Denominaremos **Sesión A** (Terminal 1) y **Sesión B** (Terminal 2).

#### Instrucciones

**3.1.** En la **Sesión A** — inicia una transacción y adquiere un bloqueo de fila sobre el pedido 1:

```sql
-- ── SESIÓN A (Terminal 1) ─────────────────────────────────────────────────
BEGIN;

UPDATE pedidos
SET estado = 'EN_PROCESO'
WHERE id = 1;

-- ⚠️ NO ejecutes COMMIT todavía. Deja la transacción abierta.
-- Verifica que la actualización fue aplicada en esta sesión:
SELECT id, estado FROM pedidos WHERE id = 1;
```

**Salida esperada en Sesión A:**

```
 id |   estado
----+------------
  1 | EN_PROCESO
(1 row)
```

**3.2.** En la **Sesión B** — intenta actualizar la misma fila:

```sql
-- ── SESIÓN B (Terminal 2) ─────────────────────────────────────────────────
BEGIN;

UPDATE pedidos
SET estado = 'EN_COLA'
WHERE id = 1;

-- La sesión B quedará ESPERANDO (bloqueada por la Sesión A).
-- No verás el prompt de retorno hasta que A haga COMMIT o ROLLBACK.
```

**Comportamiento esperado:** La Sesión B queda suspendida esperando el bloqueo. El prompt no regresa.

**3.3.** Vuelve a la **Sesión A** y diagnostica el bloqueo desde una **tercera terminal** (o abre otra conexión `psql`):

```sql
-- ── SESIÓN DE DIAGNÓSTICO (Terminal 3) ───────────────────────────────────
-- Identificar qué sesiones están bloqueadas y quién las bloquea

SELECT
    bl.pid                     AS pid_bloqueado,
    ba.usename                 AS usuario_bloqueado,
    ba.state                   AS estado_bloqueado,
    ba.wait_event_type         AS tipo_espera,
    ba.wait_event              AS evento_espera,
    left(ba.query, 60)         AS sql_bloqueado,
    kl.pid                     AS pid_bloqueador,
    ka.usename                 AS usuario_bloqueador,
    ka.state                   AS estado_bloqueador,
    left(ka.query, 60)         AS sql_bloqueador,
    bl.mode                    AS modo_lock_solicitado,
    now() - ba.xact_start      AS duracion_espera
FROM pg_locks bl
JOIN pg_stat_activity ba ON bl.pid = ba.pid
JOIN pg_locks kl
    ON  bl.locktype     = kl.locktype
    AND bl.database     IS NOT DISTINCT FROM kl.database
    AND bl.relation     IS NOT DISTINCT FROM kl.relation
    AND bl.page         IS NOT DISTINCT FROM kl.page
    AND bl.tuple        IS NOT DISTINCT FROM kl.tuple
    AND bl.classid      IS NOT DISTINCT FROM kl.classid
    AND bl.objid        IS NOT DISTINCT FROM kl.objid
    AND bl.objsubid     IS NOT DISTINCT FROM kl.objsubid
    AND bl.transactionid IS NOT DISTINCT FROM kl.transactionid
    AND bl.pid          <> kl.pid
JOIN pg_stat_activity ka ON kl.pid = ka.pid
WHERE NOT bl.granted
ORDER BY duracion_espera DESC;
```

**Salida esperada:**

```
 pid_bloqueado | usuario_bloqueado | estado_bloqueado | tipo_espera | evento_espera |    sql_bloqueado     | pid_bloqueador | usuario_bloqueador | estado_bloqueador |      sql_bloqueador       | modo_lock_solicitado | duracion_espera
---------------+-------------------+------------------+-------------+---------------+----------------------+----------------+--------------------+-------------------+---------------------------+----------------------+-----------------
         12345 | labuser           | active           | Lock        | relation      | UPDATE pedidos SET e |          12344 | labuser            | idle in transaction | UPDATE pedidos SET estado | RowExclusiveLock     | 00:00:15.23
(1 row)
```

**3.4.** Usa `pg_blocking_pids()` para una consulta más directa:

```sql
-- Forma simplificada usando pg_blocking_pids()
SELECT
    pid,
    pg_blocking_pids(pid)  AS bloqueado_por,
    wait_event_type,
    wait_event,
    state,
    left(query, 80)        AS query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

**Salida esperada:**

```
  pid  | bloqueado_por | wait_event_type | wait_event |  state  |              query
-------+---------------+-----------------+------------+---------+----------------------------------
 12345 | {12344}       | Lock            | relation   | active  | UPDATE pedidos SET estado = 'EN_COLA' WHERE id = 1
(1 row)
```

**3.5.** Observa los tipos de bloqueo en `pg_locks` para ambas sesiones:

```sql
SELECT
    l.pid,
    l.locktype,
    l.relation::regclass AS tabla,
    l.mode,
    l.granted,
    a.state
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE l.relation = 'pedidos'::regclass
ORDER BY l.granted DESC, l.pid;
```

**Salida esperada:**

```
  pid  | locktype | tabla   |       mode       | granted |  state
-------+----------+---------+------------------+---------+---------
 12344 | relation | pedidos | RowExclusiveLock | t       | idle in transaction
 12345 | relation | pedidos | RowExclusiveLock | f       | active
(2 rows)
```

**3.6.** Resuelve el bloqueo haciendo `COMMIT` en la Sesión A:

```sql
-- ── SESIÓN A (Terminal 1) ─────────────────────────────────────────────────
COMMIT;
```

Observa cómo la Sesión B inmediatamente recibe el bloqueo y completa su `UPDATE`.

```sql
-- ── SESIÓN B (Terminal 2) — ahora completa ───────────────────────────────
-- Verás la salida: UPDATE 1
-- Luego haz rollback para no alterar los datos base:
ROLLBACK;
```

#### Verificación

```sql
-- Confirmar que no quedan bloqueos pendientes
SELECT count(*) AS bloqueos_pendientes
FROM pg_locks
WHERE NOT granted;
-- Debe retornar 0
```

---

### Paso 4 — Demostrar NOWAIT y SKIP LOCKED

**Objetivo:** Observar el comportamiento alternativo ante bloqueos usando las cláusulas `NOWAIT` y `SKIP LOCKED`, que evitan esperas indefinidas.

#### Instrucciones

**4.1.** Crea nuevamente el bloqueo en la Sesión A:

```sql
-- ── SESIÓN A (Terminal 1) ─────────────────────────────────────────────────
BEGIN;
UPDATE pedidos SET estado = 'EN_PROCESO' WHERE id = 5;
-- Mantener abierto
```

**4.2.** En la Sesión B, intenta con `NOWAIT`:

```sql
-- ── SESIÓN B (Terminal 2) ─────────────────────────────────────────────────
BEGIN;

UPDATE pedidos
SET estado = 'EN_COLA'
WHERE id = 5
-- NOWAIT lanza error inmediatamente en lugar de esperar
;
-- Primero prueba sin NOWAIT para confirmar que bloquea, luego ROLLBACK y prueba con NOWAIT:
ROLLBACK;

BEGIN;
SELECT id, estado
FROM pedidos
WHERE id = 5
FOR UPDATE NOWAIT;
-- Debe fallar inmediatamente con ERROR
```

**Salida esperada con NOWAIT:**

```
ERROR:  could not obtain lock on row in relation "pedidos"
```

**4.3.** Demuestra `SKIP LOCKED` para el patrón de cola de trabajos:

```sql
-- ── SESIÓN A (Terminal 1) — toma los primeros 5 trabajos ─────────────────
ROLLBACK; -- limpia la tx anterior
BEGIN;

SELECT id, payload
FROM trabajos
WHERE estado = 'PENDIENTE'
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 5;
-- Sesión A retiene bloqueo sobre esos 5 IDs
```

```sql
-- ── SESIÓN B (Terminal 2) — toma los siguientes 5 (sin colisión) ─────────
ROLLBACK;
BEGIN;

SELECT id, payload
FROM trabajos
WHERE estado = 'PENDIENTE'
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 5;
-- Sesión B obtiene IDs distintos (los siguientes disponibles)
```

**Salida esperada:** Ambas sesiones obtienen conjuntos de IDs **distintos** sin bloquearse mutuamente.

**4.4.** Limpia las transacciones abiertas:

```sql
-- En ambas sesiones:
ROLLBACK;
```

#### Verificación

```sql
SELECT count(*) AS bloqueos_activos
FROM pg_locks
WHERE NOT granted;
-- Debe retornar 0
```

---

### Paso 5 — Configurar lock_timeout y statement_timeout en el Cluster Parameter Group

**Objetivo:** Aplicar parámetros de timeout en el Aurora Cluster Parameter Group para que el motor cancele automáticamente sesiones bloqueadas o consultas lentas, reduciendo el impacto de bloqueos prolongados.

#### Instrucciones

**5.1.** Verifica el parameter group asociado al clúster:

```bash
# En la terminal local (no psql)
aws rds describe-db-clusters \
    --db-cluster-identifier "$CLUSTER_ID" \
    --region "$AWS_REGION" \
    --query 'DBClusters[0].DBClusterParameterGroup' \
    --output text
```

**Salida esperada:**

```
aurora-lab-pg15-params
```

**5.2.** Consulta los valores actuales de los parámetros de timeout:

```bash
aws rds describe-db-cluster-parameters \
    --db-cluster-parameter-group-name "$PARAM_GROUP" \
    --region "$AWS_REGION" \
    --query "Parameters[?ParameterName=='lock_timeout' || ParameterName=='statement_timeout' || ParameterName=='idle_in_transaction_session_timeout'].[ParameterName,ParameterValue,ApplyType]" \
    --output table
```

**Salida esperada:**

```
----------------------------------------------------------------------
|                    DescribeDBClusterParameters                     |
+------------------------------------------+---------------+---------+
|  idle_in_transaction_session_timeout     |               | dynamic |
|  lock_timeout                            |               | dynamic |
|  statement_timeout                       |               | dynamic |
+------------------------------------------+---------------+---------+
```

> Los valores vacíos indican que usan el default del motor (0 = sin límite).

**5.3.** Aplica valores de timeout recomendados para entornos de laboratorio:

```bash
aws rds modify-db-cluster-parameter-group \
    --db-cluster-parameter-group-name "$PARAM_GROUP" \
    --region "$AWS_REGION" \
    --parameters \
        "ParameterName=lock_timeout,ParameterValue=5000,ApplyMethod=immediate" \
        "ParameterName=statement_timeout,ParameterValue=30000,ApplyMethod=immediate" \
        "ParameterName=idle_in_transaction_session_timeout,ParameterValue=60000,ApplyMethod=immediate"
```

> **Nota:** Los valores están en milisegundos. `5000` = 5 segundos, `30000` = 30 segundos, `60000` = 60 segundos.

**Salida esperada:**

```json
{
    "DBClusterParameterGroupName": "aurora-lab-pg15-params"
}
```

**5.4.** Verifica que los parámetros fueron aplicados:

```bash
aws rds describe-db-cluster-parameters \
    --db-cluster-parameter-group-name "$PARAM_GROUP" \
    --region "$AWS_REGION" \
    --query "Parameters[?ParameterName=='lock_timeout' || ParameterName=='statement_timeout' || ParameterName=='idle_in_transaction_session_timeout'].[ParameterName,ParameterValue]" \
    --output table
```

**Salida esperada:**

```
-----------------------------------------------------------
|              DescribeDBClusterParameters                |
+-----------------------------------------+---------------+
|  idle_in_transaction_session_timeout    |  60000        |
|  lock_timeout                           |  5000         |
|  statement_timeout                      |  30000        |
+-----------------------------------------+---------------+
```

**5.5.** Verifica los parámetros activos desde `psql` (los parámetros `dynamic` aplican sin reinicio):

```sql
-- Sesión A — Terminal 1
SHOW lock_timeout;
SHOW statement_timeout;
SHOW idle_in_transaction_session_timeout;
```

**Salida esperada:**

```
 lock_timeout
--------------
 5s
(1 row)

 statement_timeout
-------------------
 30s
(1 row)

 idle_in_transaction_session_timeout
--------------------------------------
 1min
(1 row)
```

**5.6.** Prueba el efecto del `lock_timeout`: crea un bloqueo en Sesión A y espera que Sesión B sea cancelada automáticamente:

```sql
-- ── SESIÓN A (Terminal 1) ─────────────────────────────────────────────────
BEGIN;
UPDATE pedidos SET estado = 'EN_PROCESO' WHERE id = 10;
-- Mantener abierto durante más de 5 segundos
```

```sql
-- ── SESIÓN B (Terminal 2) ─────────────────────────────────────────────────
BEGIN;
UPDATE pedidos SET estado = 'EN_COLA' WHERE id = 10;
-- Después de ~5 segundos, debe recibir:
-- ERROR:  canceling statement due to lock timeout
```

```sql
-- ── SESIÓN A (Terminal 1) ─────────────────────────────────────────────────
ROLLBACK;
```

#### Verificación

```sql
-- Confirmar parámetros activos en la sesión
SELECT name, setting, unit
FROM pg_settings
WHERE name IN ('lock_timeout', 'statement_timeout', 'idle_in_transaction_session_timeout');
```

---

### Paso 6 — Analizar sesiones idle in transaction y detectar bloqueos prolongados

**Objetivo:** Identificar sesiones en estado `idle in transaction` que retienen bloqueos y podrían impedir que el autovacuum procese tablas correctamente.

#### Instrucciones

**6.1.** Simula una sesión "idle in transaction" larga:

```sql
-- ── SESIÓN A (Terminal 1) ─────────────────────────────────────────────────
BEGIN;
UPDATE pedidos SET estado = 'EN_PROCESO' WHERE id = 20;
-- Simula que la aplicación abrió una transacción y no la cerró
-- Espera 10 segundos sin hacer nada más
```

**6.2.** Desde la Sesión de Diagnóstico, detecta la sesión problemática:

```sql
-- ── SESIÓN DE DIAGNÓSTICO (Terminal 3) ───────────────────────────────────
SELECT
    pid,
    usename,
    application_name,
    state,
    xact_start,
    now() - xact_start          AS duracion_transaccion,
    state_change,
    left(query, 80)             AS ultima_query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY duracion_transaccion DESC;
```

**Salida esperada:**

```
  pid  | usename | application_name |       state        |          xact_start           | duracion_transaccion | ...
-------+---------+------------------+--------------------+-------------------------------+----------------------+----
 12344 | labuser | psql             | idle in transaction| 2024-01-15 10:30:00.123456+00 | 00:00:12.34          | ...
(1 row)
```

**6.3.** Identifica qué bloqueos retiene esa sesión:

```sql
SELECT
    a.pid,
    a.state,
    l.locktype,
    l.relation::regclass  AS tabla,
    l.mode,
    l.granted,
    now() - a.xact_start  AS tiempo_reteniendo_lock
FROM pg_stat_activity a
JOIN pg_locks l ON a.pid = l.pid
WHERE a.state = 'idle in transaction'
  AND l.granted = true
ORDER BY tiempo_reteniendo_lock DESC;
```

**6.4.** En un entorno de producción, si fuera necesario terminar la sesión:

```sql
-- ⚠️ Solo ejecutar si se confirma que la sesión debe ser terminada
-- SELECT pg_terminate_backend(<pid>);

-- Para este lab, simplemente haz ROLLBACK en Sesión A:
```

```sql
-- ── SESIÓN A (Terminal 1) ─────────────────────────────────────────────────
ROLLBACK;
```

**6.5.** Verifica los eventos de espera activos en el sistema:

```sql
SELECT
    wait_event_type,
    wait_event,
    count(*) AS sesiones_en_espera
FROM pg_stat_activity
WHERE wait_event_type IS NOT NULL
  AND datname = 'labdb'
GROUP BY wait_event_type, wait_event
ORDER BY sesiones_en_espera DESC;
```

#### Verificación

```sql
-- No debe haber sesiones idle in transaction tras el ROLLBACK
SELECT count(*) AS idle_in_tx
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND datname = 'labdb';
-- Debe retornar 0
```

---

### Paso 7 — Analizar el autovacuum y su impacto en la concurrencia

**Objetivo:** Usar `pg_stat_user_tables` para identificar tablas con alto conteo de dead tuples y comprender cómo las transacciones largas interfieren con el autovacuum, causando bloat.

#### Instrucciones

**7.1.** Genera dead tuples mediante actualizaciones:

```sql
-- Sesión A — Terminal 1
-- Actualiza múltiples filas para generar versiones muertas (dead tuples)
UPDATE pedidos SET monto = monto * 1.05 WHERE cliente_id < 100;
UPDATE pedidos SET monto = monto * 0.95 WHERE cliente_id BETWEEN 100 AND 200;
UPDATE pedidos SET estado = 'REVISADO'  WHERE estado = 'NUEVO' AND id < 200;
```

**7.2.** Consulta el estado del autovacuum en las tablas del laboratorio:

```sql
SELECT
    schemaname,
    relname                             AS tabla,
    n_live_tup                          AS filas_vivas,
    n_dead_tup                          AS filas_muertas,
    round(100.0 * n_dead_tup
          / NULLIF(n_live_tup + n_dead_tup, 0), 2)
                                        AS pct_dead,
    last_autovacuum,
    last_autoanalyze,
    autovacuum_count,
    autoanalyze_count,
    n_mod_since_analyze
FROM pg_stat_user_tables
WHERE relname IN ('pedidos', 'trabajos')
ORDER BY n_dead_tup DESC;
```

**Salida esperada (después de las actualizaciones):**

```
 schemaname |  tabla  | filas_vivas | filas_muertas | pct_dead | last_autovacuum | last_autoanalyze | ...
------------+---------+-------------+---------------+----------+-----------------+------------------+----
 public     | pedidos |        1000 |           400 |    28.57 | [timestamp]     | [timestamp]      | ...
 public     | trabajos|         500 |             0 |     0.00 | [timestamp]     | [timestamp]      | ...
(2 rows)
```

**7.3.** Verifica si el autovacuum está actualmente corriendo:

```sql
SELECT
    pid,
    datname,
    relid::regclass         AS tabla,
    phase,
    heap_blks_scanned,
    heap_blks_vacuumed,
    index_vacuum_count
FROM pg_stat_progress_vacuum;
```

**7.4.** Revisa la configuración actual del autovacuum relevante para la tabla:

```sql
SELECT
    name,
    setting,
    unit,
    short_desc
FROM pg_settings
WHERE name IN (
    'autovacuum_vacuum_threshold',
    'autovacuum_vacuum_scale_factor',
    'autovacuum_analyze_threshold',
    'autovacuum_analyze_scale_factor',
    'autovacuum_vacuum_cost_delay'
)
ORDER BY name;
```

**7.5.** Demuestra el impacto de una transacción larga en el autovacuum:

```sql
-- Sesión A — Terminal 1
-- Una transacción abierta con un xmin antiguo impide que autovacuum
-- elimine dead tuples que son visibles para esa transacción
BEGIN;
SELECT txid_current() AS mi_xid;
-- Mantén la transacción abierta y observa:
```

```sql
-- Sesión de Diagnóstico — Terminal 3
-- El xmin más antiguo del sistema limita qué dead tuples puede limpiar autovacuum
SELECT
    pid,
    usename,
    state,
    backend_xmin,
    now() - xact_start AS duracion
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin ASC
LIMIT 5;
```

```sql
-- Sesión A — Terminal 1 — cierra la transacción
ROLLBACK;
```

#### Verificación

```sql
-- Ejecuta VACUUM manual para limpiar dead tuples y verificar el proceso
VACUUM ANALYZE pedidos;

-- Verifica que los dead tuples se redujeron
SELECT relname, n_dead_tup, last_vacuum
FROM pg_stat_user_tables
WHERE relname = 'pedidos';
```

---

### Paso 8 — Correlacionar métricas de bloqueo en CloudWatch

**Objetivo:** Observar las métricas `BlockedTransactions` y `DatabaseConnections` en CloudWatch durante los escenarios de bloqueo para correlacionar los eventos de base de datos con las métricas de monitoreo.

#### Instrucciones

**8.1.** Genera un período de bloqueo sostenido para que CloudWatch lo registre:

```sql
-- ── SESIÓN A (Terminal 1) ─────────────────────────────────────────────────
BEGIN;
UPDATE pedidos SET estado = 'BLOQUEADO_TEST' WHERE id IN (50, 51, 52, 53, 54);
-- Mantener abierto durante 2-3 minutos para que CloudWatch registre la métrica
```

```sql
-- ── SESIÓN B (Terminal 2) ─────────────────────────────────────────────────
BEGIN;
UPDATE pedidos SET estado = 'ESPERANDO' WHERE id = 50;
-- Sesión B queda bloqueada — CloudWatch registrará BlockedTransactions > 0
```

**8.2.** Mientras el bloqueo está activo, consulta las métricas desde AWS CLI:

```bash
# Obtener métricas de los últimos 5 minutos
END_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
START_TIME=$(date -u -d "5 minutes ago" +"%Y-%m-%dT%H:%M:%SZ" 2>/dev/null \
    || date -u -v-5M +"%Y-%m-%dT%H:%M:%SZ")  # macOS compatible

aws cloudwatch get-metric-statistics \
    --namespace "AWS/RDS" \
    --metric-name "BlockedTransactions" \
    --dimensions Name=DBClusterIdentifier,Value="$CLUSTER_ID" \
    --start-time "$START_TIME" \
    --end-time "$END_TIME" \
    --period 60 \
    --statistics Maximum \
    --region "$AWS_REGION" \
    --output json | jq '.Datapoints | sort_by(.Timestamp)'
```

**Salida esperada (con bloqueo activo):**

```json
[
  {
    "Timestamp": "2024-01-15T10:35:00Z",
    "Maximum": 1.0,
    "Unit": "Count"
  }
]
```

**8.3.** Consulta también la métrica de conexiones activas:

```bash
aws cloudwatch get-metric-statistics \
    --namespace "AWS/RDS" \
    --metric-name "DatabaseConnections" \
    --dimensions Name=DBClusterIdentifier,Value="$CLUSTER_ID" \
    --start-time "$START_TIME" \
    --end-time "$END_TIME" \
    --period 60 \
    --statistics Maximum Average \
    --region "$AWS_REGION" \
    --output json | jq '.Datapoints | sort_by(.Timestamp)'
```

**8.4.** Resuelve el bloqueo y verifica que la métrica regresa a 0:

```sql
-- ── SESIÓN A (Terminal 1) ─────────────────────────────────────────────────
ROLLBACK;
```

```sql
-- ── SESIÓN B (Terminal 2) ─────────────────────────────────────────────────
-- Recibirá el lock y completará. Luego:
ROLLBACK;
```

**8.5.** Accede a la consola de CloudWatch para visualizar las métricas gráficamente:

1. Navega a **AWS Console → CloudWatch → Metrics → AWS/RDS**.
2. Selecciona tu clúster bajo **DBClusterIdentifier**.
3. Agrega las métricas: `BlockedTransactions`, `DatabaseConnections`, `DeadlockCount`.
4. Configura el período de visualización a los últimos **30 minutos** con granularidad de **1 minuto**.
5. Observa el pico de `BlockedTransactions` durante el período del bloqueo.

#### Verificación

```bash
# Verificar que BlockedTransactions regresó a 0 después del ROLLBACK
aws cloudwatch get-metric-statistics \
    --namespace "AWS/RDS" \
    --metric-name "BlockedTransactions" \
    --dimensions Name=DBClusterIdentifier,Value="$CLUSTER_ID" \
    --start-time "$(date -u -d '2 minutes ago' +"%Y-%m-%dT%H:%M:%SZ" 2>/dev/null || date -u -v-2M +"%Y-%m-%dT%H:%M:%SZ")" \
    --end-time "$(date -u +"%Y-%m-%dT%H:%M:%SZ")" \
    --period 60 \
    --statistics Maximum \
    --region "$AWS_REGION" \
    --output json | jq '.Datapoints[-1].Maximum // "Sin datos aún"'
```

---

## 7. Validación y Pruebas

Ejecuta las siguientes consultas de validación para confirmar que todos los objetivos del laboratorio fueron cumplidos:

```sql
-- ── VALIDACIÓN COMPLETA DEL LABORATORIO ──────────────────────────────────

-- V1: Sin bloqueos pendientes
SELECT
    CASE
        WHEN count(*) = 0 THEN '✓ PASS: No hay bloqueos pendientes'
        ELSE '✗ FAIL: Hay ' || count(*) || ' bloqueos sin conceder'
    END AS resultado_v1
FROM pg_locks
WHERE NOT granted;

-- V2: Sin sesiones idle in transaction
SELECT
    CASE
        WHEN count(*) = 0 THEN '✓ PASS: No hay sesiones idle in transaction'
        ELSE '✗ FAIL: Hay ' || count(*) || ' sesiones idle in transaction'
    END AS resultado_v2
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND datname = 'labdb';

-- V3: Parámetros de timeout configurados correctamente
SELECT
    CASE
        WHEN setting = '5000' THEN '✓ PASS: lock_timeout = 5000ms'
        ELSE '✗ FAIL: lock_timeout = ' || setting || ' (esperado: 5000)'
    END AS resultado_v3
FROM pg_settings
WHERE name = 'lock_timeout';

SELECT
    CASE
        WHEN setting = '30000' THEN '✓ PASS: statement_timeout = 30000ms'
        ELSE '✗ FAIL: statement_timeout = ' || setting || ' (esperado: 30000)'
    END AS resultado_v4
FROM pg_settings
WHERE name = 'statement_timeout';

-- V5: Tablas de prueba con datos
SELECT
    CASE
        WHEN (SELECT count(*) FROM pedidos) >= 1000
         AND (SELECT count(*) FROM trabajos) >= 500
        THEN '✓ PASS: Tablas con datos suficientes'
        ELSE '✗ FAIL: Datos insuficientes en tablas'
    END AS resultado_v5;

-- V6: pg_blocking_pids() funciona correctamente (sin bloqueos activos = array vacío)
SELECT
    CASE
        WHEN count(*) = 0 THEN '✓ PASS: pg_blocking_pids() no reporta bloqueos activos'
        ELSE '✗ FAIL: Hay ' || count(*) || ' sesiones bloqueadas activas'
    END AS resultado_v6
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

**Salida esperada de todas las validaciones:**

```
           resultado_v1
------------------------------------------
 ✓ PASS: No hay bloqueos pendientes

           resultado_v2
------------------------------------------
 ✓ PASS: No hay sesiones idle in transaction

           resultado_v3
------------------------------------------
 ✓ PASS: lock_timeout = 5000ms

           resultado_v4
------------------------------------------
 ✓ PASS: statement_timeout = 30000ms

           resultado_v5
------------------------------------------
 ✓ PASS: Tablas con datos suficientes

           resultado_v6
------------------------------------------
 ✓ PASS: pg_blocking_pids() no reporta bloqueos activos
```

---

## 8. Resolución de Problemas

### Problema 1 — La Sesión B no muestra el estado de espera en `pg_stat_activity`

**Síntomas:**
- Al consultar `pg_stat_activity`, la Sesión B aparece con `state = 'active'` pero `wait_event` es `NULL` o el bloqueo no es visible en `pg_locks`.
- La columna `wait_event_type` no muestra `Lock`.

**Causa probable:**
El bloqueo puede resolverse antes de que ejecutes la consulta de diagnóstico (la transacción de la Sesión A fue confirmada o revertida accidentalmente), o estás consultando `pg_stat_activity` desde una **réplica lectora** de Aurora en lugar de la instancia escritora. Las réplicas lectoras no muestran la contención de escritura de la instancia primaria.

**Solución:**

```bash
# Verifica a qué instancia estás conectado
psql -c "SELECT aurora_db_instance_identifier();"
# Debe mostrar el escritor, no una réplica

# Verifica el endpoint — debe ser el writer endpoint
echo $PGHOST
# Debe contener ".cluster." no ".cluster-ro."

# Si estás en una réplica, reconéctate al writer endpoint:
export PGHOST="aurora-lab-cluster.cluster-xxxxxxxxxxxx.us-east-1.rds.amazonaws.com"
psql -c "SELECT pg_is_in_recovery();"
# Debe retornar: f (false = es el escritor)
```

```sql
-- Confirma que la Sesión A aún tiene la transacción abierta:
SELECT pid, state, xact_start, query
FROM pg_stat_activity
WHERE state IN ('idle in transaction', 'active')
  AND pid <> pg_backend_pid();
```

---

### Problema 2 — Los parámetros `lock_timeout` y `statement_timeout` no se aplican tras modificar el Parameter Group

**Síntomas:**
- `SHOW lock_timeout;` en `psql` retorna `0` (sin límite) incluso después de ejecutar `aws rds modify-db-cluster-parameter-group`.
- Las sesiones bloqueadas no son canceladas automáticamente después del tiempo configurado.
- El comando AWS CLI retornó éxito pero los valores no cambiaron.

**Causa probable:**
Aunque `lock_timeout` y `statement_timeout` son parámetros `dynamic` (no requieren reinicio del clúster), las **sesiones existentes** que se conectaron antes del cambio del parameter group pueden no haber recibido los nuevos valores. Adicionalmente, si el parámetro fue configurado explícitamente a nivel de sesión o rol con `SET`, ese valor tiene precedencia sobre el parameter group.

**Solución:**

```bash
# 1. Verifica que el parameter group fue modificado correctamente
aws rds describe-db-cluster-parameters \
    --db-cluster-parameter-group-name "$PARAM_GROUP" \
    --region "$AWS_REGION" \
    --query "Parameters[?ParameterName=='lock_timeout'].[ParameterName,ParameterValue,IsModifiable,ApplyType]" \
    --output table
```

```sql
-- 2. En la sesión psql, verifica si hay un override de sesión:
SELECT name, setting, source
FROM pg_settings
WHERE name IN ('lock_timeout', 'statement_timeout');
-- Si 'source' = 'session', hay un SET explícito que sobreescribe el parameter group

-- 3. Resetea los valores de sesión a los del parameter group:
RESET lock_timeout;
RESET statement_timeout;

-- 4. Verifica nuevamente:
SHOW lock_timeout;
-- Debe mostrar: 5s
```

```bash
# 5. Si el problema persiste, reconéctate con una nueva sesión psql
# Las nuevas conexiones siempre reciben los parámetros del parameter group
psql -c "SHOW lock_timeout;"
```

---

## 9. Limpieza de Recursos

> ⚠️ **Importante:** Ejecuta estos pasos al finalizar el laboratorio para evitar costos innecesarios y dejar el entorno limpio para el siguiente laboratorio.

### 9.1. Cerrar todas las transacciones abiertas

```sql
-- Verificar y cerrar cualquier transacción pendiente
-- En Sesión A:
ROLLBACK;

-- En Sesión B:
ROLLBACK;
```

### 9.2. Restaurar los parámetros del Parameter Group a valores por defecto

```bash
# Restaurar parámetros a sus valores por defecto (0 = sin límite)
# Nota: En producción, mantén valores de timeout razonables
aws rds reset-db-cluster-parameter-group \
    --db-cluster-parameter-group-name "$PARAM_GROUP" \
    --region "$AWS_REGION" \
    --parameters \
        "ParameterName=lock_timeout,ApplyMethod=immediate" \
        "ParameterName=statement_timeout,ApplyMethod=immediate" \
        "ParameterName=idle_in_transaction_session_timeout,ApplyMethod=immediate"
```

### 9.3. Limpiar datos de prueba del laboratorio

```sql
-- Conectado a la base de datos labdb
-- Eliminar tablas de prueba creadas en este lab
-- NOTA: Si el siguiente lab (02-00-02) las necesita, omite este paso

DROP TABLE IF EXISTS trabajos;
DROP TABLE IF EXISTS pedidos;

-- Verificar limpieza
SELECT tablename
FROM pg_tables
WHERE schemaname = 'public'
  AND tablename IN ('pedidos', 'trabajos');
-- Debe retornar 0 filas
```

### 9.4. Verificar que no quedan sesiones activas problemáticas

```sql
-- Verificar estado final del sistema
SELECT
    state,
    count(*) AS sesiones
FROM pg_stat_activity
WHERE datname = 'labdb'
  AND pid <> pg_backend_pid()
GROUP BY state;
```

### 9.5. (Opcional) Destruir infraestructura con Terraform

Si este es el último laboratorio del módulo 2 y no continuarás con el siguiente inmediatamente:

```bash
# Desde el directorio del módulo Terraform
cd terraform/module-02/
terraform destroy -auto-approve

# Confirmar destrucción
echo "Infraestructura del Módulo 2 destruida. Costo detenido."
```

---

## 10. Resumen

En este laboratorio aplicaste las técnicas de diagnóstico de bloqueos y concurrencia de Aurora PostgreSQL en escenarios controlados y representativos:

| Actividad realizada                                | Concepto aplicado                                    |
|----------------------------------------------------|------------------------------------------------------|
| Consulta de `pg_locks` y `pg_stat_activity`        | Diagnóstico de bloqueos en tiempo real               |
| Uso de `pg_blocking_pids()`                        | Identificación directa de cadenas de bloqueo         |
| Escenario `UPDATE` bloqueado entre dos sesiones    | `RowExclusiveLock`, MVCC y esperas de fila           |
| Demostración de `NOWAIT` y `SKIP LOCKED`           | Control de comportamiento ante bloqueos              |
| Configuración de `lock_timeout` en Parameter Group | Gestión proactiva de esperas prolongadas             |
| Análisis de `pg_stat_user_tables`                  | Impacto del autovacuum y dead tuples                 |
| Métricas `BlockedTransactions` en CloudWatch       | Correlación entre eventos DB y monitoreo AWS         |

### Conceptos Clave del Laboratorio

- **MVCC** permite que lectores y escritores coexistan, pero las escrituras concurrentes sobre las **mismas filas** sí generan esperas por `RowExclusiveLock`.
- **`pg_blocking_pids(pid)`** es la forma más directa de identificar qué proceso bloquea a otro, complementando la consulta detallada sobre `pg_locks`.
- **`lock_timeout`** e **`idle_in_transaction_session_timeout`** son defensas críticas contra bloqueos indefinidos en entornos de producción.
- Las sesiones **`idle in transaction`** son especialmente peligrosas: retienen bloqueos y evitan que el autovacuum limpie dead tuples, causando bloat progresivo.
- En **Aurora PostgreSQL**, toda la contención de escritura ocurre en la **instancia escritora**; las réplicas lectoras no muestran estos bloqueos.

### Próximos Pasos

- **Laboratorio 02-00-02:** Profundizar en la detección y resolución de **deadlocks** (interbloqueos circulares), que son la consecuencia más grave de patrones de bloqueo inadecuados.
- Configurar **alertas de CloudWatch** sobre `BlockedTransactions > 0` y `DatabaseConnections` cercano al máximo permitido.
- Revisar los diseños de aplicación que acceden a filas "calientes" (hot rows) con alta frecuencia y evaluar estrategias de particionamiento o colas con `SKIP LOCKED`.

### Referencias

- [Documentación de PostgreSQL: Bloqueos explícitos](https://www.postgresql.org/docs/current/explicit-locking.html)
- [Vista `pg_locks`](https://www.postgresql.org/docs/current/view-pg-locks.html)
- [Vista `pg_stat_activity`](https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-ACTIVITY-VIEW)
- [Función `pg_blocking_pids()`](https://www.postgresql.org/docs/current/functions-info.html)
- [Aurora PostgreSQL — Guía de usuario de Amazon Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.html)
- [Performance Insights para Amazon Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PerfInsights.html)
- [Parámetros de clúster Aurora PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Reference.ParameterGroups.html)

---

---

# Simulación de concurrencia y deadlocks

## Metadatos

| Campo            | Valor                          |
|------------------|-------------------------------|
| **Duración**     | 35 minutos                    |
| **Complejidad**  | Difícil                       |
| **Nivel Bloom**  | Aplicar (Apply)               |
| **Módulo**       | 2 — Bloqueos y Concurrencia   |
| **Laboratorio**  | 02-00-02 / Práctica 3         |

---

## Descripción General

En este laboratorio reproducirás de forma controlada escenarios de **deadlock** en Aurora PostgreSQL utilizando scripts Python con múltiples hilos concurrentes que acceden a recursos en orden inverso. Analizarás los mensajes de error generados en los logs de PostgreSQL, identificarás los procesos involucrados mediante `pg_stat_activity` y `pg_locks`, y aplicarás estrategias de mitigación como el ordenamiento consistente de recursos, `SELECT FOR UPDATE NOWAIT`, `SKIP LOCKED` y la selección apropiada del nivel de aislamiento. Finalmente, configurarás el parámetro `deadlock_timeout` en el cluster parameter group de Aurora y observarás su efecto en la detección y resolución de interbloqueos.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio serás capaz de:

- [ ] Reproducir deadlocks predecibles en Aurora PostgreSQL usando scripts Python con `psycopg2` y múltiples hilos concurrentes que acceden a filas en orden inverso
- [ ] Analizar los mensajes `ERROR: deadlock detected` en los logs de PostgreSQL e identificar los procesos y consultas involucrados mediante `pg_locks` y `pg_stat_activity`
- [ ] Implementar estrategias de prevención de deadlocks: ordenamiento consistente de recursos, `SELECT FOR UPDATE NOWAIT` y `SKIP LOCKED`
- [ ] Comparar el comportamiento del sistema bajo los niveles de aislamiento `READ COMMITTED`, `REPEATABLE READ` y `SERIALIZABLE` en escenarios de concurrencia alta
- [ ] Configurar `deadlock_timeout` en un Aurora Cluster Parameter Group y verificar su impacto en la detección de interbloqueos

---

## Prerrequisitos

### Conocimientos

- Haber completado el Laboratorio 02-00-01 (análisis de bloqueos básicos)
- Comprensión de transacciones ACID y niveles de aislamiento en PostgreSQL
- Familiaridad con Python 3.9+ y el módulo `threading`
- Conocimiento básico de `pg_stat_activity` y `pg_locks`

### Acceso y Recursos

- Instancia Aurora PostgreSQL (db.r6g.large recomendada, mínimo db.t3.medium) activa y accesible
- Usuario IAM con permisos sobre RDS, CloudWatch y Secrets Manager
- Base de datos de prueba `labdb` creada en el laboratorio anterior
- Permisos para modificar parámetros en el Aurora Cluster Parameter Group
- Python 3.9+ con `psycopg2-binary` instalado localmente
- `psql` (14.x o superior) disponible en la terminal
- AWS CLI 2.x configurado con el perfil del curso

---

## Entorno del Laboratorio

### Requisitos de Hardware/Software

| Componente        | Requisito Mínimo                         |
|-------------------|------------------------------------------|
| RAM local         | 8 GB (para múltiples hilos Python)       |
| CPU local         | 4 núcleos                                |
| Python            | 3.9 o superior                           |
| psycopg2-binary   | 2.9.x o superior                         |
| psql              | 14.x o superior                          |
| AWS CLI           | 2.x configurado                          |
| Aurora PostgreSQL | 14.x, instancia db.r6g.large             |

### Variables de Entorno y Configuración Inicial

Ejecuta los siguientes comandos en tu terminal antes de comenzar el laboratorio. Sustituye los valores entre `< >` con los datos de tu entorno:

```bash
# 1. Definir variables de entorno del laboratorio
export LAB_DB_HOST="<endpoint-writer-aurora>"
export LAB_DB_PORT="5432"
export LAB_DB_NAME="labdb"
export LAB_DB_USER="labuser"
export LAB_DB_PASSWORD="<tu-password>"
export LAB_CLUSTER_ID="<nombre-del-cluster-aurora>"
export LAB_PARAM_GROUP="<nombre-del-cluster-parameter-group>"
export AWS_REGION="us-east-1"   # Ajusta según tu región

# 2. Verificar conectividad con la instancia escritora
psql "host=$LAB_DB_HOST port=$LAB_DB_PORT dbname=$LAB_DB_NAME \
      user=$LAB_DB_USER password=$LAB_DB_PASSWORD sslmode=require" \
     -c "SELECT version();"

# 3. Verificar que psycopg2 está instalado
python3 -c "import psycopg2; print('psycopg2 version:', psycopg2.__version__)"

# 4. Crear directorio de trabajo para el laboratorio
mkdir -p ~/lab-02-00-02/{scripts,logs,results}
cd ~/lab-02-00-02
```

**Salida esperada del paso 2:**
```
                                                 version
---------------------------------------------------------------------------------------------------------
 PostgreSQL 14.x (compatible with Aurora PostgreSQL)
(1 row)
```

---

## Pasos del Laboratorio

---

### Paso 1: Preparar el Esquema de Prueba

**Objetivo:** Crear las tablas necesarias para los escenarios de deadlock y poblarlas con datos iniciales.

#### Instrucciones

1. Conéctate a la base de datos y crea el esquema de prueba:

```sql
-- Conectar a labdb
\c labdb

-- Crear tabla de cuentas bancarias (escenario de transferencia)
CREATE TABLE IF NOT EXISTS cuentas (
    id          SERIAL PRIMARY KEY,
    titular     VARCHAR(100) NOT NULL,
    saldo       NUMERIC(12,2) NOT NULL DEFAULT 0.00,
    version     INTEGER NOT NULL DEFAULT 0,
    updated_at  TIMESTAMP DEFAULT NOW()
);

-- Crear tabla de trabajos (escenario de cola de trabajo)
CREATE TABLE IF NOT EXISTS trabajos (
    id          SERIAL PRIMARY KEY,
    descripcion VARCHAR(200) NOT NULL,
    estado      VARCHAR(20) NOT NULL DEFAULT 'PENDIENTE',
    prioridad   INTEGER NOT NULL DEFAULT 5,
    created_at  TIMESTAMP DEFAULT NOW()
);

-- Crear tabla de auditoría de deadlocks
CREATE TABLE IF NOT EXISTS auditoria_deadlocks (
    id           SERIAL PRIMARY KEY,
    ts           TIMESTAMP DEFAULT NOW(),
    pid          INTEGER,
    hilo_nombre  VARCHAR(50),
    accion       VARCHAR(200),
    resultado    VARCHAR(50)
);

-- Insertar datos iniciales en cuentas
INSERT INTO cuentas (titular, saldo) VALUES
    ('Alice',  10000.00),
    ('Bob',     8000.00),
    ('Carlos',  5000.00),
    ('Diana',  12000.00),
    ('Eduardo', 3000.00)
ON CONFLICT DO NOTHING;

-- Insertar trabajos de prueba
INSERT INTO trabajos (descripcion, estado, prioridad)
SELECT
    'Tarea #' || i,
    'PENDIENTE',
    (RANDOM() * 9 + 1)::INTEGER
FROM generate_series(1, 50) AS i
ON CONFLICT DO NOTHING;

-- Verificar datos
SELECT COUNT(*) AS total_cuentas FROM cuentas;
SELECT COUNT(*) AS total_trabajos FROM trabajos;
```

2. Crea un índice para optimizar las consultas de cola:

```sql
CREATE INDEX IF NOT EXISTS idx_trabajos_estado_prioridad
    ON trabajos(estado, prioridad DESC)
    WHERE estado = 'PENDIENTE';

-- Confirmar estructura
\d cuentas
\d trabajos
```

#### Salida Esperada

```
 total_cuentas
---------------
             5
(1 row)

 total_trabajos
----------------
             50
(1 row)
```

#### Verificación

```sql
-- Confirmar que no hay bloqueos activos antes de comenzar
SELECT COUNT(*) AS bloqueos_activos
FROM pg_locks
WHERE NOT granted;
-- Debe retornar 0
```

---

### Paso 2: Reproducir un Deadlock Clásico (Dos Sesiones Manuales)

**Objetivo:** Entender el mecanismo de deadlock reproduciendo manualmente el escenario de "transferencia bancaria en orden inverso" antes de automatizarlo con Python.

#### Instrucciones

1. Abre **dos terminales** separadas y conéctate a la base de datos en cada una.

**Terminal A — Sesión 1:**
```sql
-- Terminal A
\c labdb
BEGIN;
-- Paso A1: Bloquear cuenta id=1 (Alice)
UPDATE cuentas SET saldo = saldo - 500, version = version + 1
WHERE id = 1;
-- *** PAUSA: NO ejecutar el siguiente UPDATE todavía ***
-- Mantener la transacción abierta y pasar a Terminal B
```

**Terminal B — Sesión 2:**
```sql
-- Terminal B
\c labdb
BEGIN;
-- Paso B1: Bloquear cuenta id=2 (Bob) PRIMERO (orden inverso a Sesión A)
UPDATE cuentas SET saldo = saldo - 300, version = version + 1
WHERE id = 2;
-- *** PAUSA: La sesión A ya tiene id=1 bloqueado ***
-- Ahora intentar bloquear id=1 (que ya tiene Sesión A)
UPDATE cuentas SET saldo = saldo + 300, version = version + 1
WHERE id = 1;
-- Esta sesión quedará esperando a Sesión A...
```

**Terminal A — Sesión 1 (continuar):**
```sql
-- Terminal A: Ahora intentar bloquear id=2 (que ya tiene Sesión B)
UPDATE cuentas SET saldo = saldo + 500, version = version + 1
WHERE id = 2;
-- PostgreSQL detectará el deadlock y cancelará una de las transacciones
```

2. Observa el mensaje de error en la sesión que es víctima del deadlock:

```
ERROR:  deadlock detected
DETAIL:  Process 12345 waits for ShareLock on transaction 789; blocked by process 67890.
         Process 67890 waits for ShareLock on transaction 456; blocked by process 12345.
HINT:  See server log for query details.
```

3. En una **tercera terminal**, consulta los logs de Aurora para ver el registro del deadlock:

```bash
# Obtener los logs recientes del cluster Aurora
aws rds describe-db-log-files \
    --db-instance-identifier "${LAB_CLUSTER_ID}-instance-1" \
    --region $AWS_REGION \
    --query 'DescribeDBLogFiles[*].LogFileName' \
    --output text

# Descargar el log más reciente (ajusta el nombre del archivo)
aws rds download-db-log-file-portion \
    --db-instance-identifier "${LAB_CLUSTER_ID}-instance-1" \
    --log-file-name "error/postgresql.log" \
    --region $AWS_REGION \
    --output text | grep -A 5 "deadlock detected" | tail -30
```

4. Hacer ROLLBACK en ambas sesiones para limpiar:

```sql
-- En ambas terminales
ROLLBACK;
```

#### Salida Esperada del Error de Deadlock

```
ERROR:  deadlock detected
DETAIL:  Process 18423 waits for ShareLock on transaction 5621; blocked by process 18456.
         Process 18456 waits for ShareLock on transaction 5618; blocked by process 18423.
HINT:  See server log for query details.
```

#### Verificación

```sql
-- Confirmar que los saldos no cambiaron (ROLLBACK exitoso)
SELECT id, titular, saldo FROM cuentas WHERE id IN (1, 2);
-- Alice debe tener 10000.00, Bob 8000.00
```

---

### Paso 3: Configurar `deadlock_timeout` en el Cluster Parameter Group

**Objetivo:** Modificar el parámetro `deadlock_timeout` en Aurora y observar cómo afecta la velocidad de detección de interbloqueos.

#### Instrucciones

1. Consulta el valor actual de `deadlock_timeout`:

```sql
-- Ver configuración actual
SHOW deadlock_timeout;
-- Por defecto: 1s
```

2. Consulta el parameter group actual del cluster:

```bash
# Identificar el cluster parameter group en uso
aws rds describe-db-clusters \
    --db-cluster-identifier $LAB_CLUSTER_ID \
    --region $AWS_REGION \
    --query 'DBClusters[0].DBClusterParameterGroup' \
    --output text
```

3. Modifica `deadlock_timeout` a 500ms para detección más rápida:

```bash
# Modificar el parámetro deadlock_timeout
aws rds modify-db-cluster-parameter-group \
    --db-cluster-parameter-group-name $LAB_PARAM_GROUP \
    --parameters "ParameterName=deadlock_timeout,ParameterValue=500,ApplyMethod=immediate" \
    --region $AWS_REGION

# Verificar que el cambio fue aceptado
aws rds describe-db-cluster-parameters \
    --db-cluster-parameter-group-name $LAB_PARAM_GROUP \
    --region $AWS_REGION \
    --query "Parameters[?ParameterName=='deadlock_timeout']" \
    --output table
```

4. Reinicia la instancia para que el parámetro surta efecto (si es necesario):

```bash
# Verificar si el parámetro requiere reinicio
aws rds describe-db-cluster-parameters \
    --db-cluster-parameter-group-name $LAB_PARAM_GROUP \
    --region $AWS_REGION \
    --query "Parameters[?ParameterName=='deadlock_timeout'].{Nombre:ParameterName,Valor:ParameterValue,Aplicacion:ApplyType}" \
    --output table
```

> **Nota:** `deadlock_timeout` es un parámetro de tipo `dynamic` en Aurora PostgreSQL, por lo que el cambio con `ApplyMethod=immediate` no requiere reinicio de la instancia.

5. Confirma el nuevo valor desde `psql`:

```sql
-- Reconectar y verificar
\c labdb
SHOW deadlock_timeout;
-- Debe mostrar: 500ms
```

#### Salida Esperada

```
 deadlock_timeout
------------------
 500ms
(1 row)
```

#### Verificación

```bash
# Confirmar desde AWS CLI que el parámetro está aplicado
aws rds describe-db-cluster-parameters \
    --db-cluster-parameter-group-name $LAB_PARAM_GROUP \
    --region $AWS_REGION \
    --query "Parameters[?ParameterName=='deadlock_timeout'].{ParameterValue:ParameterValue,IsModifiable:IsModifiable}" \
    --output table
```

---

### Paso 4: Script Python — Simulación Automatizada de Deadlocks

**Objetivo:** Automatizar la generación de deadlocks con múltiples hilos Python usando `psycopg2`, registrar los errores y medir la frecuencia de interbloqueos.

#### Instrucciones

1. Crea el script de simulación de deadlocks:

```bash
cat > ~/lab-02-00-02/scripts/simular_deadlocks.py << 'PYEOF'
#!/usr/bin/env python3
"""
Lab 02-00-02 - Simulación de Deadlocks con psycopg2
Escenario: Transferencias bancarias con acceso en orden inverso
"""

import psycopg2
import threading
import time
import logging
import os
from datetime import datetime

# ── Configuración de logging ──────────────────────────────────────────────────
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(threadName)-12s] %(levelname)-8s %(message)s',
    handlers=[
        logging.FileHandler(f'/root/lab-02-00-02/logs/deadlocks_{datetime.now().strftime("%Y%m%d_%H%M%S")}.log'),
        logging.StreamHandler()
    ]
)
log = logging.getLogger(__name__)

# ── Parámetros de conexión (leer de variables de entorno) ─────────────────────
DB_CONFIG = {
    'host':     os.environ.get('LAB_DB_HOST', 'localhost'),
    'port':     int(os.environ.get('LAB_DB_PORT', 5432)),
    'dbname':   os.environ.get('LAB_DB_NAME', 'labdb'),
    'user':     os.environ.get('LAB_DB_USER', 'labuser'),
    'password': os.environ.get('LAB_DB_PASSWORD', ''),
    'sslmode':  'require',
    'connect_timeout': 10,
    'options':  '-c lock_timeout=3000'   # 3 segundos máximo de espera por lock
}

# ── Contadores compartidos (thread-safe) ──────────────────────────────────────
contador_lock = threading.Lock()
stats = {
    'transferencias_ok':    0,
    'deadlocks_detectados': 0,
    'otros_errores':        0,
    'total_intentos':       0,
}


def transferencia_orden_inverso(hilo_id: int, cuenta_origen: int, cuenta_destino: int,
                                 monto: float, iteraciones: int = 5) -> None:
    """
    Simula transferencias bancarias accediendo a cuentas en ORDEN INVERSO
    para provocar deadlocks predecibles.
    Hilo par: bloquea cuenta_origen primero, luego cuenta_destino
    Hilo impar: bloquea cuenta_destino primero, luego cuenta_origen
    """
    conn = None
    try:
        conn = psycopg2.connect(**DB_CONFIG)
        conn.autocommit = False

        for i in range(iteraciones):
            try:
                with contador_lock:
                    stats['total_intentos'] += 1

                cur = conn.cursor()

                # Orden inverso según paridad del hilo → garantiza deadlock
                if hilo_id % 2 == 0:
                    primera, segunda = cuenta_origen, cuenta_destino
                else:
                    primera, segunda = cuenta_destino, cuenta_origen

                log.info(f"Hilo-{hilo_id} iter-{i+1}: bloqueando cuenta {primera} → {segunda}")

                # Bloquear primera cuenta
                cur.execute(
                    "UPDATE cuentas SET saldo = saldo - %s, version = version + 1 "
                    "WHERE id = %s AND saldo >= %s",
                    (monto, primera, monto)
                )
                filas = cur.rowcount
                if filas == 0:
                    log.warning(f"Hilo-{hilo_id}: saldo insuficiente en cuenta {primera}")
                    conn.rollback()
                    continue

                # Pequeña pausa para aumentar probabilidad de deadlock
                time.sleep(0.05)

                # Bloquear segunda cuenta (aquí puede ocurrir el deadlock)
                cur.execute(
                    "UPDATE cuentas SET saldo = saldo + %s, version = version + 1 "
                    "WHERE id = %s",
                    (monto, segunda)
                )

                conn.commit()
                log.info(f"Hilo-{hilo_id} iter-{i+1}: transferencia OK ({primera}→{segunda}, ${monto})")

                with contador_lock:
                    stats['transferencias_ok'] += 1

                # Registrar éxito en auditoría
                _registrar_auditoria(hilo_id, f"Transferencia {primera}→{segunda}", "OK")

            except psycopg2.errors.DeadlockDetected as e:
                conn.rollback()
                log.error(f"Hilo-{hilo_id} iter-{i+1}: DEADLOCK DETECTADO — {e.pgerror.strip()}")
                with contador_lock:
                    stats['deadlocks_detectados'] += 1
                _registrar_auditoria(hilo_id, f"Transferencia {primera}→{segunda}", "DEADLOCK")
                time.sleep(0.1)  # Backoff antes de reintentar

            except psycopg2.errors.LockNotAvailable as e:
                conn.rollback()
                log.warning(f"Hilo-{hilo_id} iter-{i+1}: LOCK TIMEOUT — {e.pgerror.strip()}")
                with contador_lock:
                    stats['otros_errores'] += 1
                _registrar_auditoria(hilo_id, f"Transferencia {primera}→{segunda}", "LOCK_TIMEOUT")

            except Exception as e:
                conn.rollback()
                log.error(f"Hilo-{hilo_id} iter-{i+1}: ERROR INESPERADO — {e}")
                with contador_lock:
                    stats['otros_errores'] += 1

    finally:
        if conn:
            conn.close()


def _registrar_auditoria(hilo_id: int, accion: str, resultado: str) -> None:
    """Registra eventos en la tabla de auditoría (conexión independiente)."""
    try:
        with psycopg2.connect(**DB_CONFIG) as conn:
            conn.autocommit = True
            with conn.cursor() as cur:
                cur.execute(
                    "INSERT INTO auditoria_deadlocks (pid, hilo_nombre, accion, resultado) "
                    "VALUES (%s, %s, %s, %s)",
                    (os.getpid(), f"Hilo-{hilo_id}", accion, resultado)
                )
    except Exception:
        pass  # No interrumpir el flujo principal por fallo de auditoría


def main():
    NUM_HILOS     = 6
    ITERACIONES   = 8
    MONTO         = 100.00

    # Pares de cuentas para transferencias cruzadas (garantizan deadlock)
    pares_cuentas = [
        (1, 2),  # Hilos 0,1 → Alice ↔ Bob
        (3, 4),  # Hilos 2,3 → Carlos ↔ Diana
        (1, 3),  # Hilos 4,5 → Alice ↔ Carlos
    ]

    log.info("=" * 60)
    log.info("INICIO SIMULACIÓN DE DEADLOCKS")
    log.info(f"  Hilos: {NUM_HILOS} | Iteraciones: {ITERACIONES} | Monto: ${MONTO}")
    log.info("=" * 60)

    inicio = time.time()
    hilos = []

    for i in range(NUM_HILOS):
        origen, destino = pares_cuentas[i % len(pares_cuentas)]
        t = threading.Thread(
            target=transferencia_orden_inverso,
            args=(i, origen, destino, MONTO, ITERACIONES),
            name=f"Hilo-{i:02d}",
            daemon=True
        )
        hilos.append(t)

    # Lanzar todos los hilos simultáneamente
    for t in hilos:
        t.start()

    for t in hilos:
        t.join(timeout=120)

    duracion = time.time() - inicio

    log.info("=" * 60)
    log.info("RESULTADOS FINALES")
    log.info(f"  Duración total:          {duracion:.2f}s")
    log.info(f"  Total intentos:          {stats['total_intentos']}")
    log.info(f"  Transferencias exitosas: {stats['transferencias_ok']}")
    log.info(f"  Deadlocks detectados:    {stats['deadlocks_detectados']}")
    log.info(f"  Otros errores:           {stats['otros_errores']}")
    log.info(f"  Tasa de deadlock:        "
             f"{stats['deadlocks_detectados']/max(stats['total_intentos'],1)*100:.1f}%")
    log.info("=" * 60)

    # Guardar resultados en JSON
    import json
    results = {**stats, 'duracion_segundos': duracion, 'timestamp': datetime.now().isoformat()}
    with open('/root/lab-02-00-02/results/deadlock_stats.json', 'w') as f:
        json.dump(results, f, indent=2)
    log.info("Resultados guardados en results/deadlock_stats.json")


if __name__ == '__main__':
    main()
PYEOF

chmod +x ~/lab-02-00-02/scripts/simular_deadlocks.py
```

2. Ejecuta el script y observa la salida en tiempo real:

```bash
cd ~/lab-02-00-02
python3 scripts/simular_deadlocks.py 2>&1 | tee logs/ejecucion_deadlocks.log
```

3. Mientras el script se ejecuta (en otra terminal), monitorea los bloqueos activos:

```sql
-- Monitor de bloqueos en tiempo real (ejecutar repetidamente o en bucle)
SELECT
    bl.pid                      AS pid_bloqueado,
    ba.application_name         AS app_bloqueado,
    ka.pid                      AS pid_bloqueador,
    now() - ba.xact_start       AS duracion_tx,
    bl.mode                     AS modo_solicitado,
    ba.query                    AS query_bloqueada,
    ka.query                    AS query_bloqueadora
FROM pg_locks bl
JOIN pg_stat_activity ba ON bl.pid = ba.pid
JOIN pg_locks kl
    ON bl.locktype = kl.locktype
   AND bl.database IS NOT DISTINCT FROM kl.database
   AND bl.relation IS NOT DISTINCT FROM kl.relation
   AND bl.page IS NOT DISTINCT FROM kl.page
   AND bl.tuple IS NOT DISTINCT FROM kl.tuple
   AND bl.classid IS NOT DISTINCT FROM kl.classid
   AND bl.objid IS NOT DISTINCT FROM kl.objid
   AND bl.objsubid IS NOT DISTINCT FROM kl.objsubid
   AND bl.transactionid IS NOT DISTINCT FROM kl.transactionid
   AND bl.pid <> kl.pid
JOIN pg_stat_activity ka ON kl.pid = ka.pid
WHERE NOT bl.granted
ORDER BY duracion_tx DESC NULLS LAST;
```

#### Salida Esperada del Script

```
2024-01-15 10:23:45,123 [Hilo-00     ] INFO     INICIO SIMULACIÓN DE DEADLOCKS
2024-01-15 10:23:45,124 [Hilo-00     ] INFO       Hilos: 6 | Iteraciones: 8 | Monto: $100.0
2024-01-15 10:23:45,891 [Hilo-01     ] ERROR    Hilo-1 iter-2: DEADLOCK DETECTADO — ERROR:  deadlock detected
...
RESULTADOS FINALES
  Duración total:          18.43s
  Total intentos:          48
  Transferencias exitosas: 31
  Deadlocks detectados:    12
  Otros errores:            5
  Tasa de deadlock:        25.0%
```

#### Verificación

```sql
-- Revisar la tabla de auditoría
SELECT resultado, COUNT(*) AS cantidad
FROM auditoria_deadlocks
GROUP BY resultado
ORDER BY cantidad DESC;

-- Ver los últimos 10 eventos de deadlock
SELECT ts, hilo_nombre, accion, resultado
FROM auditoria_deadlocks
WHERE resultado = 'DEADLOCK'
ORDER BY ts DESC
LIMIT 10;
```

---

### Paso 5: Analizar Deadlocks en los Logs de PostgreSQL

**Objetivo:** Extraer y analizar los mensajes `deadlock detected` de los logs de Aurora para identificar las consultas y procesos involucrados.

#### Instrucciones

1. Descarga y analiza los logs de PostgreSQL desde Aurora:

```bash
# Listar archivos de log disponibles
aws rds describe-db-log-files \
    --db-instance-identifier "${LAB_CLUSTER_ID}-instance-1" \
    --region $AWS_REGION \
    --query 'DescribeDBLogFiles[*].{Archivo:LogFileName,Tamaño:Size,UltimaEscritura:LastWritten}' \
    --output table

# Descargar el log de errores más reciente
aws rds download-db-log-file-portion \
    --db-instance-identifier "${LAB_CLUSTER_ID}-instance-1" \
    --log-file-name "error/postgresql.log" \
    --region $AWS_REGION \
    --output text > ~/lab-02-00-02/logs/aurora_postgresql.log

# Extraer solo los eventos de deadlock
grep -A 10 "deadlock detected" ~/lab-02-00-02/logs/aurora_postgresql.log \
    > ~/lab-02-00-02/logs/deadlocks_extraidos.log

echo "Eventos de deadlock encontrados:"
grep -c "deadlock detected" ~/lab-02-00-02/logs/aurora_postgresql.log
```

2. Analiza el patrón de los mensajes de deadlock:

```bash
# Ver los primeros 3 deadlocks completos con contexto
grep -A 8 "deadlock detected" ~/lab-02-00-02/logs/aurora_postgresql.log | head -60
```

**Estructura típica del mensaje de deadlock:**
```
2024-01-15 10:23:46 UTC [18423]: labuser@labdb ERROR:  deadlock detected
2024-01-15 10:23:46 UTC [18423]: labuser@labdb DETAIL:  Process 18423 waits for ShareLock on transaction 5621; blocked by process 18456.
        Process 18456 waits for ShareLock on transaction 5618; blocked by process 18423.
2024-01-15 10:23:46 UTC [18423]: labuser@labdb HINT:  See server log for query details.
2024-01-15 10:23:46 UTC [18423]: labuser@labdb CONTEXT:  while updating tuple (0,1) in relation "cuentas"
2024-01-15 10:23:46 UTC [18423]: labuser@labdb STATEMENT:  UPDATE cuentas SET saldo = saldo + $1, version = version + 1 WHERE id = $2
```

3. Consulta el historial de bloqueos post-ejecución:

```sql
-- Análisis de pg_locks post-simulación
SELECT
    locktype,
    relation::regclass AS tabla,
    mode,
    granted,
    COUNT(*) AS cantidad
FROM pg_locks
WHERE relation IS NOT NULL
GROUP BY locktype, relation, mode, granted
ORDER BY granted, cantidad DESC;
```

#### Salida Esperada del Análisis de Logs

```
Eventos de deadlock encontrados:
12

2024-01-15 10:23:46 UTC [18423]: labuser@labdb ERROR:  deadlock detected
DETAIL:  Process 18423 waits for ShareLock on transaction 5621; blocked by process 18456.
         Process 18456 waits for ShareLock on transaction 5618; blocked by process 18423.
CONTEXT:  while updating tuple (0,1) in relation "cuentas"
```

#### Verificación

```bash
# Contar y clasificar errores en el log
echo "=== Resumen de errores en logs ==="
echo "Deadlocks: $(grep -c 'deadlock detected' ~/lab-02-00-02/logs/aurora_postgresql.log)"
echo "Lock timeouts: $(grep -c 'lock timeout' ~/lab-02-00-02/logs/aurora_postgresql.log)"
echo "Cancels: $(grep -c 'canceling statement' ~/lab-02-00-02/logs/aurora_postgresql.log)"
```

---

### Paso 6: Implementar Estrategias de Prevención de Deadlocks

**Objetivo:** Modificar el script de transferencias para eliminar deadlocks mediante ordenamiento consistente de recursos y comparar la tasa de éxito.

#### Instrucciones

1. Crea el script con estrategias de mitigación:

```bash
cat > ~/lab-02-00-02/scripts/transferencias_seguras.py << 'PYEOF'
#!/usr/bin/env python3
"""
Lab 02-00-02 - Estrategias de Prevención de Deadlocks
Demuestra tres estrategias: ordenamiento, NOWAIT y SKIP LOCKED
"""

import psycopg2
import psycopg2.extras
import threading
import time
import logging
import os
from datetime import datetime

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(threadName)-12s] %(levelname)-8s %(message)s'
)
log = logging.getLogger(__name__)

DB_CONFIG = {
    'host':     os.environ.get('LAB_DB_HOST', 'localhost'),
    'port':     int(os.environ.get('LAB_DB_PORT', 5432)),
    'dbname':   os.environ.get('LAB_DB_NAME', 'labdb'),
    'user':     os.environ.get('LAB_DB_USER', 'labuser'),
    'password': os.environ.get('LAB_DB_PASSWORD', ''),
    'sslmode':  'require',
    'connect_timeout': 10,
}

stats_lock = threading.Lock()


# ── ESTRATEGIA 1: Ordenamiento Consistente ────────────────────────────────────
def transferencia_ordenada(hilo_id: int, cuenta_a: int, cuenta_b: int,
                            monto: float, iteraciones: int = 5) -> dict:
    """
    PREVENCIÓN: Siempre bloquear cuentas en orden ascendente de ID.
    Esto rompe el ciclo de dependencia circular que causa deadlocks.
    """
    resultados = {'ok': 0, 'deadlock': 0, 'error': 0}
    conn = None
    try:
        conn = psycopg2.connect(**DB_CONFIG)
        conn.autocommit = False

        for i in range(iteraciones):
            try:
                # CLAVE: ordenar siempre de menor a mayor ID
                primera  = min(cuenta_a, cuenta_b)
                segunda  = max(cuenta_a, cuenta_b)
                origen   = cuenta_a
                destino  = cuenta_b

                cur = conn.cursor()
                log.info(f"[ORDENADO] Hilo-{hilo_id}: bloqueando {primera} → {segunda}")

                # Bloquear en orden consistente (SELECT FOR UPDATE)
                cur.execute(
                    "SELECT id, saldo FROM cuentas WHERE id IN (%s, %s) "
                    "ORDER BY id FOR UPDATE",
                    (primera, segunda)
                )
                filas = {row[0]: row[1] for row in cur.fetchall()}

                if filas.get(origen, 0) < monto:
                    conn.rollback()
                    log.warning(f"[ORDENADO] Hilo-{hilo_id}: saldo insuficiente")
                    continue

                cur.execute("UPDATE cuentas SET saldo = saldo - %s, version = version + 1 WHERE id = %s",
                            (monto, origen))
                cur.execute("UPDATE cuentas SET saldo = saldo + %s, version = version + 1 WHERE id = %s",
                            (monto, destino))
                conn.commit()
                log.info(f"[ORDENADO] Hilo-{hilo_id} iter-{i+1}: OK")
                resultados['ok'] += 1

            except psycopg2.errors.DeadlockDetected:
                conn.rollback()
                log.error(f"[ORDENADO] Hilo-{hilo_id}: DEADLOCK (no debería ocurrir!)")
                resultados['deadlock'] += 1
            except Exception as e:
                conn.rollback()
                log.error(f"[ORDENADO] Hilo-{hilo_id}: ERROR — {e}")
                resultados['error'] += 1
    finally:
        if conn:
            conn.close()
    return resultados


# ── ESTRATEGIA 2: SELECT FOR UPDATE NOWAIT ────────────────────────────────────
def transferencia_nowait(hilo_id: int, cuenta_a: int, cuenta_b: int,
                          monto: float, iteraciones: int = 5) -> dict:
    """
    PREVENCIÓN: Usar NOWAIT para fallar inmediatamente si no se puede obtener el lock.
    Evita esperas largas y permite reintentos con backoff.
    """
    resultados = {'ok': 0, 'lock_no_disp': 0, 'reintento': 0, 'error': 0}
    conn = None
    try:
        conn = psycopg2.connect(**DB_CONFIG)
        conn.autocommit = False

        for i in range(iteraciones):
            max_reintentos = 3
            for intento in range(max_reintentos):
                try:
                    cur = conn.cursor()
                    # NOWAIT: falla inmediatamente si el lock no está disponible
                    cur.execute(
                        "SELECT id, saldo FROM cuentas WHERE id IN (%s, %s) "
                        "ORDER BY id FOR UPDATE NOWAIT",
                        (cuenta_a, cuenta_b)
                    )
                    filas = {row[0]: row[1] for row in cur.fetchall()}

                    if filas.get(cuenta_a, 0) < monto:
                        conn.rollback()
                        break

                    cur.execute("UPDATE cuentas SET saldo = saldo - %s WHERE id = %s", (monto, cuenta_a))
                    cur.execute("UPDATE cuentas SET saldo = saldo + %s WHERE id = %s", (monto, cuenta_b))
                    conn.commit()
                    log.info(f"[NOWAIT] Hilo-{hilo_id} iter-{i+1} intento-{intento+1}: OK")
                    resultados['ok'] += 1
                    break

                except psycopg2.errors.LockNotAvailable:
                    conn.rollback()
                    resultados['lock_no_disp'] += 1
                    backoff = 0.05 * (2 ** intento)
                    log.warning(f"[NOWAIT] Hilo-{hilo_id}: lock no disponible, reintento en {backoff:.2f}s")
                    time.sleep(backoff)
                    if intento == max_reintentos - 1:
                        resultados['reintento'] += 1
                except Exception as e:
                    conn.rollback()
                    log.error(f"[NOWAIT] Hilo-{hilo_id}: ERROR — {e}")
                    resultados['error'] += 1
                    break
    finally:
        if conn:
            conn.close()
    return resultados


# ── ESTRATEGIA 3: Cola con SKIP LOCKED ───────────────────────────────────────
def consumidor_skip_locked(hilo_id: int, iteraciones: int = 5) -> dict:
    """
    PATRÓN DE COLA: Usar SKIP LOCKED para que múltiples consumidores
    procesen trabajos sin colisionar entre sí.
    """
    resultados = {'trabajos_procesados': 0, 'sin_trabajo': 0, 'error': 0}
    conn = None
    try:
        conn = psycopg2.connect(**DB_CONFIG)
        conn.autocommit = False

        for i in range(iteraciones):
            try:
                cur = conn.cursor()
                # SKIP LOCKED: toma trabajos disponibles, omite los bloqueados por otros
                cur.execute("""
                    SELECT id, descripcion, prioridad
                    FROM trabajos
                    WHERE estado = 'PENDIENTE'
                    ORDER BY prioridad DESC, id
                    FOR UPDATE SKIP LOCKED
                    LIMIT 3
                """)
                trabajos = cur.fetchall()

                if not trabajos:
                    conn.rollback()
                    resultados['sin_trabajo'] += 1
                    log.info(f"[SKIP LOCKED] Hilo-{hilo_id}: sin trabajos disponibles")
                    time.sleep(0.1)
                    continue

                ids = [t[0] for t in trabajos]
                cur.execute(
                    "UPDATE trabajos SET estado = 'COMPLETADO' WHERE id = ANY(%s)",
                    (ids,)
                )
                conn.commit()
                log.info(f"[SKIP LOCKED] Hilo-{hilo_id} iter-{i+1}: procesados trabajos {ids}")
                resultados['trabajos_procesados'] += len(ids)

            except Exception as e:
                conn.rollback()
                log.error(f"[SKIP LOCKED] Hilo-{hilo_id}: ERROR — {e}")
                resultados['error'] += 1
    finally:
        if conn:
            conn.close()
    return resultados


def ejecutar_estrategia(nombre: str, func, args_list: list) -> None:
    log.info(f"\n{'='*60}")
    log.info(f"ESTRATEGIA: {nombre}")
    log.info('='*60)

    inicio = time.time()
    hilos = []
    resultados_todos = []
    r_lock = threading.Lock()

    def wrapper(func, args, resultados_todos, r_lock):
        r = func(*args)
        with r_lock:
            resultados_todos.append(r)

    for args in args_list:
        t = threading.Thread(target=wrapper, args=(func, args, resultados_todos, r_lock))
        hilos.append(t)

    for t in hilos:
        t.start()
    for t in hilos:
        t.join(timeout=60)

    duracion = time.time() - inicio
    log.info(f"\nResultados '{nombre}' ({duracion:.2f}s):")
    for k in resultados_todos[0].keys() if resultados_todos else []:
        total = sum(r.get(k, 0) for r in resultados_todos)
        log.info(f"  {k:30s}: {total}")


def main():
    # Resetear trabajos antes de la prueba de SKIP LOCKED
    with psycopg2.connect(**DB_CONFIG) as conn:
        conn.autocommit = True
        with conn.cursor() as cur:
            cur.execute("UPDATE trabajos SET estado = 'PENDIENTE' WHERE TRUE")
            log.info("Trabajos reseteados a PENDIENTE")

    # Estrategia 1: Ordenamiento consistente (6 hilos, 3 pares de cuentas)
    args_ordenado = [
        (i, [1,2,3,4,1,3][i], [2,1,4,3,3,1][i], 50.0, 6)
        for i in range(6)
    ]
    ejecutar_estrategia("ORDENAMIENTO CONSISTENTE", transferencia_ordenada, args_ordenado)

    # Estrategia 2: NOWAIT con reintentos (4 hilos)
    args_nowait = [
        (i, [1,2,1,3][i], [2,1,3,1][i], 50.0, 6)
        for i in range(4)
    ]
    ejecutar_estrategia("SELECT FOR UPDATE NOWAIT", transferencia_nowait, args_nowait)

    # Estrategia 3: Cola con SKIP LOCKED (5 consumidores)
    args_skip = [(i, 8) for i in range(5)]
    ejecutar_estrategia("COLA CON SKIP LOCKED", consumidor_skip_locked, args_skip)


if __name__ == '__main__':
    main()
PYEOF

chmod +x ~/lab-02-00-02/scripts/transferencias_seguras.py
```

2. Ejecuta el script de estrategias seguras:

```bash
python3 ~/lab-02-00-02/scripts/transferencias_seguras.py 2>&1 | tee ~/lab-02-00-02/logs/estrategias_seguras.log
```

#### Salida Esperada

```
============================================================
ESTRATEGIA: ORDENAMIENTO CONSISTENTE
============================================================
Resultados 'ORDENAMIENTO CONSISTENTE' (4.21s):
  ok                            : 35
  deadlock                      : 0      ← Sin deadlocks con ordenamiento
  error                         : 1

ESTRATEGIA: SELECT FOR UPDATE NOWAIT
  ok                            : 22
  lock_no_disp                  : 6
  reintento                     : 2
  error                         : 0

ESTRATEGIA: COLA CON SKIP LOCKED
  trabajos_procesados           : 50
  sin_trabajo                   : 3
  error                         : 0
```

#### Verificación

```sql
-- Confirmar que no hubo deadlocks con ordenamiento consistente
SELECT resultado, COUNT(*) FROM auditoria_deadlocks
WHERE ts > NOW() - INTERVAL '5 minutes'
GROUP BY resultado;

-- Verificar estado de trabajos procesados
SELECT estado, COUNT(*) FROM trabajos GROUP BY estado;
```

---

### Paso 7: Comparar Niveles de Aislamiento

**Objetivo:** Observar el comportamiento del sistema bajo `READ COMMITTED`, `REPEATABLE READ` y `SERIALIZABLE` en escenarios de lectura y escritura concurrentes.

#### Instrucciones

1. Ejecuta las siguientes pruebas en dos sesiones `psql` simultáneas para observar las diferencias:

**Prueba A — READ COMMITTED (comportamiento por defecto):**

```sql
-- Sesión 1: READ COMMITTED (default)
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
SELECT saldo FROM cuentas WHERE id = 1;
-- Anotar saldo inicial: ej. 9450.00

-- (Sesión 2 ejecuta UPDATE y hace COMMIT aquí)

-- Sesión 1 vuelve a leer: verá el nuevo valor (lectura no repetible)
SELECT saldo FROM cuentas WHERE id = 1;
COMMIT;
```

```sql
-- Sesión 2 (ejecutar mientras Sesión 1 está abierta):
BEGIN;
UPDATE cuentas SET saldo = saldo + 1000 WHERE id = 1;
COMMIT;
```

**Prueba B — REPEATABLE READ:**

```sql
-- Sesión 1: REPEATABLE READ
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT saldo FROM cuentas WHERE id = 1;
-- Anotar saldo: ej. 10450.00

-- (Sesión 2 ejecuta UPDATE y hace COMMIT aquí)

-- Sesión 1 vuelve a leer: verá el MISMO valor (snapshot fijo)
SELECT saldo FROM cuentas WHERE id = 1;
COMMIT;
```

**Prueba C — SERIALIZABLE (con conflicto de escritura):**

```sql
-- Sesión 1: SERIALIZABLE
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT SUM(saldo) AS total FROM cuentas;

-- (Sesión 2 hace INSERT de nueva cuenta aquí)

-- Sesión 1 intenta hacer UPDATE basado en la suma anterior
UPDATE cuentas SET saldo = saldo * 1.01 WHERE TRUE;
COMMIT;
-- Puede fallar con: ERROR: could not serialize access due to concurrent update
```

2. Script Python para demostrar el error de serialización:

```bash
cat > ~/lab-02-00-02/scripts/test_isolation.py << 'PYEOF'
#!/usr/bin/env python3
"""Demostración de niveles de aislamiento y sus efectos."""

import psycopg2
import threading
import time
import os
import logging

logging.basicConfig(level=logging.INFO,
                    format='%(asctime)s [%(threadName)-10s] %(message)s')
log = logging.getLogger(__name__)

DB_CONFIG = {
    'host': os.environ.get('LAB_DB_HOST', 'localhost'),
    'port': int(os.environ.get('LAB_DB_PORT', 5432)),
    'dbname': os.environ.get('LAB_DB_NAME', 'labdb'),
    'user': os.environ.get('LAB_DB_USER', 'labuser'),
    'password': os.environ.get('LAB_DB_PASSWORD', ''),
    'sslmode': 'require',
}

def test_isolation_level(nivel: str, cuenta_id: int = 1) -> None:
    """Prueba si el nivel de aislamiento ve cambios de transacciones concurrentes."""
    barrier = threading.Barrier(2)
    resultados = {}

    def lector():
        with psycopg2.connect(**DB_CONFIG) as conn:
            conn.autocommit = False
            with conn.cursor() as cur:
                cur.execute(f"SET TRANSACTION ISOLATION LEVEL {nivel}")
                cur.execute("SELECT saldo FROM cuentas WHERE id = %s", (cuenta_id,))
                saldo_antes = cur.fetchone()[0]
                resultados['saldo_antes'] = float(saldo_antes)
                log.info(f"[{nivel}] Lector: saldo inicial = {saldo_antes}")

                barrier.wait()  # Sincronizar: esperar a que escritor haga COMMIT
                time.sleep(0.3)

                cur.execute("SELECT saldo FROM cuentas WHERE id = %s", (cuenta_id,))
                saldo_despues = cur.fetchone()[0]
                resultados['saldo_despues'] = float(saldo_despues)
                log.info(f"[{nivel}] Lector: saldo después del commit externo = {saldo_despues}")

                if saldo_antes == saldo_despues:
                    log.info(f"[{nivel}] → SNAPSHOT FIJO: no vio el cambio externo ✓")
                else:
                    log.info(f"[{nivel}] → VIO EL CAMBIO: lectura no repetible (diferencia: {saldo_despues - saldo_antes:+.2f})")
            conn.rollback()

    def escritor():
        barrier.wait()  # Esperar a que lector haya hecho su primera lectura
        with psycopg2.connect(**DB_CONFIG) as conn:
            conn.autocommit = False
            with conn.cursor() as cur:
                cur.execute("UPDATE cuentas SET saldo = saldo + 999 WHERE id = %s", (cuenta_id,))
            conn.commit()
            log.info(f"[{nivel}] Escritor: COMMIT de +999 en cuenta {cuenta_id}")
        # Revertir para no afectar otras pruebas
        with psycopg2.connect(**DB_CONFIG) as conn:
            conn.autocommit = False
            with conn.cursor() as cur:
                cur.execute("UPDATE cuentas SET saldo = saldo - 999 WHERE id = %s", (cuenta_id,))
            conn.commit()

    t_lector  = threading.Thread(target=lector,  name="Lector")
    t_escritor = threading.Thread(target=escritor, name="Escritor")
    t_lector.start(); t_escritor.start()
    t_lector.join(); t_escritor.join()
    return resultados

print("\n" + "="*60)
print("COMPARACIÓN DE NIVELES DE AISLAMIENTO")
print("="*60)

for nivel in ["READ COMMITTED", "REPEATABLE READ"]:
    print(f"\n--- Probando: {nivel} ---")
    test_isolation_level(nivel)
    time.sleep(1)
PYEOF

python3 ~/lab-02-00-02/scripts/test_isolation.py
```

#### Tabla Resumen de Comportamientos

| Fenómeno                    | READ COMMITTED | REPEATABLE READ | SERIALIZABLE |
|-----------------------------|:--------------:|:---------------:|:------------:|
| Lectura sucia               | ✗ No           | ✗ No            | ✗ No         |
| Lectura no repetible        | ✓ Posible      | ✗ No            | ✗ No         |
| Lectura fantasma            | ✓ Posible      | ✗ No*           | ✗ No         |
| Anomalía de serialización   | ✓ Posible      | ✓ Posible       | ✗ No         |
| Deadlocks posibles          | ✓ Sí           | ✓ Sí            | ✓ (+ errores de serialización) |

*Aurora PostgreSQL usa MVCC que previene lecturas fantasma en REPEATABLE READ.

#### Verificación

```sql
-- Confirmar nivel de aislamiento por defecto del cluster
SHOW default_transaction_isolation;
-- Debe mostrar: read committed
```

---

## Validación y Pruebas

Ejecuta las siguientes verificaciones para confirmar que los objetivos del laboratorio se han cumplido:

```sql
-- ── VALIDACIÓN 1: Confirmar que se generaron deadlocks durante la simulación ──
SELECT
    resultado,
    COUNT(*) AS total,
    MIN(ts) AS primer_evento,
    MAX(ts) AS ultimo_evento
FROM auditoria_deadlocks
GROUP BY resultado
ORDER BY total DESC;
-- Debe mostrar al menos una fila con resultado='DEADLOCK'

-- ── VALIDACIÓN 2: Integridad de saldos (suma debe ser constante) ──────────────
SELECT
    SUM(saldo) AS suma_total_saldos,
    COUNT(*) AS num_cuentas
FROM cuentas;
-- La suma debe ser igual a la suma inicial (10000+8000+5000+12000+3000 = 38000)
-- Si hay diferencia, alguna transacción no hizo ROLLBACK correctamente

-- ── VALIDACIÓN 3: Verificar configuración de deadlock_timeout ─────────────────
SHOW deadlock_timeout;
-- Debe mostrar: 500ms

-- ── VALIDACIÓN 4: Estado de los trabajos procesados ───────────────────────────
SELECT estado, COUNT(*) FROM trabajos GROUP BY estado ORDER BY estado;
-- Debe mostrar trabajos en estado COMPLETADO (de la prueba SKIP LOCKED)

-- ── VALIDACIÓN 5: No hay bloqueos activos al finalizar ────────────────────────
SELECT COUNT(*) AS bloqueos_pendientes
FROM pg_locks WHERE NOT granted;
-- Debe retornar 0

-- ── VALIDACIÓN 6: Confirmar que los logs muestran deadlock_timeout correcto ───
SELECT name, setting, unit
FROM pg_settings
WHERE name IN ('deadlock_timeout', 'lock_timeout', 'statement_timeout');
```

```bash
# ── VALIDACIÓN 7: Verificar que los archivos de resultados existen ─────────────
ls -la ~/lab-02-00-02/results/deadlock_stats.json
cat ~/lab-02-00-02/results/deadlock_stats.json | jq '.deadlocks_detectados'
# Debe mostrar un número > 0

# ── VALIDACIÓN 8: Comparar tasa de deadlock entre scripts ─────────────────────
echo "=== Comparación de resultados ==="
echo "Script con deadlocks (orden inverso):"
cat ~/lab-02-00-02/results/deadlock_stats.json | jq '{
    intentos: .total_intentos,
    exitosas: .transferencias_ok,
    deadlocks: .deadlocks_detectados,
    tasa_deadlock: ((.deadlocks_detectados / .total_intentos * 100) | round)
}'
```

---

## Resolución de Problemas

### Problema 1: El script Python no genera deadlocks (tasa de deadlock = 0%)

**Síntoma:** El script `simular_deadlocks.py` completa todas las iteraciones sin registrar ningún `DEADLOCK DETECTADO` en los logs. La tabla `auditoria_deadlocks` no tiene filas con `resultado='DEADLOCK'`.

**Causa:** La pausa `time.sleep(0.05)` entre los dos `UPDATE` puede ser demasiado corta o el sistema es tan rápido que las transacciones se completan antes de que los hilos se crucen. También puede ocurrir si `deadlock_timeout` es demasiado alto y PostgreSQL resuelve el bloqueo antes de detectarlo como deadlock, o si `lock_timeout` cancela las transacciones antes de que se forme el ciclo.

**Solución:**

```python
# Aumentar la pausa entre los dos UPDATE para dar tiempo al cruce de hilos
# En simular_deadlocks.py, cambiar:
time.sleep(0.05)
# Por:
time.sleep(0.15)   # Mayor ventana de oportunidad para el deadlock

# También reducir el número de iteraciones y aumentar hilos:
NUM_HILOS   = 8    # Más contención
ITERACIONES = 10
```

```bash
# Verificar que deadlock_timeout no sea excesivamente alto
psql "host=$LAB_DB_HOST dbname=$LAB_DB_NAME user=$LAB_DB_USER password=$LAB_DB_PASSWORD" \
     -c "SHOW deadlock_timeout;"
# Debe ser 500ms (configurado en Paso 3)

# Si lock_timeout está activo y muy bajo, puede estar cancelando antes del deadlock
psql "host=$LAB_DB_HOST dbname=$LAB_DB_NAME user=$LAB_DB_USER password=$LAB_DB_PASSWORD" \
     -c "SHOW lock_timeout;"
# Si muestra un valor muy bajo (ej. 500ms), aumentarlo en DB_CONFIG:
# 'options': '-c lock_timeout=5000'   # 5 segundos
```

---

### Problema 2: Error de conexión SSL o `could not connect to server`

**Síntoma:** El script Python falla con `psycopg2.OperationalError: could not connect to server: Connection refused` o `SSL connection has been closed unexpectedly`. Todos los hilos fallan inmediatamente.

**Causa:** Las variables de entorno `LAB_DB_HOST`, `LAB_DB_USER` o `LAB_DB_PASSWORD` no están correctamente definidas, el Security Group de Aurora no permite tráfico en el puerto 5432 desde la IP del cliente, o el certificado SSL de Aurora no está siendo aceptado.

**Solución:**

```bash
# 1. Verificar que las variables de entorno están definidas
echo "Host:     $LAB_DB_HOST"
echo "Puerto:   $LAB_DB_PORT"
echo "DB:       $LAB_DB_NAME"
echo "Usuario:  $LAB_DB_USER"
# Si alguna está vacía, redefinirla con los valores correctos del laboratorio

# 2. Probar conectividad de red al puerto 5432
nc -zv $LAB_DB_HOST 5432
# Si falla, revisar el Security Group en la consola AWS:
# VPC → Security Groups → Inbound Rules → debe permitir TCP/5432 desde tu IP

# 3. Probar conexión manual con psql (más descriptivo en errores)
psql "host=$LAB_DB_HOST port=$LAB_DB_PORT dbname=$LAB_DB_NAME \
      user=$LAB_DB_USER password=$LAB_DB_PASSWORD sslmode=require" \
     -c "SELECT 1 AS ok;"

# 4. Si el problema es SSL, probar con sslmode=prefer (menos estricto)
# En DB_CONFIG del script, cambiar temporalmente:
# 'sslmode': 'prefer'

# 5. Verificar que la instancia Aurora está disponible
aws rds describe-db-instances \
    --db-instance-identifier "${LAB_CLUSTER_ID}-instance-1" \
    --region $AWS_REGION \
    --query 'DBInstances[0].DBInstanceStatus' \
    --output text
# Debe mostrar: available
```

---

## Limpieza de Recursos

Ejecuta los siguientes pasos al finalizar el laboratorio para limpiar los recursos y controlar costos:

```sql
-- 1. Limpiar tablas de prueba (mantener estructura para laboratorios futuros)
TRUNCATE TABLE auditoria_deadlocks RESTART IDENTITY;
UPDATE trabajos SET estado = 'PENDIENTE' WHERE TRUE;

-- Restaurar saldos originales si hubo diferencias
UPDATE cuentas SET saldo = CASE id
    WHEN 1 THEN 10000.00
    WHEN 2 THEN  8000.00
    WHEN 3 THEN  5000.00
    WHEN 4 THEN 12000.00
    WHEN 5 THEN  3000.00
END,
version = 0
WHERE id IN (1,2,3,4,5);

-- Verificar restauración
SELECT id, titular, saldo FROM cuentas ORDER BY id;
```

```bash
# 2. Restaurar deadlock_timeout al valor por defecto (1000ms = 1s)
aws rds modify-db-cluster-parameter-group \
    --db-cluster-parameter-group-name $LAB_PARAM_GROUP \
    --parameters "ParameterName=deadlock_timeout,ParameterValue=1000,ApplyMethod=immediate" \
    --region $AWS_REGION

# Verificar restauración
psql "host=$LAB_DB_HOST dbname=$LAB_DB_NAME user=$LAB_DB_USER password=$LAB_DB_PASSWORD" \
     -c "SHOW deadlock_timeout;"
# Debe mostrar: 1s

# 3. Archivar logs y resultados del laboratorio
cd ~/lab-02-00-02
tar -czf ../lab-02-00-02-resultados-$(date +%Y%m%d).tar.gz logs/ results/
echo "Archivado: ../lab-02-00-02-resultados-$(date +%Y%m%d).tar.gz"

# 4. Limpiar archivos temporales de logs locales (opcional)
# rm -f logs/*.log   # Descomenta si no necesitas conservar los logs

# 5. Si este es el último laboratorio del módulo, destruir infraestructura Terraform
# cd ~/terraform/modulo-02
# terraform destroy -auto-approve
# ADVERTENCIA: Solo ejecutar si has terminado TODOS los labs del módulo 2
```

---

## Resumen

En este laboratorio aplicaste de forma práctica los conceptos de concurrencia y bloqueos en Aurora PostgreSQL:

| Concepto Practicado                        | Técnica Aplicada                                              |
|--------------------------------------------|---------------------------------------------------------------|
| Reproducción de deadlocks                  | Acceso a filas en orden inverso con múltiples hilos Python    |
| Análisis de mensajes de error              | Logs de Aurora + `pg_locks` + `pg_stat_activity`             |
| Prevención por ordenamiento                | `SELECT ... ORDER BY id FOR UPDATE` (orden ascendente fijo)   |
| Fallo rápido sin espera                    | `SELECT FOR UPDATE NOWAIT` + reintentos con backoff           |
| Cola de trabajo sin colisiones             | `SELECT FOR UPDATE SKIP LOCKED`                               |
| Comparación de niveles de aislamiento      | READ COMMITTED vs REPEATABLE READ vs SERIALIZABLE             |
| Configuración de parámetros Aurora         | `deadlock_timeout` en Cluster Parameter Group                 |

### Puntos Clave

- Un **deadlock** ocurre cuando dos o más transacciones esperan mutuamente por recursos que la otra retiene; PostgreSQL lo detecta automáticamente y cancela una de las transacciones víctima.
- La estrategia más efectiva para prevenir deadlocks es el **ordenamiento consistente de recursos**: siempre acceder a filas/tablas en el mismo orden en todas las transacciones.
- `SELECT FOR UPDATE NOWAIT` permite detectar contención inmediatamente y aplicar lógica de reintento en la aplicación, evitando esperas largas.
- `SKIP LOCKED` es ideal para patrones de cola de trabajo donde múltiples consumidores procesan elementos independientes sin interferirse.
- `deadlock_timeout` controla cuánto tiempo espera PostgreSQL antes de verificar si existe un deadlock; valores más bajos detectan deadlocks más rápido pero aumentan el overhead de verificación.
- **REPEATABLE READ** y **SERIALIZABLE** ofrecen mayor aislamiento pero pueden generar más errores de serialización que deben manejarse en la aplicación.

### Recursos Adicionales

- [PostgreSQL — Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [PostgreSQL — Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [psycopg2 — Error Classes](https://www.psycopg.org/docs/errors.html)
- [Aurora PostgreSQL — Parameter Groups](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_WorkingWithParamGroups.html)
- [AWS Blog — Deadlock Analysis in Aurora PostgreSQL](https://aws.amazon.com/blogs/database/)

---
*Laboratorio 02-00-02 | Módulo 2: Bloqueos y Concurrencia | Nivel: Difícil | Duración: 35 minutos*

---

# Alta disponibilidad y failover

## Metadatos

| Campo            | Detalle                                      |
|------------------|----------------------------------------------|
| **Duración**     | 45 minutos                                   |
| **Complejidad**  | Difícil                                      |
| **Nivel Bloom**  | Aplicar                                      |
| **Módulo**       | 2 — Concurrencia, bloqueos y alta disponibilidad |
| **Laboratorio**  | 02-00-03 (Práctica 4)                        |

---

## Descripción General

En este laboratorio ejecutarás un failover manual sobre un clúster Aurora PostgreSQL, medirás el tiempo de recuperación (RTO) y analizarás el comportamiento de las conexiones de aplicación durante y después del evento. Explorarás los tres tipos de endpoints de Aurora (cluster, reader, instancia-específico) y verificarás cómo cada uno responde ante el cambio de rol writer/reader. Finalmente, revisarás la arquitectura de Aurora Global Database como estrategia de disaster recovery multi-región, correlacionando los conceptos de concurrencia y bloqueos aprendidos en la lección 2.1 con el estado del clúster durante el proceso de failover.

---

## Objetivos de Aprendizaje

Al completar este laboratorio serás capaz de:

- [ ] Ejecutar un failover manual en un clúster Aurora PostgreSQL y medir el RTO con precisión usando AWS CLI y scripts de conectividad continua.
- [ ] Verificar el comportamiento de los endpoints de Aurora (cluster endpoint, reader endpoint, instancia-específico) durante y después de un failover.
- [ ] Analizar los logs de Aurora y los eventos de CloudWatch/EventBridge generados durante el proceso de failover para reconstruir la secuencia de eventos.
- [ ] Configurar reconexión automática en una aplicación Python para tolerar failovers sin intervención manual.
- [ ] Explorar la latencia de replicación en Aurora Global Database y evaluar su impacto en el RTO/RPO para escenarios de DR multi-región.

---

## Prerrequisitos

### Conocimiento previo
- Arquitectura Aurora PostgreSQL: almacenamiento compartido, instancias writer/reader, endpoints.
- Conceptos de MVCC, bloqueos de fila y de tabla (lección 2.1).
- Comandos básicos de `psql`, `pg_stat_activity` y `pg_stat_replication`.
- Manejo de AWS CLI v2 y conceptos de IAM (políticas RDS, CloudWatch).

### Acceso y permisos requeridos
- Usuario IAM con políticas: `rds:FailoverDBCluster`, `rds:DescribeDBClusters`, `rds:DescribeDBInstances`, `rds:DescribeEvents`, `cloudwatch:GetMetricData`, `logs:FilterLogEvents`.
- Clúster Aurora PostgreSQL aprovisionado con **al menos una réplica de lectura** (instancia writer + 1 reader mínimo).
- Terraform state del módulo 2 aplicado (`terraform apply` completado).
- Acceso SSH o terminal con `psql`, `pgbench`, Python 3.9+ y `jq` instalados.
- Variable de entorno `PGPASSWORD` o archivo `~/.pgpass` configurado.

---

## Entorno de Laboratorio

### Hardware recomendado

| Componente       | Mínimo              | Recomendado         |
|------------------|---------------------|---------------------|
| RAM              | 8 GB                | 16 GB               |
| CPU              | 4 núcleos           | 8 núcleos           |
| Almacenamiento   | 10 GB libres        | 20 GB libres        |
| Red              | 10 Mbps             | 25 Mbps             |

### Software requerido

| Herramienta            | Versión mínima |
|------------------------|----------------|
| AWS CLI                | 2.x            |
| psql                   | 14.x           |
| pgbench                | 14.x           |
| Python                 | 3.9+           |
| jq                     | 1.6+           |
| DBeaver Community      | 23.x           |

### Configuración inicial del entorno

Ejecuta los siguientes comandos en tu terminal antes de comenzar los pasos del laboratorio:

```bash
# 1. Verificar versión de AWS CLI
aws --version

# 2. Confirmar región activa
aws configure get region

# 3. Definir variables de entorno para el laboratorio
export AWS_REGION="us-east-1"                          # Ajusta a tu región
export CLUSTER_ID="aurora-lab-cluster"                 # Nombre de tu clúster
export DB_NAME="labdb"
export DB_USER="labadmin"

# 4. Obtener endpoints del clúster y guardarlos en variables
export CLUSTER_ENDPOINT=$(aws rds describe-db-clusters \
  --db-cluster-identifier $CLUSTER_ID \
  --query "DBClusters[0].Endpoint" \
  --output text)

export READER_ENDPOINT=$(aws rds describe-db-clusters \
  --db-cluster-identifier $CLUSTER_ID \
  --query "DBClusters[0].ReaderEndpoint" \
  --output text)

echo "Cluster Endpoint : $CLUSTER_ENDPOINT"
echo "Reader Endpoint  : $READER_ENDPOINT"

# 5. Verificar conectividad básica
psql -h $CLUSTER_ENDPOINT -U $DB_USER -d $DB_NAME -c "SELECT version();"
```

> **⚠️ Aviso de costos:** Las instancias Aurora `db.r6g.large` generan cargos por hora. Ejecuta `terraform destroy` al finalizar la sesión. Costo estimado para este laboratorio: **$2–4 USD**.

---

## Pasos del Laboratorio

---

### Paso 1: Verificar la topología del clúster Aurora

**Objetivo:** Confirmar que el clúster tiene al menos una instancia writer y una reader activas antes de iniciar el failover.

#### Instrucciones

1. Lista todas las instancias del clúster y sus roles actuales:

```bash
aws rds describe-db-clusters \
  --db-cluster-identifier $CLUSTER_ID \
  --query "DBClusters[0].DBClusterMembers[*].{ID:DBInstanceIdentifier,Writer:IsClusterWriter,Status:DBClusterParameterGroupStatus}" \
  --output table
```

2. Identifica explícitamente cuál instancia es el writer actual:

```bash
export WRITER_INSTANCE=$(aws rds describe-db-clusters \
  --db-cluster-identifier $CLUSTER_ID \
  --query "DBClusters[0].DBClusterMembers[?IsClusterWriter==\`true\`].DBInstanceIdentifier" \
  --output text)

export READER_INSTANCE=$(aws rds describe-db-clusters \
  --db-cluster-identifier $CLUSTER_ID \
  --query "DBClusters[0].DBClusterMembers[?IsClusterWriter==\`false\`].DBInstanceIdentifier | [0]" \
  --output text)

echo "Writer actual : $WRITER_INSTANCE"
echo "Reader actual : $READER_INSTANCE"
```

3. Desde `psql`, verifica el rol de la instancia conectada a través del cluster endpoint:

```sql
-- Conectar al cluster endpoint (siempre apunta al writer)
\c labdb

-- Verificar rol
SELECT pg_is_in_recovery() AS es_replica,
       inet_server_addr()  AS ip_instancia,
       current_setting('server_version') AS version_pg;
```

4. Verifica el estado de replicación desde la instancia writer:

```sql
-- Solo disponible en la instancia writer
SELECT client_addr,
       state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       (sent_lsn - replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

5. Verifica el estado de la réplica lectora conectando al reader endpoint:

```bash
psql -h $READER_ENDPOINT -U $DB_USER -d $DB_NAME \
  -c "SELECT pg_is_in_recovery() AS es_replica, inet_server_addr() AS ip;"
```

#### Salida esperada

```
Writer actual : aurora-lab-cluster-instance-1
Reader actual : aurora-lab-cluster-instance-2

 es_replica | ip_instancia  | version_pg
------------+---------------+------------
 f          | 10.0.1.45     | 14.x
(1 row)

-- pg_stat_replication mostrará 1 fila con state = 'streaming'

-- Reader endpoint:
 es_replica | ip
------------+-----------
 t          | 10.0.1.67
```

#### Verificación

```bash
# Confirmar que el cluster está en estado 'available'
aws rds describe-db-clusters \
  --db-cluster-identifier $CLUSTER_ID \
  --query "DBClusters[0].Status" \
  --output text
# Resultado esperado: available
```

---

### Paso 2: Preparar el script de monitoreo de conectividad continua

**Objetivo:** Crear un script Python que intente conectarse al cluster endpoint cada segundo y registre cuándo pierde y recupera la conexión, permitiendo medir el RTO con precisión.

#### Instrucciones

1. Crea el archivo `monitor_failover.py`:

```python
#!/usr/bin/env python3
"""
monitor_failover.py
Monitorea la conectividad al cluster endpoint de Aurora durante un failover.
Registra timestamps de pérdida y recuperación de conexión para calcular RTO.
"""

import psycopg2
import time
import os
import sys
from datetime import datetime

# Configuración — ajusta con tus valores
CLUSTER_ENDPOINT = os.environ.get("CLUSTER_ENDPOINT", "tu-cluster-endpoint.rds.amazonaws.com")
DB_NAME    = os.environ.get("DB_NAME", "labdb")
DB_USER    = os.environ.get("DB_USER", "labadmin")
DB_PASS    = os.environ.get("PGPASSWORD", "")
DB_PORT    = 5432
INTERVAL   = 1  # segundos entre intentos

log_file = "failover_log.txt"

def ts():
    return datetime.utcnow().strftime("%Y-%m-%dT%H:%M:%S.%f")[:-3] + "Z"

def try_connect():
    """Intenta conectar y ejecutar una query simple. Retorna (True, writer_ip) o (False, error)."""
    try:
        conn = psycopg2.connect(
            host=CLUSTER_ENDPOINT, dbname=DB_NAME,
            user=DB_USER, password=DB_PASS,
            port=DB_PORT, connect_timeout=3,
            options="-c statement_timeout=2000"
        )
        conn.autocommit = True
        cur = conn.cursor()
        cur.execute("SELECT inet_server_addr(), pg_is_in_recovery();")
        ip, is_replica = cur.fetchone()
        cur.close()
        conn.close()
        return True, f"ip={ip} replica={is_replica}"
    except Exception as e:
        return False, str(e)

def main():
    print(f"[{ts()}] Iniciando monitoreo → {CLUSTER_ENDPOINT}")
    print(f"[{ts()}] Log en: {log_file}")
    print("Presiona Ctrl+C para detener.\n")

    last_status = None
    down_start  = None
    attempt     = 0

    with open(log_file, "w") as f:
        f.write(f"timestamp,attempt,status,detail\n")

        while True:
            attempt += 1
            ok, detail = try_connect()
            status = "UP" if ok else "DOWN"
            line = f"{ts()},{attempt},{status},{detail}"
            f.write(line + "\n")
            f.flush()

            if status != last_status:
                if status == "DOWN":
                    down_start = time.time()
                    print(f"\n🔴 [{ts()}] CONEXIÓN PERDIDA — Attempt #{attempt}")
                    print(f"   Detalle: {detail}")
                elif status == "UP" and down_start:
                    rto = time.time() - down_start
                    print(f"\n🟢 [{ts()}] CONEXIÓN RECUPERADA — RTO: {rto:.2f}s")
                    print(f"   Detalle: {detail}")
                    down_start = None
                else:
                    print(f"🟢 [{ts()}] Conectado — {detail}")
                last_status = status
            else:
                # Indicador de progreso sin nueva línea
                sym = "." if ok else "x"
                print(sym, end="", flush=True)

            time.sleep(INTERVAL)

if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print(f"\n\n[{ts()}] Monitoreo detenido.")
        sys.exit(0)
```

2. Instala la dependencia requerida:

```bash
pip3 install psycopg2-binary
```

3. Prueba el script antes del failover para confirmar que funciona correctamente:

```bash
python3 monitor_failover.py
# Deberías ver líneas con 🟢 y puntos "." cada segundo
# Deja el script corriendo en esta terminal — NO lo detengas
```

4. Abre una **segunda terminal** para ejecutar los pasos siguientes mientras el monitor corre en la primera.

#### Salida esperada (terminal de monitoreo)

```
[2024-01-15T14:22:01.123Z] Iniciando monitoreo → aurora-lab-cluster.cluster-xyz.us-east-1.rds.amazonaws.com
[2024-01-15T14:22:01.124Z] Log en: failover_log.txt

🟢 [2024-01-15T14:22:01.987Z] Conectado — ip=10.0.1.45 replica=False
..........
```

#### Verificación

```bash
# Confirmar que el log se está escribiendo
head -5 failover_log.txt
# Debe mostrar encabezado CSV y filas con status=UP
```

---

### Paso 3: Ejecutar el failover manual y medir el RTO

**Objetivo:** Iniciar un failover manual del clúster Aurora usando AWS CLI, observar el comportamiento en el monitor de conectividad y medir el tiempo de recuperación.

#### Instrucciones

1. Registra el timestamp exacto de inicio del failover:

```bash
echo "Failover iniciado: $(date -u +%Y-%m-%dT%H:%M:%SZ)"
FAILOVER_START=$(date +%s)
```

2. Ejecuta el failover manual usando AWS CLI (desde la segunda terminal):

```bash
aws rds failover-db-cluster \
  --db-cluster-identifier $CLUSTER_ID \
  --target-db-instance-identifier $READER_INSTANCE \
  --region $AWS_REGION
```

> **Nota:** El parámetro `--target-db-instance-identifier` fuerza la promoción de una réplica específica. Si se omite, Aurora elige automáticamente la réplica con mayor prioridad de failover (Tier).

3. Monitorea el estado del clúster en tiempo real con polling cada 5 segundos:

```bash
watch -n 5 "aws rds describe-db-clusters \
  --db-cluster-identifier $CLUSTER_ID \
  --query 'DBClusters[0].{Status:Status,Members:DBClusterMembers[*].{ID:DBInstanceIdentifier,Writer:IsClusterWriter}}' \
  --output json | jq ."
```

4. Cuando el clúster vuelva al estado `available`, registra el tiempo de recuperación:

```bash
FAILOVER_END=$(date +%s)
RTO_SECONDS=$((FAILOVER_END - FAILOVER_START))
echo "RTO aproximado (desde CLI): ${RTO_SECONDS} segundos"
```

5. Verifica que los roles writer/reader se han invertido:

```bash
# Nuevo writer (debería ser la antigua réplica)
aws rds describe-db-clusters \
  --db-cluster-identifier $CLUSTER_ID \
  --query "DBClusters[0].DBClusterMembers[?IsClusterWriter==\`true\`].DBInstanceIdentifier" \
  --output text

# Confirmar desde psql que el cluster endpoint apunta al nuevo writer
psql -h $CLUSTER_ENDPOINT -U $DB_USER -d $DB_NAME \
  -c "SELECT inet_server_addr() AS ip_writer, pg_is_in_recovery() AS es_replica;"
```

6. Analiza el log del monitor de conectividad para calcular el RTO preciso:

```bash
# Contar segundos en estado DOWN
grep ",DOWN," failover_log.txt | wc -l

# Ver el momento exacto de pérdida y recuperación
grep -E ",DOWN,|,UP," failover_log.txt | head -20
```

#### Salida esperada (terminal de monitoreo durante failover)

```
..........
🔴 [2024-01-15T14:25:03.421Z] CONEXIÓN PERDIDA — Attempt #182
   Detalle: could not connect to server: Connection refused

xxxxxxxxxxxxxxxxxxxxxxxxxx

🟢 [2024-01-15T14:25:33.887Z] CONEXIÓN RECUPERADA — RTO: 30.47s
   Detalle: ip=10.0.1.67 replica=False
```

#### Salida esperada (CLI post-failover)

```json
{
  "Status": "available",
  "Members": [
    {"ID": "aurora-lab-cluster-instance-2", "Writer": true},
    {"ID": "aurora-lab-cluster-instance-1", "Writer": false}
  ]
}
```

#### Verificación

```bash
# El nuevo writer NO debe ser una réplica
psql -h $CLUSTER_ENDPOINT -U $DB_USER -d $DB_NAME \
  -c "SELECT pg_is_in_recovery();"
# Resultado esperado: f (false = es writer)

# El reader endpoint debe apuntar a la antigua instancia writer (ahora reader)
psql -h $READER_ENDPOINT -U $DB_USER -d $DB_NAME \
  -c "SELECT pg_is_in_recovery();"
# Resultado esperado: t (true = es réplica)
```

---

### Paso 4: Analizar los logs de Aurora durante el failover

**Objetivo:** Revisar los eventos de RDS y los logs de PostgreSQL para reconstruir la secuencia de eventos del failover y entender qué ocurrió con las conexiones y bloqueos activos.

#### Instrucciones

1. Consulta los eventos del clúster en la ventana de tiempo del failover:

```bash
# Eventos de los últimos 30 minutos
aws rds describe-events \
  --source-identifier $CLUSTER_ID \
  --source-type db-cluster \
  --duration 30 \
  --query "Events[*].{Time:Date,Message:Message}" \
  --output table
```

2. Consulta los eventos de las instancias individuales (writer y reader):

```bash
for INSTANCE in $WRITER_INSTANCE $READER_INSTANCE; do
  echo "=== Eventos de instancia: $INSTANCE ==="
  aws rds describe-events \
    --source-identifier $INSTANCE \
    --source-type db-instance \
    --duration 30 \
    --query "Events[*].{Time:Date,Message:Message}" \
    --output table
done
```

3. Descarga el log de PostgreSQL de la instancia que era writer antes del failover:

```bash
# Listar logs disponibles
aws rds describe-db-log-files \
  --db-instance-identifier $WRITER_INSTANCE \
  --query "DescribeDBLogFiles[-3:].{Name:LogFileName,Size:Size}" \
  --output table

# Descargar el log más reciente (ajusta el nombre según la salida anterior)
aws rds download-db-log-file-portion \
  --db-instance-identifier $WRITER_INSTANCE \
  --log-file-name "error/postgresql.log.2024-01-15-14" \
  --output text > writer_failover.log

# Buscar eventos clave del failover
grep -iE "failover|shutdown|promoted|recovery|checkpoint" writer_failover.log | tail -30
```

4. Conecta al nuevo writer y verifica el estado de las sesiones post-failover:

```sql
-- Sesiones activas post-failover
SELECT pid,
       usename,
       application_name,
       client_addr,
       state,
       wait_event_type,
       wait_event,
       backend_start,
       now() - backend_start AS session_age
FROM pg_stat_activity
WHERE backend_type = 'client backend'
ORDER BY backend_start;
```

5. Verifica si quedaron transacciones "idle in transaction" o bloqueos residuales después del failover (concepto de la lección 2.1):

```sql
-- Detectar sesiones idle in transaction post-failover
SELECT pid,
       usename,
       xact_start,
       now() - xact_start AS duracion_tx,
       state,
       query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY duracion_tx DESC;

-- Verificar bloqueos activos
SELECT
  bl.pid       AS pid_bloqueado,
  ka.pid       AS pid_bloqueador,
  l.mode       AS modo_bloqueo,
  now() - ba.xact_start AS dur_tx_bloqueado,
  ba.query     AS sql_bloqueado
FROM pg_locks bl
JOIN pg_stat_activity ba ON bl.pid = ba.pid
JOIN pg_locks kl ON bl.locktype = kl.locktype
  AND bl.database IS NOT DISTINCT FROM kl.database
  AND bl.relation IS NOT DISTINCT FROM kl.relation
  AND bl.page IS NOT DISTINCT FROM kl.page
  AND bl.tuple IS NOT DISTINCT FROM kl.tuple
  AND bl.classid IS NOT DISTINCT FROM kl.classid
  AND bl.objid IS NOT DISTINCT FROM kl.objid
  AND bl.objsubid IS NOT DISTINCT FROM kl.objsubid
  AND bl.transactionid IS NOT DISTINCT FROM kl.transactionid
  AND bl.pid <> kl.pid
JOIN pg_stat_activity ka ON kl.pid = ka.pid
WHERE NOT bl.granted;
```

#### Salida esperada (eventos RDS)

```
----------------------------------------------------------------------
|                         DescribeEvents                             |
+---------------------------+-----------------------------------------+
| Time                      | Message                                 |
+---------------------------+-----------------------------------------+
| 2024-01-15T14:25:01.000Z  | Failover started for cluster ...        |
| 2024-01-15T14:25:02.000Z  | DB instance instance-1 is being...      |
| 2024-01-15T14:25:28.000Z  | Completed failover to DB instance...    |
| 2024-01-15T14:25:30.000Z  | DB cluster is available                 |
+---------------------------+-----------------------------------------+
```

#### Verificación

```bash
# Confirmar que el log descargado contiene entradas de failover
grep -c "failover\|promoted" writer_failover.log
# Debe retornar un número > 0
```

---

### Paso 5: Configurar reconexión automática en la aplicación

**Objetivo:** Implementar un patrón de reconexión con backoff exponencial en Python para que la aplicación tolere failovers de Aurora sin intervención manual.

#### Instrucciones

1. Crea el archivo `app_resiliente.py` con manejo robusto de reconexión:

```python
#!/usr/bin/env python3
"""
app_resiliente.py
Demuestra reconexión automática con backoff exponencial durante failover Aurora.
Implementa el patrón recomendado para aplicaciones que usan el cluster endpoint.
"""

import psycopg2
import time
import os
import random
from datetime import datetime

# Configuración
CLUSTER_ENDPOINT = os.environ.get("CLUSTER_ENDPOINT")
DB_NAME          = os.environ.get("DB_NAME", "labdb")
DB_USER          = os.environ.get("DB_USER", "labadmin")
DB_PASS          = os.environ.get("PGPASSWORD", "")

# Parámetros de reconexión
MAX_RETRIES     = 20
BASE_DELAY      = 0.5   # segundos
MAX_DELAY       = 30.0  # segundos
JITTER_FACTOR   = 0.3   # 30% de jitter aleatorio

def ts():
    return datetime.utcnow().strftime("%H:%M:%S.%f")[:-3]

def get_connection(attempt=0):
    """Obtiene conexión con backoff exponencial y jitter."""
    delay = min(BASE_DELAY * (2 ** attempt), MAX_DELAY)
    jitter = delay * JITTER_FACTOR * random.random()
    wait = delay + jitter

    for retry in range(MAX_RETRIES):
        try:
            conn = psycopg2.connect(
                host=CLUSTER_ENDPOINT,
                dbname=DB_NAME,
                user=DB_USER,
                password=DB_PASS,
                port=5432,
                connect_timeout=5,
                # Importante: deshabilitar keepalive largo para detectar rápido la caída
                keepalives=1,
                keepalives_idle=10,
                keepalives_interval=3,
                keepalives_count=3,
                options="-c statement_timeout=5000 -c lock_timeout=3000"
            )
            conn.autocommit = False
            print(f"[{ts()}] ✅ Conexión establecida (intento {retry+1})")
            return conn
        except psycopg2.OperationalError as e:
            current_wait = min(BASE_DELAY * (2 ** retry), MAX_DELAY)
            current_wait += current_wait * JITTER_FACTOR * random.random()
            print(f"[{ts()}] ⚠️  Intento {retry+1}/{MAX_RETRIES} fallido: {e}")
            if retry < MAX_RETRIES - 1:
                print(f"[{ts()}]    Reintentando en {current_wait:.2f}s...")
                time.sleep(current_wait)
    raise Exception(f"No se pudo conectar después de {MAX_RETRIES} intentos")

def ejecutar_operacion(conn, op_id):
    """Ejecuta una operación de negocio simple con manejo de errores."""
    try:
        with conn.cursor() as cur:
            # Simular operación de lectura/escritura
            cur.execute("""
                SELECT inet_server_addr() AS writer_ip,
                       pg_is_in_recovery() AS es_replica,
                       now() AS ts_servidor
            """)
            row = cur.fetchone()
            conn.commit()
            print(f"[{ts()}] Op#{op_id:04d} OK → writer={row[0]} replica={row[1]}")
            return True
    except (psycopg2.OperationalError, psycopg2.InterfaceError) as e:
        print(f"[{ts()}] Op#{op_id:04d} ERROR (reconectando): {e}")
        try:
            conn.rollback()
        except Exception:
            pass
        return False

def main():
    print(f"[{ts()}] Iniciando aplicación resiliente")
    print(f"[{ts()}] Endpoint: {CLUSTER_ENDPOINT}\n")

    conn = get_connection()
    op_id = 0

    while True:
        op_id += 1
        success = ejecutar_operacion(conn, op_id)

        if not success:
            print(f"[{ts()}] 🔄 Iniciando reconexión...")
            try:
                conn.close()
            except Exception:
                pass
            conn = get_connection()

        time.sleep(1)

if __name__ == "__main__":
    main()
```

2. Ejecuta la aplicación resiliente en una tercera terminal:

```bash
python3 app_resiliente.py
```

3. Ejecuta un **segundo failover** para probar la reconexión automática (ahora volviendo a la instancia original):

```bash
# Obtener el reader actual (que ahora es la instancia original)
NEW_READER=$(aws rds describe-db-clusters \
  --db-cluster-identifier $CLUSTER_ID \
  --query "DBClusters[0].DBClusterMembers[?IsClusterWriter==\`false\`].DBInstanceIdentifier | [0]" \
  --output text)

echo "Iniciando segundo failover hacia: $NEW_READER"
aws rds failover-db-cluster \
  --db-cluster-identifier $CLUSTER_ID \
  --target-db-instance-identifier $NEW_READER \
  --region $AWS_REGION
```

4. Observa cómo la aplicación detecta la caída, reintenta con backoff y reconecta automáticamente.

5. Configura los timeouts recomendados a nivel de sesión (aplica el concepto de la lección 2.1):

```sql
-- Conectar al nuevo writer y configurar timeouts para prevenir bloqueos prolongados
ALTER ROLE labadmin SET lock_timeout = '3s';
ALTER ROLE labadmin SET statement_timeout = '30s';
ALTER ROLE labadmin SET idle_in_transaction_session_timeout = '60s';

-- Verificar la configuración
SELECT name, setting, unit
FROM pg_settings
WHERE name IN ('lock_timeout', 'statement_timeout', 'idle_in_transaction_session_timeout');
```

#### Salida esperada (app_resiliente.py durante failover)

```
[14:35:01.123] Iniciando aplicación resiliente
[14:35:01.124] Endpoint: aurora-lab-cluster.cluster-xyz.us-east-1.rds.amazonaws.com

[14:35:01.987] ✅ Conexión establecida (intento 1)
[14:35:02.012] Op#0001 OK → writer=10.0.1.67 replica=False
[14:35:03.015] Op#0002 OK → writer=10.0.1.67 replica=False
...
[14:35:45.321] Op#0043 ERROR (reconectando): server closed the connection unexpectedly
[14:35:45.322] 🔄 Iniciando reconexión...
[14:35:45.323] ⚠️  Intento 1/20 fallido: could not connect to server
[14:35:45.823]    Reintentando en 1.23s...
[14:35:47.891] ⚠️  Intento 2/20 fallido: could not connect to server
...
[14:36:12.445] ✅ Conexión establecida (intento 7)
[14:36:12.512] Op#0044 OK → writer=10.0.1.45 replica=False
```

#### Verificación

```bash
# Confirmar que los timeouts están configurados
psql -h $CLUSTER_ENDPOINT -U $DB_USER -d $DB_NAME \
  -c "SHOW lock_timeout; SHOW statement_timeout; SHOW idle_in_transaction_session_timeout;"
```

---

### Paso 6: Explorar Aurora Global Database y latencia de replicación

**Objetivo:** Comprender la arquitectura de Aurora Global Database como estrategia de DR multi-región y verificar métricas de latencia de replicación usando CloudWatch.

> **⚠️ Nota:** Este paso es **exploratorio/observacional** si no tienes una Aurora Global Database aprovisionada. Si tienes acceso a una, ejecuta todos los comandos. Si no, ejecuta solo los comandos de consulta de métricas y revisa la arquitectura.

#### Instrucciones

1. Verifica si existe una Aurora Global Database asociada al clúster:

```bash
aws rds describe-global-clusters \
  --query "GlobalClusters[*].{ID:GlobalClusterIdentifier,Status:Status,Engine:Engine,Members:GlobalClusterMembers[*].{ARN:DBClusterArn,Writer:IsWriter}}" \
  --output json | jq .
```

2. Si existe una Global Database, consulta la latencia de replicación entre regiones desde CloudWatch:

```bash
# Latencia de replicación Aurora Global Database (últimos 30 minutos)
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name AuroraGlobalDBReplicationLag \
  --dimensions Name=DBClusterIdentifier,Value=$CLUSTER_ID \
  --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || \
                 date -u -v-30M +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Average,Maximum \
  --query "Datapoints[*].{Time:Timestamp,Avg:Average,Max:Maximum}" \
  --output table
```

3. Consulta métricas adicionales del clúster primario relevantes para DR:

```bash
# RPO estimado: AuroraGlobalDBRPOLag
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name AuroraGlobalDBRPOLag \
  --dimensions Name=DBClusterIdentifier,Value=$CLUSTER_ID \
  --start-time $(date -u -d '30 minutes ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || \
                 date -u -v-30M +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Average,Maximum \
  --output table 2>/dev/null || echo "Métrica no disponible (Global Database no configurada)"
```

4. Desde el clúster Aurora, verifica los parámetros relevantes para recuperación y autovacuum post-failover:

```sql
-- Parámetros de autovacuum relevantes para recuperación
SELECT name,
       setting,
       unit,
       short_desc
FROM pg_settings
WHERE name IN (
  'autovacuum_vacuum_cost_delay',
  'autovacuum_vacuum_scale_factor',
  'autovacuum_analyze_scale_factor',
  'checkpoint_completion_target',
  'wal_level',
  'max_wal_senders',
  'wal_keep_size'
)
ORDER BY name;
```

5. Verifica el estado del WAL y la posición actual del LSN (útil para entender el RPO):

```sql
-- Posición actual del WAL (en la instancia writer)
SELECT pg_current_wal_lsn()        AS lsn_actual,
       pg_walfile_name(pg_current_wal_lsn()) AS archivo_wal,
       pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0') AS bytes_wal_total;

-- Verificar que el writer no está en recovery (confirmación post-failover)
SELECT pg_is_in_recovery(),
       pg_last_wal_receive_lsn(),
       pg_last_wal_replay_lsn(),
       pg_last_xact_replay_timestamp();
```

6. Documenta los valores de RTO y RPO observados en el laboratorio:

```bash
cat << EOF > rto_rpo_summary.txt
=== Resumen RTO/RPO del Laboratorio 02-00-03 ===
Fecha: $(date -u)
Cluster: $CLUSTER_ID
Región: $AWS_REGION

Failover #1:
  - RTO medido por script Python: $(grep -c ",DOWN," failover_log.txt) segundos aprox.
  - Instancia promovida: $READER_INSTANCE

Failover #2:
  - Instancia promovida: $NEW_READER

Configuración de timeouts aplicada:
  - lock_timeout: 3s
  - statement_timeout: 30s
  - idle_in_transaction_session_timeout: 60s

Aurora Global Database:
  - Latencia de replicación típica: < 1 segundo (mismo continente)
  - RPO teórico: ~1 segundo
  - RTO con managed failover: ~60-120 segundos
EOF

cat rto_rpo_summary.txt
```

#### Salida esperada

```
=== Resumen RTO/RPO del Laboratorio 02-00-03 ===
Fecha: Mon Jan 15 14:55:00 UTC 2024
Cluster: aurora-lab-cluster
Región: us-east-1

Failover #1:
  - RTO medido por script Python: 28 segundos aprox.
  - Instancia promovida: aurora-lab-cluster-instance-2
...
```

#### Verificación

```bash
# Confirmar que el clúster está estable y disponible
aws rds describe-db-clusters \
  --db-cluster-identifier $CLUSTER_ID \
  --query "DBClusters[0].{Status:Status,MultiAZ:MultiAZ,Engine:Engine,EngineVersion:EngineVersion}" \
  --output table
```

---

## Validación y Pruebas

Ejecuta las siguientes verificaciones para confirmar que el laboratorio se completó correctamente:

```bash
#!/bin/bash
# validate_lab.sh — Ejecutar al finalizar todos los pasos

echo "=============================="
echo "Validación Lab 02-00-03"
echo "=============================="

PASS=0
FAIL=0

check() {
  local desc="$1"
  local cmd="$2"
  local expected="$3"
  result=$(eval "$cmd" 2>/dev/null)
  if echo "$result" | grep -q "$expected"; then
    echo "✅ PASS: $desc"
    ((PASS++))
  else
    echo "❌ FAIL: $desc (obtenido: $result)"
    ((FAIL++))
  fi
}

# 1. Clúster disponible
check "Clúster en estado available" \
  "aws rds describe-db-clusters --db-cluster-identifier $CLUSTER_ID --query 'DBClusters[0].Status' --output text" \
  "available"

# 2. Cluster endpoint apunta al writer (no es réplica)
check "Cluster endpoint apunta al writer" \
  "psql -h $CLUSTER_ENDPOINT -U $DB_USER -d $DB_NAME -t -c 'SELECT pg_is_in_recovery();'" \
  "f"

# 3. Reader endpoint apunta a una réplica
check "Reader endpoint apunta a réplica" \
  "psql -h $READER_ENDPOINT -U $DB_USER -d $DB_NAME -t -c 'SELECT pg_is_in_recovery();'" \
  "t"

# 4. Log de failover existe y tiene entradas DOWN
check "Log de failover contiene eventos DOWN" \
  "grep -c ',DOWN,' failover_log.txt" \
  "[0-9]"

# 5. Timeout de lock configurado
check "lock_timeout configurado en 3s" \
  "psql -h $CLUSTER_ENDPOINT -U $DB_USER -d $DB_NAME -t -c 'SHOW lock_timeout;'" \
  "3s"

# 6. idle_in_transaction_session_timeout configurado
check "idle_in_transaction_session_timeout configurado" \
  "psql -h $CLUSTER_ENDPOINT -U $DB_USER -d $DB_NAME -t -c 'SHOW idle_in_transaction_session_timeout;'" \
  "60s"

# 7. pg_stat_replication tiene al menos una fila (réplica activa)
check "Réplica de streaming activa" \
  "psql -h $CLUSTER_ENDPOINT -U $DB_USER -d $DB_NAME -t -c 'SELECT count(*) FROM pg_stat_replication;'" \
  "[1-9]"

# 8. Archivo de resumen RTO/RPO creado
check "Archivo rto_rpo_summary.txt existe" \
  "test -f rto_rpo_summary.txt && echo found" \
  "found"

echo ""
echo "=============================="
echo "Resultado: $PASS PASS | $FAIL FAIL"
echo "=============================="
```

```bash
chmod +x validate_lab.sh
bash validate_lab.sh
```

**Resultado esperado:** 8/8 PASS. Si alguna verificación falla, revisa el paso correspondiente.

---

## Resolución de Problemas

### Problema 1: El failover tarda más de 60 segundos y el clúster queda en estado "failing-over"

**Síntomas:**
- El comando `aws rds failover-db-cluster` retorna exitosamente, pero el clúster permanece en estado `failing-over` durante más de 60 segundos.
- El script de monitoreo continúa mostrando `x` (DOWN) sin recuperarse.
- `aws rds describe-db-clusters` muestra `"Status": "failing-over"` de forma persistente.

**Causa probable:**
La réplica de lectura objetivo tiene conexiones activas con transacciones largas (`idle in transaction`) que retienen bloqueos, o el parámetro de prioridad de failover (Tier) no está configurado correctamente. En clústeres con carga activa, las sesiones que mantienen bloqueos de tabla (`AccessExclusiveLock`) pueden retrasar la finalización del checkpoint previo a la promoción.

**Solución:**

```bash
# 1. Verificar el estado detallado de las instancias
aws rds describe-db-instances \
  --filters Name=db-cluster-id,Values=$CLUSTER_ID \
  --query "DBInstances[*].{ID:DBInstanceIdentifier,Status:DBInstanceStatus,Tier:PromotionTier}" \
  --output table

# 2. Si hay sesiones bloqueantes en el writer, terminarlas antes del failover
psql -h $CLUSTER_ENDPOINT -U $DB_USER -d $DB_NAME << 'SQL'
-- Identificar y terminar sesiones idle in transaction > 5 minutos
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - xact_start > interval '5 minutes';
SQL

# 3. Ajustar el Tier de la réplica para que tenga prioridad 0 (máxima)
aws rds modify-db-instance \
  --db-instance-identifier $READER_INSTANCE \
  --promotion-tier 0 \
  --apply-immediately

# 4. Reintentar el failover
aws rds failover-db-cluster \
  --db-cluster-identifier $CLUSTER_ID \
  --region $AWS_REGION
```

---

### Problema 2: La aplicación Python no reconecta automáticamente después del failover (sigue fallando con "SSL connection has been closed unexpectedly")

**Síntomas:**
- `app_resiliente.py` muestra repetidamente `ERROR: SSL connection has been closed unexpectedly` pero no logra reconectar.
- El script de monitoreo `monitor_failover.py` ya muestra `🟢 CONEXIÓN RECUPERADA`, pero la app sigue fallando.
- El DNS del cluster endpoint ya resuelve a la nueva IP del writer.

**Causa probable:**
La conexión Python tiene caché de DNS a nivel del sistema operativo o de la librería `psycopg2`/`libpq` que sigue resolviendo a la IP antigua del writer (ahora reader). Adicionalmente, el objeto `conn` de psycopg2 puede estar en un estado inconsistente donde `conn.closed` es `0` (aparentemente abierto) pero la conexión TCP subyacente ya fue terminada por Aurora durante el failover.

**Solución:**

```python
# Agregar verificación explícita del estado de la conexión
# y forzar resolución DNS fresca en cada reconexión

import socket

def is_connection_alive(conn):
    """Verifica si la conexión está realmente activa."""
    if conn is None or conn.closed != 0:
        return False
    try:
        conn.cursor().execute("SELECT 1")
        return True
    except Exception:
        return False

def flush_dns_cache(hostname):
    """Fuerza resolución DNS fresca (evita caché del SO)."""
    try:
        # Resolver manualmente para verificar que el DNS ya propagó
        ip = socket.gethostbyname(hostname)
        print(f"DNS resuelto: {hostname} → {ip}")
        return ip
    except socket.gaierror as e:
        print(f"DNS no resuelto aún: {e}")
        return None

# En el loop principal, antes de reconectar:
# flush_dns_cache(CLUSTER_ENDPOINT)
# time.sleep(2)  # Dar tiempo a que el DNS propague completamente
# conn = get_connection()
```

```bash
# Verificar desde la terminal que el DNS ya propagó al nuevo writer
# (comparar con la IP que reportó el script de monitoreo)
dig +short $CLUSTER_ENDPOINT
nslookup $CLUSTER_ENDPOINT

# Si el DNS aún retorna la IP antigua, esperar 5-10 segundos más
# Aurora actualiza el DNS del cluster endpoint en ~30 segundos post-failover

# Limpiar caché DNS del sistema operativo si es necesario
# En Linux:
sudo systemd-resolve --flush-caches 2>/dev/null || \
sudo /etc/init.d/nscd restart 2>/dev/null || \
echo "Caché DNS limpiada manualmente"

# Verificar TTL del DNS de Aurora (debe ser bajo, ~5 segundos)
dig $CLUSTER_ENDPOINT | grep -E "ANSWER|TTL"
```

---

## Limpieza de Recursos

> **⚠️ Importante:** Ejecuta estos pasos al finalizar el laboratorio para evitar cargos innecesarios.

```bash
# 1. Detener todos los scripts de monitoreo (Ctrl+C en sus terminales)
# monitor_failover.py, app_resiliente.py

# 2. Verificar el estado final del clúster
aws rds describe-db-clusters \
  --db-cluster-identifier $CLUSTER_ID \
  --query "DBClusters[0].{Status:Status,Members:DBClusterMembers[*].{ID:DBInstanceIdentifier,Writer:IsClusterWriter}}" \
  --output json | jq .

# 3. Revertir los timeouts configurados si es necesario para otros labs
psql -h $CLUSTER_ENDPOINT -U $DB_USER -d $DB_NAME << 'SQL'
-- Opcional: revertir timeouts al valor por defecto del parameter group
ALTER ROLE labadmin RESET lock_timeout;
ALTER ROLE labadmin RESET statement_timeout;
ALTER ROLE labadmin RESET idle_in_transaction_session_timeout;

-- Verificar
SELECT rolname, rolconfig FROM pg_roles WHERE rolname = 'labadmin';
SQL

# 4. Guardar los artefactos del laboratorio
mkdir -p lab_02_00_03_artifacts
mv failover_log.txt rto_rpo_summary.txt writer_failover.log lab_02_00_03_artifacts/ 2>/dev/null
echo "Artefactos guardados en: lab_02_00_03_artifacts/"

# 5. Si no continuarás con el siguiente laboratorio, destruir infraestructura con Terraform
# SOLO ejecutar si es el fin de la sesión completa del módulo 2
# cd ~/terraform/modulo-02
# terraform destroy -auto-approve

# Si CONTINÚAS con el siguiente lab, NO destruyas la infraestructura
echo "¿Continúas con el siguiente laboratorio? (s/n)"
read -r respuesta
if [ "$respuesta" = "n" ]; then
  echo "Ejecuta: cd ~/terraform/modulo-02 && terraform destroy -auto-approve"
fi
```

---

## Resumen

En este laboratorio aplicaste los conceptos de alta disponibilidad y failover en Aurora PostgreSQL de forma práctica y medible:

| Actividad realizada | Herramienta/Concepto clave |
|---------------------|---------------------------|
| Verificación de topología writer/reader | `pg_stat_replication`, AWS CLI |
| Medición de RTO con script Python | `psycopg2`, backoff exponencial |
| Failover manual y análisis de secuencia | `aws rds failover-db-cluster` |
| Análisis de logs y eventos post-failover | CloudWatch Events, `pg_stat_activity` |
| Detección de bloqueos residuales | `pg_locks`, `idle in transaction` |
| Reconexión automática con tolerancia a fallos | Python + keepalives + timeouts |
| Exploración de Aurora Global Database | `AuroraGlobalDBReplicationLag` |
| Configuración de timeouts preventivos | `lock_timeout`, `statement_timeout` |

### Conceptos clave reforzados

- **RTO típico de Aurora:** 20–40 segundos para failover dentro de la misma región (AZ diferente). El DNS del cluster endpoint se actualiza en ~30 segundos.
- **Endpoints de Aurora:** el cluster endpoint siempre apunta al writer actual; el reader endpoint balancea entre réplicas. Usar el endpoint correcto es crítico para la correcta reconexión post-failover.
- **Bloqueos y failover:** las sesiones `idle in transaction` que retienen bloqueos (`AccessExclusiveLock`, bloqueos de fila) pueden retrasar el failover. Los timeouts configurados en la lección 2.1 (`lock_timeout`, `idle_in_transaction_session_timeout`) son también una medida de resiliencia ante failover.
- **Aurora Global Database:** ofrece RPO < 1 segundo y RTO de 60–120 segundos para DR multi-región, con replicación física a nivel de almacenamiento.

### Recursos adicionales

- [Amazon Aurora — Failover for Aurora DB Clusters](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html)
- [Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
- [Aurora PostgreSQL Endpoints](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Endpoints.html)
- [psycopg2 — Connection parameters](https://www.psycopg.org/docs/module.html#psycopg2.connect)
- [PostgreSQL — Lock Monitoring](https://wiki.postgresql.org/wiki/Lock_Monitoring)

---
