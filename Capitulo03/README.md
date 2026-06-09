# Ajuste de parámetros y optimización

## Metadatos

| Atributo         | Valor                                      |
|------------------|--------------------------------------------|
| **Duración**     | 35 minutos                                 |
| **Complejidad**  | Media                                      |
| **Nivel Bloom**  | Aplicar (Apply)                            |
| **Módulo**       | 3 — Optimización de parámetros y conexiones|
| **Práctica**     | 5                                          |

---

## Descripción General

En este laboratorio crearás un DB parameter group personalizado para Aurora PostgreSQL 14, ajustarás parámetros críticos de memoria, escritura y autovacuum, y medirás el impacto de cada configuración usando `pgbench` como generador de carga OLTP. Partirás de un baseline con la configuración por defecto, aplicarás cambios de forma sistemática y documentarás los resultados en una matriz de parámetros que compara TPS (transacciones por segundo) y latencia media. El laboratorio refuerza el flujo completo: crear grupo → modificar parámetros → asociar a instancia → validar con `psql` y CloudWatch.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio serás capaz de:

- [ ] Crear y aplicar un DB parameter group personalizado con parámetros críticos de PostgreSQL optimizados para cargas OLTP.
- [ ] Ajustar parámetros de memoria (`work_mem`, `effective_cache_size`) y medir su impacto en TPS y latencia con `pgbench`.
- [ ] Configurar autovacuum agresivo (`autovacuum_vacuum_scale_factor`, `autovacuum_analyze_scale_factor`, `autovacuum_vacuum_cost_delay`) y verificar su efecto en `pg_stat_user_tables`.
- [ ] Interpretar métricas de `pg_stat_bgwriter` y CloudWatch (WriteIOPS, FreeableMemory) en correlación con los parámetros ajustados.
- [ ] Documentar una matriz de parámetros con valores baseline, ajustados e impacto medido.

---

## Prerrequisitos

### Conocimiento previo
- Familiaridad con los conceptos de DB parameter groups (parámetros dinámicos vs. estáticos) cubiertos en la Lección 3.1.
- Comprensión básica de los parámetros de memoria de PostgreSQL y sus unidades (kB, MB, 8kB blocks).
- Experiencia básica con `psql` y AWS CLI.

### Acceso y recursos AWS
- Clúster Aurora PostgreSQL 14 activo con al menos una instancia writer (`db.r6g.large` o superior).
- Usuario IAM con permisos sobre: `rds:CreateDBParameterGroup`, `rds:ModifyDBParameterGroup`, `rds:ModifyDBInstance`, `rds:RebootDBInstance`, `rds:DescribeDBParameters`, `cloudwatch:GetMetricStatistics`.
- Base de datos `pgbench` inicializada con escala 10 (ver sección de entorno).
- `pgbench`, `psql` y AWS CLI 2.x instalados en la máquina local.

> **⚠️ Costo estimado:** Una instancia `db.r6g.large` cuesta aproximadamente $0.26 USD/hora. Destruye los recursos al finalizar la sesión.

---

## Entorno del Laboratorio

### Hardware recomendado

| Recurso          | Mínimo                        |
|------------------|-------------------------------|
| RAM              | 8 GB                          |
| CPU              | 4 núcleos                     |
| Almacenamiento   | 10 GB libres                  |
| Red              | 10 Mbps hacia AWS             |

### Software requerido

| Herramienta      | Versión          | Uso en este lab                    |
|------------------|------------------|------------------------------------|
| AWS CLI          | 2.x              | Crear/modificar parameter groups   |
| psql             | 14.x o superior  | Validar parámetros efectivos       |
| pgbench          | 14.x o superior  | Generar carga OLTP reproducible    |
| jq               | 1.6 o superior   | Parsear salidas JSON de AWS CLI    |
| DBeaver / pgAdmin| 23.x / 7.x       | Opcional: visualización de métricas|

### Variables de entorno — configuración inicial

Antes de comenzar, define las variables de entorno que se usarán durante todo el laboratorio. Reemplaza los valores entre `<>` con los datos de tu entorno:

```bash
# ── Identificadores del clúster ──────────────────────────────────────────────
export CLUSTER_ID="aurora-lab-cluster"          # Nombre de tu clúster Aurora
export WRITER_ID="aurora-lab-cluster-instance-1" # Nombre de la instancia writer
export AWS_REGION="us-east-1"                    # Región donde está el clúster
export PG_FAMILY="aurora-postgresql14"           # Familia de parámetros

# ── Conexión a la base de datos ──────────────────────────────────────────────
export PGHOST="<writer-endpoint>.rds.amazonaws.com"
export PGPORT="5432"
export PGUSER="postgres"
export PGPASSWORD="<tu-password>"
export PGDATABASE="pgbench_db"

# ── Nombre del parameter group personalizado ─────────────────────────────────
export PG_GROUP="apg14-tuning-oltp"
```

### Inicialización de la base de datos de prueba

Si aún no has inicializado la base de datos de `pgbench`, ejecuta:

```bash
# Crear la base de datos destino (si no existe)
psql -h $PGHOST -U $PGUSER -d postgres -c "CREATE DATABASE pgbench_db;"

# Inicializar pgbench con escala 10 (~1.5 millones de filas en pgbench_accounts)
pgbench -h $PGHOST -U $PGUSER -d pgbench_db -i -s 10
```

**Salida esperada de inicialización:**
```
dropping old tables...
creating tables...
generating data (client-side)...
1000000 of 1000000 tuples (100%) done (elapsed 45.23 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 58.41 s (drop tables 0.01 s, create tables 0.06 s, ...)
```

---

## Pasos del Laboratorio

### Paso 1 — Capturar el Baseline con la Configuración por Defecto

**Objetivo:** Ejecutar una prueba de rendimiento con `pgbench` usando la configuración por defecto de Aurora PostgreSQL 14 y registrar los valores de referencia (TPS y latencia) que servirán como punto de comparación.

#### Instrucciones

1. Verifica la configuración actual de parámetros clave antes de cualquier cambio:

```sql
-- Ejecutar en psql
\c pgbench_db

SELECT name, setting, unit, source
FROM pg_settings
WHERE name IN (
  'work_mem',
  'effective_cache_size',
  'wal_buffers',
  'checkpoint_completion_target',
  'synchronous_commit',
  'max_connections',
  'autovacuum_vacuum_scale_factor',
  'autovacuum_analyze_scale_factor',
  'autovacuum_vacuum_cost_delay'
)
ORDER BY name;
```

2. Registra los valores baseline en la **Tabla de Matriz de Parámetros** (ver Paso 5).

3. Ejecuta la prueba baseline con `pgbench` — 3 clientes, 2 hilos, 60 segundos:

```bash
# Prueba baseline — guardar resultado en archivo
pgbench \
  -h $PGHOST \
  -U $PGUSER \
  -d pgbench_db \
  -c 3 \
  -j 2 \
  -T 60 \
  -P 10 \
  --report-per-command \
  2>&1 | tee /tmp/pgbench_baseline.txt
```

4. Captura métricas del `pg_stat_bgwriter` como referencia:

```sql
SELECT
  checkpoints_timed,
  checkpoints_req,
  checkpoint_write_time,
  checkpoint_sync_time,
  buffers_checkpoint,
  buffers_clean,
  maxwritten_clean,
  buffers_backend,
  buffers_alloc,
  stats_reset
FROM pg_stat_bgwriter;
```

> Guarda esta salida. Después de cada prueba resetearás las estadísticas con:
> ```sql
> SELECT pg_stat_reset_shared('bgwriter');
> ```

#### Salida esperada

```
pgbench (14.x)
starting vacuum...end.
progress: 10.0 s, 245.3 tps, lat 12.23 ms stddev 4.56
progress: 20.0 s, 251.7 tps, lat 11.92 ms stddev 4.21
...
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
query mode: simple
number of clients: 3
number of threads: 2
duration: 60 s
number of transactions actually processed: 14923
latency average = 12.06 ms
latency stddev = 4.38 ms
tps = 248.71 (including connections establishing)
tps = 248.85 (excluding connections establishing)
```

#### Verificación

```bash
# Confirmar que el archivo baseline fue guardado
cat /tmp/pgbench_baseline.txt | grep "tps ="
```

---

### Paso 2 — Crear el DB Parameter Group Personalizado

**Objetivo:** Crear un DB parameter group basado en la familia `aurora-postgresql14` que servirá como contenedor para todos los ajustes del laboratorio.

#### Instrucciones

1. Crea el parameter group personalizado:

```bash
aws rds create-db-parameter-group \
  --db-parameter-group-name "$PG_GROUP" \
  --db-parameter-group-family "$PG_FAMILY" \
  --description "Tuning OLTP para Aurora PostgreSQL 14 - Lab 03-00-01" \
  --region $AWS_REGION
```

2. Verifica que el grupo fue creado correctamente:

```bash
aws rds describe-db-parameter-groups \
  --db-parameter-group-name "$PG_GROUP" \
  --region $AWS_REGION \
  --query 'DBParameterGroups[0].{Nombre:DBParameterGroupName,Familia:DBParameterGroupFamily,Desc:Description}' \
  --output table
```

3. Inspecciona los parámetros disponibles en el grupo (filtra solo los modificables):

```bash
aws rds describe-db-parameters \
  --db-parameter-group-name "$PG_GROUP" \
  --source "user" \
  --region $AWS_REGION \
  --query 'Parameters[*].{Param:ParameterName,Valor:ParameterValue,Metodo:ApplyMethod,Tipo:IsModifiable}' \
  --output table | head -40
```

4. Consulta los valores actuales de parámetros específicos que modificaremos:

```bash
aws rds describe-db-parameters \
  --db-parameter-group-name "$PG_GROUP" \
  --region $AWS_REGION \
  --query 'Parameters[?contains(`["work_mem","effective_cache_size","wal_buffers","checkpoint_completion_target","autovacuum_vacuum_scale_factor","autovacuum_analyze_scale_factor","autovacuum_vacuum_cost_delay","synchronous_commit"]`, ParameterName)]' \
  --output json | jq '.[] | {param: .ParameterName, value: .ParameterValue, apply: .ApplyMethod}'
```

#### Salida esperada

```
---------------------------------------------------------
|           DescribeDBParameterGroups                   |
+---------------------------+---------------------------+
|  Nombre                   |  apg14-tuning-oltp        |
|  Familia                  |  aurora-postgresql14       |
|  Desc                     |  Tuning OLTP para Aurora  |
+---------------------------+---------------------------+
```

