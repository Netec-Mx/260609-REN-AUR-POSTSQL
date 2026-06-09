# Observabilidad avanzada con Performance Insights

## Metadatos

| Campo            | Detalle                                      |
|------------------|----------------------------------------------|
| **Duración**     | 45 minutos                                   |
| **Complejidad**  | Media                                        |
| **Nivel Bloom**  | Aplicar (Apply)                              |
| **Módulo**       | 4 — Observabilidad y Optimización Avanzada   |
| **Laboratorio**  | 04-00-01 / Práctica 7                        |

---

## Descripción General

En este laboratorio habilitarás y utilizarás Amazon RDS Performance Insights (PI) sobre una instancia Aurora PostgreSQL para identificar cuellos de botella reales generados por carga controlada. Generarás patrones de espera observables con `pgbench` y consultas problemáticas diseñadas (full table scans, operaciones sin índice, sorts en memoria insuficiente), luego navegarás la interfaz de PI para correlacionar el gráfico de Average Active Sessions (AAS) con los wait events predominantes y las consultas responsables. Finalmente, configurarás `pg_stat_statements` con `track_io_timing = on` y crearás alarmas en CloudWatch para detectar degradación de rendimiento de forma proactiva.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio serás capaz de:

- [ ] Habilitar Performance Insights con retención extendida y navegar el gráfico de AAS para identificar el Top SQL por carga.
- [ ] Identificar y clasificar wait events predominantes (Lock, IO, CPU) correlacionándolos con consultas específicas en la dimensión SQL de PI.
- [ ] Configurar `pg_stat_statements` con `track_io_timing = on` y construir consultas analíticas para detectar las consultas con mayor `total_exec_time`, `mean_exec_time` y `stddev_exec_time`.
- [ ] Utilizar Enhanced Monitoring para analizar métricas de sistema operativo a nivel de proceso con granularidad de 1 segundo.
- [ ] Crear alarmas en CloudWatch basadas en métricas de PI para detectar proactivamente degradación de rendimiento.

---

## Prerrequisitos

### Conocimiento previo
- Familiaridad con la consola de AWS (RDS, CloudWatch).
- Conocimiento básico de SQL y PostgreSQL (extensiones, parameter groups).
- Haber revisado la lección 4.1 sobre Performance Insights y análisis de waits.
- Comprensión del concepto AAS y su relación con vCPU.

### Acceso y recursos
- Usuario IAM con políticas para RDS, CloudWatch, Secrets Manager y VPC (sin credenciales root).
- Clúster Aurora PostgreSQL activo (writer `db.r6g.large` o mínimo `db.t3.medium`) con Performance Insights **habilitado**.
- Enhanced Monitoring habilitado con granularidad de **1 segundo**.
- Extensión `pg_stat_statements` instalada en la base de datos objetivo.
- `pgbench`, `psql` y AWS CLI 2.x disponibles en la máquina local.
- Scripts del repositorio del curso clonados localmente.

> ⚠️ **Importante sobre costos:** Las instancias `db.r6g.large` generan costos por hora. Ejecuta `terraform destroy` al finalizar la sesión. Estima $3–5 USD para este laboratorio individual.

---

## Entorno del Laboratorio

### Hardware recomendado

| Componente       | Mínimo requerido                                    |
|------------------|-----------------------------------------------------|
| RAM              | 8 GB (para ejecutar pgbench y scripts de carga)     |
| Almacenamiento   | 10 GB libres (logs, scripts, resultados)            |
| CPU              | 4 núcleos (simulaciones de concurrencia)            |
| Red              | 10 Mbps estables hacia AWS                          |
| Pantalla         | 1920×1080 (consola AWS y herramientas de monitoreo) |

### Software requerido

| Herramienta         | Versión mínima | Propósito                                  |
|---------------------|----------------|--------------------------------------------|
| AWS CLI             | 2.x            | Comandos RDS, PI API, CloudWatch           |
| psql                | 14.x           | Conexión y consultas a Aurora              |
| pgbench             | 14.x           | Generación de carga observable             |
| Python              | 3.9+           | Scripts de carga problemática              |
| jq                  | 1.6+           | Procesamiento de salida JSON de la PI API  |
| Navegador web       | Última versión | Consola AWS (Performance Insights)         |

### Configuración inicial del entorno

**Paso 0.1 — Clonar el repositorio del curso y configurar variables de entorno**

```bash
# Clonar repositorio (si no se hizo en laboratorios anteriores)
git clone https://github.com/curso-aurora-postgresql/labs.git ~/aurora-labs
cd ~/aurora-labs/04-00-01

# Configurar variables de entorno para el laboratorio
export AWS_REGION="us-east-1"
export CLUSTER_ID="aurora-lab-cluster"
export WRITER_INSTANCE_ID="aurora-lab-writer-1"
export DB_NAME="labdb"
export DB_USER="labadmin"
export DB_HOST=$(aws rds describe-db-clusters \
  --db-cluster-identifier $CLUSTER_ID \
  --query 'DBClusters[0].Endpoint' \
  --output text)
export DB_PORT="5432"

# Verificar conectividad
psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME -c "SELECT version();"
```

**Paso 0.2 — Aprovisionar infraestructura base con Terraform**

```bash
cd ~/aurora-labs/04-00-01/terraform
terraform init
terraform apply -auto-approve

# Capturar el ARN de la instancia writer (necesario para la PI API)
export WRITER_ARN=$(aws rds describe-db-instances \
  --db-instance-identifier $WRITER_INSTANCE_ID \
  --query 'DBInstances[0].DBInstanceArn' \
  --output text)
echo "Writer ARN: $WRITER_ARN"
```

**Paso 0.3 — Verificar que Performance Insights y Enhanced Monitoring estén activos**

```bash
aws rds describe-db-instances \
  --db-instance-identifier $WRITER_INSTANCE_ID \
  --query 'DBInstances[0].{PI_Enabled:PerformanceInsightsEnabled,
    PI_Retention:PerformanceInsightsRetentionPeriod,
    EM_Interval:MonitoringInterval,
    Instance_Class:DBInstanceClass}' \
  --output table
```

Salida esperada:

```
-----------------------------------------------------------
|               DescribeDBInstances                       |
+------------------+--------------------------------------+
|  EM_Interval     |  1                                   |
|  Instance_Class  |  db.r6g.large                        |
|  PI_Enabled      |  True                                |
|  PI_Retention    |  7                                   |
+------------------+--------------------------------------+
```

> Si `PI_Enabled` aparece como `False` o `EM_Interval` es mayor que 1, revisa la sección de Solución de Problemas antes de continuar.

---

## Pasos del Laboratorio

---

### Paso 1: Configurar el Parameter Group para pg_stat_statements con track_io_timing

**Objetivo:** Asegurar que `pg_stat_statements` esté configurado con los parámetros óptimos para recolectar estadísticas de tiempo de I/O y cobertura completa de consultas, de modo que Performance Insights pueda mostrar Top SQL enriquecido.

#### Instrucciones

**1.1 — Verificar la configuración actual del parameter group**

```bash
# Obtener el nombre del parameter group de la instancia
export PG_GROUP=$(aws rds describe-db-instances \
  --db-instance-identifier $WRITER_INSTANCE_ID \
  --query 'DBInstances[0].DBParameterGroups[0].DBParameterGroupName' \
  --output text)
echo "Parameter Group: $PG_GROUP"

# Verificar parámetros actuales de pg_stat_statements
aws rds describe-db-parameters \
  --db-parameter-group-name $PG_GROUP \
  --query 'Parameters[?contains(ParameterName, `pg_stat_statements`) || 
           contains(ParameterName, `track_io_timing`) ||
           contains(ParameterName, `shared_preload_libraries`)]
          .{Name:ParameterName,Value:ParameterValue,Source:Source}' \
  --output table
```

**1.2 — Aplicar parámetros requeridos**

```bash
# Modificar el parameter group con los parámetros necesarios
aws rds modify-db-parameter-group \
  --db-parameter-group-name $PG_GROUP \
  --parameters \
    "ParameterName=pg_stat_statements.max,ParameterValue=10000,ApplyMethod=pending-reboot" \
    "ParameterName=pg_stat_statements.track,ParameterValue=all,ApplyMethod=pending-reboot" \
    "ParameterName=pg_stat_statements.track_utility,ParameterValue=off,ApplyMethod=pending-reboot" \
    "ParameterName=track_io_timing,ParameterValue=on,ApplyMethod=immediate" \
    "ParameterName=track_activity_query_size,ParameterValue=4096,ApplyMethod=pending-reboot"
```

> **Nota:** `track_io_timing = on` se puede aplicar de forma inmediata. Los parámetros de `pg_stat_statements` que requieren `pending-reboot` ya deben estar activos si la instancia fue provisionada con Terraform. Verifica con el siguiente comando.

**1.3 — Verificar que pg_stat_statements esté instalado y activo**

```sql
-- Conectarse a la base de datos
psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME

-- Verificar extensión instalada
SELECT name, default_version, installed_version 
FROM pg_available_extensions 
WHERE name = 'pg_stat_statements';

-- Instalar si no está presente
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Verificar configuración activa
SHOW pg_stat_statements.track;
SHOW track_io_timing;
SHOW pg_stat_statements.max;

-- Resetear estadísticas para partir de cero en este laboratorio
SELECT pg_stat_statements_reset();
\q
```

**Salida esperada:**

```
        name         | default_version | installed_version
---------------------+-----------------+-------------------
 pg_stat_statements  | 1.10            | 1.10
(1 row)

pg_stat_statements.track
--------------------------
 all
(1 row)

track_io_timing
-----------------
 on
(1 row)
```

**Verificación:**

```bash
# Confirmar que track_io_timing está activo consultando una sesión real
psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME \
  -c "SELECT name, setting, unit FROM pg_settings 
      WHERE name IN ('track_io_timing','pg_stat_statements.track',
                     'pg_stat_statements.max','track_activity_query_size')
      ORDER BY name;"
```

---

### Paso 2: Preparar los datos y generar carga observable con pgbench

**Objetivo:** Inicializar el esquema de pgbench y generar carga de trabajo que produzca patrones claros y diferenciados en Performance Insights (IO, CPU, Lock).

#### Instrucciones

**2.1 — Inicializar el esquema de pgbench**

```bash
# Inicializar pgbench con factor de escala 50 (~750 MB de datos)
pgbench -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME \
  --initialize \
  --scale=50 \
  --foreign-keys \
  --no-vacuum

echo "Inicialización completada. Ejecutando ANALYZE..."
psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME \
  -c "ANALYZE pgbench_accounts, pgbench_branches, pgbench_tellers;"
```

**2.2 — Crear tablas adicionales para escenarios problemáticos**

```sql
psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME << 'EOF'

-- Tabla para simular full table scans (sin índices en columnas de filtro)
CREATE TABLE IF NOT EXISTS lab_orders (
    order_id     BIGSERIAL PRIMARY KEY,
    customer_id  INTEGER NOT NULL,
    product_code VARCHAR(50),
    amount       NUMERIC(12,2),
    status       VARCHAR(20),
    created_at   TIMESTAMP DEFAULT NOW(),
    description  TEXT
);

-- Poblar con 2 millones de registros
INSERT INTO lab_orders (customer_id, product_code, amount, status, description)
SELECT 
    (random() * 100000)::INTEGER,
    'PROD-' || (random() * 1000)::INTEGER::TEXT,
    (random() * 10000)::NUMERIC(12,2),
    CASE (random() * 4)::INTEGER 
        WHEN 0 THEN 'pending'
        WHEN 1 THEN 'processing'
        WHEN 2 THEN 'shipped'
        ELSE 'delivered'
    END,
    repeat('descripcion de orden de compra para cliente ', 5) || generate_series
FROM generate_series(1, 2000000);

-- IMPORTANTE: No crear índice en customer_id ni status intencionalmente
-- para forzar sequential scans observables en PI

-- Tabla para simular contención de locks
CREATE TABLE IF NOT EXISTS lab_inventory (
    product_id  INTEGER PRIMARY KEY,
    stock       INTEGER NOT NULL DEFAULT 100,
    reserved    INTEGER NOT NULL DEFAULT 0,
    updated_at  TIMESTAMP DEFAULT NOW()
);

INSERT INTO lab_inventory (product_id, stock)
SELECT generate_series(1, 1000);

COMMIT;
EOF
```

**2.3 — Iniciar carga de trabajo con pgbench (carga OLTP estándar en background)**

```bash
# Terminal 1: Carga OLTP estándar (genera CPU + IO mix)
pgbench -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME \
  --time=300 \
  --client=20 \
  --jobs=4 \
  --progress=10 \
  --log \
  --log-prefix=/tmp/pgbench_oltp \
  > /tmp/pgbench_oltp_results.txt 2>&1 &

PGBENCH_PID=$!
echo "pgbench iniciado con PID: $PGBENCH_PID"
echo "Esperando 30 segundos para que se establezca la carga base..."
sleep 30
```

**2.4 — Ejecutar consultas problemáticas diseñadas (en paralelo)**

```bash
# Script para generar full table scans repetidos (fuerza IO:DataFileRead)
cat > /tmp/run_full_scans.sh << 'SCRIPT'
#!/bin/bash
for i in $(seq 1 50); do
  psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME -c \
    "SELECT COUNT(*), SUM(amount) 
     FROM lab_orders 
     WHERE status = 'pending' AND customer_id > 50000;" \
    > /dev/null 2>&1
  sleep 1
done
SCRIPT
chmod +x /tmp/run_full_scans.sh
/tmp/run_full_scans.sh &

# Script para generar sorts en memoria insuficiente (fuerza CPU + IO spill)
cat > /tmp/run_sorts.sh << 'SCRIPT'
#!/bin/bash
psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME << 'SQL'
SET work_mem = '1MB';  -- Forzar sort spill a disco
SELECT customer_id, product_code, SUM(amount) as total,
       COUNT(*) as num_orders,
       AVG(amount) as avg_amount
FROM lab_orders
GROUP BY customer_id, product_code
ORDER BY total DESC
LIMIT 100;
SQL
SCRIPT
chmod +x /tmp/run_sorts.sh
/tmp/run_sorts.sh &

# Script para generar contención de locks (fuerza Lock:transactionid)
cat > /tmp/run_lock_contention.sh << 'SCRIPT'
#!/bin/bash
for i in $(seq 1 30); do
  # Sesión 1: Actualización lenta que mantiene lock
  psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME << 'SQL' &
BEGIN;
UPDATE lab_inventory SET stock = stock - 1, reserved = reserved + 1 
WHERE product_id = 1;
SELECT pg_sleep(3);
COMMIT;
SQL
  # Sesión 2: Intenta actualizar el mismo registro
  psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME << 'SQL' &
UPDATE lab_inventory SET stock = stock - 1, reserved = reserved + 1 
WHERE product_id = 1;
SQL
  sleep 2
done
SCRIPT
chmod +x /tmp/run_lock_contention.sh
/tmp/run_lock_contention.sh &

echo "Cargas problemáticas iniciadas. Esperando 60 segundos para acumular datos en PI..."
sleep 60
```

**Salida esperada** (pgbench progress):

```
progress: 10.0 s, 842.3 tps, lat 23.7 ms stddev 8.2, 0 failed
progress: 20.0 s, 831.5 tps, lat 24.1 ms stddev 9.1, 0 failed
progress: 30.0 s, 756.2 tps, lat 26.4 ms stddev 12.3, 0 failed
```

**Verificación:**

```bash
# Confirmar que hay sesiones activas generando carga
psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME -c \
  "SELECT state, wait_event_type, wait_event, COUNT(*) 
   FROM pg_stat_activity 
   WHERE state != 'idle' AND pid != pg_backend_pid()
   GROUP BY state, wait_event_type, wait_event
   ORDER BY count DESC;"
```

---

### Paso 3: Navegar Performance Insights — Análisis de AAS y Wait Events

**Objetivo:** Utilizar la consola de Performance Insights para identificar el gráfico de AAS, clasificar los wait events dominantes y correlacionar con las consultas responsables.

#### Instrucciones

**3.1 — Abrir Performance Insights en la consola AWS**

1. En tu navegador, navega a: **AWS Console → RDS → Databases**.
2. Selecciona la instancia writer (`aurora-lab-writer-1`).
3. Haz clic en la pestaña **"Performance Insights"**.
4. Ajusta el intervalo de tiempo a **"Last 15 minutes"** (ícono de reloj en la parte superior derecha).
5. Asegúrate de que la dimensión seleccionada sea **"Wait"** en el selector sobre el gráfico apilado.

**3.2 — Analizar el gráfico de AAS**

Observa el gráfico apilado y responde las siguientes preguntas en tu cuaderno de laboratorio:

| Pregunta de análisis                                        | Tu observación |
|-------------------------------------------------------------|----------------|
| ¿Cuántas vCPU tiene la instancia? (línea de referencia)     |                |
| ¿El AAS supera la línea de vCPU? ¿En qué momentos?          |                |
| ¿Qué color/categoría domina el gráfico apilado?             |                |
| ¿Hay picos súbitos o la carga es sostenida?                 |                |

> **Referencia:** Para `db.r6g.large`, la línea de referencia de vCPU = 2. Si AAS > 2, hay sesiones esperando recursos.

**3.3 — Identificar wait events específicos**

En la tabla **"Top waits"** debajo del gráfico:

1. Observa los 5 wait events con mayor contribución al AAS.
2. Identifica si aparecen: `IO:DataFileRead`, `Lock:transactionid`, `CPU`, `LWLock:BufferContent`.
3. Haz clic en `IO:DataFileRead` (si aparece) para filtrar el gráfico solo por ese wait.

Referencia de interpretación:

| Wait Event observado       | Causa probable en este lab                          |
|----------------------------|-----------------------------------------------------|
| `IO:DataFileRead`          | Full table scan en `lab_orders` (sin índice)        |
| `Lock:transactionid`       | Contención en `lab_inventory` (script de locks)     |
| `CPU`                      | Sort spill + carga OLTP de pgbench                  |
| `LWLock:BufferContent`     | Contención de buffer por hot pages en pgbench       |

**3.4 — Cambiar a la dimensión SQL y correlacionar**

1. En el selector de dimensión, cambia de **"Wait"** a **"SQL"**.
2. Observa el **Top SQL** ordenado por carga (AAS contribution).
3. Identifica la consulta de `lab_orders` con el full table scan.
4. Haz clic sobre esa consulta para ver:
   - Su contribución por tipo de wait.
   - El texto SQL normalizado.
   - Las métricas: calls, avg latency, rows examined.
5. Regresa a dimensión **"Wait"**, filtra por `Lock:transactionid` y luego cambia a **"SQL"** para ver qué consulta genera los locks.

**3.5 — Usar la PI API para exportar métricas programáticamente**

```bash
# Obtener las últimas 15 minutos de db.load.avg desglosado por wait_event_type
START_TIME=$(date -u -d '15 minutes ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || \
             date -u -v-15M +%Y-%m-%dT%H:%M:%SZ)
END_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)

aws pi get-resource-metrics \
  --service-type RDS \
  --identifier $WRITER_ARN \
  --metric-queries '[
    {
      "Metric": "db.load.avg",
      "GroupBy": {
        "Group": "db.wait_event_type",
        "Dimensions": ["db.wait_event_type"],
        "Limit": 5
      }
    }
  ]' \
  --start-time $START_TIME \
  --end-time $END_TIME \
  --period-in-seconds 60 \
  --output json | jq '
    .MetricList[0].DataPoints[] | 
    {
      timestamp: .Timestamp,
      wait_type: (.Value | keys[0]),
      aas_value: (.Value | to_entries[0].value)
    }
  ' | head -50
```

**Salida esperada (ejemplo):**

```json
{
  "timestamp": "2026-06-09T13:05:00Z",
  "wait_type": "IO",
  "aas_value": 3.42
}
{
  "timestamp": "2026-06-09T13:05:00Z",
  "wait_type": "Lock",
  "aas_value": 1.87
}
```

```bash
# Obtener Top SQL por db.load.avg en el mismo período
aws pi describe-dimension-keys \
  --service-type RDS \
  --identifier $WRITER_ARN \
  --start-time $START_TIME \
  --end-time $END_TIME \
  --metric db.load.avg \
  --group-by '{"Group":"db.sql_tokenized","Dimensions":["db.sql_tokenized.statement"],"Limit":5}' \
  --period-in-seconds 900 \
  --output json | jq '.Keys[] | {sql: .Dimensions["db.sql_tokenized.statement"], total_aas: .Total}' 
```

**Verificación:**

```bash
# Confirmar que la PI API retorna datos (código de salida 0)
echo "Exit code de PI API: $?"
# Debe retornar 0 (éxito)
```

---

### Paso 4: Análisis profundo con pg_stat_statements

**Objetivo:** Construir consultas analíticas sobre `pg_stat_statements` para identificar las consultas con mayor impacto acumulado, mayor variabilidad de latencia y mayor consumo de I/O.

#### Instrucciones

**4.1 — Consultas Top por tiempo total de ejecución**

```sql
psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME << 'EOF'

-- Top 10 consultas por total_exec_time
SELECT
    LEFT(query, 80) AS query_snippet,
    calls,
    ROUND(total_exec_time::numeric, 2) AS total_exec_ms,
    ROUND(mean_exec_time::numeric, 2)  AS mean_exec_ms,
    ROUND(stddev_exec_time::numeric, 2) AS stddev_exec_ms,
    ROUND((total_exec_time / SUM(total_exec_time) OVER ()) * 100, 2) AS pct_total,
    rows
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat_statements%'
  AND query NOT LIKE '%pg_sleep%'
ORDER BY total_exec_time DESC
LIMIT 10;

EOF
```

**4.2 — Consultas con mayor variabilidad de latencia (candidatas a optimización)**

```sql
psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME << 'EOF'

-- Consultas con alta desviación estándar (comportamiento inconsistente)
SELECT
    LEFT(query, 80) AS query_snippet,
    calls,
    ROUND(mean_exec_time::numeric, 2)   AS mean_exec_ms,
    ROUND(stddev_exec_time::numeric, 2) AS stddev_exec_ms,
    ROUND(min_exec_time::numeric, 2)    AS min_exec_ms,
    ROUND(max_exec_time::numeric, 2)    AS max_exec_ms,
    -- Coeficiente de variación: stddev/mean (>1 indica alta variabilidad)
    ROUND((stddev_exec_time / NULLIF(mean_exec_time, 0))::numeric, 3) AS cv_ratio
FROM pg_stat_statements
WHERE calls > 10
  AND mean_exec_time > 10
  AND query NOT LIKE '%pg_stat_statements%'
ORDER BY cv_ratio DESC
LIMIT 10;

EOF
```

**4.3 — Consultas con mayor consumo de I/O (requiere track_io_timing = on)**

```sql
psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME << 'EOF'

-- Top consultas por tiempo de I/O (blk_read_time + blk_write_time)
SELECT
    LEFT(query, 80) AS query_snippet,
    calls,
    ROUND(total_exec_time::numeric, 2)  AS total_exec_ms,
    ROUND(blk_read_time::numeric, 2)    AS total_read_ms,
    ROUND(blk_write_time::numeric, 2)   AS total_write_ms,
    ROUND((blk_read_time + blk_write_time)::numeric, 2) AS total_io_ms,
    -- Porcentaje del tiempo en I/O
    ROUND(((blk_read_time + blk_write_time) / 
           NULLIF(total_exec_time, 0) * 100)::numeric, 2) AS io_pct,
    shared_blks_read,
    shared_blks_hit,
    -- Cache hit ratio por consulta
    ROUND((shared_blks_hit::numeric / 
           NULLIF(shared_blks_hit + shared_blks_read, 0) * 100), 2) AS cache_hit_pct
FROM pg_stat_statements
WHERE (blk_read_time + blk_write_time) > 0
  AND query NOT LIKE '%pg_stat_statements%'
ORDER BY total_io_ms DESC
LIMIT 10;

EOF
```

**4.4 — Identificar consultas con sequential scans masivos**

```sql
psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME << 'EOF'

-- Correlacionar pg_stat_statements con pg_stat_user_tables para sequential scans
SELECT
    relname AS table_name,
    seq_scan,
    seq_tup_read,
    idx_scan,
    ROUND(seq_tup_read::numeric / NULLIF(seq_scan, 0), 0) AS avg_rows_per_seqscan,
    n_live_tup AS estimated_rows,
    last_seq_scan
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC
LIMIT 10;

EOF
```

**Salida esperada (ejemplo de 4.1):**

```
                query_snippet                  | calls | total_exec_ms | mean_exec_ms | stddev_exec_ms | pct_total | rows
-----------------------------------------------+-------+---------------+--------------+----------------+-----------+------
 SELECT COUNT(*), SUM(amount) FROM lab_orders  |    50 |     245832.41 |      4916.65 |        1203.22 |     68.42 |   50
 SELECT customer_id, product_code, SUM(amoun   |     3 |      89234.12 |     29744.71 |        8821.33 |     24.85 |  100
 UPDATE pgbench_accounts SET abalance = abala  |  8432 |      12341.22 |         1.46 |           0.89 |      3.44 | 8432
```

**Verificación:**

```bash
# Confirmar que track_io_timing está produciendo datos
psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME \
  -c "SELECT COUNT(*) as queries_with_io_timing 
      FROM pg_stat_statements 
      WHERE (blk_read_time + blk_write_time) > 0;"
# Debe retornar > 0 si track_io_timing está activo y hubo I/O
```

---

### Paso 5: Enhanced Monitoring — Análisis a nivel de proceso del SO

**Objetivo:** Utilizar Enhanced Monitoring para observar métricas de sistema operativo con granularidad de 1 segundo e identificar procesos PostgreSQL que consumen CPU y memoria durante la carga generada.

#### Instrucciones

**5.1 — Acceder a Enhanced Monitoring en la consola**

1. Navega a **AWS Console → RDS → Databases → aurora-lab-writer-1**.
2. Selecciona la pestaña **"Monitoring"**.
3. En el menú desplegable de monitoreo, selecciona **"Enhanced monitoring"**.
4. Ajusta el período a **"1 second"** (granularidad máxima).
5. Observa las siguientes métricas durante la carga activa:
   - **CPU Utilization** (por proceso: `postgres`, `aurora background`)
   - **Memory** (freeable, active)
   - **Disk I/O** (read/write IOPS, throughput)
   - **Process list** (muestra procesos individuales del SO)

**5.2 — Obtener métricas de Enhanced Monitoring via CloudWatch Logs**

```bash
# Enhanced Monitoring publica métricas en CloudWatch Logs
# Log group: RDSOSMetrics
LOG_GROUP="/aws/rds/cluster/$CLUSTER_ID/osaggregated"

# Obtener los últimos eventos del stream de la instancia writer
aws logs describe-log-streams \
  --log-group-name "RDSOSMetrics" \
  --log-stream-name-prefix $WRITER_INSTANCE_ID \
  --query 'logStreams[0].logStreamName' \
  --output text | xargs -I {} aws logs get-log-events \
  --log-group-name "RDSOSMetrics" \
  --log-stream-name {} \
  --limit 5 \
  --output json | jq '.events[0].message | fromjson | 
    {
      timestamp: .timestamp,
      cpu_used: .cpuUtilization.total,
      memory_free_mb: (.memory.free / 1024),
      read_iops: .diskIO[0].readIOsPS,
      write_iops: .diskIO[0].writeIOsPS
    }'
```

**5.3 — Analizar el Process List de Enhanced Monitoring**

```bash
# Extraer la lista de procesos con mayor consumo de CPU
aws logs get-log-events \
  --log-group-name "RDSOSMetrics" \
  --log-stream-name $(aws logs describe-log-streams \
    --log-group-name "RDSOSMetrics" \
    --log-stream-name-prefix $WRITER_INSTANCE_ID \
    --query 'logStreams[0].logStreamName' \
    --output text) \
  --limit 3 \
  --output json | jq '
    .events[-1].message | fromjson | 
    .processList | 
    sort_by(-.cpuUsedPc) | 
    .[:5] | 
    .[] | 
    {pid: .id, name: .name, cpu_pct: .cpuUsedPc, mem_rss_kb: .rss, vss_kb: .vss}
  '
```

**Salida esperada (ejemplo):**

```json
{"pid": 12847, "name": "postgres: labadmin labdb", "cpu_pct": 42.3, "mem_rss_kb": 524288, "vss_kb": 786432}
{"pid": 12901, "name": "postgres: labadmin labdb", "cpu_pct": 38.1, "mem_rss_kb": 262144, "vss_kb": 524288}
{"pid": 12756, "name": "postgres: checkpointer", "cpu_pct": 8.4, "mem_rss_kb": 16384, "vss_kb": 32768}
```

**Verificación:**

```bash
# Verificar que Enhanced Monitoring está publicando datos recientes (< 60 segundos)
aws logs describe-log-streams \
  --log-group-name "RDSOSMetrics" \
  --log-stream-name-prefix $WRITER_INSTANCE_ID \
  --query 'logStreams[0].lastEventTimestamp' \
  --output text | xargs -I {} python3 -c "
import datetime, sys
ts = int('{}') / 1000
last = datetime.datetime.utcfromtimestamp(ts)
now = datetime.datetime.utcnow()
diff = (now - last).seconds
print(f'Último dato hace {diff} segundos')
print('OK' if diff < 120 else 'ADVERTENCIA: datos desactualizados')
"
```

---

### Paso 6: Configurar Alarmas en CloudWatch para Detección Proactiva

**Objetivo:** Crear alarmas en CloudWatch basadas en métricas de Performance Insights y RDS para detectar automáticamente degradación de rendimiento antes de que impacte a los usuarios.

#### Instrucciones

**6.1 — Crear alarma para AAS elevado (saturación de CPU)**

```bash
# Obtener el Resource ID de PI (diferente del ARN de la instancia)
PI_RESOURCE_ID=$(aws rds describe-db-instances \
  --db-instance-identifier $WRITER_INSTANCE_ID \
  --query 'DBInstances[0].DbiResourceId' \
  --output text)
echo "PI Resource ID: $PI_RESOURCE_ID"

# Crear alarma: AAS > 1.5x vCPU (db.r6g.large = 2 vCPU, threshold = 3)
aws cloudwatch put-metric-alarm \
  --alarm-name "Aurora-PI-HighAAS-${WRITER_INSTANCE_ID}" \
  --alarm-description "AAS supera 1.5x vCPU en instancia writer Aurora" \
  --namespace "AWS/RDS" \
  --metric-name "DBLoad" \
  --dimensions Name=DbiResourceId,Value=$PI_RESOURCE_ID \
  --statistic Average \
  --period 60 \
  --threshold 3.0 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --datapoints-to-alarm 2 \
  --treat-missing-data notBreaching \
  --alarm-actions "arn:aws:sns:${AWS_REGION}:$(aws sts get-caller-identity --query Account --output text):aurora-lab-alerts"
```

**6.2 — Crear alarma para latencia de lectura elevada**

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "Aurora-HighReadLatency-${WRITER_INSTANCE_ID}" \
  --alarm-description "Latencia de lectura supera 20ms en instancia Aurora writer" \
  --namespace "AWS/RDS" \
  --metric-name "ReadLatency" \
  --dimensions Name=DBInstanceIdentifier,Value=$WRITER_INSTANCE_ID \
  --statistic Average \
  --period 60 \
  --threshold 0.020 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --datapoints-to-alarm 2 \
  --treat-missing-data notBreaching
```

**6.3 — Crear alarma compuesta para detección de contención de locks**

```bash
# Crear métrica personalizada para conteo de locks usando un script de monitoreo
cat > /tmp/publish_lock_metric.sh << 'SCRIPT'
#!/bin/bash
LOCK_COUNT=$(psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME -t -c \
  "SELECT COUNT(*) FROM pg_locks l 
   JOIN pg_stat_activity a ON l.pid = a.pid
   WHERE NOT l.granted AND a.state = 'active';")

aws cloudwatch put-metric-data \
  --namespace "AuroraLab/Locks" \
  --metric-name "BlockedSessions" \
  --value $LOCK_COUNT \
  --dimensions DBInstance=$WRITER_INSTANCE_ID \
  --region $AWS_REGION

echo "$(date): Blocked sessions = $LOCK_COUNT"
SCRIPT
chmod +x /tmp/publish_lock_metric.sh

# Ejecutar cada 30 segundos durante el laboratorio
for i in $(seq 1 10); do
  /tmp/publish_lock_metric.sh
  sleep 30
done &

# Crear alarma sobre la métrica personalizada
aws cloudwatch put-metric-alarm \
  --alarm-name "Aurora-BlockedSessions-${WRITER_INSTANCE_ID}" \
  --alarm-description "Sesiones bloqueadas por locks superan umbral crítico" \
  --namespace "AuroraLab/Locks" \
  --metric-name "BlockedSessions" \
  --dimensions Name=DBInstance,Value=$WRITER_INSTANCE_ID \
  --statistic Maximum \
  --period 30 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --datapoints-to-alarm 2 \
  --treat-missing-data notBreaching
```

**6.4 — Verificar el estado de las alarmas creadas**

```bash
aws cloudwatch describe-alarms \
  --alarm-name-prefix "Aurora-" \
  --query 'MetricAlarms[].{Name:AlarmName,State:StateValue,Threshold:Threshold,Metric:MetricName}' \
  --output table
```

**Salida esperada:**

```
-------------------------------------------------------------------------------------------
|                                    DescribeAlarms                                       |
+------------------------------------------+-------+------------+-----------------------+
|  Name                                    | State | Threshold  | Metric                |
+------------------------------------------+-------+------------+-----------------------+
|  Aurora-PI-HighAAS-aurora-lab-writer-1   | OK    |  3.0       | DBLoad                |
|  Aurora-HighReadLatency-aurora-lab-writer| ALARM |  0.02      | ReadLatency           |
|  Aurora-BlockedSessions-aurora-lab-writer| ALARM |  5.0       | BlockedSessions       |
+------------------------------------------+-------+------------+-----------------------+
```

**Verificación:**

```bash
# Confirmar que al menos 2 alarmas están creadas
ALARM_COUNT=$(aws cloudwatch describe-alarms \
  --alarm-name-prefix "Aurora-" \
  --query 'length(MetricAlarms)' \
  --output text)
echo "Alarmas creadas: $ALARM_COUNT"
[ "$ALARM_COUNT" -ge 2 ] && echo "✓ Verificación exitosa" || echo "✗ Crear alarmas faltantes"
```

---

## Validación y Pruebas

Una vez completados todos los pasos, ejecuta esta validación integral para confirmar que el laboratorio está correctamente configurado y que los datos observables están disponibles.

```bash
#!/bin/bash
# Script de validación integral del laboratorio 04-00-01
echo "=========================================="
echo "  VALIDACIÓN LAB 04-00-01"
echo "=========================================="

PASS=0
FAIL=0

check() {
    local desc="$1"
    local result="$2"
    if [ "$result" = "true" ] || [ "$result" -gt 0 ] 2>/dev/null; then
        echo "✓ $desc"
        PASS=$((PASS+1))
    else
        echo "✗ FALLO: $desc"
        FAIL=$((FAIL+1))
    fi
}

# 1. Performance Insights habilitado
PI_STATUS=$(aws rds describe-db-instances \
  --db-instance-identifier $WRITER_INSTANCE_ID \
  --query 'DBInstances[0].PerformanceInsightsEnabled' \
  --output text)
check "Performance Insights habilitado" "$PI_STATUS"

# 2. Enhanced Monitoring con granularidad de 1 segundo
EM_INTERVAL=$(aws rds describe-db-instances \
  --db-instance-identifier $WRITER_INSTANCE_ID \
  --query 'DBInstances[0].MonitoringInterval' \
  --output text)
check "Enhanced Monitoring con intervalo de 1 segundo" "$([ $EM_INTERVAL -eq 1 ] && echo true)"

# 3. pg_stat_statements instalado
EXT_STATUS=$(psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME -t \
  -c "SELECT COUNT(*) FROM pg_extension WHERE extname = 'pg_stat_statements';" | tr -d ' ')
check "Extensión pg_stat_statements instalada" "$EXT_STATUS"

# 4. track_io_timing activo
IO_TIMING=$(psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME -t \
  -c "SHOW track_io_timing;" | tr -d ' \n')
check "track_io_timing habilitado" "$([ "$IO_TIMING" = "on" ] && echo true)"

# 5. Datos en pg_stat_statements
STMT_COUNT=$(psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME -t \
  -c "SELECT COUNT(*) FROM pg_stat_statements WHERE calls > 0;" | tr -d ' ')
check "pg_stat_statements tiene datos ($STMT_COUNT consultas)" "$STMT_COUNT"

# 6. Tabla lab_orders poblada
ORDER_COUNT=$(psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME -t \
  -c "SELECT COUNT(*) FROM lab_orders;" | tr -d ' ')
check "lab_orders poblada ($ORDER_COUNT registros)" "$([ $ORDER_COUNT -gt 1000000 ] && echo true)"

# 7. Alarmas CloudWatch creadas
ALARM_COUNT=$(aws cloudwatch describe-alarms \
  --alarm-name-prefix "Aurora-" \
  --query 'length(MetricAlarms)' \
  --output text)
check "Alarmas CloudWatch creadas ($ALARM_COUNT)" "$([ $ALARM_COUNT -ge 2 ] && echo true)"

# 8. PI API responde correctamente
PI_RESPONSE=$(aws pi get-resource-metrics \
  --service-type RDS \
  --identifier $WRITER_ARN \
  --metric-queries '[{"Metric":"db.load.avg","GroupBy":{"Group":"db.wait_event_type","Limit":3}}]' \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -v-10M +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period-in-seconds 60 \
  --query 'length(MetricList)' \
  --output text 2>/dev/null)
check "PI API retorna datos" "$([ "${PI_RESPONSE:-0}" -gt 0 ] && echo true)"

echo "------------------------------------------"
echo "Resultados: $PASS pasaron, $FAIL fallaron"
[ $FAIL -eq 0 ] && echo "✓ LABORATORIO COMPLETADO EXITOSAMENTE" || echo "✗ Revisar los ítems fallidos"
echo "=========================================="
```

### Preguntas de Reflexión

Responde estas preguntas en tu cuaderno de laboratorio como evidencia de comprensión:

1. **Sobre AAS:** Durante la carga generada, ¿el AAS superó la línea de vCPU de la instancia? ¿Qué implica esto para los usuarios finales?

2. **Sobre wait events:** ¿Cuál fue el wait event dominante durante la ejecución del script de full table scans? ¿Coincide con `IO:DataFileRead`? ¿Por qué?

3. **Sobre pg_stat_statements:** ¿Qué consulta tuvo el mayor `cv_ratio` (coeficiente de variación)? ¿Qué tipo de problema indica una alta variabilidad de latencia?

4. **Sobre Enhanced Monitoring:** ¿Qué proceso del SO consumió más CPU durante la carga? ¿Era un proceso `postgres: backend` o un proceso de background?

5. **Sobre correlación:** Si en PI observas `IO:DataFileRead` alto y en `pg_stat_statements` identificas la consulta responsable, ¿qué acción de optimización aplicarías primero?

---

## Solución de Problemas

### Problema 1: Performance Insights no muestra datos de Top SQL (tabla vacía)

**Síntomas:**
- El gráfico de AAS muestra actividad, pero la tabla "Top SQL" aparece vacía o con "No data available".
- La dimensión SQL no muestra consultas aunque hay carga activa.

**Causa:**
La extensión `pg_stat_statements` no está cargada en `shared_preload_libraries`, o el parámetro `pg_stat_statements.track` no está configurado como `all`. Performance Insights requiere `pg_stat_statements` activo para mostrar el texto de las consultas en Top SQL. También puede ocurrir si la extensión fue instalada pero la instancia no fue reiniciada después de modificar `shared_preload_libraries`.

**Solución:**

```bash
# Paso 1: Verificar que shared_preload_libraries incluye pg_stat_statements
psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME \
  -c "SHOW shared_preload_libraries;"
# Debe incluir 'pg_stat_statements'

# Paso 2: Si no está, modificar el parameter group (requiere reboot)
aws rds modify-db-parameter-group \
  --db-parameter-group-name $PG_GROUP \
  --parameters \
    "ParameterName=shared_preload_libraries,ParameterValue=pg_stat_statements,ApplyMethod=pending-reboot"

# Reiniciar la instancia (solo si es aceptable en el entorno de lab)
aws rds reboot-db-instance \
  --db-instance-identifier $WRITER_INSTANCE_ID

# Esperar a que esté disponible
aws rds wait db-instance-available \
  --db-instance-identifier $WRITER_INSTANCE_ID

# Paso 3: Reinstalar la extensión después del reboot
psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME \
  -c "CREATE EXTENSION IF NOT EXISTS pg_stat_statements;"

# Paso 4: Generar algo de carga y esperar 2-3 minutos para que PI actualice
pgbench -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME \
  --time=60 --client=5 > /dev/null 2>&1
```

---

### Problema 2: Enhanced Monitoring no muestra datos o el log group no existe

**Síntomas:**
- La pestaña "Monitoring" de la instancia no muestra la opción "Enhanced monitoring" con granularidad de 1 segundo.
- El comando `aws logs describe-log-streams --log-group-name "RDSOSMetrics"` retorna error `ResourceNotFoundException`.
- El Process List de Enhanced Monitoring aparece vacío.

**Causa:**
Enhanced Monitoring requiere un rol IAM específico (`rds-monitoring-role`) con la política `AmazonRDSEnhancedMonitoringRole`. Si la instancia fue creada sin este rol, o si el rol no tiene los permisos para publicar en CloudWatch Logs, los datos no se recolectarán. También puede ocurrir si el intervalo de monitoreo está configurado en `0` (deshabilitado).

**Solución:**

```bash
# Paso 1: Verificar el rol de monitoreo actual
aws rds describe-db-instances \
  --db-instance-identifier $WRITER_INSTANCE_ID \
  --query 'DBInstances[0].{MonitoringInterval:MonitoringInterval,
           MonitoringRoleArn:MonitoringRoleArn}' \
  --output table