#### Verificación

```bash
# Confirmar existencia del grupo
aws rds describe-db-parameter-groups \
  --db-parameter-group-name "$PG_GROUP" \
  --region $AWS_REGION \
  --query 'DBParameterGroups[0].DBParameterGroupName' \
  --output text
# Debe retornar: apg14-tuning-oltp
```

---

### Paso 3 — Aplicar Parámetros de Memoria y Escritura (Dinámicos)

**Objetivo:** Configurar parámetros de memoria y escritura que aplican de forma inmediata (`ApplyMethod=immediate`) sin necesidad de reinicio, y medir su impacto en TPS y latencia.

#### Instrucciones

1. Aplica los parámetros dinámicos de memoria y escritura en un solo comando:

```bash
aws rds modify-db-parameter-group \
  --db-parameter-group-name "$PG_GROUP" \
  --region $AWS_REGION \
  --parameters \
    "ParameterName=work_mem,ParameterValue=16384,ApplyMethod=immediate" \
    "ParameterName=effective_cache_size,ParameterValue=3145728,ApplyMethod=immediate" \
    "ParameterName=wal_buffers,ParameterValue=2048,ApplyMethod=immediate" \
    "ParameterName=checkpoint_completion_target,ParameterValue=0.9,ApplyMethod=immediate" \
    "ParameterName=synchronous_commit,ParameterValue=off,ApplyMethod=immediate"
```

> **Nota sobre unidades:** En Aurora PostgreSQL, `work_mem` se expresa en **kB** (16384 kB = 16 MB). `effective_cache_size` en kB (3145728 kB ≈ 3 GB, adecuado para `db.r6g.large` con 16 GB RAM). `wal_buffers` en bloques de 8 kB (2048 bloques = 16 MB).

> **⚠️ Advertencia sobre `synchronous_commit=off`:** Mejora el rendimiento de escritura pero introduce un riesgo de pérdida de hasta `wal_writer_delay` (200ms por defecto) de transacciones ante un crash. Úsalo solo en entornos donde esta pérdida sea aceptable (carga de datos, entornos de prueba).

2. Asocia el parameter group a la instancia writer:

```bash
aws rds modify-db-instance \
  --db-instance-identifier "$WRITER_ID" \
  --db-parameter-group-name "$PG_GROUP" \
  --apply-immediately \
  --region $AWS_REGION
```

3. Espera a que el estado de la instancia sea `available` y el parameter group esté `in-sync`:

```bash
# Monitorear estado — puede tardar 1-3 minutos
watch -n 15 "aws rds describe-db-instances \
  --db-instance-identifier $WRITER_ID \
  --region $AWS_REGION \
  --query 'DBInstances[0].{Estado:DBInstanceStatus,ParamGroup:DBParameterGroups[0].ParameterApplyStatus}' \
  --output table"
```

4. Verifica que los parámetros están activos en la sesión de psql:

```sql
SHOW work_mem;
SHOW effective_cache_size;
SHOW wal_buffers;
SHOW checkpoint_completion_target;
SHOW synchronous_commit;
```

5. Resetea las estadísticas del bgwriter antes de la prueba:

```sql
SELECT pg_stat_reset_shared('bgwriter');
```

6. Ejecuta la prueba con los parámetros de memoria ajustados:

```bash
pgbench \
  -h $PGHOST \
  -U $PGUSER \
  -d pgbench_db \
  -c 3 \
  -j 2 \
  -T 60 \
  -P 10 \
  2>&1 | tee /tmp/pgbench_memoria_ajustada.txt
```

7. Captura nuevamente las estadísticas del bgwriter:

```sql
SELECT
  checkpoints_timed,
  checkpoints_req,
  checkpoint_write_time,
  checkpoint_sync_time,
  buffers_checkpoint,
  buffers_backend,
  buffers_alloc
FROM pg_stat_bgwriter;
```

#### Salida esperada

```
-- psql: verificación de parámetros
 work_mem
----------
 16MB
(1 row)

 effective_cache_size
----------------------
 3GB
(1 row)

-- pgbench: mejora esperada con synchronous_commit=off
tps = 312.45 (including connections establishing)
latency average = 9.60 ms
```

#### Verificación

```bash
# Comparar TPS baseline vs. ajustado
echo "=== BASELINE ===" && grep "tps =" /tmp/pgbench_baseline.txt
echo "=== MEMORIA AJUSTADA ===" && grep "tps =" /tmp/pgbench_memoria_ajustada.txt
```

---

### Paso 4 — Configurar y Verificar Autovacuum Agresivo

**Objetivo:** Ajustar los parámetros de autovacuum para que opere con mayor frecuencia bajo carga OLTP, reduciendo el riesgo de table bloat y manteniendo estadísticas actualizadas. Verificar el efecto en `pg_stat_user_tables`.

#### Instrucciones

1. Consulta el estado actual del autovacuum en las tablas de pgbench:

```sql
SELECT
  schemaname,
  relname,
  n_live_tup,
  n_dead_tup,
  last_autovacuum,
  last_autoanalyze,
  autovacuum_count,
  autoanalyze_count
FROM pg_stat_user_tables
WHERE relname LIKE 'pgbench%'
ORDER BY n_dead_tup DESC;
```

2. Aplica los parámetros de autovacuum optimizados:

```bash
aws rds modify-db-parameter-group \
  --db-parameter-group-name "$PG_GROUP" \
  --region $AWS_REGION \
  --parameters \
    "ParameterName=autovacuum_vacuum_scale_factor,ParameterValue=0.01,ApplyMethod=immediate" \
    "ParameterName=autovacuum_analyze_scale_factor,ParameterValue=0.005,ApplyMethod=immediate" \
    "ParameterName=autovacuum_vacuum_cost_delay,ParameterValue=2,ApplyMethod=immediate" \
    "ParameterName=autovacuum_vacuum_cost_limit,ParameterValue=400,ApplyMethod=immediate" \
    "ParameterName=autovacuum_max_workers,ParameterValue=4,ApplyMethod=immediate"
```

> **Explicación de los valores:**
> - `autovacuum_vacuum_scale_factor=0.01` → Dispara vacuum cuando el 1% de las filas son dead tuples (vs. 20% por defecto).
> - `autovacuum_analyze_scale_factor=0.005` → Dispara analyze cuando el 0.5% de las filas cambian (vs. 10% por defecto).
> - `autovacuum_vacuum_cost_delay=2` → Reduce la pausa entre rondas de I/O de vacuum (de 20ms a 2ms), haciéndolo más agresivo.
> - `autovacuum_vacuum_cost_limit=400` → Aumenta el trabajo permitido por ronda (de 200 a 400).

3. Verifica que los parámetros están efectivos:

```sql
SHOW autovacuum_vacuum_scale_factor;
SHOW autovacuum_analyze_scale_factor;
SHOW autovacuum_vacuum_cost_delay;
```

4. Genera carga de UPDATE para crear dead tuples y observar el autovacuum:

```bash
# Ejecutar pgbench con mayor concurrencia para generar dead tuples
pgbench \
  -h $PGHOST \
  -U $PGUSER \
  -d pgbench_db \
  -c 10 \
  -j 4 \
  -T 30 \
  2>&1 | tee /tmp/pgbench_carga_autovacuum.txt
```

5. Monitorea el autovacuum en tiempo real durante la carga:

```sql
-- Ejecutar en una segunda sesión psql mientras corre pgbench
SELECT
  pid,
  datname,
  relname,
  phase,
  heap_blks_scanned,
  heap_blks_vacuumed,
  index_vacuum_count
FROM pg_stat_progress_vacuum;
```

6. Después de la carga, verifica el estado de las tablas:

```sql
SELECT
  relname,
  n_live_tup,
  n_dead_tup,
  ROUND(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS pct_dead,
  last_autovacuum,
  last_autoanalyze,
  autovacuum_count,
  autoanalyze_count
FROM pg_stat_user_tables
WHERE relname LIKE 'pgbench%'
ORDER BY n_dead_tup DESC;
```

#### Salida esperada

```sql
-- pg_stat_user_tables después de carga y autovacuum agresivo
     relname      | n_live_tup | n_dead_tup | pct_dead |       last_autovacuum        | autovacuum_count
------------------+------------+------------+----------+------------------------------+-----------------
 pgbench_accounts |    1000000 |        342 |     0.03 | 2024-01-15 14:23:45.123+00   |               8
 pgbench_history  |      18450 |          0 |     0.00 | 2024-01-15 14:23:12.456+00   |               3
 pgbench_tellers  |        100 |          2 |     1.96 | 2024-01-15 14:23:45.789+00   |               8
 pgbench_branches |         10 |          0 |     0.00 | 2024-01-15 14:22:58.321+00   |               2
```

#### Verificación

```sql
-- Confirmar que autovacuum_count aumentó respecto al inicio del laboratorio
-- y que n_dead_tup se mantiene bajo
SELECT relname, autovacuum_count, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'pgbench_accounts';
```

---

### Paso 5 — Prueba de Carga con Mayor Concurrencia y Análisis de max_connections

**Objetivo:** Simular una carga OLTP con mayor número de clientes para observar el impacto de `max_connections` en el rendimiento y la relación con el consumo de memoria.

#### Instrucciones

1. Consulta el valor actual de `max_connections`:

```sql
SHOW max_connections;
-- En Aurora db.r6g.large el valor por defecto suele ser ~683
```

2. Verifica el número de conexiones activas actuales:

```sql
SELECT
  count(*) AS total_conexiones,
  count(*) FILTER (WHERE state = 'active') AS activas,
  count(*) FILTER (WHERE state = 'idle') AS inactivas,
  count(*) FILTER (WHERE state = 'idle in transaction') AS idle_en_transaccion
FROM pg_stat_activity
WHERE datname = 'pgbench_db';
```

3. Ejecuta una prueba con mayor concurrencia (20 clientes):

```bash
pgbench \
  -h $PGHOST \
  -U $PGUSER \
  -d pgbench_db \
  -c 20 \
  -j 4 \
  -T 60 \
  -P 10 \
  2>&1 | tee /tmp/pgbench_20clientes.txt
```

4. Mientras corre pgbench, en otra terminal monitorea el consumo de memoria de conexiones:

```sql
-- Memoria estimada por conexión (work_mem * conexiones activas = presión de memoria)
SELECT
  count(*) AS conexiones_activas,
  current_setting('work_mem') AS work_mem_por_sesion,
  pg_size_pretty(
    count(*) * pg_size_bytes(current_setting('work_mem'))
  ) AS memoria_work_mem_total_estimada
FROM pg_stat_activity
WHERE state = 'active' AND datname = 'pgbench_db';
```