# Paso 2: Crear el rol si no existe
aws iam create-role \
  --role-name rds-enhanced-monitoring-lab \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{
      "Effect":"Allow",
      "Principal":{"Service":"monitoring.rds.amazonaws.com"},
      "Action":"sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name rds-enhanced-monitoring-lab \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole

MONITORING_ROLE_ARN=$(aws iam get-role \
  --role-name rds-enhanced-monitoring-lab \
  --query 'Role.Arn' --output text)

# Paso 3: Habilitar Enhanced Monitoring con granularidad de 1 segundo
aws rds modify-db-instance \
  --db-instance-identifier $WRITER_INSTANCE_ID \
  --monitoring-interval 1 \
  --monitoring-role-arn $MONITORING_ROLE_ARN \
  --apply-immediately

# Paso 4: Esperar 2-3 minutos y verificar que el log group existe
sleep 120
aws logs describe-log-groups \
  --log-group-name-prefix "RDSOSMetrics" \
  --query 'logGroups[].logGroupName' \
  --output text
```

---

## Limpieza de Recursos

> ⚠️ **Ejecuta la limpieza al finalizar el laboratorio** para evitar costos innecesarios. Los recursos Aurora generan costos por hora incluso cuando están inactivos.

```bash
# Paso 1: Detener todos los procesos de carga en background
kill $PGBENCH_PID 2>/dev/null
pkill -f "run_full_scans.sh" 2>/dev/null
pkill -f "run_sorts.sh" 2>/dev/null
pkill -f "run_lock_contention.sh" 2>/dev/null
pkill -f "publish_lock_metric.sh" 2>/dev/null
echo "Procesos de carga detenidos."

# Paso 2: Eliminar tablas del laboratorio
psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME << 'EOF'
DROP TABLE IF EXISTS lab_orders CASCADE;
DROP TABLE IF EXISTS lab_inventory CASCADE;
DROP TABLE IF EXISTS pgbench_accounts CASCADE;
DROP TABLE IF EXISTS pgbench_branches CASCADE;
DROP TABLE IF EXISTS pgbench_tellers CASCADE;
DROP TABLE IF EXISTS pgbench_history CASCADE;
COMMIT;
EOF
echo "Tablas del laboratorio eliminadas."

# Paso 3: Eliminar alarmas de CloudWatch
aws cloudwatch delete-alarms \
  --alarm-names \
    "Aurora-PI-HighAAS-${WRITER_INSTANCE_ID}" \
    "Aurora-HighReadLatency-${WRITER_INSTANCE_ID}" \
    "Aurora-BlockedSessions-${WRITER_INSTANCE_ID}"
echo "Alarmas CloudWatch eliminadas."

# Paso 4: Limpiar archivos temporales
rm -f /tmp/pgbench_oltp* /tmp/run_*.sh /tmp/publish_lock_metric.sh
echo "Archivos temporales eliminados."

# Paso 5: Destruir infraestructura con Terraform
cd ~/aurora-labs/04-00-01/terraform
terraform destroy -auto-approve
echo "Infraestructura Terraform destruida."

# Paso 6: Verificar que no quedan instancias activas
aws rds describe-db-instances \
  --query 'DBInstances[?DBInstanceStatus==`available`].{ID:DBInstanceIdentifier,Status:DBInstanceStatus,Class:DBInstanceClass}' \
  --output table
```

> **Nota:** Si planeas continuar con el laboratorio 04-00-02 en la misma sesión, puedes omitir el paso 5 (Terraform destroy) y mantener la infraestructura activa.

---

## Resumen

En este laboratorio aplicaste las herramientas de observabilidad nativas de AWS para Aurora PostgreSQL en un escenario de carga real controlada. Los conceptos clave practicados fueron:

| Concepto | Lo que hiciste | Resultado observable |
|---|---|---|
| **AAS y saturación** | Generaste carga con pgbench y consultas problemáticas | Gráfico AAS superando la línea de vCPU |
| **Wait events** | Ejecutaste full scans, sorts y contención de locks | `IO:DataFileRead`, `Lock:transactionid` visibles en PI |
| **Top SQL en PI** | Correlacionaste wait events con consultas específicas | Identificaste la consulta de `lab_orders` como responsable del IO |
| **pg_stat_statements** | Configuraste `track_io_timing = on` y construiste queries analíticas | Ranking de consultas por `total_exec_time`, `cv_ratio` y `io_pct` |
| **Enhanced Monitoring** | Observaste métricas de proceso del SO a 1 segundo | Process list con procesos postgres y su consumo de CPU/memoria |
| **CloudWatch Alarms** | Creaste alarmas para AAS, latencia y sesiones bloqueadas | Detección proactiva de degradación de rendimiento |

### Flujo de Troubleshooting Aprendido

```
Pico en AAS
    ↓
¿Domina IO, Lock, CPU o LWLock?
    ↓
Cambiar dimensión a SQL en PI → Identificar consulta responsable
    ↓
Confirmar con pg_stat_statements (total_exec_time, blk_read_time, cv_ratio)
    ↓
Aplicar corrección focalizada (índice, batch, parámetro, work_mem)
    ↓
Validar reducción en PI (mismo intervalo, carga similar)
```

### Recursos Adicionales

- [Documentación oficial: Performance Insights para Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_PerfInsights.html)
- [API de Performance Insights: describe-dimension-keys](https://docs.aws.amazon.com/cli/latest/reference/pi/describe-dimension-keys.html)
- [pg_stat_statements — PostgreSQL 14 Documentation](https://www.postgresql.org/docs/14/pgstatstatements.html)
- [Wait Events de PostgreSQL](https://www.postgresql.org/docs/current/monitoring-stats.html#WAIT-EVENT-TABLE)
- [Enhanced Monitoring para Amazon RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_Monitoring.OS.html)
- [Blog AWS: Using Performance Insights API to retrieve performance data](https://aws.amazon.com/blogs/database/using-the-performance-insights-api-to-retrieve-performance-data/)

---
*Lab 04-00-01 — Módulo 4: Observabilidad y Optimización Avanzada en Aurora PostgreSQL*

---

# Diagnóstico y optimización de rendimiento

## Metadatos

| Campo            | Detalle                          |
|------------------|----------------------------------|
| **Duración**     | 45 minutos                       |
| **Complejidad**  | Alta                             |
| **Nivel Bloom**  | Aplicar (Apply)                  |
| **Módulo**       | 4 — Observabilidad y Optimización|
| **Laboratorio**  | 04-00-02 (Práctica 8)            |

---

## Visión General

Este laboratorio integra todas las herramientas de observabilidad de Aurora PostgreSQL —Performance Insights, `pg_stat_statements` y `EXPLAIN ANALYZE`— en un flujo de trabajo de diagnóstico y optimización *end-to-end*. Recibirás una base de datos con problemas de rendimiento pre-configurados (consultas sin índices apropiados, estadísticas desactualizadas, `work_mem` insuficiente para operaciones de ordenamiento y patrones N+1) y deberás diagnosticarlos, resolverlos y verificar la mejora con métricas cuantitativas *before/after*. El laboratorio culmina con la documentación estructurada del proceso de optimización.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio serás capaz de:

- [ ] Diagnosticar consultas lentas identificadas en Performance Insights y `pg_stat_statements` aplicando el ciclo completo: identificar → analizar plan → implementar solución → verificar mejora.
- [ ] Optimizar consultas problemáticas mediante creación de índices de cobertura, índices parciales, reescritura de SQL y ajuste de estadísticas con `ANALYZE`.
- [ ] Identificar y resolver cuellos de botella de I/O mediante análisis de patrones de acceso, uso de índices y estrategias de caching (`shared_buffers`/`work_mem`).
- [ ] Ajustar `work_mem` a nivel de sesión para eliminar *disk spills* en operaciones de ordenamiento y hash join.
- [ ] Documentar el proceso de optimización con métricas *before/after* utilizando `pg_stat_statements` y Performance Insights.

---

## Prerrequisitos

### Conocimiento previo

- Haber completado el **Laboratorio 7 (04-00-01)**: dominio de herramientas de observabilidad (Performance Insights, Enhanced Monitoring, `pg_stat_statements`).
- Comprensión de `EXPLAIN ANALYZE` y lectura de planes de ejecución (Seq Scan, Index Scan, Hash Join, Sort).
- Familiaridad con creación de índices en PostgreSQL (B-tree, índices parciales, índices de cobertura con `INCLUDE`).
- Conocimiento básico de AWS CLI y consola de RDS.

### Acceso y recursos

- Clúster Aurora PostgreSQL activo (writer + 1 reader), instancia mínima **db.r6g.large**.
- **Performance Insights habilitado** con al menos 1 hora de datos históricos acumulados.
- `pg_stat_statements` habilitado en el parameter group y con datos de carga previa (carga generada en Lab 7).
- Usuario IAM con permisos sobre RDS, CloudWatch, Secrets Manager y VPC.
- Base de datos `labdb` con el esquema de problemas pre-configurados (disponible en el repositorio del curso: `labs/04-00-02/`).
- Herramientas locales: `psql`, `pgbench`, AWS CLI 2.x, Python 3.9+, `jq`.

---

## Entorno de Laboratorio

### Hardware recomendado

| Componente      | Mínimo requerido                              |
|-----------------|-----------------------------------------------|
| RAM             | 8 GB (para scripts de carga concurrente)      |
| CPU             | 4 núcleos                                     |
| Almacenamiento  | 10 GB libres (logs y resultados)              |
| Red             | 10 Mbps hacia AWS                             |

### Software requerido

| Herramienta              | Versión          |
|--------------------------|------------------|
| AWS CLI                  | 2.x              |
| psql / pgbench           | 14.x o superior  |
| Python                   | 3.9 o superior   |
| DBeaver Community        | 23.x o superior  |
| jq                       | 1.6 o superior   |
| Navegador web            | Chrome/Firefox   |

### Configuración inicial del entorno

> **⚠️ Importante:** Ejecuta estos comandos antes de iniciar los pasos del laboratorio. Si ya completaste el Lab 7 y el clúster está activo, ve directamente al Paso 1.

**1. Verificar variables de entorno y conectividad:**

```bash
# Exportar variables de entorno base
export AWS_REGION="us-east-1"
export CLUSTER_ID="aurora-lab-cluster"
export DB_WRITER_ENDPOINT=$(aws rds describe-db-clusters \
  --db-cluster-identifier $CLUSTER_ID \
  --query 'DBClusters[0].Endpoint' \
  --output text)
export DB_PORT=5432
export DB_NAME="labdb"
export DB_USER="labadmin"

echo "Writer endpoint: $DB_WRITER_ENDPOINT"

# Verificar conectividad
psql -h $DB_WRITER_ENDPOINT -p $DB_PORT -U $DB_USER -d $DB_NAME -c "SELECT version();"
```

**2. Cargar el esquema con problemas pre-configurados (si no está cargado):**

```bash
# Clonar repositorio del curso (si no está clonado)
git clone https://github.com/curso-aurora-pg/labs.git ~/aurora-labs
cd ~/aurora-labs/04-00-02

# Cargar esquema con problemas de rendimiento
psql -h $DB_WRITER_ENDPOINT -p $DB_PORT -U $DB_USER -d $DB_NAME \
  -f setup/01_create_schema_with_issues.sql

# Cargar datos de prueba (~2M filas)
psql -h $DB_WRITER_ENDPOINT -p $DB_PORT -U $DB_USER -d $DB_NAME \
  -f setup/02_load_test_data.sql

echo "Esquema y datos cargados correctamente."
```

**3. Verificar que `pg_stat_statements` esté activo:**

```bash
psql -h $DB_WRITER_ENDPOINT -p $DB_PORT -U $DB_USER -d $DB_NAME -c \
  "SELECT name, setting FROM pg_settings WHERE name LIKE 'pg_stat_statements%';"
```

Resultado esperado: parámetros `pg_stat_statements.track = all` y `pg_stat_statements.max = 10000`.

---

## Pasos del Laboratorio

---

### Paso 1: Generar carga de trabajo y capturar línea base en Performance Insights

**Objetivo:** Generar carga con las consultas problemáticas pre-configuradas y establecer una línea base de métricas AAS, waits y top SQL en Performance Insights antes de aplicar cualquier optimización.

#### Instrucciones

**1.1. Resetear estadísticas de `pg_stat_statements` para obtener métricas limpias:**

```sql
-- Conectar a la base de datos
psql -h $DB_WRITER_ENDPOINT -p $DB_PORT -U $DB_USER -d $DB_NAME

-- Resetear estadísticas previas
SELECT pg_stat_statements_reset();

-- Verificar reset
SELECT count(*) FROM pg_stat_statements;
-- Debe retornar 0 o muy pocas filas del reset mismo
```

**1.2. Ejecutar el script de carga que simula los problemas pre-configurados:**

```bash
# En una terminal separada, ejecutar el script de carga durante 5 minutos
cd ~/aurora-labs/04-00-02

python3 scripts/generate_load_with_issues.py \
  --host $DB_WRITER_ENDPOINT \
  --port $DB_PORT \
  --dbname $DB_NAME \
  --user $DB_USER \
  --duration 300 \
  --concurrency 20

# El script simula:
# - Consultas con sequential scan en tablas grandes (problema I/O)
# - Consultas con sort sin work_mem suficiente (disk spill)
# - Patrón N+1 (muchas queries individuales)
# - Consultas con estadísticas desactualizadas
```

**1.3. Mientras corre la carga, capturar snapshot de métricas baseline en `pg_stat_statements`:**

```sql
-- Guardar baseline en tabla temporal para comparación posterior
CREATE TABLE IF NOT EXISTS lab_baseline_stats AS
SELECT
    queryid,
    LEFT(query, 120)                         AS query_preview,
    calls,
    total_exec_time,
    mean_exec_time,
    stddev_exec_time,
    rows,
    shared_blks_hit,
    shared_blks_read,
    shared_blks_dirtied,
    temp_blks_read,
    temp_blks_written,
    blk_read_time,
    blk_write_time,
    NOW()                                    AS captured_at
FROM pg_stat_statements
WHERE calls > 5
ORDER BY total_exec_time DESC
LIMIT 20;

SELECT count(*) AS queries_capturadas FROM lab_baseline_stats;
```

**1.4. Capturar el ARN de la instancia para usar con la PI API:**

```bash
export DB_INSTANCE_ARN=$(aws rds describe-db-instances \
  --filters "Name=db-cluster-id,Values=$CLUSTER_ID" \
  --query 'DBInstances[?contains(DBInstanceIdentifier, `writer`) || DBInstanceArn != null] | [0].DBInstanceArn' \
  --output text)

# Alternativa directa
export DB_INSTANCE_ARN=$(aws rds describe-db-instances \
  --query "DBInstances[?DBClusterIdentifier=='$CLUSTER_ID' && DBInstanceRole=='WRITER'].DBInstanceArn | [0]" \
  --output text)

echo "Instance ARN: $DB_INSTANCE_ARN"
```

**1.5. Consultar métricas AAS baseline desde la PI API:**

```bash
# Obtener AAS desglosado por wait_event_type durante la ventana de carga
START_TIME=$(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || \
             date -u -v-10M +%Y-%m-%dT%H:%M:%SZ)
END_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)

aws pi get-resource-metrics \
  --service-type RDS \
  --identifier "$DB_INSTANCE_ARN" \
  --metric db.load.avg \
  --start-time "$START_TIME" \
  --end-time "$END_TIME" \
  --period-in-seconds 60 \
  --metric-queries '[
    {
      "Metric": "db.load.avg",
      "GroupBy": {
        "Group": "db.wait_event_type",
        "Dimensions": ["db.wait_event_type"],
        "Limit": 5
      }
    }
  ]' | jq '.MetricList[].DataPoints[] | {timestamp: .Timestamp, value: .Value}' \
  | head -30
```

#### Salida esperada

```
Esquema y datos cargados correctamente.
queries_capturadas
--------------------
                 20
Instance ARN: arn:aws:rds:us-east-1:123456789012:db:aurora-lab-writer-1
```

En Performance Insights (consola), deberías ver AAS > 2 con dominancia de `IO:DataFileRead` y posiblemente `CPU`.

#### Verificación

```bash
# Confirmar que hay datos en la tabla baseline
psql -h $DB_WRITER_ENDPOINT -p $DB_PORT -U $DB_USER -d $DB_NAME -c \
  "SELECT query_preview, calls, ROUND(mean_exec_time::numeric, 2) AS mean_ms,
          temp_blks_written AS disk_spill_blks
   FROM lab_baseline_stats
   ORDER BY mean_exec_time DESC LIMIT 5;"
```

Debes ver al menos 3 consultas con `mean_ms > 500` y algunas con `disk_spill_blks > 0`.

---

### Paso 2: Identificar las consultas problemáticas con `pg_stat_statements` y Performance Insights

**Objetivo:** Usar `pg_stat_statements` y la consola de Performance Insights para identificar con precisión las 4 consultas problemáticas pre-configuradas y clasificar el tipo de problema de cada una.

#### Instrucciones

**2.1. Identificar las top queries por tiempo total de ejecución:**

```sql
-- Top 10 queries por tiempo total (candidatas a optimizar)
SELECT
    queryid,
    LEFT(query, 100)                                AS query_preview,
    calls,
    ROUND(total_exec_time::numeric, 2)              AS total_ms,
    ROUND(mean_exec_time::numeric, 2)               AS mean_ms,
    ROUND(stddev_exec_time::numeric, 2)             AS stddev_ms,
    ROUND((shared_blks_read * 100.0 /
           NULLIF(shared_blks_hit + shared_blks_read, 0))::numeric, 2)
                                                    AS cache_miss_pct,
    temp_blks_written                               AS disk_spill_blks,
    rows
FROM pg_stat_statements
WHERE calls > 5
  AND query NOT LIKE '%pg_stat%'
ORDER BY total_exec_time DESC
LIMIT 10;
```

**2.2. Identificar consultas con disk spills (work_mem insuficiente):**

```sql
-- Consultas que generan escritura en disco temporal
SELECT
    queryid,
    LEFT(query, 100)                               AS query_preview,
    calls,
    ROUND(mean_exec_time::numeric, 2)              AS mean_ms,
    temp_blks_written,
    temp_blks_read,
    ROUND((temp_blks_written * 8192.0 / 1024 / 1024)::numeric, 2)
                                                   AS disk_spill_mb
FROM pg_stat_statements
WHERE temp_blks_written > 0
  AND calls > 3
ORDER BY temp_blks_written DESC;
```

**2.3. Identificar consultas con alto cache miss (candidatas a índices o shared_buffers):**

```sql
-- Consultas con alta proporción de lecturas desde disco
SELECT
    queryid,
    LEFT(query, 100)                               AS query_preview,
    calls,
    shared_blks_hit,
    shared_blks_read,
    ROUND((shared_blks_read * 100.0 /
           NULLIF(shared_blks_hit + shared_blks_read, 0))::numeric, 2)
                                                   AS cache_miss_pct,
    ROUND(mean_exec_time::numeric, 2)              AS mean_ms
FROM pg_stat_statements
WHERE (shared_blks_hit + shared_blks_read) > 1000
  AND calls > 3
ORDER BY cache_miss_pct DESC, shared_blks_read DESC
LIMIT 10;
```

**2.4. Identificar el patrón N+1 (muchas llamadas con bajo tiempo individual pero alto tiempo total):**

```sql
-- Patrón N+1: muchas llamadas, bajo mean_ms, alto total_ms
SELECT
    queryid,
    LEFT(query, 100)                               AS query_preview,
    calls,
    ROUND(mean_exec_time::numeric, 4)              AS mean_ms,
    ROUND(total_exec_time::numeric, 2)             AS total_ms,
    rows
FROM pg_stat_statements
WHERE calls > 100
  AND mean_exec_time < 50
  AND total_exec_time > 5000
ORDER BY calls DESC
LIMIT 5;
```

**2.5. Analizar en Performance Insights (consola AWS):**

1. Navega a **RDS Console → Performance Insights** → selecciona la instancia writer.
2. Establece el intervalo en **los últimos 15 minutos**.
3. Observa el gráfico AAS apilado: identifica la categoría dominante (`IO`, `CPU`, `Lock`).
4. Cambia la dimensión a **SQL** para ver las top queries que contribuyen a la carga.
5. Haz clic en la query con mayor contribución → observa su desglose de waits.
6. **Documenta** en tu cuaderno de laboratorio:
   - Categoría de wait dominante
   - Los 3 queryids con mayor contribución a AAS
   - Si alguna query muestra `IO:DataFileRead` como wait principal

#### Salida esperada

Ejemplo representativo de la salida del paso 2.1:

```
 queryid | query_preview                                          | calls | total_ms  | mean_ms | cache_miss_pct | disk_spill_blks
---------+--------------------------------------------------------+-------+-----------+---------+----------------+-----------------
  123456 | SELECT * FROM orders o JOIN customers c ON ...         |   450 | 234500.00 |  521.11 |          87.50 |               0
  234567 | SELECT o.* FROM orders o WHERE status = 'pending' ...  |   380 | 198000.00 |  521.05 |          92.30 |               0
  345678 | SELECT id, name FROM products ORDER BY price DESC ...  |   200 |  96000.00 |  480.00 |          15.20 |            8450
  456789 | SELECT * FROM customers WHERE customer_id = $1         |  4500 |  45000.00 |   10.00 |           8.10 |               0
```

#### Verificación

```bash
# Confirmar que identificaste los 4 tipos de problemas
psql -h $DB_WRITER_ENDPOINT -p $DB_PORT -U $DB_USER -d $DB_NAME -c "
SELECT
  CASE
    WHEN temp_blks_written > 0 THEN 'DISK_SPILL (work_mem)'
    WHEN (shared_blks_read * 100.0 / NULLIF(shared_blks_hit + shared_blks_read,0)) > 80
         THEN 'HIGH_IO (missing index)'
    WHEN calls > 500 AND mean_exec_time < 20 THEN 'N+1 PATTERN'
    ELSE 'OTHER'
  END AS problem_type,
  count(*) AS query_count
FROM pg_stat_statements
WHERE calls > 5 AND query NOT LIKE '%pg_stat%'
GROUP BY 1
ORDER BY 2 DESC;"
```

---

### Paso 3: Analizar planes de ejecución con `EXPLAIN ANALYZE BUFFERS`

**Objetivo:** Obtener y analizar el plan de ejecución detallado de cada consulta problemática para confirmar la causa raíz antes de implementar cualquier solución.

#### Instrucciones

**3.1. Analizar la consulta con alto I/O (Seq Scan en tabla grande):**

```sql
-- Habilitar timing de I/O para el análisis
SET track_io_timing = on;

-- Analizar la consulta problemática de I/O
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.order_id, o.order_date, o.total_amount, c.customer_name, c.email
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'pending'
  AND o.order_date >= NOW() - INTERVAL '30 days';
```

> **Qué buscar en el plan:**
> - `Seq Scan on orders` con `rows=` alto → falta índice en `status` o `order_date`.
> - `Buffers: shared read=XXXXX` con número alto → lecturas desde disco.
> - `actual time=` alto en el nodo de Seq Scan.

**3.2. Analizar la consulta con disk spill (Sort/Hash Join sin work_mem suficiente):**

```sql
-- Identificar el spill
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT p.product_id, p.product_name, p.category, p.price,
       COUNT(oi.order_item_id) AS total_sold,
       SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name, p.category, p.price
ORDER BY total_revenue DESC;
```

> **Qué buscar en el plan:**
> - `Sort Method: external merge  Disk: XXXXX kB` → disk spill por sort.
> - O bien `Hash Batches: N  (original: 1)` con N > 1 → hash join spill.
> - `Temp Files` en el nodo Sort o HashAggregate.

**3.3. Analizar la consulta con estadísticas desactualizadas:**

```sql
-- Verificar estadísticas de la tabla
SELECT
    schemaname,
    tablename,
    last_analyze,
    last_autoanalyze,
    n_live_tup,
    n_dead_tup,
    n_mod_since_analyze
FROM pg_stat_user_tables
WHERE tablename IN ('orders', 'customers', 'products', 'order_items')
ORDER BY n_mod_since_analyze DESC;
```

```sql
-- Analizar la consulta afectada por estadísticas desactualizadas
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT customer_id, COUNT(*) AS order_count, SUM(total_amount) AS lifetime_value
FROM orders
WHERE status IN ('completed', 'shipped')
GROUP BY customer_id
HAVING COUNT(*) > 5
ORDER BY lifetime_value DESC
LIMIT 100;
```

> **Qué buscar:** `rows=` estimado muy diferente de `rows=` actual (actual rows=XXXX vs estimated rows=YY). Esto indica estadísticas obsoletas que llevan al planificador a elegir planes subóptimos.

**3.4. Examinar el uso actual de índices en las tablas problemáticas:**

```sql
-- Índices existentes y su uso
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE tablename IN ('orders', 'customers', 'products', 'order_items')
ORDER BY tablename, idx_scan DESC;
```

#### Salida esperada

Para el paso 3.1, deberías ver algo similar a:

```
Gather  (cost=1000.00..45234.56 rows=12500 width=68)
  ->  Hash Join  (cost=...)
        ->  Seq Scan on orders  (cost=0.00..38000.00 rows=12500 width=44)
              Filter: ((status = 'pending') AND (order_date >= ...))
              Rows Removed by Filter: 1987500
              Buffers: shared hit=234 read=18766
        ->  Hash  (...)
              ->  Seq Scan on customers  (...)
Planning Time: 2.345 ms
Execution Time: 4823.567 ms
```

El `Seq Scan` con `Rows Removed by Filter: 1987500` y `Buffers: shared read=18766` confirma la necesidad de un índice.

#### Verificación

```bash
# Guardar los planes de ejecución en archivos para referencia posterior
psql -h $DB_WRITER_ENDPOINT -p $DB_PORT -U $DB_USER -d $DB_NAME \
  -c "SET track_io_timing=on; EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
      SELECT o.order_id, o.order_date FROM orders o
      WHERE o.status = 'pending' AND o.order_date >= NOW() - INTERVAL '30 days';" \
  > ~/aurora-labs/04-00-02/results/plan_before_io_fix.json

echo "Plan guardado en results/plan_before_io_fix.json"
```

---

### Paso 4: Implementar optimizaciones — Índices, estadísticas y `work_mem`

**Objetivo:** Aplicar las soluciones identificadas en los pasos anteriores: crear índices de cobertura e índices parciales, actualizar estadísticas con target aumentado y ajustar `work_mem` para eliminar disk spills.

#### Instrucciones

**4.1. Crear índice parcial para la consulta con filtro selectivo en `status`:**

```sql
-- Problema: Seq Scan en orders filtrando por status='pending' y order_date
-- Solución: Índice parcial (solo filas 'pending') + columnas de cobertura

-- ANTES: medir tiempo sin índice
\timing on

SELECT o.order_id, o.order_date, o.total_amount, c.customer_name, c.email
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status = 'pending'
  AND o.order_date >= NOW() - INTERVAL '30 days'
LIMIT 100;

-- Crear índice parcial de cobertura
CREATE INDEX CONCURRENTLY idx_orders_pending_date_cover
ON orders (order_date DESC)
INCLUDE (customer_id, total_amount)
WHERE status = 'pending';

-- Verificar creación
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'orders' AND indexname = 'idx_orders_pending_date_cover';
```

**4.2. Crear índice de cobertura para eliminar heap fetches en la consulta de `customers`:**

```sql
-- Problema: N+1 pattern - SELECT * FROM customers WHERE customer_id = $1
-- Solución: Verificar si el índice existente es de cobertura o requiere heap fetch

-- Ver el plan de la consulta N+1
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, customer_name, email, phone, city
FROM customers
WHERE customer_id = 12345;

-- Si hay "Heap Fetches" alto en Index Scan, crear índice de cobertura
CREATE INDEX CONCURRENTLY idx_customers_id_cover
ON customers (customer_id)
INCLUDE (customer_name, email, phone, city);

-- Verificar uso del nuevo índice
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, customer_name, email, phone, city
FROM customers
WHERE customer_id = 12345;
-- Debe mostrar "Index Only Scan" con Heap Fetches: 0
```

**4.3. Actualizar estadísticas con `statistics_target` aumentado para la tabla `orders`:**

```sql
-- Problema: Estimaciones incorrectas del planificador por estadísticas obsoletas
-- Solución: Aumentar statistics_target para columnas con distribución no uniforme

-- Ver el target actual
SELECT attname, attstattarget
FROM pg_attribute
WHERE attrelid = 'orders'::regclass
  AND attname IN ('status', 'order_date', 'customer_id', 'total_amount');

-- Aumentar statistics_target para columnas críticas
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;
ALTER TABLE orders ALTER COLUMN order_date SET STATISTICS 500;
ALTER TABLE orders ALTER COLUMN total_amount SET STATISTICS 300;

-- Ejecutar ANALYZE con el nuevo target
ANALYZE VERBOSE orders;

-- Verificar que las estadísticas se actualizaron
SELECT
    tablename,
    attname,
    n_distinct,
    correlation,
    most_common_vals,
    most_common_freqs
FROM pg_stats
WHERE tablename = 'orders'
  AND attname IN ('status', 'order_date', 'total_amount')
ORDER BY attname;
```

**4.4. Resolver disk spill ajustando `work_mem` a nivel de sesión:**

```sql
-- Problema: Sort/Hash Join con disk spill por work_mem insuficiente
-- Verificar el valor actual
SHOW work_mem;

-- Ajustar work_mem SOLO para esta sesión (no cambiar globalmente)
SET work_mem = '64MB';

-- Re-ejecutar la consulta problemática con el nuevo work_mem
EXPLAIN (ANALYZE, BUFFERS)
SELECT p.product_id, p.product_name, p.category, p.price,
       COUNT(oi.order_item_id) AS total_sold,
       SUM(oi.quantity * oi.unit_price) AS total_revenue
FROM products p
JOIN order_items oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name, p.category, p.price
ORDER BY total_revenue DESC;

-- Verificar que desapareció el disk spill:
-- "Sort Method: quicksort  Memory: XXXXkB" (sin "Disk:")
-- "Batches: 1" en HashAggregate (sin batches adicionales)
```

**4.5. Habilitar `auto_explain` para captura automática de planes lentos:**

```sql
-- Habilitar auto_explain para queries > 1 segundo (útil para monitoreo continuo)
LOAD 'auto_explain';
SET auto_explain.log_min_duration = '1000';   -- 1 segundo
SET auto_explain.log_analyze = true;
SET auto_explain.log_buffers = true;
SET auto_explain.log_format = 'text';

-- Confirmar configuración
SHOW auto_explain.log_min_duration;
SHOW auto_explain.log_analyze;
```

> **Nota:** Para habilitar `auto_explain` de forma persistente en Aurora, agrégalo a `shared_preload_libraries` en el parameter group y reinicia la instancia. El ajuste a nivel de sesión es suficiente para este laboratorio.

#### Salida esperada

```
CREATE INDEX
Index "public.idx_orders_pending_date_cover"
  Column     | Type
  order_date | timestamp without time zone
  (customer_id, total_amount included)
  Predicate: (status = 'pending')

ANALYZE
INFO:  analyzing "public.orders"
INFO:  "orders": scanned 30000 of 30000 pages, containing 2000000 live rows and 0 dead rows;
       30000 rows in sample, 2000000 estimated total rows

work_mem
---------
 64MB
```

#### Verificación

```sql
-- Confirmar que los índices fueron creados y son válidos
SELECT
    indexname,
    indexdef,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_indexes
WHERE tablename IN ('orders', 'customers')
  AND indexname IN ('idx_orders_pending_date_cover', 'idx_customers_id_cover')
ORDER BY tablename;
```

---

### Paso 5: Verificar mejoras con métricas *before/after*

**Objetivo:** Cuantificar el impacto de las optimizaciones comparando métricas de `pg_stat_statements` y Performance Insights antes y después de los cambios.

#### Instrucciones

**5.1. Resetear `pg_stat_statements` y re-ejecutar la carga de trabajo:**

```bash
# Resetear estadísticas para medir solo el período post-optimización
psql -h $DB_WRITER_ENDPOINT -p $DB_PORT -U $DB_USER -d $DB_NAME \
  -c "SELECT pg_stat_statements_reset();"

# Re-ejecutar el mismo script de carga
python3 ~/aurora-labs/04-00-02/scripts/generate_load_with_issues.py \
  --host $DB_WRITER_ENDPOINT \
  --port $DB_PORT \
  --dbname $DB_NAME \
  --user $DB_USER \
  --duration 300 \
  --concurrency 20
```

**5.2. Capturar métricas post-optimización:**

```sql
-- Capturar snapshot post-optimización
CREATE TABLE IF NOT EXISTS lab_after_stats AS
SELECT
    queryid,
    LEFT(query, 120)                         AS query_preview,
    calls,
    total_exec_time,
    mean_exec_time,
    stddev_exec_time,
    rows,
    shared_blks_hit,
    shared_blks_read,
    temp_blks_read,
    temp_blks_written,
    blk_read_time,
    blk_write_time,
    NOW()                                    AS captured_at
FROM pg_stat_statements
WHERE calls > 5
ORDER BY total_exec_time DESC
LIMIT 20;
```

**5.3. Comparar métricas before/after:**

```sql
-- Comparación cuantitativa before vs after
SELECT
    b.query_preview,
    b.calls                                                    AS calls_before,
    a.calls                                                    AS calls_after,
    ROUND(b.mean_exec_time::numeric, 2)                        AS mean_ms_before,
    ROUND(a.mean_exec_time::numeric, 2)                        AS mean_ms_after,
    ROUND(((b.mean_exec_time - a.mean_exec_time) /
           NULLIF(b.mean_exec_time, 0) * 100)::numeric, 1)    AS improvement_pct,
    b.temp_blks_written                                        AS disk_spill_before,
    COALESCE(a.temp_blks_written, 0)                           AS disk_spill_after,
    ROUND(b.shared_blks_read::numeric, 0)                      AS blks_read_before,
    ROUND(COALESCE(a.shared_blks_read, 0)::numeric, 0)         AS blks_read_after
FROM lab_baseline_stats b
LEFT JOIN lab_after_stats a
  ON b.query_preview = a.query_preview
WHERE b.mean_exec_time > 100
ORDER BY improvement_pct DESC NULLS LAST;
```

**5.4. Verificar uso de los nuevos índices:**

```sql
-- Confirmar que los nuevos índices están siendo utilizados
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan                                                AS scans_used,
    idx_tup_read                                            AS tuples_read,
    idx_tup_fetch                                           AS tuples_fetched,
    pg_size_pretty(pg_relation_size(indexrelid))            AS index_size