5. Ejecuta también una prueba con 50 clientes para observar la degradación:

```bash
pgbench \
  -h $PGHOST \
  -U $PGUSER \
  -d pgbench_db \
  -c 50 \
  -j 8 \
  -T 60 \
  -P 10 \
  2>&1 | tee /tmp/pgbench_50clientes.txt
```

6. Compara los resultados de las tres pruebas:

```bash
echo "╔══════════════════════════════════════════╗"
echo "║     COMPARATIVA DE RENDIMIENTO           ║"
echo "╚══════════════════════════════════════════╝"
echo ""
echo "--- BASELINE (3 clientes, config default) ---"
grep "tps =\|latency average" /tmp/pgbench_baseline.txt

echo ""
echo "--- MEMORIA AJUSTADA (3 clientes) ---"
grep "tps =\|latency average" /tmp/pgbench_memoria_ajustada.txt

echo ""
echo "--- 20 CLIENTES (config ajustada) ---"
grep "tps =\|latency average" /tmp/pgbench_20clientes.txt

echo ""
echo "--- 50 CLIENTES (config ajustada) ---"
grep "tps =\|latency average" /tmp/pgbench_50clientes.txt
```

#### Salida esperada

```
╔══════════════════════════════════════════╗
║     COMPARATIVA DE RENDIMIENTO           ║
╚══════════════════════════════════════════╝

--- BASELINE (3 clientes, config default) ---
latency average = 12.06 ms
tps = 248.71 (including connections establishing)

--- MEMORIA AJUSTADA (3 clientes) ---
latency average = 9.60 ms
tps = 312.45 (including connections establishing)

--- 20 CLIENTES (config ajustada) ---
latency average = 8.32 ms
tps = 2403.12 (including connections establishing)

--- 50 CLIENTES (config ajustada) ---
latency average = 11.45 ms
tps = 4367.89 (including connections establishing)
```

> **Nota:** Con 50 clientes es normal observar un aumento de latencia respecto a 20 clientes debido a la contención en locks de las tablas `pgbench_branches` y `pgbench_tellers`. Esto ilustra el límite práctico de concurrencia antes de necesitar connection pooling (RDS Proxy).

#### Verificación

```bash
# Extraer TPS de cada archivo para la matriz
for f in baseline memoria_ajustada 20clientes 50clientes; do
  echo -n "$f: "
  grep "tps = " /tmp/pgbench_${f}.txt | tail -1
done
```

---

### Paso 6 — Documentar la Matriz de Parámetros

**Objetivo:** Consolidar todos los resultados medidos en una matriz estructurada que compare valores baseline vs. ajustados e impacto en rendimiento.

#### Instrucciones

1. Ejecuta la siguiente consulta para capturar el estado final de todos los parámetros ajustados:

```sql
SELECT
  name AS parametro,
  setting AS valor_actual,
  unit AS unidad,
  source AS fuente,
  context AS contexto_aplicacion
FROM pg_settings
WHERE name IN (
  'work_mem',
  'effective_cache_size',
  'wal_buffers',
  'checkpoint_completion_target',
  'synchronous_commit',
  'max_connections',
  'autovacuum_vacuum_scale_factor',
  'autovacuum_analyze_scale_factor',
  'autovacuum_vacuum_cost_delay',
  'autovacuum_vacuum_cost_limit',
  'autovacuum_max_workers'
)
ORDER BY name;
```

2. Completa la siguiente **Matriz de Parámetros** con tus resultados medidos:

```
╔══════════════════════════════════════════════════════════════════════════════════════════════════╗
║                    MATRIZ DE PARÁMETROS — LAB 03-00-01                                         ║
╠══════════════════════════════════════════════╦══════════════════╦══════════════════╦════════════╣
║ Parámetro                                    ║ Valor Baseline   ║ Valor Ajustado   ║ Impacto    ║
╠══════════════════════════════════════════════╬══════════════════╬══════════════════╬════════════╣
║ work_mem                                     ║ 4 MB             ║ 16 MB            ║ +TPS sorts ║
║ effective_cache_size                         ║ 1 GB             ║ 3 GB             ║ +plan cache ║
║ wal_buffers                                  ║ 4 MB             ║ 16 MB            ║ -WriteIOPS ║
║ checkpoint_completion_target                 ║ 0.5              ║ 0.9              ║ -I/O spikes║
║ synchronous_commit                           ║ on               ║ off              ║ +TPS write ║
║ autovacuum_vacuum_scale_factor               ║ 0.2              ║ 0.01             ║ -dead_tup  ║
║ autovacuum_analyze_scale_factor              ║ 0.1              ║ 0.005            ║ +stats freq║
║ autovacuum_vacuum_cost_delay (ms)            ║ 20               ║ 2                ║ +vac speed ║
║ autovacuum_vacuum_cost_limit                 ║ 200              ║ 400              ║ +vac I/O   ║
╠══════════════════════════════════════════════╬══════════════════╬══════════════════╬════════════╣
║ TPS (3 clientes)                             ║ ~248             ║ ~312             ║ +25%       ║
║ Latencia media (3 clientes, ms)              ║ ~12.1            ║ ~9.6             ║ -20%       ║
╚══════════════════════════════════════════════╩══════════════════╩══════════════════╩════════════╝
```

3. Guarda la matriz en un archivo de texto:

```bash
# Capturar el estado final de parámetros a un archivo
psql -h $PGHOST -U $PGUSER -d pgbench_db -c "
SELECT name, setting, unit, source
FROM pg_settings
WHERE name IN (
  'work_mem','effective_cache_size','wal_buffers',
  'checkpoint_completion_target','synchronous_commit',
  'max_connections','autovacuum_vacuum_scale_factor',
  'autovacuum_analyze_scale_factor','autovacuum_vacuum_cost_delay'
)
ORDER BY name;" > /tmp/matriz_parametros_final.txt

echo "Matriz guardada en /tmp/matriz_parametros_final.txt"
cat /tmp/matriz_parametros_final.txt
```

#### Salida esperada

```
                  name                   | setting |  unit  |    source
-----------------------------------------+---------+--------+--------------
 autovacuum_analyze_scale_factor         | 0.005   |        | configuration
 autovacuum_vacuum_cost_delay            | 2       | ms     | configuration
 autovacuum_vacuum_scale_factor          | 0.01    |        | configuration
 checkpoint_completion_target            | 0.9     |        | configuration
 effective_cache_size                    | 3145728 | 8kB    | configuration
 max_connections                         | 683     |        | configuration
 synchronous_commit                      | off     |        | configuration
 wal_buffers                             | 2048    | 8kB    | configuration
 work_mem                                | 16384   | kB     | configuration
```

#### Verificación

```bash
# Confirmar que todos los parámetros muestran source = 'configuration'
grep -c "configuration" /tmp/matriz_parametros_final.txt
# Debe retornar: 9 (o el número de parámetros ajustados)
```

---

## Validación y Pruebas

Ejecuta las siguientes verificaciones finales para confirmar que el laboratorio fue completado correctamente:

### Verificación 1 — Parameter Group asociado y en sincronía

```bash
aws rds describe-db-instances \
  --db-instance-identifier "$WRITER_ID" \
  --region $AWS_REGION \
  --query 'DBInstances[0].DBParameterGroups' \
  --output table
```

**Resultado esperado:** El campo `ParameterApplyStatus` debe mostrar `in-sync`.

### Verificación 2 — Parámetros efectivos en la instancia

```sql
-- Todos deben mostrar source = 'configuration' (no 'default')
SELECT name, setting, source
FROM pg_settings
WHERE name IN ('work_mem', 'autovacuum_vacuum_scale_factor', 'checkpoint_completion_target')
  AND source != 'default';
```

**Resultado esperado:** 3 filas con `source = 'configuration'`.

### Verificación 3 — Autovacuum activo en tablas pgbench

```sql
SELECT relname, autovacuum_count, last_autovacuum
FROM pg_stat_user_tables
WHERE relname LIKE 'pgbench%'
  AND autovacuum_count > 0;
```

**Resultado esperado:** Al menos 3 tablas con `autovacuum_count > 0`.

### Verificación 4 — Archivos de resultados pgbench presentes

```bash
ls -lh /tmp/pgbench_*.txt
```

**Resultado esperado:** 4 archivos (baseline, memoria_ajustada, 20clientes, 50clientes).

### Verificación 5 — Comparación de TPS (mejora ≥ 10%)

```bash
BASELINE_TPS=$(grep "tps = " /tmp/pgbench_baseline.txt | head -1 | awk '{print $3}')
ADJUSTED_TPS=$(grep "tps = " /tmp/pgbench_memoria_ajustada.txt | head -1 | awk '{print $3}')
echo "Baseline TPS: $BASELINE_TPS"
echo "Adjusted TPS: $ADJUSTED_TPS"
python3 -c "b=$BASELINE_TPS; a=$ADJUSTED_TPS; print(f'Mejora: {((a-b)/b)*100:.1f}%')"
```

---

## Solución de Problemas

### Problema 1 — El parameter group muestra `pending-reboot` después de aplicar parámetros dinámicos

**Síntoma:**
```bash
aws rds describe-db-instances \
  --db-instance-identifier "$WRITER_ID" \
  --query 'DBInstances[0].DBParameterGroups[0].ParameterApplyStatus' \
  --output text
# Retorna: pending-reboot
```

**Causa:** Alguno de los parámetros modificados fue declarado como estático (`ApplyMethod=pending-reboot`) en la familia `aurora-postgresql14`, aunque en otras versiones sea dinámico. Esto es común con `wal_buffers` en ciertas versiones de Aurora, o si accidentalmente se incluyó un parámetro estático en el lote.

**Solución:**

```bash
# 1. Identificar qué parámetros están pendientes de reinicio
aws rds describe-db-parameters \
  --db-parameter-group-name "$PG_GROUP" \
  --source "user" \
  --region $AWS_REGION \
  --query 'Parameters[?ApplyMethod==`pending-reboot`].{Param:ParameterName,Valor:ParameterValue}' \
  --output table

# 2. Si el parámetro es realmente estático, planificar reinicio
# OPCIÓN A: Reinicio inmediato (causa ~30 seg de indisponibilidad)
aws rds reboot-db-instance \
  --db-instance-identifier "$WRITER_ID" \
  --region $AWS_REGION

# OPCIÓN B: Esperar ventana de mantenimiento (sin impacto inmediato)
# El parámetro se aplicará en el próximo mantenimiento programado.

# 3. Verificar después del reinicio
aws rds describe-db-instances \
  --db-instance-identifier "$WRITER_ID" \
  --region $AWS_REGION \
  --query 'DBInstances[0].DBParameterGroups[0].ParameterApplyStatus' \
  --output text
# Debe retornar: in-sync
```