FROM pg_stat_user_indexes
WHERE indexname IN ('idx_orders_pending_date_cover', 'idx_customers_id_cover')
ORDER BY idx_scan DESC;
```

**5.5. Comparar AAS en Performance Insights post-optimización:**

```bash
# Obtener AAS post-optimización
START_TIME_AFTER=$(date -u -d '8 minutes ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || \
                   date -u -v-8M +%Y-%m-%dT%H:%M:%SZ)
END_TIME_AFTER=$(date -u +%Y-%m-%dT%H:%M:%SZ)

aws pi get-resource-metrics \
  --service-type RDS \
  --identifier "$DB_INSTANCE_ARN" \
  --metric db.load.avg \
  --start-time "$START_TIME_AFTER" \
  --end-time "$END_TIME_AFTER" \
  --period-in-seconds 60 \
  --metric-queries '[
    {
      "Metric": "db.load.avg",
      "GroupBy": {
        "Group": "db.wait_event_type",
        "Dimensions": ["db.wait_event_type"],
        "Limit": 5
      }
    }
  ]' | jq '.MetricList[0].DataPoints | map({t: .Timestamp, v: (.Value | . * 100 | round / 100)})' \
  | head -20
```

#### Salida esperada

Ejemplo de comparación before/after:

```
query_preview                              | mean_ms_before | mean_ms_after | improvement_pct | disk_spill_before | disk_spill_after
-------------------------------------------+----------------+---------------+-----------------+-------------------+------------------
SELECT o.order_id, o.order_date...         |         521.11 |         18.34 |            96.5 |                 0 |                0
SELECT p.product_id...ORDER BY total_rev.. |         480.00 |         95.20 |            80.2 |              8450 |                0
SELECT id, name FROM products ORDER BY..   |         320.50 |         45.10 |            85.9 |              2100 |                0
SELECT * FROM customers WHERE customer_..  |          10.00 |          0.85 |            91.5 |                 0 |                0
```

#### Verificación

```sql
-- Verificación final: ninguna consulta principal debe tener disk spill
SELECT
    LEFT(query, 80) AS query,
    calls,
    ROUND(mean_exec_time::numeric, 2) AS mean_ms,
    temp_blks_written AS disk_spill
FROM pg_stat_statements
WHERE temp_blks_written > 0
  AND calls > 5
  AND query NOT LIKE '%pg_stat%'
ORDER BY temp_blks_written DESC;
-- Resultado esperado: 0 filas (sin disk spills)
```

---

### Paso 6: Generar informe de optimización estructurado

**Objetivo:** Documentar el proceso completo de optimización con métricas cuantitativas en un informe estructurado que pueda ser referenciado en entornos productivos.

#### Instrucciones

**6.1. Generar el informe de optimización desde SQL:**

```sql
-- Informe de optimización consolidado
\o ~/aurora-labs/04-00-02/results/optimization_report.txt

SELECT '=== INFORME DE OPTIMIZACIÓN - Lab 04-00-02 ===' AS seccion;
SELECT 'Generado: ' || NOW()::text AS timestamp_reporte;

SELECT '--- RESUMEN DE MEJORAS ---' AS seccion;
SELECT
    b.query_preview                                             AS consulta,
    ROUND(b.mean_exec_time::numeric, 2)                         AS latencia_antes_ms,
    ROUND(COALESCE(a.mean_exec_time, b.mean_exec_time)::numeric, 2)
                                                                AS latencia_despues_ms,
    ROUND(((b.mean_exec_time - COALESCE(a.mean_exec_time, b.mean_exec_time)) /
           NULLIF(b.mean_exec_time, 0) * 100)::numeric, 1)     AS mejora_pct,
    CASE
        WHEN b.temp_blks_written > 0 AND COALESCE(a.temp_blks_written, 0) = 0
             THEN 'Disk spill eliminado'
        WHEN (b.shared_blks_read * 100.0 /
              NULLIF(b.shared_blks_hit + b.shared_blks_read, 0)) > 80
             THEN 'Índice creado - I/O reducido'
        ELSE 'Optimización aplicada'
    END                                                         AS solucion_aplicada
FROM lab_baseline_stats b
LEFT JOIN lab_after_stats a ON b.query_preview = a.query_preview
WHERE b.mean_exec_time > 50
ORDER BY mejora_pct DESC NULLS LAST;

SELECT '--- ÍNDICES CREADOS ---' AS seccion;
SELECT indexname, indexdef, pg_size_pretty(pg_relation_size(indexrelid)) AS tamano
FROM pg_indexes
WHERE indexname IN ('idx_orders_pending_date_cover', 'idx_customers_id_cover')
ORDER BY indexname;

SELECT '--- USO DE ÍNDICES NUEVOS ---' AS seccion;
SELECT indexname, idx_scan AS veces_usado, idx_tup_read AS tuplas_leidas
FROM pg_stat_user_indexes
WHERE indexname IN ('idx_orders_pending_date_cover', 'idx_customers_id_cover');

\o
\echo 'Informe guardado en results/optimization_report.txt'
```

**6.2. Verificar el informe generado:**

```bash
cat ~/aurora-labs/04-00-02/results/optimization_report.txt
```

**6.3. Exportar métricas PI para el informe (opcional, para análisis histórico):**

```bash
# Exportar AAS antes y después a CSV para visualización
python3 ~/aurora-labs/04-00-02/scripts/export_pi_metrics.py \
  --instance-arn "$DB_INSTANCE_ARN" \
  --region "$AWS_REGION" \
  --output ~/aurora-labs/04-00-02/results/pi_metrics_comparison.csv

echo "Métricas PI exportadas a results/pi_metrics_comparison.csv"
```

#### Salida esperada

```
=== INFORME DE OPTIMIZACIÓN - Lab 04-00-02 ===
Generado: 2026-06-09 14:32:15.123456+00

--- RESUMEN DE MEJORAS ---
consulta                    | latencia_antes_ms | latencia_despues_ms | mejora_pct | solucion_aplicada
----------------------------+-------------------+---------------------+------------+----------------------------
SELECT o.order_id...        |            521.11 |               18.34 |       96.5 | Índice creado - I/O reducido
SELECT p.product_id...      |            480.00 |               95.20 |       80.2 | Disk spill eliminado
SELECT * FROM customers...  |             10.00 |                0.85 |       91.5 | Índice creado - I/O reducido

--- ÍNDICES CREADOS ---
idx_orders_pending_date_cover | CREATE INDEX ... WHERE (status = 'pending') | 2048 kB
idx_customers_id_cover        | CREATE INDEX ... INCLUDE (customer_name...) | 1536 kB

Informe guardado en results/optimization_report.txt
```

#### Verificación

```bash
# Verificar que el informe existe y tiene contenido
wc -l ~/aurora-labs/04-00-02/results/optimization_report.txt
# Debe tener al menos 30 líneas

ls -lh ~/aurora-labs/04-00-02/results/
```

---

## Validación y Pruebas

Ejecuta las siguientes verificaciones para confirmar que el laboratorio fue completado exitosamente:

```sql
-- VERIFICACIÓN 1: Todos los índices creados están activos y siendo usados
SELECT
    indexname,
    idx_scan > 0 AS fue_usado,
    pg_size_pretty(pg_relation_size(indexrelid)) AS tamano
FROM pg_stat_user_indexes
WHERE indexname IN ('idx_orders_pending_date_cover', 'idx_customers_id_cover');
-- Esperado: fue_usado = true para ambos índices

-- VERIFICACIÓN 2: No hay disk spills en las consultas optimizadas
SELECT COUNT(*) AS queries_con_disk_spill
FROM pg_stat_statements
WHERE temp_blks_written > 100
  AND calls > 5
  AND query NOT LIKE '%pg_stat%';
-- Esperado: 0

-- VERIFICACIÓN 3: Mejora de latencia > 70% en las consultas principales
SELECT
    COUNT(*) AS queries_mejoradas
FROM lab_baseline_stats b
JOIN lab_after_stats a ON b.query_preview = a.query_preview
WHERE ((b.mean_exec_time - a.mean_exec_time) / NULLIF(b.mean_exec_time, 0)) > 0.70;
-- Esperado: al menos 3 queries con mejora > 70%

-- VERIFICACIÓN 4: Estadísticas actualizadas en tabla orders
SELECT
    tablename,
    last_analyze > NOW() - INTERVAL '1 hour' AS analyze_reciente,
    n_mod_since_analyze < 100 AS estadisticas_frescas
FROM pg_stat_user_tables
WHERE tablename = 'orders';
-- Esperado: analyze_reciente = true
```

```bash
# VERIFICACIÓN 5: Informe de optimización generado
test -f ~/aurora-labs/04-00-02/results/optimization_report.txt && \
  echo "✅ Informe generado correctamente" || \
  echo "❌ Informe NO encontrado"

# VERIFICACIÓN 6: Performance Insights muestra reducción de AAS
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier "$DB_INSTANCE_ARN" \
  --metric db.load.avg \
  --start-time "$(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || date -u -v-5M +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --period-in-seconds 60 \
  --metric-queries '[{"Metric":"db.load.avg"}]' \
  | jq '.MetricList[0].DataPoints | map(.Value) | add / length' 2>/dev/null
# Esperado: valor AAS < 2.0 (reducción respecto al baseline)
```

---

## Solución de Problemas

### Problema 1: El índice parcial no es utilizado por el planificador

**Síntoma:** Después de crear `idx_orders_pending_date_cover`, `EXPLAIN ANALYZE` sigue mostrando `Seq Scan on orders` en lugar de `Index Scan` o `Index Only Scan`. El tiempo de ejecución no mejora.

**Causa raíz:** El planificador de PostgreSQL puede ignorar un índice si:
- Las estadísticas de la tabla están obsoletas y subestima la selectividad del filtro `status = 'pending'`.
- El parámetro `enable_indexscan` o `enable_indexonlyscan` está desactivado en la sesión.
- El índice aún no fue analizado por `autovacuum` y no tiene estadísticas propias.
- La tabla cabe completamente en `shared_buffers` y el planificador prefiere el Seq Scan en memoria.

**Solución:**

```sql
-- Paso 1: Actualizar estadísticas de la tabla Y del índice
ANALYZE VERBOSE orders;

-- Paso 2: Verificar que el planificador tiene acceso al índice
SET enable_seqscan = off;   -- Solo para diagnóstico, NO en producción
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.order_id, o.order_date, o.total_amount
FROM orders o
WHERE o.status = 'pending'
  AND o.order_date >= NOW() - INTERVAL '30 days';

-- Si con enable_seqscan=off usa el índice y es más rápido,
-- el problema es de estadísticas. Aumentar el target:
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 1000;
ANALYZE orders;

-- Paso 3: Restaurar configuración normal
RESET enable_seqscan;

-- Verificar selectividad estimada
SELECT
    attname,
    n_distinct,
    most_common_vals,
    most_common_freqs
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
-- Si n_distinct es -1 (fracción) y la frecuencia de 'pending' es baja,
-- el índice debería ser preferido automáticamente tras el ANALYZE.
```

---

### Problema 2: `pg_stat_statements` no muestra datos o retorna vacío

**Síntoma:** Las consultas a `pg_stat_statements` retornan 0 filas o un error `ERROR: relation "pg_stat_statements" does not exist`. No se pueden obtener métricas baseline ni comparar before/after.

**Causa raíz:** La extensión `pg_stat_statements` no está cargada en `shared_preload_libraries` del parameter group de Aurora, o fue cargada pero no creada como extensión en la base de datos `labdb`. También puede ocurrir que el parameter group modificado no fue asociado al clúster o que el cambio requiere reinicio y este no se ha realizado.

**Solución:**

```bash
# Paso 1: Verificar el parameter group asociado al clúster
aws rds describe-db-clusters \
  --db-cluster-identifier $CLUSTER_ID \
  --query 'DBClusters[0].DBClusterParameterGroup' \
  --output text

# Paso 2: Verificar el valor de shared_preload_libraries en el parameter group
PARAM_GROUP=$(aws rds describe-db-clusters \
  --db-cluster-identifier $CLUSTER_ID \
  --query 'DBClusters[0].DBClusterParameterGroup' \
  --output text)

aws rds describe-db-cluster-parameters \
  --db-cluster-parameter-group-name $PARAM_GROUP \
  --query "Parameters[?ParameterName=='shared_preload_libraries']" \
  --output table
```

```sql
-- Paso 3: Verificar desde PostgreSQL si la librería está cargada
SHOW shared_preload_libraries;
-- Debe incluir 'pg_stat_statements'

-- Paso 4: Crear la extensión si no existe (requiere superuser o rds_superuser)
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Paso 5: Verificar que la extensión existe
SELECT * FROM pg_extension WHERE extname = 'pg_stat_statements';
```

```bash
# Paso 6: Si shared_preload_libraries no incluye pg_stat_statements,
# modificar el parameter group (requiere reinicio de instancia)
aws rds modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name $PARAM_GROUP \
  --parameters "ParameterName=shared_preload_libraries,ParameterValue=pg_stat_statements,ApplyMethod=pending-reboot"

# Reiniciar la instancia writer (planificar ventana de mantenimiento)
aws rds reboot-db-instance \
  --db-instance-identifier aurora-lab-writer-1

echo "Esperando reinicio..."
aws rds wait db-instance-available \
  --db-instance-identifier aurora-lab-writer-1
echo "Instancia disponible. Crear la extensión y reiniciar el laboratorio."
```

---

## Limpieza de Recursos

> **⚠️ Importante:** Ejecuta la limpieza al finalizar el laboratorio para evitar costos innecesarios. Los índices creados pueden mantenerse si continuarás con el Lab 04-00-03, ya que construye sobre este entorno.

**Limpieza de objetos temporales del laboratorio:**

```sql
-- Eliminar tablas temporales de estadísticas del laboratorio
DROP TABLE IF EXISTS lab_baseline_stats;
DROP TABLE IF EXISTS lab_after_stats;

-- Opcional: Eliminar índices si NO continuarás con el siguiente lab
-- DROP INDEX CONCURRENTLY IF EXISTS idx_orders_pending_date_cover;
-- DROP INDEX CONCURRENTLY IF EXISTS idx_customers_id_cover;

-- Resetear parámetros de sesión
RESET work_mem;
RESET track_io_timing;
RESET auto_explain.log_min_duration;

-- Confirmar limpieza
SELECT tablename FROM pg_tables
WHERE tablename IN ('lab_baseline_stats', 'lab_after_stats')
  AND schemaname = 'public';
-- Debe retornar 0 filas
```

**Limpieza de archivos locales (opcional):**

```bash
# Comprimir resultados para referencia futura
cd ~/aurora-labs/04-00-02
tar -czf results_lab_04-00-02_$(date +%Y%m%d).tar.gz results/
echo "Resultados comprimidos: results_lab_04-00-02_$(date +%Y%m%d).tar.gz"

# Si NO continuarás con laboratorios adicionales hoy, destruir infraestructura
# cd ~/aurora-labs/terraform/module-04
# terraform destroy -auto-approve
# echo "Infraestructura destruida. Costos detenidos."
```

**Verificar que no quedan recursos activos innecesarios:**

```bash
# Verificar estado del clúster
aws rds describe-db-clusters \
  --db-cluster-identifier $CLUSTER_ID \
  --query 'DBClusters[0].{Status:Status, Engine:Engine, InstanceCount:length(DBClusterMembers)}' \
  --output table
```

---

## Resumen

### Lo que aprendiste en este laboratorio

En este laboratorio aplicaste el ciclo completo de diagnóstico y optimización de rendimiento en Aurora PostgreSQL:

| Fase                  | Herramienta utilizada                          | Resultado obtenido                                    |
|-----------------------|------------------------------------------------|-------------------------------------------------------|
| **Identificación**    | `pg_stat_statements`, Performance Insights AAS | Clasificación de 4 tipos de problemas de rendimiento  |
| **Análisis**          | `EXPLAIN ANALYZE BUFFERS`, `pg_stats`          | Confirmación de causa raíz (Seq Scan, disk spill)     |
| **Implementación**    | Índices parciales, índices de cobertura, ANALYZE | Reducción de I/O y eliminación de disk spills        |
| **Verificación**      | Comparación before/after en `pg_stat_statements` | Mejoras cuantificadas: 80–96% reducción de latencia  |
| **Documentación**     | Informe estructurado SQL + PI API export       | Registro auditables de las optimizaciones aplicadas   |

### Conceptos clave consolidados

- **Índice parcial** (`WHERE clause`): ideal para tablas con columnas de baja cardinalidad y acceso frecuente a un subconjunto específico de valores.
- **Índice de cobertura** (`INCLUDE`): elimina heap fetches al incluir las columnas proyectadas en el índice, habilitando `Index Only Scan`.
- **`work_mem` por sesión**: ajuste quirúrgico para operaciones específicas sin impactar el presupuesto de memoria global del servidor.
- **`statistics_target` aumentado**: mejora la precisión del planificador en columnas con distribución no uniforme o alta cardinalidad.
- **Ciclo PI**: detectar categoría dominante (IO/CPU/Lock) → identificar SQL → aplicar corrección → validar reducción de AAS.

### Recursos adicionales

- [Documentación oficial: CREATE INDEX con INCLUDE](https://www.postgresql.org/docs/current/sql-createindex.html)
- [Documentación oficial: pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)
- [AWS Blog: Optimizing Aurora PostgreSQL with Performance Insights](https://aws.amazon.com/blogs/database/optimizing-amazon-aurora-postgresql-with-performance-insights/)
- [Documentación Aurora: Mejores prácticas de rendimiento](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.BestPractices.html)
- [Guía de EXPLAIN de PostgreSQL (depesz)](https://explain.depesz.com/)
- [Referencia de wait events en PostgreSQL](https://www.postgresql.org/docs/current/monitoring-stats.html#WAIT-EVENT-TABLE)

---
*Lab 04-00-02 — Módulo 4: Observabilidad y Optimización en Aurora PostgreSQL*

---

# Escenarios Avanzados Aurora — Optimización Integral, Multi-Región y Observabilidad

## Metadatos

| Campo            | Detalle                                      |
|------------------|----------------------------------------------|
| **Duración**     | 45 minutos                                   |
| **Complejidad**  | Alta                                         |
| **Nivel Bloom**  | Crear (Síntesis y diseño de soluciones)      |
| **Módulo**       | 4 — Observabilidad Avanzada y DR             |
| **Laboratorio**  | 04-00-03 (Práctica 9 — Laboratorio de Cierre)|

---

## Descripción General

Este laboratorio integrador cierra el curso asumiendo el rol de **arquitecto de base de datos** para una aplicación e-commerce de alta concurrencia. En tres fases encadenadas, aplicarás simultáneamente todas las técnicas del curso — afinación de parámetros, indexación, RDS Proxy, autovacuum y pg_stat_statements — medirás su impacto acumulado con pgbench, configurarás Aurora Global Database para un plan de Disaster Recovery (DR) multi-región con RTO/RPO definidos, y construirás un dashboard de CloudWatch que consolide Performance Insights, Enhanced Monitoring y métricas personalizadas. El entregable final es un documento técnico de arquitectura que demuestra capacidad de síntesis creativa.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio serás capaz de:

- [ ] **Diseñar e implementar** una estrategia integral de optimización OLTP aplicando simultáneamente parameter tuning, indexación, RDS Proxy y autovacuum, y midiendo el impacto acumulado con pgbench.
- [ ] **Crear y validar** un plan de Disaster Recovery multi-región usando Aurora Global Database con Managed Planned Failover, midiendo RPO y RTO reales.
- [ ] **Construir** un dashboard de CloudWatch personalizado que integre métricas de Performance Insights API, Enhanced Monitoring y pg_stat_statements para observabilidad continua.
- [ ] **Evaluar y documentar** el impacto acumulado de todas las optimizaciones mediante pruebas de carga comparativas antes/después.
- [ ] **Proponer y justificar** técnicamente una arquitectura Aurora PostgreSQL optimizada para el escenario empresarial complejo descrito.

---

## Prerrequisitos

### Conocimiento previo
- Haber completado los laboratorios 01-00-01 al 04-00-02 del curso.
- Comprensión de: parameter groups de Aurora, índices B-tree/parciales, RDS Proxy, autovacuum tuning, pg_stat_statements, Performance Insights y Enhanced Monitoring.
- Familiaridad con pgbench (scripts personalizados del repositorio del curso) y AWS CLI v2.

### Acceso y recursos AWS
- Usuario IAM con políticas para: RDS, Aurora, CloudWatch, Secrets Manager, VPC, PI (Performance Insights).
- Cuotas de servicio habilitadas en **dos regiones AWS** (p. ej., `us-east-1` como primaria y `us-west-2` como secundaria).
- Aurora Global Database disponible o permisos para crearla.
- Instancias mínimo `db.r6g.large` (Performance Insights no disponible en `db.t3.micro`).
- Terraform ≥ 1.5.x con el módulo del curso clonado y aplicado para el Módulo 4.

---

## Entorno de Laboratorio

### Hardware recomendado

| Recurso          | Mínimo                          |
|------------------|---------------------------------|
| RAM              | 8 GB                            |
| CPU              | 4 núcleos                       |
| Almacenamiento   | 10 GB libres (logs y resultados)|
| Conexión         | 10 Mbps estable                 |

### Software requerido

| Herramienta            | Versión        |
|------------------------|----------------|
| AWS CLI                | 2.x            |
| psql / pgbench         | 14.x o superior|
| Python                 | 3.9 o superior |
| jq                     | 1.6 o superior |
| Terraform              | 1.5.x          |
| DBeaver Community      | 23.x           |
| Navegador Web          | Chrome/Firefox (última versión)|

### Variables de entorno — configurar antes de iniciar

Ejecuta los siguientes comandos en tu terminal para definir las variables que usarán todos los pasos del laboratorio:

```bash
# === CONFIGURACIÓN GLOBAL ===
export AWS_PRIMARY_REGION="us-east-1"
export AWS_SECONDARY_REGION="us-west-2"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# === CLUSTER AURORA PRIMARY ===
export AURORA_CLUSTER_ID="aurora-lab-cluster"
export AURORA_WRITER_ID="aurora-lab-cluster-instance-1"
export AURORA_WRITER_ENDPOINT=$(aws rds describe-db-clusters \
  --db-cluster-identifier $AURORA_CLUSTER_ID \
  --region $AWS_PRIMARY_REGION \
  --query 'DBClusters[0].Endpoint' --output text)
export AURORA_READER_ENDPOINT=$(aws rds describe-db-clusters \
  --db-cluster-identifier $AURORA_CLUSTER_ID \
  --region $AWS_PRIMARY_REGION \
  --query 'DBClusters[0].ReaderEndpoint' --output text)

# === BASE DE DATOS ===
export DB_NAME="ecommerce_lab"
export DB_USER="labadmin"
export DB_PASSWORD="LabPassword2024!"   # Usar Secrets Manager en producción
export PGPASSWORD=$DB_PASSWORD

# === RDS PROXY ===
export PROXY_ENDPOINT=$(aws rds describe-db-proxies \
  --region $AWS_PRIMARY_REGION \
  --query 'DBProxies[?DBProxyName==`aurora-lab-proxy`].Endpoint' \
  --output text)

# === PERFORMANCE INSIGHTS ===
export PI_RESOURCE_ID=$(aws rds describe-db-instances \
  --db-instance-identifier $AURORA_WRITER_ID \
  --region $AWS_PRIMARY_REGION \
  --query 'DBInstances[0].DbiResourceId' --output text)
export PI_ARN="arn:aws:rds:${AWS_PRIMARY_REGION}:${AWS_ACCOUNT_ID}:db:${AURORA_WRITER_ID}"

echo "=== Variables configuradas ==="
echo "Writer:  $AURORA_WRITER_ENDPOINT"
echo "Proxy:   $PROXY_ENDPOINT"
echo "PI ID:   $PI_RESOURCE_ID"
```

### Verificación del entorno base

```bash
# Verificar conectividad al writer (directo)
psql -h $AURORA_WRITER_ENDPOINT -U $DB_USER -d $DB_NAME -c "SELECT version();"

# Verificar conectividad vía RDS Proxy
psql -h $PROXY_ENDPOINT -U $DB_USER -d $DB_NAME -c "SELECT current_database(), now();"

# Verificar que pg_stat_statements esté activo
psql -h $AURORA_WRITER_ENDPOINT -U $DB_USER -d $DB_NAME \
  -c "SELECT name, setting FROM pg_settings WHERE name LIKE 'pg_stat_statements%';"
```

---

## Desarrollo del Laboratorio

---

### FASE 1: Optimización Integral con Medición de Impacto Acumulado

---

#### Paso 1.1 — Establecer Línea Base de Rendimiento (Benchmark Inicial)

**Objetivo:** Capturar métricas de rendimiento antes de aplicar optimizaciones, para comparación posterior.

**Instrucciones:**

1. Crea el directorio de resultados del laboratorio:

```bash
mkdir -p ~/lab-04-00-03/{baseline,optimized,dr,dashboard,architecture}
cd ~/lab-04-00-03
```

2. Ejecuta el benchmark de línea base con pgbench usando el script OLTP personalizado del repositorio:

```bash
# Inicializar pgbench con factor de escala 50 (≈ 5M filas en pgbench_accounts)
pgbench -h $AURORA_WRITER_ENDPOINT -U $DB_USER -d $DB_NAME \
  -i -s 50 --foreign-keys 2>&1 | tee baseline/pgbench_init.log

# Benchmark de lectura/escritura mixto — línea base (sin optimizaciones)
# -c 20: 20 clientes concurrentes | -j 4: 4 hilos | -T 120: 120 segundos
pgbench -h $AURORA_WRITER_ENDPOINT -U $DB_USER -d $DB_NAME \
  -c 20 -j 4 -T 120 -P 10 \
  --report-per-command \
  2>&1 | tee baseline/pgbench_baseline_results.txt

echo "=== Línea base capturada en baseline/pgbench_baseline_results.txt ==="
```

3. Captura el estado inicial de pg_stat_statements:

```bash
psql -h $AURORA_WRITER_ENDPOINT -U $DB_USER -d $DB_NAME << 'EOF' | tee baseline/pg_stat_statements_baseline.txt
SELECT
  substring(query, 1, 80)  AS query_short,
  calls,
  round(total_exec_time::numeric, 2)  AS total_ms,
  round(mean_exec_time::numeric, 4)   AS mean_ms,
  round(stddev_exec_time::numeric, 4) AS stddev_ms,
  rows,
  shared_blks_hit,
  shared_blks_read,
  round(100.0 * shared_blks_hit /
    NULLIF(shared_blks_hit + shared_blks_read, 0), 2) AS cache_hit_pct
FROM pg_stat_statements
WHERE calls > 10
ORDER BY total_exec_time DESC
LIMIT 20;
EOF
```

4. Consulta la PI API para capturar AAS de línea base:

```bash
START_TIME=$(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || \
             date -u -v-5M +%Y-%m-%dT%H:%M:%SZ)
END_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)

aws pi get-resource-metrics \
  --service-type RDS \
  --identifier $PI_ARN \
  --metric db.load.avg \
  --start-time $START_TIME \
  --end-time $END_TIME \
  --period-in-seconds 60 \
  --metric-queries '[
    {
      "Metric": "db.load.avg",
      "GroupBy": {
        "Group": "db.wait_event_type",
        "Dimensions": ["db.wait_event_type"],
        "Limit": 5
      }
    }
  ]' \
  --region $AWS_PRIMARY_REGION \
  --output json | jq '.' > baseline/pi_aas_baseline.json

echo "AAS línea base guardado en baseline/pi_aas_baseline.json"
cat baseline/pi_aas_baseline.json | jq '.MetricList[].DataPoints[] | {timestamp: .Timestamp, value: .Value}'
```

**Salida esperada:**
```
pgbench (14.x)
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 50
...
tps = 245.32 (including connections establishment)
tps = 248.11 (excluding connections establishment)
```

**Verificación:**
```bash
# Confirmar que los archivos de línea base existen y tienen contenido
ls -lh baseline/
grep "tps =" baseline/pgbench_baseline_results.txt
```

---

#### Paso 1.2 — Aplicar Stack de Optimización Integral

**Objetivo:** Aplicar simultáneamente todas las optimizaciones del curso y verificar cada capa.

**Instrucciones:**

1. **Parameter Group — Aplicar configuración final optimizada:**

```bash
# Obtener el nombre del cluster parameter group
PARAM_GROUP=$(aws rds describe-db-clusters \
  --db-cluster-identifier $AURORA_CLUSTER_ID \
  --region $AWS_PRIMARY_REGION \
  --query 'DBClusters[0].DBClusterParameterGroup' --output text)

echo "Parameter Group activo: $PARAM_GROUP"

# Aplicar parámetros críticos de rendimiento OLTP
aws rds modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name $PARAM_GROUP \
  --region $AWS_PRIMARY_REGION \
  --parameters \
    "ParameterName=shared_buffers,ParameterValue=4096,ApplyMethod=pending-reboot" \
    "ParameterName=work_mem,ParameterValue=65536,ApplyMethod=immediate" \
    "ParameterName=maintenance_work_mem,ParameterValue=524288,ApplyMethod=immediate" \
    "ParameterName=effective_cache_size,ParameterValue=12288,ApplyMethod=immediate" \
    "ParameterName=checkpoint_completion_target,ParameterValue=0.9,ApplyMethod=immediate" \
    "ParameterName=wal_buffers,ParameterValue=2048,ApplyMethod=pending-reboot" \
    "ParameterName=random_page_cost,ParameterValue=1.1,ApplyMethod=immediate" \
    "ParameterName=effective_io_concurrency,ParameterValue=200,ApplyMethod=immediate" \
    "ParameterName=autovacuum_vacuum_scale_factor,ParameterValue=0.01,ApplyMethod=immediate" \
    "ParameterName=autovacuum_analyze_scale_factor,ParameterValue=0.005,ApplyMethod=immediate" \
    "ParameterName=autovacuum_vacuum_cost_delay,ParameterValue=2,ApplyMethod=immediate" \
    "ParameterName=log_min_duration_statement,ParameterValue=1000,ApplyMethod=immediate"

echo "Parámetros aplicados. Verificando estado..."
aws rds describe-db-cluster-parameters \
  --db-cluster-parameter-group-name $PARAM_GROUP \
  --region $AWS_PRIMARY_REGION \
  --query 'Parameters[?ParameterName==`work_mem` || ParameterName==`random_page_cost`].[ParameterName,ParameterValue,ApplyType]' \
  --output table
```

2. **Indexación estratégica — crear índices críticos para carga OLTP:**

```bash
psql -h $AURORA_WRITER_ENDPOINT -U $DB_USER -d $DB_NAME << 'EOF'
-- Índices para las tablas de pgbench (simula e-commerce)
-- Índice parcial para cuentas activas con balance alto (hot path)
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_accounts_active_balance
  ON pgbench_accounts (aid, abalance)
  WHERE abalance > 0;

-- Índice compuesto para historial por cuenta y tiempo
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_history_aid_mtime
  ON pgbench_history (aid, mtime DESC);

-- Índice en branches para joins frecuentes
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_branches_bid_bbalance
  ON pgbench_branches (bid, bbalance);

-- Verificar índices creados
SELECT
  schemaname,
  tablename,
  indexname,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE indexname LIKE 'idx_%'
ORDER BY pg_relation_size(indexrelid) DESC;
EOF
```

3. **Verificar configuración de RDS Proxy para producción:**

```bash
# Verificar estado del proxy y su target group
aws rds describe-db-proxies \
  --region $AWS_PRIMARY_REGION \
  --query 'DBProxies[?DBProxyName==`aurora-lab-proxy`].{Name:DBProxyName,Status:Status,Endpoint:Endpoint}' \
  --output table

# Verificar target group del proxy apunta al cluster correcto
aws rds describe-db-proxy-targets \
  --db-proxy-name aurora-lab-proxy \
  --region $AWS_PRIMARY_REGION \
  --query 'Targets[].{Type:Type,Endpoint:Endpoint,Port:Port,State:TargetHealth.State}' \
  --output table

# Verificar configuración de connection pooling
aws rds describe-db-proxy-target-groups \
  --db-proxy-name aurora-lab-proxy \
  --region $AWS_PRIMARY_REGION \
  --query 'TargetGroups[0].ConnectionPoolConfig' \
  --output json
```

4. **Forzar ANALYZE en tablas críticas post-indexación:**

```bash
psql -h $AURORA_WRITER_ENDPOINT -U $DB_USER -d $DB_NAME << 'EOF'
ANALYZE VERBOSE pgbench_accounts;
ANALYZE VERBOSE pgbench_history;
ANALYZE VERBOSE pgbench_branches;
ANALYZE VERBOSE pgbench_tellers;

-- Verificar estadísticas actualizadas
SELECT
  relname,
  n_live_tup,
  n_dead_tup,
  last_vacuum,
  last_autovacuum,
  last_analyze,
  last_autoanalyze
FROM pg_stat_user_tables
WHERE relname LIKE 'pgbench%'
ORDER BY n_live_tup DESC;
EOF
```

**Salida esperada:**
```
INFO:  analyzing "public.pgbench_accounts"
INFO:  "pgbench_accounts": scanned 30000 of 66667 pages, containing 4500000 live rows...
```

**Verificación:**
```bash
psql -h $AURORA_WRITER_ENDPOINT -U $DB_USER -d $DB_NAME \
  -c "SELECT count(*) FROM pg_indexes WHERE tablename LIKE 'pgbench%';"
```

---

#### Paso 1.3 — Benchmark Post-Optimización y Análisis de Impacto

**Objetivo:** Medir el impacto acumulado de todas las optimizaciones y analizar con Performance Insights.

**Instrucciones:**

1. Ejecuta el benchmark optimizado **vía RDS Proxy** (connection pooling activo):

```bash
# Benchmark optimizado — vía Proxy con mayor concurrencia
pgbench -h $PROXY_ENDPOINT -U $DB_USER -d $DB_NAME \
  -c 50 -j 8 -T 120 -P 10 \
  --report-per-command \
  2>&1 | tee optimized/pgbench_optimized_results.txt

echo "=== Benchmark optimizado completado ==="
```

2. Captura métricas post-optimización de pg_stat_statements:

```bash
psql -h $AURORA_WRITER_ENDPOINT -U $DB_USER -d $DB_NAME << 'EOF' | tee optimized/pg_stat_statements_optimized.txt
SELECT
  substring(query, 1, 80)  AS query_short,
  calls,
  round(total_exec_time::numeric, 2)  AS total_ms,
  round(mean_exec_time::numeric, 4)   AS mean_ms,
  round(100.0 * shared_blks_hit /
    NULLIF(shared_blks_hit + shared_blks_read, 0), 2) AS cache_hit_pct
FROM pg_stat_statements
WHERE calls > 10
ORDER BY total_exec_time DESC
LIMIT 20;
EOF
```

3. Genera el reporte comparativo de impacto:

```bash
cat << 'PYEOF' > ~/lab-04-00-03/compare_results.py
#!/usr/bin/env python3
"""Compara resultados de pgbench baseline vs optimizado."""
import re, sys

def extract_tps(filepath):
    try:
        with open(filepath) as f:
            content = f.read()
        # Busca la línea "tps = X.XX (excluding connections establishment)"
        match = re.search(r'tps = ([\d.]+) \(excluding connections', content)
        return float(match.group(1)) if match else None
    except FileNotFoundError:
        return None

baseline_tps  = extract_tps('baseline/pgbench_baseline_results.txt')
optimized_tps = extract_tps('optimized/pgbench_optimized_results.txt')

if baseline_tps and optimized_tps:
    improvement = ((optimized_tps - baseline_tps) / baseline_tps) * 100
    print(f"\n{'='*50}")
    print(f"  REPORTE DE IMPACTO ACUMULADO")
    print(f"{'='*50}")
    print(f"  TPS Línea Base:    {baseline_tps:>10.2f}")
    print(f"  TPS Optimizado:    {optimized_tps:>10.2f}")
    print(f"  Mejora:            {improvement:>+10.1f}%")
    print(f"{'='*50}")
    if improvement > 20:
        print("  ✓ Mejora significativa (>20%) — optimizaciones efectivas")
    elif improvement > 5:
        print("  ~ Mejora moderada (5-20%) — revisar cuellos de botella")
    else:
        print("  ✗ Mejora mínima — investigar con Performance Insights")
else:
    print("No se pudieron leer los archivos de resultados.")
    sys.exit(1)
PYEOF

python3 ~/lab-04-00-03/compare_results.py | tee optimized/impact_report.txt
```

4. Consulta Performance Insights para validar reducción de waits:

```bash
START_TIME=$(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || \
             date -u -v-10M +%Y-%m-%dT%H:%M:%SZ)
END_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)

aws pi get-resource-metrics \
  --service-type RDS \
  --identifier $PI_ARN \
  --metric db.load.avg \
  --start-time $START_TIME \
  --end-time $END_TIME \
  --period-in-seconds 60 \
  --metric-queries '[
    {
      "Metric": "db.load.avg",
      "GroupBy": {
        "Group": "db.wait_event",
        "Dimensions": ["db.wait_event"],
        "Limit": 10
      }
    }
  ]' \
  --region $AWS_PRIMARY_REGION \
  --output json | jq '
    .MetricList[].DataPoints |
    map(select(.Value > 0)) |
    group_by(.Timestamp) |
    map({
      timestamp: .[0].Timestamp,
      total_aas: (map(.Value) | add)
    })
  ' | tee optimized/pi_aas_optimized.json

echo "=== Datos PI post-optimización guardados ==="
```

**Salida esperada:**
```
==================================================
  REPORTE DE IMPACTO ACUMULADO
==================================================
  TPS Línea Base:        245.32
  TPS Optimizado:        387.45
  Mejora:                +57.9%
==================================================
  ✓ Mejora significativa (>20%) — optimizaciones efectivas
```

**Verificación:**
```bash
grep "tps =" optimized/pgbench_optimized_results.txt
cat optimized/impact_report.txt
```

---

### FASE 2: Estrategia Multi-Región y Disaster Recovery

---

#### Paso 2.1 — Configurar Aurora Global Database

**Objetivo:** Añadir una región secundaria al cluster Aurora existente para habilitar DR multi-región.

**Instrucciones:**

1. Verifica si ya existe una Aurora Global Database o créala:

```bash
# Verificar si existe Global Database
GLOBAL_DB=$(aws rds describe-global-clusters \
  --region $AWS_PRIMARY_REGION \
  --query 'GlobalClusters[?contains(GlobalClusterMembers[].DBClusterArn, `aurora-lab-cluster`)].GlobalClusterIdentifier' \
  --output text)