---

### Problema 2 — pgbench falla con "too many connections" o errores de conexión bajo alta concurrencia

**Síntoma:**
```
pgbench: error: connection to server at "xxx.rds.amazonaws.com" (x.x.x.x),
port 5432 failed: FATAL:  remaining connection slots are reserved for
non-replication superuser connections
```
O bien, TPS cae drásticamente a 0 con muchos errores en la prueba de 50 clientes.

**Causa:** El número de clientes de pgbench supera los connection slots disponibles. Aurora reserva conexiones para el superusuario y la replicación. Con `max_connections=683` y múltiples herramientas conectadas simultáneamente (psql, DBeaver, pgAdmin), los slots disponibles para pgbench pueden ser insuficientes.

**Solución:**

```bash
# 1. Verificar conexiones activas antes de lanzar pgbench
psql -h $PGHOST -U $PGUSER -d pgbench_db -c "
SELECT count(*), state
FROM pg_stat_activity
GROUP BY state
ORDER BY count DESC;"

# 2. Cerrar conexiones idle no necesarias desde otras herramientas
# (cerrar DBeaver/pgAdmin si están conectados)

# 3. Reducir el número de clientes en pgbench a un valor seguro
# Regla práctica: no superar max_connections * 0.7
MAX_CONN=$(psql -h $PGHOST -U $PGUSER -d pgbench_db -t -c "SHOW max_connections;" | tr -d ' ')
SAFE_CLIENTS=$(python3 -c "print(int($MAX_CONN * 0.7))")
echo "Clientes seguros para pgbench: $SAFE_CLIENTS"

# 4. Relanzar pgbench con número seguro de clientes
pgbench \
  -h $PGHOST \
  -U $PGUSER \
  -d pgbench_db \
  -c 30 \
  -j 4 \
  -T 60 \
  2>&1 | tee /tmp/pgbench_30clientes_retry.txt

# 5. (Opcional) Verificar que no hay conexiones colgadas
psql -h $PGHOST -U $PGUSER -d pgbench_db -c "
SELECT pid, state, query_start, state_change, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND state_change < NOW() - INTERVAL '5 minutes';"
```

---

## Limpieza de Recursos

> **⚠️ Importante:** Ejecuta estos pasos al finalizar el laboratorio para evitar cargos innecesarios.

### Paso 1 — Restaurar el parameter group por defecto en la instancia

```bash
# Obtener el nombre del parameter group default para aurora-postgresql14
DEFAULT_PG=$(aws rds describe-db-parameter-groups \
  --region $AWS_REGION \
  --query "DBParameterGroups[?DBParameterGroupFamily=='aurora-postgresql14' && contains(DBParameterGroupName,'default')].DBParameterGroupName" \
  --output text | head -1)

echo "Parameter group por defecto: $DEFAULT_PG"

# Restaurar el parameter group default en la instancia
aws rds modify-db-instance \
  --db-instance-identifier "$WRITER_ID" \
  --db-parameter-group-name "$DEFAULT_PG" \
  --apply-immediately \
  --region $AWS_REGION
```

### Paso 2 — Eliminar el parameter group personalizado

```bash
# Esperar a que la instancia esté en sync con el default
sleep 30

# Eliminar el parameter group del laboratorio
aws rds delete-db-parameter-group \
  --db-parameter-group-name "$PG_GROUP" \
  --region $AWS_REGION

echo "Parameter group '$PG_GROUP' eliminado."
```

### Paso 3 — Limpiar la base de datos de prueba (opcional)

```bash
# Si no necesitas pgbench_db para laboratorios posteriores
psql -h $PGHOST -U $PGUSER -d postgres -c "DROP DATABASE IF EXISTS pgbench_db;"
```

### Paso 4 — Verificar la limpieza

```bash
# Confirmar que el parameter group fue eliminado
aws rds describe-db-parameter-groups \
  --db-parameter-group-name "$PG_GROUP" \
  --region $AWS_REGION 2>&1 | grep -i "not found\|DBParameterGroupNotFound"
# Debe mostrar un error de "not found" confirmando la eliminación
```

### Paso 5 — Limpiar archivos locales temporales

```bash
rm -f /tmp/pgbench_*.txt /tmp/matriz_parametros_final.txt
echo "Archivos temporales eliminados."
```

> **Nota:** Si el clúster Aurora fue creado exclusivamente para este módulo y no lo necesitas para laboratorios posteriores, ejecuta `terraform destroy` desde el directorio del módulo 3 para eliminar toda la infraestructura y detener completamente los costos.

---

## Resumen

En este laboratorio aplicaste el flujo completo de gestión de DB parameter groups en Aurora PostgreSQL 14:

1. **Capturaste un baseline reproducible** con `pgbench` para tener métricas de referencia comparables.
2. **Creaste un DB parameter group personalizado** (`apg14-tuning-oltp`) con la familia correcta `aurora-postgresql14`.
3. **Ajustaste parámetros de memoria y escritura** (`work_mem`, `effective_cache_size`, `wal_buffers`, `checkpoint_completion_target`, `synchronous_commit`) y mediste una mejora típica del 20-25% en TPS.
4. **Configuraste autovacuum agresivo** reduciendo `autovacuum_vacuum_scale_factor` y `autovacuum_analyze_scale_factor`, verificando su efecto en `pg_stat_user_tables` con dead tuples controlados.
5. **Analizaste el impacto de la concurrencia** en `max_connections` y la relación entre `work_mem` y presión de memoria bajo alta concurrencia.
6. **Documentaste una matriz de parámetros** que sirve como referencia para optimizaciones futuras.

### Conceptos Clave para Recordar

| Concepto | Takeaway |
|----------|----------|
| Parámetros dinámicos vs. estáticos | Los dinámicos aplican sin reinicio; los estáticos requieren planificación de ventana de mantenimiento |
| `work_mem` y concurrencia | Cada sort/hash por conexión puede usar hasta `work_mem`; con muchas conexiones, el consumo total puede ser enorme |
| `synchronous_commit=off` | Mejora escritura pero acepta pérdida de hasta ~200ms de transacciones ante crash |
| Autovacuum agresivo | Escala factors bajos + cost_delay bajo = vacuum frecuente; impacto en I/O debe monitorearse |
| `shared_buffers` en Aurora | A diferencia de PostgreSQL on-premises, Aurora gestiona el buffer pool de forma diferente; el parámetro existe pero su impacto es menor que en instalaciones estándar |

### Recursos Adicionales