if [ -z "$GLOBAL_DB" ] || [ "$GLOBAL_DB" = "None" ]; then
  echo "Creando Aurora Global Database desde el cluster existente..."
  aws rds create-global-cluster \
    --global-cluster-identifier "aurora-lab-global" \
    --source-db-cluster-identifier \
      "arn:aws:rds:${AWS_PRIMARY_REGION}:${AWS_ACCOUNT_ID}:cluster:${AURORA_CLUSTER_ID}" \
    --region $AWS_PRIMARY_REGION
  export GLOBAL_DB="aurora-lab-global"
  echo "Global Database creada: $GLOBAL_DB"
else
  echo "Global Database existente: $GLOBAL_DB"
fi

# Esperar a que esté disponible
echo "Esperando disponibilidad de Global Database..."
aws rds wait db-cluster-available \
  --db-cluster-identifier $AURORA_CLUSTER_ID \
  --region $AWS_PRIMARY_REGION
echo "Cluster primario disponible."
```

2. Añade el cluster secundario en la región de DR:

```bash
# Obtener VPC y subnets en la región secundaria (previamente aprovisionadas con Terraform)
SECONDARY_VPC=$(aws ec2 describe-vpcs \
  --region $AWS_SECONDARY_REGION \
  --filters "Name=tag:Name,Values=aurora-lab-vpc" \
  --query 'Vpcs[0].VpcId' --output text)

SECONDARY_SUBNET_GROUP=$(aws rds describe-db-subnet-groups \
  --region $AWS_SECONDARY_REGION \
  --query 'DBSubnetGroups[?DBSubnetGroupName==`aurora-lab-subnet-group`].DBSubnetGroupName' \
  --output text)

SECONDARY_SG=$(aws ec2 describe-security-groups \
  --region $AWS_SECONDARY_REGION \
  --filters "Name=tag:Name,Values=aurora-lab-sg" \
  --query 'SecurityGroups[0].GroupId' --output text)

echo "VPC secundaria: $SECONDARY_VPC"
echo "Subnet group: $SECONDARY_SUBNET_GROUP"
echo "Security group: $SECONDARY_SG"

# Añadir región secundaria a la Global Database
aws rds create-db-cluster \
  --db-cluster-identifier "aurora-lab-cluster-secondary" \
  --global-cluster-identifier $GLOBAL_DB \
  --engine aurora-postgresql \
  --engine-version "14.9" \
  --db-cluster-parameter-group-name "default.aurora-postgresql14" \
  --db-subnet-group-name $SECONDARY_SUBNET_GROUP \
  --vpc-security-group-ids $SECONDARY_SG \
  --region $AWS_SECONDARY_REGION

# Crear instancia en el cluster secundario
aws rds create-db-instance \
  --db-instance-identifier "aurora-lab-secondary-instance-1" \
  --db-cluster-identifier "aurora-lab-cluster-secondary" \
  --db-instance-class "db.r6g.large" \
  --engine aurora-postgresql \
  --region $AWS_SECONDARY_REGION

echo "Cluster secundario en $AWS_SECONDARY_REGION creado. Esperando replicación inicial..."
echo "(Este proceso puede tardar 10-15 minutos)"
```

3. Monitorea el estado de la Global Database:

```bash
# Verificar estado de ambas regiones
watch -n 30 "aws rds describe-global-clusters \
  --global-cluster-identifier aurora-lab-global \
  --region $AWS_PRIMARY_REGION \
  --query 'GlobalClusters[0].GlobalClusterMembers[].{ARN:DBClusterArn,Writer:IsWriter,Status:Readers}' \
  --output table"
# Presiona Ctrl+C cuando ambas regiones muestren estado "available"
```

**Salida esperada:**
```
---------------------------------------------------------------------------
|                        DescribeGlobalClusters                           |
+------------------------------------------------------------------+-------+
|  ARN                                                             |Writer |
+------------------------------------------------------------------+-------+
|  arn:aws:rds:us-east-1:123456789012:cluster:aurora-lab-cluster  |True   |
|  arn:aws:rds:us-west-2:123456789012:cluster:aurora-lab-cluster-secondary|False|
+------------------------------------------------------------------+-------+
```

**Verificación:**
```bash
aws rds describe-global-clusters \
  --global-cluster-identifier aurora-lab-global \
  --region $AWS_PRIMARY_REGION \
  --output json | jq '.GlobalClusters[0] | {id: .GlobalClusterIdentifier, status: .Status, members: (.GlobalClusterMembers | length)}'
```

---

#### Paso 2.2 — Medir RPO Real y Ejecutar Managed Planned Failover

**Objetivo:** Medir el RPO real entre regiones y ejecutar un failover controlado documentando RTO.

**Instrucciones:**

1. Inserta datos de referencia para medir RPO:

```bash
# Crear tabla de marcadores RPO en la región primaria
psql -h $AURORA_WRITER_ENDPOINT -U $DB_USER -d $DB_NAME << 'EOF'
CREATE TABLE IF NOT EXISTS dr_rpo_markers (
  marker_id     SERIAL PRIMARY KEY,
  region        VARCHAR(20) NOT NULL,
  inserted_at   TIMESTAMPTZ DEFAULT now(),
  sequence_num  BIGINT,
  payload       TEXT
);

-- Insertar marcadores cada segundo durante 30 segundos
DO $$
DECLARE i INTEGER;
BEGIN
  FOR i IN 1..30 LOOP
    INSERT INTO dr_rpo_markers (region, sequence_num, payload)
    VALUES ('us-east-1', i, 'rpo_test_' || i::TEXT || '_' || now()::TEXT);
    PERFORM pg_sleep(1);
  END LOOP;
END;
$$;

SELECT count(*) AS total_markers, max(inserted_at) AS last_insert FROM dr_rpo_markers;
EOF
```

2. Verifica replicación en región secundaria **antes** del failover:

```bash
SECONDARY_READER=$(aws rds describe-db-clusters \
  --db-cluster-identifier "aurora-lab-cluster-secondary" \
  --region $AWS_SECONDARY_REGION \
  --query 'DBClusters[0].ReaderEndpoint' --output text)

echo "Reader secundario: $SECONDARY_READER"

# Contar marcadores replicados en la región secundaria
psql -h $SECONDARY_READER -U $DB_USER -d $DB_NAME \
  -c "SELECT count(*) AS replicated_markers, max(inserted_at) AS last_replicated FROM dr_rpo_markers;"

# Calcular lag de replicación
psql -h $AURORA_WRITER_ENDPOINT -U $DB_USER -d $DB_NAME \
  -c "SELECT EXTRACT(EPOCH FROM (now() - max(inserted_at))) AS lag_seconds FROM dr_rpo_markers;" | tee dr/rpo_measurement.txt
```

3. Ejecuta el Managed Planned Failover y mide el RTO:

```bash
echo "=== INICIANDO MANAGED PLANNED FAILOVER ==="
echo "Timestamp inicio: $(date -u +%Y-%m-%dT%H:%M:%SZ)" | tee dr/failover_log.txt

FAILOVER_START=$(date +%s)

# Ejecutar failover gestionado a la región secundaria
aws rds failover-global-cluster \
  --global-cluster-identifier aurora-lab-global \
  --target-db-cluster-identifier \
    "arn:aws:rds:${AWS_SECONDARY_REGION}:${AWS_ACCOUNT_ID}:cluster:aurora-lab-cluster-secondary" \
  --allow-data-loss \
  --region $AWS_PRIMARY_REGION

echo "Failover iniciado. Monitoreando estado..."

# Monitorear hasta que la región secundaria sea writer
while true; do
  STATUS=$(aws rds describe-global-clusters \
    --global-cluster-identifier aurora-lab-global \
    --region $AWS_PRIMARY_REGION \
    --query 'GlobalClusters[0].GlobalClusterMembers[?IsWriter==`true`].DBClusterArn' \
    --output text 2>/dev/null)

  echo "$(date -u +%H:%M:%S) - Writer actual: $STATUS"

  if echo "$STATUS" | grep -q "$AWS_SECONDARY_REGION"; then
    FAILOVER_END=$(date +%s)
    RTO=$((FAILOVER_END - FAILOVER_START))
    echo "=== FAILOVER COMPLETADO ===" | tee -a dr/failover_log.txt
    echo "RTO medido: ${RTO} segundos" | tee -a dr/failover_log.txt
    break
  fi
  sleep 10
done
```

4. Verifica datos en la nueva región primaria (antes: secundaria) y calcula RPO real:

```bash
NEW_PRIMARY_ENDPOINT=$(aws rds describe-db-clusters \
  --db-cluster-identifier "aurora-lab-cluster-secondary" \
  --region $AWS_SECONDARY_REGION \
  --query 'DBClusters[0].Endpoint' --output text)

psql -h $NEW_PRIMARY_ENDPOINT -U $DB_USER -d $DB_NAME << 'EOF' | tee dr/rpo_final_measurement.txt
-- Contar marcadores disponibles post-failover
SELECT
  count(*)                                          AS markers_available,
  max(sequence_num)                                 AS max_sequence,
  max(inserted_at)                                  AS last_data_timestamp,
  now() - max(inserted_at)                          AS data_age,
  EXTRACT(EPOCH FROM (now() - max(inserted_at)))    AS rpo_seconds
FROM dr_rpo_markers;
EOF

echo ""
echo "=== RESUMEN DR ==="
cat dr/failover_log.txt
cat dr/rpo_final_measurement.txt
```

**Salida esperada:**
```
=== FAILOVER COMPLETADO ===
RTO medido: 47 segundos

 markers_available | max_sequence | rpo_seconds
-------------------+--------------+-------------
                29 |           29 |        1.23
```

**Verificación:**
```bash
# Confirmar que la región secundaria ahora es writer
aws rds describe-global-clusters \
  --global-cluster-identifier aurora-lab-global \
  --region $AWS_PRIMARY_REGION \
  --query 'GlobalClusters[0].GlobalClusterMembers[].{ARN:DBClusterArn,Writer:IsWriter}' \
  --output table
```

---

### FASE 3: Dashboard de Observabilidad y Entregable de Arquitectura

---

#### Paso 3.1 — Construir Dashboard de CloudWatch Personalizado

**Objetivo:** Crear un dashboard de CloudWatch que integre métricas de PI, Enhanced Monitoring y pg_stat_statements.

**Instrucciones:**

1. Genera el JSON del dashboard con todas las métricas críticas:

```bash
cat << 'EOF' > dashboard/aurora_observability_dashboard.json
{
  "widgets": [
    {
      "type": "metric",
      "x": 0, "y": 0, "width": 12, "height": 6,
      "properties": {
        "title": "Aurora - TPS y Latencia (Rendimiento OLTP)",
        "metrics": [
          ["AWS/RDS", "CommitThroughput",    "DBClusterIdentifier", "aurora-lab-cluster"],
          ["AWS/RDS", "DMLThroughput",       "DBClusterIdentifier", "aurora-lab-cluster"],
          ["AWS/RDS", "SelectThroughput",    "DBClusterIdentifier", "aurora-lab-cluster"]
        ],
        "period": 60,
        "stat": "Average",
        "view": "timeSeries",
        "region": "us-east-1"
      }
    },
    {
      "type": "metric",
      "x": 12, "y": 0, "width": 12, "height": 6,
      "properties": {
        "title": "Performance Insights - AAS por Tipo de Espera",
        "metrics": [
          ["AWS/RDS", "DBLoad",           "DBInstanceIdentifier", "aurora-lab-cluster-instance-1"],
          ["AWS/RDS", "DBLoadCPU",        "DBInstanceIdentifier", "aurora-lab-cluster-instance-1"],
          ["AWS/RDS", "DBLoadNonCPU",     "DBInstanceIdentifier", "aurora-lab-cluster-instance-1"]
        ],
        "period": 60,
        "stat": "Average",
        "view": "timeSeries",
        "region": "us-east-1"
      }
    },
    {
      "type": "metric",
      "x": 0, "y": 6, "width": 8, "height": 6,
      "properties": {
        "title": "Utilización CPU y Memoria",
        "metrics": [
          ["AWS/RDS", "CPUUtilization",          "DBClusterIdentifier", "aurora-lab-cluster"],
          ["AWS/RDS", "FreeableMemory",           "DBClusterIdentifier", "aurora-lab-cluster"]
        ],
        "period": 60,
        "stat": "Average",
        "view": "timeSeries",
        "region": "us-east-1"
      }
    },
    {
      "type": "metric",
      "x": 8, "y": 6, "width": 8, "height": 6,
      "properties": {
        "title": "Conexiones - Direct vs RDS Proxy",
        "metrics": [
          ["AWS/RDS",   "DatabaseConnections", "DBClusterIdentifier", "aurora-lab-cluster"],
          ["AWS/RDS",   "DatabaseConnections", "ProxyName", "aurora-lab-proxy"]
        ],
        "period": 60,
        "stat": "Average",
        "view": "timeSeries",
        "region": "us-east-1"
      }
    },
    {
      "type": "metric",
      "x": 16, "y": 6, "width": 8, "height": 6,
      "properties": {
        "title": "I/O - Latencia de Lectura y Escritura",
        "metrics": [
          ["AWS/RDS", "ReadLatency",   "DBClusterIdentifier", "aurora-lab-cluster"],
          ["AWS/RDS", "WriteLatency",  "DBClusterIdentifier", "aurora-lab-cluster"],
          ["AWS/RDS", "ReadIOPS",      "DBClusterIdentifier", "aurora-lab-cluster"],
          ["AWS/RDS", "WriteIOPS",     "DBClusterIdentifier", "aurora-lab-cluster"]
        ],
        "period": 60,
        "stat": "Average",
        "view": "timeSeries",
        "region": "us-east-1"
      }
    },
    {
      "type": "metric",
      "x": 0, "y": 12, "width": 12, "height": 6,
      "properties": {
        "title": "Aurora Global Database - Replication Lag (DR)",
        "metrics": [
          ["AWS/RDS", "AuroraGlobalDBReplicationLag",
           "DBClusterIdentifier", "aurora-lab-cluster-secondary",
           {"region": "us-west-2"}]
        ],
        "period": 60,
        "stat": "Maximum",
        "view": "timeSeries",
        "annotations": {
          "horizontal": [{"value": 1000, "label": "RPO Objetivo (1s)", "color": "#ff0000"}]
        }
      }
    },
    {
      "type": "metric",
      "x": 12, "y": 12, "width": 12, "height": 6,
      "properties": {
        "title": "Autovacuum y Bloat - Salud de Tablas",
        "metrics": [
          ["AWS/RDS", "MaximumUsedTransactionIDs", "DBClusterIdentifier", "aurora-lab-cluster"],
          ["AWS/RDS", "OldestReplicationSlotLag",  "DBClusterIdentifier", "aurora-lab-cluster"]
        ],
        "period": 300,
        "stat": "Maximum",
        "view": "timeSeries",
        "region": "us-east-1"
      }
    }
  ]
}
EOF

# Crear el dashboard en CloudWatch
aws cloudwatch put-dashboard \
  --dashboard-name "Aurora-OLTP-Observabilidad-Integral" \
  --dashboard-body file://dashboard/aurora_observability_dashboard.json \
  --region $AWS_PRIMARY_REGION

echo "Dashboard creado. URL:"
echo "https://${AWS_PRIMARY_REGION}.console.aws.amazon.com/cloudwatch/home?region=${AWS_PRIMARY_REGION}#dashboards:name=Aurora-OLTP-Observabilidad-Integral"
```

2. Configura alertas de EventBridge para automatización de DR:

```bash
# Alarma CloudWatch para replication lag crítico
aws cloudwatch put-metric-alarm \
  --alarm-name "Aurora-GlobalDB-ReplicationLag-Critical" \
  --alarm-description "Replication lag supera 5 segundos — evaluar DR" \
  --metric-name AuroraGlobalDBReplicationLag \
  --namespace AWS/RDS \
  --dimensions Name=DBClusterIdentifier,Value=aurora-lab-cluster-secondary \
  --statistic Maximum \
  --period 60 \
  --evaluation-periods 3 \
  --threshold 5000 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions "arn:aws:sns:${AWS_SECONDARY_REGION}:${AWS_ACCOUNT_ID}:aurora-dr-alerts" \
  --region $AWS_SECONDARY_REGION

# Alarma para AAS > vCPU (saturación de base de datos)
aws cloudwatch put-metric-alarm \
  --alarm-name "Aurora-DBLoad-Saturacion-CPU" \
  --alarm-description "AAS supera vCPU disponibles — revisar en Performance Insights" \
  --metric-name DBLoad \
  --namespace AWS/RDS \
  --dimensions Name=DBInstanceIdentifier,Value=$AURORA_WRITER_ID \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 8 \
  --comparison-operator GreaterThanThreshold \
  --region $AWS_PRIMARY_REGION

echo "Alarmas de observabilidad configuradas."
```

**Salida esperada:**
```json
{
    "DashboardValidationMessages": []
}
Dashboard creado. URL:
https://us-east-1.console.aws.amazon.com/cloudwatch/home?region=us-east-1#dashboards:name=Aurora-OLTP-Observabilidad-Integral
```

**Verificación:**
```bash
aws cloudwatch list-dashboards \
  --region $AWS_PRIMARY_REGION \
  --query 'DashboardEntries[?DashboardName==`Aurora-OLTP-Observabilidad-Integral`].[DashboardName,LastModified]' \
  --output table
```

---

#### Paso 3.2 — Generar Documento Técnico de Arquitectura

**Objetivo:** Producir el entregable de arquitectura que justifica técnicamente todas las decisiones de diseño.

**Instrucciones:**

1. Recopila todos los datos del laboratorio para el documento:

```bash
cat << 'PYEOF' > ~/lab-04-00-03/generate_architecture_doc.py
#!/usr/bin/env python3
"""Genera el documento técnico de arquitectura Aurora PostgreSQL."""
import json, os, subprocess
from datetime import datetime

def read_file(path, default="N/A"):
    try:
        with open(path) as f:
            return f.read().strip()
    except:
        return default

def run_cmd(cmd):
    try:
        result = subprocess.run(cmd, shell=True, capture_output=True, text=True, timeout=30)
        return result.stdout.strip()
    except:
        return "N/A"

# Leer resultados de benchmarks
baseline_content  = read_file('baseline/pgbench_baseline_results.txt')
optimized_content = read_file('optimized/pgbench_optimized_results.txt')
impact_report     = read_file('optimized/impact_report.txt')
failover_log      = read_file('dr/failover_log.txt')
rpo_measurement   = read_file('dr/rpo_final_measurement.txt')

doc = f"""# Documento Técnico de Arquitectura Aurora PostgreSQL
## E-Commerce de Alta Concurrencia — Solución Optimizada

**Fecha:** {datetime.now().strftime('%Y-%m-%d %H:%M UTC')}
**Autor:** Arquitecto de Base de Datos — Lab 04-00-03

---

## 1. Resumen Ejecutivo

Este documento describe la arquitectura Aurora PostgreSQL optimizada para una aplicación
e-commerce OLTP de alta concurrencia, implementada y validada durante el laboratorio
04-00-03. Se cubren cuatro dimensiones: rendimiento, disponibilidad, observabilidad y DR.

---

## 2. Arquitectura Propuesta

### 2.1 Topología de Cluster

```
[Aplicación]
     |
[RDS Proxy — Connection Pooling]
     |
[Aurora PostgreSQL Cluster — us-east-1 (Primario)]
  ├── Writer Instance (db.r6g.large)
  └── Reader Instance (db.r6g.large)
     |
     | (Replicación Global — <1s lag)
     |
[Aurora PostgreSQL Cluster — us-west-2 (Secundario DR)]
  └── Reader Instance (db.r6g.large)
```

### 2.2 Justificación de Parámetros Críticos

| Parámetro                        | Valor Configurado | Justificación                                      |
|----------------------------------|-------------------|---------------------------------------------------|
| work_mem                         | 64 MB             | Evita spills a disco en sorts/joins OLTP           |
| shared_buffers                   | 4 GB              | 25% de RAM para cache de páginas frecuentes        |
| effective_cache_size             | 12 GB             | Orienta al planificador sobre cache OS disponible  |
| random_page_cost                 | 1.1               | SSD/NVMe Aurora — favorece index scans             |
| effective_io_concurrency         | 200               | Paralelismo I/O para Aurora storage                |
| checkpoint_completion_target     | 0.9               | Distribuye checkpoints, reduce spikes I/O          |
| autovacuum_vacuum_scale_factor   | 0.01              | Vacuums más frecuentes en tablas OLTP grandes      |
| autovacuum_vacuum_cost_delay     | 2 ms              | Reduce impacto de autovacuum en carga activa       |

---

## 3. Estrategia de Indexación

### 3.1 Índices Implementados y Justificación

- **idx_accounts_active_balance (parcial):** Cubre el hot path de consultas sobre cuentas
  activas con balance positivo. Reduce 80% del scan de pgbench_accounts.
- **idx_history_aid_mtime (compuesto):** Optimiza queries de historial por cuenta ordenado
  por tiempo — patrón frecuente en e-commerce (historial de pedidos).
- **idx_branches_bid_bbalance:** Elimina sequential scans en joins de branches.

### 3.2 Principios Aplicados

- Índices CONCURRENTLY para no bloquear producción.
- Índices parciales donde la cardinalidad selectiva lo justifica (abalance > 0).
- ANALYZE post-creación para actualizar estadísticas del planificador.

---

## 4. Configuración RDS Proxy

- **MaxConnectionsPercent:** 100 (proxy gestiona el pool completo)
- **MaxIdleConnectionsPercent:** 50 (mantiene conexiones calientes)
- **ConnectionBorrowTimeout:** 120s
- **InitQuery:** SET search_path TO public; SET application_name TO 'ecommerce-proxy'
- **Beneficio medido:** Soporte de 50 clientes concurrentes sin degradación vs 20 directos.

---

## 5. Plan de Disaster Recovery

### 5.1 Objetivos Definidos

| Métrica | Objetivo | Resultado Medido |
|---------|----------|-----------------|
| RTO     | < 60s    | {run_cmd("grep 'RTO medido' dr/failover_log.txt || echo 'ver failover_log.txt'")} |
| RPO     | < 5s     | {run_cmd("grep 'rpo_seconds' dr/rpo_final_measurement.txt || echo 'ver rpo_final_measurement.txt'")} |

### 5.2 Procedimiento de Failover

1. Detectar degradación en región primaria (alarma CloudWatch AuroraGlobalDBReplicationLag).
2. Verificar que el lag en región secundaria sea < 1s antes de failover.
3. Ejecutar: `aws rds failover-global-cluster --global-cluster-identifier aurora-lab-global --target-db-cluster-identifier <arn-secundario>`
4. Actualizar DNS/endpoints de aplicación (Route 53 health check automatizado).
5. Validar escrituras en nueva región primaria con tabla dr_rpo_markers.
6. Documentar RTO real y comparar con SLA.

---

## 6. Impacto de Optimizaciones

{impact_report}

---

## 7. Observabilidad — Dashboard de CloudWatch

### 7.1 Métricas Clave Monitoreadas

| Métrica                       | Fuente                  | Umbral de Alerta       |
|-------------------------------|-------------------------|------------------------|
| DBLoad / DBLoadCPU            | Performance Insights    | > vCPU (8) por 5 min   |
| AuroraGlobalDBReplicationLag  | CloudWatch RDS          | > 5,000 ms             |
| DatabaseConnections           | CloudWatch RDS          | > 80% max_connections  |
| ReadLatency / WriteLatency    | CloudWatch RDS          | > 20 ms promedio       |
| CPUUtilization                | CloudWatch RDS          | > 80% por 10 min       |

### 7.2 Ciclo de Diagnóstico con Performance Insights

1. Detectar pico en AAS (gráfico apilado por wait events).
2. Identificar categoría dominante: CPU / IO / Lock / LWLock.
3. Cambiar dimensión a SQL para identificar query causante.
4. Aplicar corrección focalizada (índice, batch, parámetro).
5. Validar reducción de AAS en PI post-cambio.

---

## 8. Conclusiones y Recomendaciones

- La combinación de parameter tuning + indexación estratégica + RDS Proxy produjo una
  mejora de TPS > 50% en carga OLTP de 50 clientes concurrentes.
- Aurora Global Database demostró cumplir el objetivo de RPO < 5s y RTO < 60s.
- Performance Insights es la herramienta principal de diagnóstico: habilitar siempre en
  producción con retención extendida y pg_stat_statements activo.
- Se recomienda revisión trimestral de índices con pg_stat_user_indexes para eliminar
  índices no utilizados y ajustar autovacuum por tabla según tasa de cambio.
"""

output_path = 'architecture/arquitectura_aurora_optimizada.md'
with open(output_path, 'w') as f:
    f.write(doc)

print(f"✓ Documento técnico generado: {output_path}")
print(f"  Tamaño: {os.path.getsize(output_path)} bytes")
PYEOF

python3 ~/lab-04-00-03/generate_architecture_doc.py
```

2. Verifica el documento generado:

```bash
# Mostrar las primeras 50 líneas del documento
head -50 ~/lab-04-00-03/architecture/arquitectura_aurora_optimizada.md

# Listar todos los entregables del laboratorio
echo ""
echo "=== ENTREGABLES DEL LABORATORIO ==="
find ~/lab-04-00-03 -type f | sort | while read f; do
  echo "  $(ls -lh $f | awk '{print $5, $9}')"
done
```

**Salida esperada:**
```
✓ Documento técnico generado: architecture/arquitectura_aurora_optimizada.md
  Tamaño: 4832 bytes

=== ENTREGABLES DEL LABORATORIO ===
  baseline/pgbench_baseline_results.txt
  baseline/pg_stat_statements_baseline.txt
  baseline/pi_aas_baseline.json
  optimized/pgbench_optimized_results.txt
  optimized/impact_report.txt
  dr/failover_log.txt
  dr/rpo_final_measurement.txt
  dashboard/aurora_observability_dashboard.json
  architecture/arquitectura_aurora_optimizada.md
```

**Verificación:**
```bash
wc -l ~/lab-04-00-03/architecture/arquitectura_aurora_optimizada.md
grep "RTO\|RPO\|TPS\|Mejora" ~/lab-04-00-03/architecture/arquitectura_aurora_optimizada.md
```

---

## Validación y Pruebas Finales

Ejecuta la siguiente secuencia de validación para confirmar que el laboratorio está completo:

```bash
#!/bin/bash
echo "============================================"
echo "  VALIDACIÓN FINAL — LAB 04-00-03"
echo "============================================"

PASS=0; FAIL=0

check() {
  local desc="$1"; local cmd="$2"
  if eval "$cmd" &>/dev/null; then
    echo "  ✓ PASS: $desc"; ((PASS++))
  else
    echo "  ✗ FAIL: $desc"; ((FAIL++))
  fi
}

# Fase 1: Optimización
check "Línea base capturada" \
  "grep -q 'tps =' ~/lab-04-00-03/baseline/pgbench_baseline_results.txt"

check "Benchmark optimizado completado" \
  "grep -q 'tps =' ~/lab-04-00-03/optimized/pgbench_optimized_results.txt"

check "Índices creados en DB" \
  "psql -h $AURORA_WRITER_ENDPOINT -U $DB_USER -d $DB_NAME -t -c \"SELECT count(*) FROM pg_indexes WHERE indexname LIKE 'idx_%';\" | grep -qv '^0$'"

check "pg_stat_statements activo" \
  "psql -h $AURORA_WRITER_ENDPOINT -U $DB_USER -d $DB_NAME -t -c \"SELECT count(*) FROM pg_stat_statements;\" | grep -qv '^0$'"

check "Reporte de impacto generado" \
  "test -s ~/lab-04-00-03/optimized/impact_report.txt"

# Fase 2: DR
check "Aurora Global Database existe" \
  "aws rds describe-global-clusters --global-cluster-identifier aurora-lab-global --region $AWS_PRIMARY_REGION"

check "Failover log existe" \
  "test -s ~/lab-04-00-03/dr/failover_log.txt"

check "Medición RPO existe" \
  "test -s ~/lab-04-00-03/dr/rpo_final_measurement.txt"

# Fase 3: Dashboard y Arquitectura
check "Dashboard CloudWatch creado" \
  "aws cloudwatch list-dashboards --region $AWS_PRIMARY_REGION --query 'DashboardEntries[?DashboardName==\`Aurora-OLTP-Observabilidad-Integral\`]' --output text | grep -q Aurora"

check "Documento de arquitectura generado" \
  "test -s ~/lab-04-00-03/architecture/arquitectura_aurora_optimizada.md"

check "Alarma replication lag configurada" \
  "aws cloudwatch describe-alarms --alarm-names Aurora-GlobalDB-ReplicationLag-Critical --region $AWS_SECONDARY_REGION --query 'MetricAlarms[0].AlarmName' --output text | grep -q Aurora"

echo "--------------------------------------------"
echo "  Resultado: ${PASS} PASS | ${FAIL} FAIL"
echo "============================================"
[ $FAIL -eq 0 ] && echo "  ✓ Laboratorio COMPLETADO exitosamente" || echo "  ✗ Revisar los ítems fallidos"
```

---

## Solución de Problemas

### Problema 1: El Managed Planned Failover falla con error "InvalidDBClusterStateFault"

**Síntomas:**
```
An error occurred (InvalidDBClusterStateFault) when calling the FailoverGlobalCluster
operation: Global cluster aurora-lab-global is not in available state.
```

**Causa:** El cluster secundario aún está en estado de inicialización o replicación inicial no completada. Aurora Global Database requiere que ambos clusters estén en estado `available` y que el lag de replicación sea bajo antes de permitir un planned failover.

**Solución:**
```bash
# 1. Verificar estado de ambos clusters
aws rds describe-global-clusters \
  --global-cluster-identifier aurora-lab-global \
  --region $AWS_PRIMARY_REGION \
  --query 'GlobalClusters[0].{Status:Status,Members:GlobalClusterMembers[].{ARN:DBClusterArn,Writer:IsWriter}}' \
  --output json

# 2. Esperar a que el cluster secundario esté "available"
aws rds wait db-cluster-available \
  --db-cluster-identifier aurora-lab-cluster-secondary \
  --region $AWS_SECONDARY_REGION

# 3. Verificar lag de replicación < 1000ms antes de reintentar
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name AuroraGlobalDBReplicationLag \
  --dimensions Name=DBClusterIdentifier,Value=aurora-lab-cluster-secondary \
  --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Maximum \
  --region $AWS_SECONDARY_REGION \
  --query 'Datapoints[0].Maximum'

# 4. Reintentar el failover solo cuando el lag sea < 1000ms
```

---

### Problema 2: Performance Insights muestra "No data available" para Top SQL

**Síntomas:** El gráfico de AAS muestra actividad pero la pestaña "Top SQL" aparece vacía o con el mensaje "pg_stat_statements not enabled".

**Causa:** La extensión `pg_stat_statements` no está cargada en `shared_preload_libraries`, o el parameter group fue modificado pero la instancia no fue reiniciada para aplicar el cambio (parámetro de tipo `pending-reboot`).

**Solución:**
```bash
# 1. Verificar si pg_stat_statements está en shared_preload_libraries
psql -h $AURORA_WRITER_ENDPOINT -U $DB_USER -d $DB_NAME \
  -c "SHOW shared_preload_libraries;"

# 2. Si no está, modificar el parameter group
PARAM_GROUP=$(aws rds describe-db-clusters \
  --db-cluster-identifier $AURORA_CLUSTER_ID \
  --region $AWS_PRIMARY_REGION \
  --query 'DBClusters[0].DBClusterParameterGroup' --output text)

aws rds modify-db-cluster-parameter-group \
  --db-cluster-parameter-group-name $PARAM_GROUP \
  --parameters \
    "ParameterName=shared_preload_libraries,ParameterValue=pg_stat_statements,ApplyMethod=pending-reboot" \
    "ParameterName=pg_stat_statements.track,ParameterValue=all,ApplyMethod=immediate" \
  --region $AWS_PRIMARY_REGION

# 3. Reiniciar la instancia writer para aplicar shared_preload_libraries
aws rds reboot-db-instance \
  --db-instance-identifier $AURORA_WRITER_ID \
  --region $AWS_PRIMARY_REGION

# Esperar reinicio
aws rds wait db-instance-available \
  --db-instance-identifier $AURORA_WRITER_ID \
  --region $AWS_PRIMARY_REGION

# 4. Crear la extensión en la base de datos
psql -h $AURORA_WRITER_ENDPOINT -U $DB_USER -d $DB_NAME \
  -c "CREATE EXTENSION IF NOT EXISTS pg_stat_statements;"

# 5. Verificar que está activa
psql -h $AURORA_WRITER_ENDPOINT -U $DB_USER -d $DB_NAME \
  -c "SELECT count(*) FROM pg_stat_statements;"
```

---

## Limpieza de Recursos

> ⚠️ **Importante:** Ejecuta la limpieza al finalizar para evitar costos innecesarios. Los recursos de Aurora Global Database generan cargos en ambas regiones.

```bash
echo "=== INICIANDO LIMPIEZA DE RECURSOS ==="

# 1. Guardar todos los entregables localmente antes de destruir
tar -czf ~/lab-04-00-03-entregables-$(date +%Y%m%d).tar.gz ~/lab-04-00-03/
echo "✓ Entregables archivados en ~/lab-04-00-03-entregables-$(date +%Y%m%d).tar.gz"

# 2. Eliminar instancia secundaria
aws rds delete-db-instance \
  --db-instance-identifier aurora-lab-secondary-instance-1 \
  --skip-final-snapshot \
  --region $AWS_SECONDARY_REGION
echo "Eliminando instancia secundaria..."
aws rds wait db-instance-deleted \
  --db-instance-identifier aurora-lab-secondary-instance-1 \
  --region $AWS_SECONDARY_REGION

# 3. Eliminar cluster secundario de la Global Database
aws rds remove-from-global-cluster \
  --global-cluster-identifier aurora-lab-global \
  --db-cluster-identifier \
    "arn:aws:rds:${AWS_SECONDARY_REGION}:${AWS_ACCOUNT_ID}:cluster:aurora-lab-cluster-secondary" \
  --region $AWS_PRIMARY_REGION

# 4. Eliminar cluster secundario
aws rds delete-db-cluster \
  --db-cluster-identifier aurora-lab-cluster-secondary \
  --skip-final-snapshot \
  --region $AWS_SECONDARY_REGION

# 5. Eliminar Global Database
aws rds delete-global-cluster \
  --global-cluster-identifier aurora-lab-global \
  --region $AWS_PRIMARY_REGION

# 6. Eliminar dashboard de CloudWatch
aws cloudwatch delete-dashboards \
  --dashboard-names "Aurora-OLTP-Observabilidad-Integral" \
  --region $AWS_PRIMARY_REGION

# 7. Eliminar alarmas
aws cloudwatch delete-alarms \
  --alarm-names \
    "Aurora-GlobalDB-ReplicationLag-Critical" \
  --region $AWS_SECONDARY_REGION

aws cloudwatch delete-alarms \
  --alarm-names \
    "Aurora-DBLoad-Saturacion-CPU" \
  --region $AWS_PRIMARY_REGION

# 8. Destruir infraestructura base con Terraform
cd ~/curso-aurora/terraform/modulo-04
terraform destroy -auto-approve

echo ""
echo "=== LIMPIEZA COMPLETADA ==="
echo "Verificar en la consola AWS que no queden recursos activos en:"
echo "  - us-east-1: RDS > Clusters, RDS > Proxies"
echo "  - us-west-2: RDS > Clusters"
echo "  - CloudWatch > Dashboards, Alarms"
```

---

## Resumen

En este laboratorio de cierre integraste todos los conceptos del curso en un escenario empresarial complejo de e-commerce OLTP. A través de tres fases encadenadas:

**Fase 1 — Optimización Integral:** Estableciste una línea base con pgbench, aplicaste simultáneamente parameter tuning (work_mem, random_page_cost, autovacuum), indexación estratégica (índices parciales y compuestos con CONCURRENTLY), y validaste la mejora acumulada de TPS vía RDS Proxy. La PI API te permitió correlacionar la reducción de waits con las mejoras de rendimiento.

**Fase 2 — Disaster Recovery Multi-Región:** Configuraste Aurora Global Database con un cluster secundario en una segunda región, mediste el RPO real insertando marcadores de referencia, y ejecutaste un Managed Planned Failover documentando RTO y RPO obtenidos frente a los objetivos definidos.

**Fase 3 — Observabilidad y Arquitectura:** Construiste un dashboard de CloudWatch que integra métricas de Performance Insights (DBLoad, DBLoadCPU), Enhanced Monitoring y métricas operacionales (latencia, conexiones, replication lag), configuraste alertas automatizadas con EventBridge, y generaste un documento técnico de arquitectura que justifica cada decisión de diseño.

### Conceptos Clave Demostrados

| Concepto                    | Técnica Aplicada                                        |
|-----------------------------|---------------------------------------------------------|
| Medición de impacto         | pgbench antes/después con métricas cuantitativas        |
| Observabilidad de waits     | Performance Insights API — AAS por wait_event_type      |
| Connection pooling          | RDS Proxy — benchmark con 50 clientes concurrentes      |
| DR multi-región             | Aurora Global Database + Managed Planned Failover       |
| RPO/RTO real                | Marcadores de datos + cronometría de failover           |
| Dashboard integrado         | CloudWatch con PI, Enhanced Monitoring y alarmas        |

### Recursos Adicionales

- [Aurora Global Database — Documentación AWS](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
- [Managed Planned Failover para Aurora Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database-disaster-recovery.html#aurora-global-database-failover)
- [Performance Insights API Reference](https://docs.aws.amazon.com/performance-insights/latest/APIReference/Welcome.html)
- [Buenas prácticas de Aurora PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.BestPractices.html)
- [pgbench — Documentación PostgreSQL](https://www.postgresql.org/docs/current/pgbench.html)
- [CloudWatch Dashboards — Guía del usuario](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Dashboards.html)

---
*Lab 04-00-03 — Módulo 4: Observabilidad Avanzada y Disaster Recovery en Aurora PostgreSQL*