- [Trabajar con grupos de parámetros de BD — AWS Docs](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithParamGroups.html)
- [Parámetros de Aurora PostgreSQL — AWS Docs](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Reference.ParameterGroups.html)
- [pgbench — PostgreSQL Documentation](https://www.postgresql.org/docs/14/pgbench.html)
- [Tuning autovacuum en PostgreSQL — AWS Blog](https://aws.amazon.com/blogs/database/understanding-autovacuum-in-amazon-rds-for-postgresql-environments/)
- [CLI: describe-db-parameters](https://docs.aws.amazon.com/cli/latest/reference/rds/describe-db-parameters.html)

---

*Lab 03-00-01 — Módulo 3: Optimización de parámetros y conexiones | Aurora PostgreSQL Performance Engineering*

---

# Gestión de conexiones con RDS Proxy

## Metadatos

| Campo            | Valor                                      |
|------------------|--------------------------------------------|
| **Duración**     | 35 minutos                                 |
| **Complejidad**  | Media                                      |
| **Nivel Bloom**  | Aplicar (Apply)                            |
| **Módulo**       | 3 — Optimización de parámetros y conexiones |
| **Laboratorio**  | 03-00-02                                   |

---

## Descripción General

En este laboratorio desplegarás RDS Proxy como capa de gestión de conexiones frente a un clúster Aurora PostgreSQL, configurando los parámetros clave de connection pooling (`max_connections_percent`, `connection_borrow_timeout`, `idle_client_timeout`). Realizarás pruebas comparativas con `pgbench` y un script Python para medir el comportamiento bajo distintos niveles de concurrencia (10, 50, 100 y 500 conexiones) tanto en conexión directa como a través del proxy. Finalmente, analizarás las métricas específicas de RDS Proxy en CloudWatch (`ClientConnections`, `DatabaseConnections`, `QueryRequests`) para comprender cómo el proxy absorbe picos de carga y protege al motor Aurora.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio serás capaz de:

- [ ] Configurar y desplegar RDS Proxy para un clúster Aurora PostgreSQL, verificando la conectividad a través del endpoint del proxy.
- [ ] Demostrar cuantitativamente el beneficio del connection pooling de RDS Proxy bajo carga de alta concurrencia comparado con conexión directa.
- [ ] Implementar buenas prácticas de gestión de conexiones: `idle_client_timeout`, `max_connections_percent` y modos de pooling (transaction vs. session).
- [ ] Simular un escenario de saturación de conexiones y verificar cómo RDS Proxy mitiga el problema.
- [ ] Analizar métricas de RDS Proxy en CloudWatch para identificar el comportamiento del pool bajo carga.

---

## Prerrequisitos

### Conocimientos Previos

- Comprensión de DB parameter groups en Aurora PostgreSQL (Lección 3.1).
- Familiaridad con `pgbench` y conexiones PostgreSQL mediante `psql`.
- Conocimiento básico de AWS IAM, Secrets Manager y VPC.
- Experiencia básica con Python y la librería `psycopg2`.

### Acceso y Recursos AWS

- Clúster Aurora PostgreSQL (versión 14 o 15) operativo con al menos una instancia writer (`db.r6g.large` recomendada).
- AWS Secrets Manager con un secreto que contenga las credenciales de la base de datos (usuario/contraseña).
- IAM Role con las políticas `AmazonRDSFullAccess` y permisos sobre Secrets Manager (`secretsmanager:GetSecretValue`).
- VPC con al menos dos subnets privadas en zonas de disponibilidad distintas.
- Subnet group de RDS que incluya esas subnets privadas.
- Security Group que permita tráfico TCP en el puerto 5432 desde la instancia/máquina del estudiante hacia el proxy.
- AWS CLI 2.x configurada con credenciales IAM de estudiante (no root).
- Python 3.9+ con `psycopg2-binary` y `boto3` instalados.
- `pgbench` 14.x incluido en las herramientas cliente de PostgreSQL.

---

## Entorno de Laboratorio

### Hardware Mínimo Recomendado

| Recurso      | Mínimo                                        |
|--------------|-----------------------------------------------|
| RAM          | 8 GB (para simular concurrencia local)        |
| CPU          | 4 núcleos                                     |
| Almacenamiento | 10 GB libres (logs y resultados)            |
| Red          | 10 Mbps estables hacia AWS                    |
| Pantalla     | 1920×1080                                     |

### Software Requerido

| Herramienta         | Versión          |
|---------------------|------------------|
| AWS CLI             | 2.x              |
| psql                | 14.x o superior  |
| pgbench             | 14.x o superior  |
| Python              | 3.9 o superior   |
| psycopg2-binary     | 2.9.x            |
| boto3               | 1.26.x o superior|
| jq                  | 1.6 o superior   |
| DBeaver / pgAdmin 4 | Opcional         |

### Variables de Entorno Globales

Antes de comenzar, exporta las siguientes variables en tu terminal. Sustitúyelas con los valores reales de tu entorno:

```bash
# ─── Configuración del clúster Aurora ───────────────────────────────────────
export AWS_REGION="us-east-1"
export CLUSTER_ID="aurora-pg-lab-cluster"
export WRITER_ENDPOINT="aurora-pg-lab-cluster.cluster-xxxxxxxxxxxx.us-east-1.rds.amazonaws.com"
export DB_PORT="5432"
export DB_NAME="labdb"
export DB_USER="labadmin"

# ─── Configuración de RDS Proxy ─────────────────────────────────────────────
export PROXY_NAME="aurora-pg-proxy-lab"
export SECRET_ARN="arn:aws:secretsmanager:us-east-1:123456789012:secret:aurora-pg-lab-secret-XXXXXX"
export PROXY_ROLE_ARN="arn:aws:iam::123456789012:role/rds-proxy-lab-role"
export SUBNET_IDS="subnet-aaaa1111,subnet-bbbb2222"
export SECURITY_GROUP_ID="sg-0123456789abcdef0"

# ─── Directorio de trabajo ───────────────────────────────────────────────────
export LAB_DIR="$HOME/lab-03-00-02"
mkdir -p "$LAB_DIR/results"
cd "$LAB_DIR"
```

### Verificación del Entorno Inicial

```bash
# Verificar AWS CLI configurada
aws sts get-caller-identity --query "Account" --output text

# Verificar conectividad al writer de Aurora
psql "host=$WRITER_ENDPOINT port=$DB_PORT dbname=$DB_NAME user=$DB_USER" \
  -c "SELECT version();" -c "SHOW max_connections;"

# Verificar pgbench disponible
pgbench --version

# Verificar Python y librerías
python3 -c "import psycopg2; import boto3; print('OK')"
```

---

## Pasos del Laboratorio

---

### Paso 1 — Preparar la Base de Datos y Revisar Conexiones Actuales

**Objetivo:** Inicializar el esquema de `pgbench` en Aurora y establecer una línea base del comportamiento de conexiones directas antes de introducir el proxy.

#### Instrucciones

**1.1 — Inicializar el esquema de pgbench:**

```bash
pgbench -h "$WRITER_ENDPOINT" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" \
  --initialize --scale=10 2>&1 | tee "$LAB_DIR/results/pgbench_init.log"
```

> **Nota:** `--scale=10` genera aproximadamente 1 millón de filas en `pgbench_accounts`, suficiente para pruebas de concurrencia representativas.

**1.2 — Revisar el valor actual de `max_connections` en Aurora:**

```bash
psql "host=$WRITER_ENDPOINT port=$DB_PORT dbname=$DB_NAME user=$DB_USER" <<'EOF'
SELECT name, setting, unit, context
FROM pg_settings
WHERE name IN ('max_connections', 'superuser_reserved_connections');

SELECT count(*) AS conexiones_activas FROM pg_stat_activity;
EOF
```

**1.3 — Registrar la línea base de conexiones disponibles:**

```bash
psql "host=$WRITER_ENDPOINT port=$DB_PORT dbname=$DB_NAME user=$DB_USER" \
  -c "SELECT current_setting('max_connections')::int - count(*) AS conexiones_disponibles
      FROM pg_stat_activity;" \
  2>&1 | tee "$LAB_DIR/results/baseline_connections.txt"
```

**1.4 — Ejecutar una prueba de carga directa de referencia (10 clientes, 60 segundos):**

```bash
pgbench -h "$WRITER_ENDPOINT" -p "$DB_PORT" -U "$DB_USER" -d "$DB_NAME" \
  --client=10 --jobs=4 --time=60 --protocol=prepared \
  --report-per-command \
  2>&1 | tee "$LAB_DIR/results/direct_10clients.txt"
```

#### Salida Esperada

```
starting vacuum...end.
...
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 10
number of clients: 10
number of threads: 4
duration: 60 s
number of transactions actually processed: XXXX
latency average = XX.XXX ms
tps = XX.XXXXXX (without initial connection time)
```

#### Verificación

```bash
# Confirmar que las tablas de pgbench existen
psql "host=$WRITER_ENDPOINT port=$DB_PORT dbname=$DB_NAME user=$DB_USER" \
  -c "\dt pgbench_*"
```

Debes ver: `pgbench_accounts`, `pgbench_branches`, `pgbench_history`, `pgbench_tellers`.

---

### Paso 2 — Crear el RDS Proxy mediante AWS CLI

**Objetivo:** Desplegar RDS Proxy configurado con los parámetros de pooling adecuados para Aurora PostgreSQL.

#### Instrucciones

**2.1 — Crear el RDS Proxy:**

```bash
aws rds create-db-proxy \
  --db-proxy-name "$PROXY_NAME" \
  --engine-family POSTGRESQL \
  --auth '[
    {
      "AuthScheme": "SECRETS",
      "SecretArn": "'"$SECRET_ARN"'",
      "IAMAuth": "DISABLED"
    }
  ]' \
  --role-arn "$PROXY_ROLE_ARN" \
  --vpc-subnet-ids $(echo $SUBNET_IDS | tr ',' ' ') \
  --vpc-security-group-ids "$SECURITY_GROUP_ID" \
  --require-tls \
  --idle-client-timeout 1800 \
  --debug-logging false \
  --tags Key=Lab,Value=03-00-02 Key=Environment,Value=training \
  --region "$AWS_REGION" \
  2>&1 | tee "$LAB_DIR/results/proxy_creation.json"
```

**2.2 — Asociar el proxy al clúster Aurora (target group):**

```bash
# Obtener el nombre del target group creado automáticamente
TARGET_GROUP_NAME=$(aws rds describe-db-proxy-target-groups \
  --db-proxy-name "$PROXY_NAME" \
  --region "$AWS_REGION" \
  --query "TargetGroups[0].TargetGroupName" \
  --output text)

echo "Target Group: $TARGET_GROUP_NAME"

# Asociar el clúster Aurora al target group con parámetros de pooling
aws rds modify-db-proxy-target-group \
  --db-proxy-name "$PROXY_NAME" \
  --target-group-name "$TARGET_GROUP_NAME" \
  --connection-pool-config '{
    "MaxConnectionsPercent": 80,
    "MaxIdleConnectionsPercent": 50,
    "ConnectionBorrowTimeout": 120,
    "SessionPinningFilters": []
  }' \
  --region "$AWS_REGION" \
  2>&1 | tee "$LAB_DIR/results/target_group_config.json"
```

> **Parámetros clave:**
> - `MaxConnectionsPercent=80`: El proxy usará como máximo el 80% de `max_connections` de Aurora.
> - `MaxIdleConnectionsPercent=50`: Mantiene hasta el 50% de conexiones idle en el pool.
> - `ConnectionBorrowTimeout=120`: Espera hasta 120 segundos para obtener una conexión del pool antes de retornar error.

**2.3 — Registrar el clúster como target:**

```bash
aws rds register-db-proxy-targets \
  --db-proxy-name "$PROXY_NAME" \
  --db-cluster-identifiers "$CLUSTER_ID" \
  --region "$AWS_REGION"
```

**2.4 — Esperar a que el proxy esté disponible:**

```bash
echo "Esperando que el proxy esté en estado 'available'..."
aws rds wait db-proxy-available \
  --db-proxy-name "$PROXY_NAME" \
  --region "$AWS_REGION"

# Obtener el endpoint del proxy
PROXY_ENDPOINT=$(aws rds describe-db-proxies \
  --db-proxy-name "$PROXY_NAME" \
  --region "$AWS_REGION" \
  --query "DBProxies[0].Endpoint" \
  --output text)

echo "Proxy endpoint: $PROXY_ENDPOINT"
export PROXY_ENDPOINT
```

> **Tiempo estimado:** El proxy tarda entre 5 y 10 minutos en estar disponible. Mientras esperas, revisa la consola AWS en **RDS → Proxies** para observar el progreso.

#### Salida Esperada

```json
{
    "DBProxy": {
        "DBProxyName": "aurora-pg-proxy-lab",
        "DBProxyArn": "arn:aws:rds:us-east-1:...",
        "Status": "creating",
        "Endpoint": "aurora-pg-proxy-lab.proxy-xxxxxxxxxxxx.us-east-1.rds.amazonaws.com",
        ...
    }
}
```

#### Verificación

```bash
# Verificar estado del proxy
aws rds describe-db-proxies \
  --db-proxy-name "$PROXY_NAME" \
  --region "$AWS_REGION" \
  --query "DBProxies[0].{Estado:Status,Endpoint:Endpoint,TLS:RequireTLS}" \
  --output table

# Verificar estado de los targets
aws rds describe-db-proxy-targets \
  --db-proxy-name "$PROXY_NAME" \
  --region "$AWS_REGION" \
  --query "Targets[*].{Estado:TargetHealth.State,Descripcion:TargetHealth.Description,Endpoint:Endpoint}" \
  --output table
```

El estado debe ser `available` para el proxy y `AVAILABLE` para el target.

---

### Paso 3 — Verificar Conectividad a través del Proxy

**Objetivo:** Confirmar que el proxy enruta correctamente las conexiones al clúster Aurora y que la autenticación funciona.

#### Instrucciones

**3.1 — Conectar al clúster a través del proxy con psql:**

```bash
psql "host=$PROXY_ENDPOINT port=$DB_PORT dbname=$DB_NAME user=$DB_USER sslmode=require" \
  -c "SELECT 'Conectado via RDS Proxy' AS estado;" \
  -c "SELECT inet_server_addr() AS ip_servidor;" \
  -c "SELECT count(*) AS tablas_pgbench FROM pg_tables WHERE tablename LIKE 'pgbench%';"
```

**3.2 — Verificar desde Aurora que la conexión aparece con origen del proxy:**

```bash
# Conectar directamente al writer para ver pg_stat_activity
psql "host=$WRITER_ENDPOINT port=$DB_PORT dbname=$DB_NAME user=$DB_USER" <<'EOF'
SELECT pid,
       usename,
       application_name,
       client_addr,
       state,
       backend_type
FROM pg_stat_activity
WHERE usename = current_user
ORDER BY backend_start DESC
LIMIT 10;
EOF
```

> Las conexiones originadas desde el proxy aparecerán con una IP de la VPC (no la IP de tu máquina local), lo que confirma que el proxy actúa como intermediario.

**3.3 — Verificar la configuración de pooling aplicada:**

```bash
aws rds describe-db-proxy-target-groups \
  --db-proxy-name "$PROXY_NAME" \
  --region "$AWS_REGION" \
  --query "TargetGroups[0].ConnectionPoolConfig" \
  --output table
```

#### Salida Esperada

```
 estado
────────────────────────
 Conectado via RDS Proxy
(1 row)

─────────────────────────────────────────
 MaxConnectionsPercent      │ 80
 MaxIdleConnectionsPercent  │ 50
 ConnectionBorrowTimeout    │ 120
─────────────────────────────────────────
```

#### Verificación

```bash
# Prueba rápida de latencia a través del proxy
time psql "host=$PROXY_ENDPOINT port=$DB_PORT dbname=$DB_NAME user=$DB_USER sslmode=require" \
  -c "SELECT 1;" -q
```

El tiempo de respuesta debe ser inferior a 100 ms en condiciones normales.

---

### Paso 4 — Prueba Comparativa de Rendimiento: Conexión Directa vs. RDS Proxy

**Objetivo:** Medir cuantitativamente el impacto del connection pooling bajo diferentes niveles de concurrencia usando `pgbench`.

#### Instrucciones

**4.1 — Crear el script de prueba comparativa:**

```bash
cat > "$LAB_DIR/run_comparison.sh" << 'SCRIPT'
#!/bin/bash
# Prueba comparativa: conexión directa vs RDS Proxy
# Parámetros: $1=endpoint, $2=etiqueta, $3=clientes, $4=duración

ENDPOINT=$1
LABEL=$2
CLIENTS=$3
DURATION=${4:-30}

echo "=== Prueba: $LABEL | Clientes: $CLIENTS | Duración: ${DURATION}s ==="
pgbench -h "$ENDPOINT" \
        -p "${DB_PORT}" \
        -U "${DB_USER}" \
        -d "${DB_NAME}" \
        --client="$CLIENTS" \
        --jobs=4 \
        --time="$DURATION" \
        --protocol=prepared \
        --no-vacuum \
        2>&1 | grep -E "(tps|latency average|number of clients|connection time)"
echo ""
SCRIPT
chmod +x "$LAB_DIR/run_comparison.sh"
```

**4.2 — Ejecutar pruebas con 10 clientes (directa y proxy):**

```bash
# Conexión directa - 10 clientes
"$LAB_DIR/run_comparison.sh" "$WRITER_ENDPOINT" "DIRECTA" 10 30 \
  2>&1 | tee "$LAB_DIR/results/direct_c10.txt"

# A través del proxy - 10 clientes
"$LAB_DIR/run_comparison.sh" "$PROXY_ENDPOINT" "PROXY" 10 30 \
  2>&1 | tee "$LAB_DIR/results/proxy_c10.txt"
```

**4.3 — Ejecutar pruebas con 50 clientes:**

```bash
"$LAB_DIR/run_comparison.sh" "$WRITER_ENDPOINT" "DIRECTA" 50 30 \
  2>&1 | tee "$LAB_DIR/results/direct_c50.txt"

"$LAB_DIR/run_comparison.sh" "$PROXY_ENDPOINT" "PROXY" 50 30 \
  2>&1 | tee "$LAB_DIR/results/proxy_c50.txt"
```

**4.4 — Ejecutar pruebas con 100 clientes:**

```bash
"$LAB_DIR/run_comparison.sh" "$WRITER_ENDPOINT" "DIRECTA" 100 30 \
  2>&1 | tee "$LAB_DIR/results/direct_c100.txt"

"$LAB_DIR/run_comparison.sh" "$PROXY_ENDPOINT" "PROXY" 100 30 \
  2>&1 | tee "$LAB_DIR/results/proxy_c100.txt"
```

**4.5 — Consolidar y mostrar resultados comparativos:**

```bash
echo ""
echo "═══════════════════════════════════════════════════════════"
echo "  RESUMEN COMPARATIVO: CONEXIÓN DIRECTA vs RDS PROXY"
echo "═══════════════════════════════════════════════════════════"
echo ""
for CLIENTS in 10 50 100; do
  echo "--- $CLIENTS Clientes ---"
  echo -n "  DIRECTA  → TPS: "
  grep "tps = " "$LAB_DIR/results/direct_c${CLIENTS}.txt" | awk '{print $3}' | head -1
  echo -n "  PROXY    → TPS: "
  grep "tps = " "$LAB_DIR/results/proxy_c${CLIENTS}.txt" | awk '{print $3}' | head -1
  echo ""
done
```

#### Salida Esperada

```
═══════════════════════════════════════════════════════════
  RESUMEN COMPARATIVO: CONEXIÓN DIRECTA vs RDS PROXY
═══════════════════════════════════════════════════════════

--- 10 Clientes ---
  DIRECTA  → TPS: 245.123456
  PROXY    → TPS: 238.456789

--- 50 Clientes ---
  DIRECTA  → TPS: 312.456789
  PROXY    → TPS: 358.123456

--- 100 Clientes ---
  DIRECTA  → TPS: 287.654321
  PROXY    → TPS: 342.987654
```

> **Interpretación:** A bajo número de clientes (10), el overhead del proxy puede ser marginal. A medida que aumenta la concurrencia (50-100), el proxy debería mostrar ventajas gracias al connection pooling, reduciendo el overhead de establecimiento de conexiones en Aurora.

#### Verificación

```bash
# Monitorear conexiones activas en Aurora durante las pruebas
psql "host=$WRITER_ENDPOINT port=$DB_PORT dbname=$DB_NAME user=$DB_USER" \
  -c "SELECT count(*) AS total_conexiones,
             count(*) FILTER (WHERE state = 'active') AS activas,
             count(*) FILTER (WHERE state = 'idle') AS idle
      FROM pg_stat_activity
      WHERE backend_type = 'client backend';"
```

---

### Paso 5 — Simulación de Saturación de Conexiones

**Objetivo:** Demostrar cómo RDS Proxy protege a Aurora cuando el número de conexiones cliente supera la capacidad del motor.

#### Instrucciones

**5.1 — Crear el script Python de saturación:**

```bash
cat > "$LAB_DIR/connection_saturation_test.py" << 'PYTHON'
#!/usr/bin/env python3
"""
Prueba de saturación de conexiones:
Intenta abrir N conexiones simultáneas al endpoint dado y mide cuántas
tienen éxito, cuántas fallan y el tiempo total.
"""
import psycopg2
import threading
import time
import os
import sys

ENDPOINT   = sys.argv[1] if len(sys.argv) > 1 else os.environ.get("WRITER_ENDPOINT")
NUM_CONNS  = int(sys.argv[2]) if len(sys.argv) > 2 else 200
DB_PORT    = int(os.environ.get("DB_PORT", 5432))
DB_NAME    = os.environ.get("DB_NAME", "labdb")
DB_USER    = os.environ.get("DB_USER", "labadmin")
DB_PASS    = os.environ.get("PGPASSWORD", "")

results = {"success": 0, "failed": 0, "errors": []}
lock = threading.Lock()

def open_connection(conn_id):
    try:
        conn = psycopg2.connect(
            host=ENDPOINT,
            port=DB_PORT,
            dbname=DB_NAME,
            user=DB_USER,
            password=DB_PASS,
            connect_timeout=10,
            sslmode="require"
        )
        # Mantener conexión abierta 15 segundos para simular sesión activa
        cur = conn.cursor()
        cur.execute("SELECT pg_sleep(15), pg_backend_pid()")
        cur.fetchone()
        conn.close()
        with lock:
            results["success"] += 1
    except Exception as e:
        with lock:
            results["failed"] += 1
            if len(results["errors"]) < 5:
                results["errors"].append(str(e))

print(f"\n{'='*60}")
print(f"  PRUEBA DE SATURACIÓN: {NUM_CONNS} conexiones → {ENDPOINT}")
print(f"{'='*60}")

threads = []
start = time.time()

for i in range(NUM_CONNS):
    t = threading.Thread(target=open_connection, args=(i,))
    threads.append(t)
    t.start()
    time.sleep(0.02)  # 20ms entre conexiones para no saturar el OS

for t in threads:
    t.join(timeout=30)

elapsed = time.time() - start

print(f"\n  Conexiones exitosas : {results['success']}")
print(f"  Conexiones fallidas : {results['failed']}")
print(f"  Tasa de éxito       : {results['success']/NUM_CONNS*100:.1f}%")
print(f"  Tiempo total        : {elapsed:.2f}s")
if results["errors"]:
    print(f"\n  Errores (primeros 5):")
    for err in results["errors"]:
        print(f"    - {err}")
print(f"{'='*60}\n")
PYTHON
chmod +x "$LAB_DIR/connection_saturation_test.py"
```

**5.2 — Obtener la contraseña de la base de datos desde Secrets Manager:**

```bash
export PGPASSWORD=$(aws secretsmanager get-secret-value \
  --secret-id "$SECRET_ARN" \
  --region "$AWS_REGION" \
  --query "SecretString" \
  --output text | jq -r '.password')
```

**5.3 — Ejecutar prueba de saturación con conexión directa (200 conexiones):**

```bash
echo "Probando saturación con CONEXIÓN DIRECTA (200 conexiones)..."
python3 "$LAB_DIR/connection_saturation_test.py" "$WRITER_ENDPOINT" 200 \
  2>&1 | tee "$LAB_DIR/results/saturation_direct.txt"
```

**5.4 — Ejecutar la misma prueba a través del proxy:**

```bash
echo "Probando saturación con RDS PROXY (200 conexiones)..."
python3 "$LAB_DIR/connection_saturation_test.py" "$PROXY_ENDPOINT" 200 \
  2>&1 | tee "$LAB_DIR/results/saturation_proxy.txt"
```

**5.5 — Monitorear conexiones en Aurora durante la prueba del proxy:**

```bash
# Ejecutar en una terminal separada mientras corre la prueba del proxy
watch -n 2 'psql "host=$WRITER_ENDPOINT port=$DB_PORT dbname=$DB_NAME user=$DB_USER" \
  -t -c "SELECT count(*) AS \"Conexiones en Aurora\" FROM pg_stat_activity WHERE backend_type='"'"'client backend'"'"';"'
```

#### Salida Esperada

```
============================================================
  PRUEBA DE SATURACIÓN: 200 conexiones → aurora-pg-proxy-lab...
============================================================

  Conexiones exitosas : 198
  Conexiones fallidas : 2
  Tasa de éxito       : 99.0%
  Tiempo total        : 19.43s
============================================================
```

> **Resultado clave:** Con conexión directa, es probable que veas fallos cuando el número de conexiones supere `max_connections`. Con el proxy, el pooling gestiona las conexiones y la tasa de éxito debe ser significativamente mayor, mientras que las conexiones reales en Aurora se mantienen dentro del límite configurado (80% de `max_connections`).

#### Verificación

```bash
# Comparar resultados de saturación
echo "=== COMPARATIVA DE SATURACIÓN ==="
echo "DIRECTA:"
grep -E "(exitosas|fallidas|Tasa)" "$LAB_DIR/results/saturation_direct.txt"
echo ""
echo "PROXY:"
grep -E "(exitosas|fallidas|Tasa)" "$LAB_DIR/results/saturation_proxy.txt"
```

---

### Paso 6 — Análisis de Métricas de RDS Proxy en CloudWatch

**Objetivo:** Consultar y analizar las métricas específicas de RDS Proxy en CloudWatch para comprender el comportamiento del pool durante las pruebas realizadas.

#### Instrucciones

**6.1 — Crear script de consulta de métricas CloudWatch:**

```bash
cat > "$LAB_DIR/query_proxy_metrics.sh" << 'SCRIPT'
#!/bin/bash
# Consulta métricas de RDS Proxy en CloudWatch
# Período: últimos 30 minutos, granularidad: 1 minuto

PROXY_NAME=${PROXY_NAME:-"aurora-pg-proxy-lab"}
REGION=${AWS_REGION:-"us-east-1"}
END_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
START_TIME=$(date -u -d "30 minutes ago" +"%Y-%m-%dT%H:%M:%SZ" 2>/dev/null || \
             date -u -v-30M +"%Y-%m-%dT%H:%M:%SZ")  # macOS compat

echo "=== Métricas RDS Proxy: $PROXY_NAME ==="
echo "Período: $START_TIME → $END_TIME"
echo ""

for METRIC in "ClientConnections" "DatabaseConnections" "QueryRequests" "ClientConnectionsClosed" "MaxDatabaseConnectionsAllowed"; do
  echo "── $METRIC ──"
  aws cloudwatch get-metric-statistics \
    --namespace "AWS/RDS" \
    --metric-name "$METRIC" \
    --dimensions Name=ProxyName,Value="$PROXY_NAME" \
    --start-time "$START_TIME" \
    --end-time "$END_TIME" \
    --period 60 \
    --statistics Average Maximum \
    --region "$REGION" \
    --query "sort_by(Datapoints, &Timestamp)[*].{Tiempo:Timestamp,Promedio:Average,Maximo:Maximum}" \
    --output table 2>/dev/null || echo "  (Sin datos disponibles aún)"
  echo ""
done
SCRIPT
chmod +x "$LAB_DIR/query_proxy_metrics.sh"
```

**6.2 — Ejecutar la consulta de métricas:**

```bash
"$LAB_DIR/query_proxy_metrics.sh" 2>&1 | tee "$LAB_DIR/results/proxy_metrics.txt"
```

**6.3 — Consultar la métrica `MaxDatabaseConnectionsAllowed` para calcular el límite real:**

```bash
# Esta métrica indica el máximo de conexiones que el proxy puede abrir a Aurora
# Debería ser max_connections * MaxConnectionsPercent / 100
MAX_ALLOWED=$(aws cloudwatch get-metric-statistics \
  --namespace "AWS/RDS" \
  --metric-name "MaxDatabaseConnectionsAllowed" \
  --dimensions Name=ProxyName,Value="$PROXY_NAME" \
  --start-time "$(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -v-5M +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --period 300 \
  --statistics Maximum \
  --region "$AWS_REGION" \
  --query "Datapoints[0].Maximum" \
  --output text)

echo "Conexiones máximas permitidas por el proxy hacia Aurora: $MAX_ALLOWED"
```

**6.4 — Revisar métricas en la Consola AWS (instrucciones visuales):**

```
1. Navegar a: AWS Console → RDS → Proxies → aurora-pg-proxy-lab
2. Hacer clic en la pestaña "Monitoring"
3. Observar los siguientes gráficos:
   - ClientConnections: Conexiones desde aplicaciones al proxy
   - DatabaseConnections: Conexiones reales del proxy a Aurora
   - QueryRequests: Solicitudes procesadas por segundo
4. Comparar ClientConnections vs DatabaseConnections:
   - La diferencia = conexiones multiplexadas (beneficio del pooling)
5. Ajustar el rango de tiempo a "Last 30 minutes"
```

**6.5 — Verificar el modo de pooling configurado:**

```bash
# El modo transaction pooling es el recomendado para la mayoría de workloads OLTP
# Session pooling mantiene la conexión durante toda la sesión del cliente
aws rds describe-db-proxy-target-groups \
  --db-proxy-name "$PROXY_NAME" \
  --region "$AWS_REGION" \
  --query "TargetGroups[0].{Modo:ConnectionPoolConfig,Filtros:ConnectionPoolConfig.SessionPinningFilters}" \
  --output json
```

#### Salida Esperada

```
=== Métricas RDS Proxy: aurora-pg-proxy-lab ===

── ClientConnections ──
| Tiempo              | Promedio | Maximo |
|---------------------|----------|--------|
| 2024-01-15T10:30:00 | 45.0     | 200.0  |
| 2024-01-15T10:31:00 | 12.0     | 50.0   |

── DatabaseConnections ──
| Tiempo              | Promedio | Maximo |
|---------------------|----------|--------|
| 2024-01-15T10:30:00 | 18.0     | 32.0   |  ← Significativamente menor
| 2024-01-15T10:31:00 | 8.0      | 15.0   |
```

> **Observación clave:** `DatabaseConnections` (conexiones reales a Aurora) debe ser significativamente menor que `ClientConnections` (conexiones de aplicaciones al proxy), demostrando el efecto de multiplexación del connection pooling.

#### Verificación

```bash
# Confirmar que las métricas se están recopilando
aws cloudwatch list-metrics \
  --namespace "AWS/RDS" \
  --dimensions Name=ProxyName,Value="$PROXY_NAME" \
  --region "$AWS_REGION" \
  --query "Metrics[*].MetricName" \
  --output table
```

---

## Validación y Pruebas Finales

Ejecuta este bloque de validación completo para confirmar que todos los objetivos del laboratorio se han cumplido:

```bash
echo ""
echo "╔══════════════════════════════════════════════════════════════╗"
echo "║         VALIDACIÓN FINAL - LAB 03-00-02                     ║"
echo "╚══════════════════════════════════════════════════════════════╝"

# ── Validación 1: Proxy existe y está disponible ──────────────────
PROXY_STATUS=$(aws rds describe-db-proxies \
  --db-proxy-name "$PROXY_NAME" \
  --region "$AWS_REGION" \
  --query "DBProxies[0].Status" \
  --output text 2>/dev/null)

if [ "$PROXY_STATUS" = "available" ]; then
  echo "✅ [1/5] RDS Proxy disponible (Status: available)"
else
  echo "❌ [1/5] RDS Proxy NO disponible (Status: $PROXY_STATUS)"
fi

# ── Validación 2: Target registrado y saludable ───────────────────
TARGET_STATUS=$(aws rds describe-db-proxy-targets \
  --db-proxy-name "$PROXY_NAME" \
  --region "$AWS_REGION" \
  --query "Targets[0].TargetHealth.State" \
  --output text 2>/dev/null)

if [ "$TARGET_STATUS" = "AVAILABLE" ]; then
  echo "✅ [2/5] Target Aurora registrado y saludable"
else
  echo "❌ [2/5] Target no disponible (State: $TARGET_STATUS)"
fi

# ── Validación 3: Conectividad a través del proxy ─────────────────
PROXY_QUERY=$(psql "host=$PROXY_ENDPOINT port=$DB_PORT dbname=$DB_NAME \
  user=$DB_USER sslmode=require" \
  -t -c "SELECT 'OK'" 2>/dev/null | tr -d ' ')

if [ "$PROXY_QUERY" = "OK" ]; then
  echo "✅ [3/5] Conectividad a través del proxy verificada"
else
  echo "❌ [3/5] No se pudo conectar a través del proxy"
fi

# ── Validación 4: Resultados de pruebas generados ────────────────
RESULT_FILES=$(ls "$LAB_DIR/results/"*.txt 2>/dev/null | wc -l)
if [ "$RESULT_FILES" -ge 5 ]; then
  echo "✅ [4/5] Resultados de pruebas generados ($RESULT_FILES archivos)"
else
  echo "⚠️  [4/5] Pocos archivos de resultados ($RESULT_FILES/5+ esperados)"
fi

# ── Validación 5: Configuración de pooling correcta ──────────────
MAX_CONN_PCT=$(aws rds describe-db-proxy-target-groups \
  --db-proxy-name "$PROXY_NAME" \
  --region "$AWS_REGION" \
  --query "TargetGroups[0].ConnectionPoolConfig.MaxConnectionsPercent" \
  --output text 2>/dev/null)

if [ "$MAX_CONN_PCT" = "80" ]; then
  echo "✅ [5/5] MaxConnectionsPercent configurado correctamente (80%)"
else
  echo "❌ [5/5] MaxConnectionsPercent incorrecto (valor: $MAX_CONN_PCT, esperado: 80)"
fi

echo ""
echo "Archivos de resultados generados:"
ls -lh "$LAB_DIR/results/"
echo ""
```

**Criterios de éxito:**
- Los 5 checks deben mostrar ✅.
- Los resultados de pruebas deben mostrar que `DatabaseConnections` < `ClientConnections` en las métricas de CloudWatch.
- La tasa de éxito en la prueba de saturación a través del proxy debe ser ≥ 95%.

---

## Resolución de Problemas

### Problema 1 — El proxy permanece en estado `creating` por más de 15 minutos

**Síntoma:**
```
aws rds describe-db-proxies → Status: "creating"
# Después de 15+ minutos sin avanzar a "available"
```

**Causa probable:**
El IAM Role asignado al proxy no tiene los permisos necesarios para leer el secreto de Secrets Manager, o la VPC/Security Group no permite la comunicación entre el proxy y la instancia Aurora en el puerto 5432.

**Solución:**

```bash
# 1. Verificar que el role tiene la política correcta
aws iam get-role-policy \
  --role-name "rds-proxy-lab-role" \
  --policy-name "RDSProxySecretsManagerPolicy" 2>/dev/null || \
aws iam list-attached-role-policies \
  --role-name "rds-proxy-lab-role" \
  --query "AttachedPolicies[*].PolicyName" \
  --output table

# 2. Verificar que el secreto existe y el role puede leerlo
aws secretsmanager get-secret-value \
  --secret-id "$SECRET_ARN" \
  --region "$AWS_REGION" \
  --query "SecretString" \
  --output text | jq 'keys'

# 3. Verificar reglas de Security Group (debe permitir 5432 saliente)
aws ec2 describe-security-groups \
  --group-ids "$SECURITY_GROUP_ID" \
  --region "$AWS_REGION" \
  --query "SecurityGroups[0].IpPermissionsEgress" \
  --output table

# 4. Si el role tiene permisos incorrectos, adjuntar política inline
aws iam put-role-policy \
  --role-name "rds-proxy-lab-role" \
  --policy-name "SecretsManagerReadPolicy" \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"],
      "Resource": "'"$SECRET_ARN"'"
    }]
  }'

# 5. Verificar eventos de error del proxy en CloudWatch Logs
aws logs describe-log-groups \
  --log-group-name-prefix "/aws/rds/proxy/$PROXY_NAME" \
  --region "$AWS_REGION"
```

---

### Problema 2 — Error `FATAL: remaining connection slots are reserved` en prueba de saturación directa

**Síntoma:**
```
psycopg2.OperationalError: FATAL: remaining connection slots are reserved
for non-replication superuser connections
# O bien:
FATAL: sorry, too many clients already
```

**Causa probable:**
El número de conexiones simultáneas ha alcanzado el límite de `max_connections` de la instancia Aurora. Este es el comportamiento **esperado** en la prueba de conexión directa y demuestra exactamente por qué RDS Proxy es necesario. Sin embargo, si ocurre también a través del proxy, el parámetro `MaxConnectionsPercent` puede estar mal configurado o el valor de `max_connections` en Aurora es demasiado bajo.

**Solución:**

```bash
# 1. Verificar max_connections actual en Aurora
psql "host=$WRITER_ENDPOINT port=$DB_PORT dbname=$DB_NAME user=$DB_USER" \
  -c "SHOW max_connections;"

# 2. Verificar conexiones activas en este momento
psql "host=$WRITER_ENDPOINT port=$DB_PORT dbname=$DB_NAME user=$DB_USER" \
  -c "SELECT count(*), state FROM pg_stat_activity GROUP BY state;"

# 3. Si el error ocurre a través del proxy, verificar MaxConnectionsPercent
aws rds describe-db-proxy-target-groups \
  --db-proxy-name "$PROXY_NAME" \
  --region "$AWS_REGION" \
  --query "TargetGroups[0].ConnectionPoolConfig.MaxConnectionsPercent"

# 4. Si es necesario, reducir el número de conexiones de prueba
# (el error en conexión directa con 200 conexiones es ESPERADO y demuestra el problema)
# Para el proxy, reducir NUM_CONNS a 100 si max_connections es bajo (ej: 100)

# 5. Terminar conexiones idle que puedan estar bloqueando
psql "host=$WRITER_ENDPOINT port=$DB_PORT dbname=$DB_NAME user=$DB_USER" \
  -c "SELECT pg_terminate_backend(pid)
      FROM pg_stat_activity
      WHERE state = 'idle'
        AND backend_type = 'client backend'
        AND query_start < now() - interval '5 minutes';"

# 6. Si max_connections es muy bajo, ajustar el DB parameter group
# (requiere reinicio - ver Lección 3.1)
aws rds modify-db-parameter-group \
  --db-parameter-group-name "$(aws rds describe-db-instances \
    --db-instance-identifier "${CLUSTER_ID}-instance-1" \
    --query "DBInstances[0].DBParameterGroups[0].DBParameterGroupName" \
    --output text)" \
  --parameters "ParameterName=max_connections,ParameterValue=200,ApplyMethod=pending-reboot"
```

---

## Limpieza de Recursos

> ⚠️ **Importante:** Ejecuta la limpieza al finalizar el laboratorio para evitar costos innecesarios. RDS Proxy genera cargos por hora adicionales a los del clúster Aurora.

```bash
echo "Iniciando limpieza de recursos del Lab 03-00-02..."

# ── 1. Deregistrar targets del proxy ─────────────────────────────
echo "1/4 Desregistrando targets..."
aws rds deregister-db-proxy-targets \
  --db-proxy-name "$PROXY_NAME" \
  --db-cluster-identifiers "$CLUSTER_ID" \
  --region "$AWS_REGION" 2>/dev/null && echo "   ✓ Targets desregistrados"

# ── 2. Eliminar el RDS Proxy ──────────────────────────────────────
echo "2/4 Eliminando RDS Proxy..."
aws rds delete-db-proxy \
  --db-proxy-name "$PROXY_NAME" \
  --region "$AWS_REGION" 2>/dev/null && echo "   ✓ Proxy en proceso de eliminación"

# ── 3. Limpiar datos de pgbench (opcional) ────────────────────────
echo "3/4 Limpiando esquema pgbench (opcional)..."
read -p "   ¿Eliminar tablas pgbench_* de Aurora? [s/N]: " CONFIRM
if [[ "$CONFIRM" =~ ^[Ss]$ ]]; then
  psql "host=$WRITER_ENDPOINT port=$DB_PORT dbname=$DB_NAME user=$DB_USER" \
    -c "DROP TABLE IF EXISTS pgbench_accounts, pgbench_branches, pgbench_history, pgbench_tellers CASCADE;"
  echo "   ✓ Tablas pgbench eliminadas"
fi

# ── 4. Archivar resultados locales ────────────────────────────────
echo "4/4 Archivando resultados..."
tar -czf "$HOME/lab-03-00-02-results-$(date +%Y%m%d_%H%M%S).tar.gz" \
  "$LAB_DIR/results/" 2>/dev/null && echo "   ✓ Resultados archivados en $HOME/"

echo ""
echo "✅ Limpieza completada."
echo ""
echo "NOTA: El clúster Aurora ($CLUSTER_ID) NO fue eliminado."
echo "Si no continuarás con el siguiente laboratorio, ejecuta:"
echo "  aws rds delete-db-cluster --db-cluster-identifier $CLUSTER_ID \\"
echo "    --skip-final-snapshot --region $AWS_REGION"
```

**Verificar que el proxy fue eliminado:**

```bash
# Esperar eliminación completa (puede tomar 2-3 minutos)
sleep 60
aws rds describe-db-proxies \
  --db-proxy-name "$PROXY_NAME" \
  --region "$AWS_REGION" 2>&1 | grep -c "DBProxyNotFoundFault" && \
  echo "Proxy eliminado correctamente" || \
  echo "El proxy aún existe - verificar en consola AWS"
```

---

## Resumen

En este laboratorio aplicaste RDS Proxy como capa de gestión de conexiones para Aurora PostgreSQL, completando los siguientes logros:

| Objetivo | Resultado |
|----------|-----------|
| Desplegar RDS Proxy con AWS CLI | ✅ Proxy creado con `max_connections_percent=80`, `idle_client_timeout=1800s` |
| Verificar conectividad a través del proxy | ✅ Conexión psql y validación de enrutamiento confirmados |
| Prueba comparativa directa vs proxy | ✅ pgbench ejecutado con 10, 50 y 100 clientes simultáneos |
| Simular saturación de conexiones | ✅ 200 conexiones simultáneas; proxy demostró mayor tasa de éxito |
| Analizar métricas CloudWatch | ✅ `ClientConnections` vs `DatabaseConnections` comparados |

### Conceptos Clave Reforzados

- **Connection Pooling en Transaction Mode:** El proxy reutiliza conexiones a Aurora entre transacciones de diferentes clientes, reduciendo el overhead de establecimiento de conexiones.
- **`MaxConnectionsPercent`:** Controla qué fracción de `max_connections` de Aurora puede usar el proxy, protegiendo al motor de saturación.
- **`ConnectionBorrowTimeout`:** Define cuánto espera una aplicación por una conexión del pool antes de recibir un error, evitando bloqueos indefinidos.
- **`idle_client_timeout`:** Cierra conexiones de clientes inactivos al proxy, liberando recursos del pool.
- **Multiplexación:** La diferencia entre `ClientConnections` y `DatabaseConnections` en CloudWatch cuantifica el beneficio real del pooling.

### Relación con DB Parameter Groups (Lección 3.1)

El parámetro `max_connections` configurado en el DB parameter group de Aurora (Lección 3.1) es la base sobre la que `MaxConnectionsPercent` del proxy calcula su límite. Un ajuste inadecuado de `max_connections` (parámetro estático que requiere reinicio) afecta directamente la capacidad del proxy para gestionar conexiones.

### Recursos Adicionales

- [Documentación oficial: RDS Proxy para Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/rds-proxy.html)
- [Métricas de CloudWatch para RDS Proxy](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/rds-proxy.monitoring.html)
- [Configuración de connection pooling en RDS Proxy](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/rds-proxy-connections.html)
- [Buenas prácticas de gestión de conexiones en Aurora PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.BestPractices.html)
- [CLI: create-db-proxy](https://docs.aws.amazon.com/cli/latest/reference/rds/create-db-proxy.html)
- [CLI: modify-db-proxy-target-group](https://docs.aws.amazon.com/cli/latest/reference/rds/modify-db-proxy-target-group.html)

---
*Lab 03-00-02 — Módulo 3: Optimización de Parámetros y Gestión de Conexiones en Aurora PostgreSQL*
