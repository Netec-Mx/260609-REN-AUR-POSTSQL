# Análisis de rendimiento con logs y tuning

## Metadatos

| Campo            | Detalle                                      |
|------------------|----------------------------------------------|
| **Duración**     | 45 minutos                                   |
| **Complejidad**  | Media                                        |
| **Nivel Bloom**  | Aplicar (Apply)                              |
| **Módulo**       | 1 — Fundamentos de rendimiento en Aurora PostgreSQL |
| **Laboratorio**  | 01-00-01                                     |

---

## Descripción General

En este laboratorio configurarás un entorno Aurora PostgreSQL para capturar consultas lentas mediante `log_min_duration_statement`, crearás una base de datos de prueba con tablas representativas de carga OLTP y analizarás planes de ejecución usando `EXPLAIN` y `EXPLAIN ANALYZE`. Además, implementarás una estructura de roles y privilegios siguiendo el principio de mínimo privilegio, separando el rol propietario de los roles operativos y verificando los accesos con las vistas del catálogo de PostgreSQL.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:

- [ ] Configurar `log_min_duration_statement` en un Parameter Group de Aurora PostgreSQL para capturar slow queries
- [ ] Crear índices B-tree simples, compuestos, parciales y de cobertura, y comparar su impacto en los planes de ejecución
- [ ] Interpretar la salida de `EXPLAIN` y `EXPLAIN ANALYZE` para identificar escaneos ineficientes y seleccionar el índice adecuado
- [ ] Implementar una jerarquía de roles con `GRANT/REVOKE` y `ALTER DEFAULT PRIVILEGES` siguiendo principios de mínimo privilegio
- [ ] Verificar privilegios efectivos con `information_schema.table_privileges` y `pg_roles`

---

## Prerrequisitos

### Conocimiento previo
- Sintaxis básica de SQL (SELECT, INSERT, UPDATE, CREATE TABLE)
- Concepto general de índices en bases de datos relacionales
- Familiaridad con la línea de comandos Linux/macOS o PowerShell
- Haber leído la lección 1.1 sobre Roles y privilegios en PostgreSQL

### Acceso y recursos
- Clúster Aurora PostgreSQL 14.x o superior provisionado y accesible (endpoint anotado)
- AWS CLI 2.x configurado con perfil que tenga permisos sobre RDS y CloudWatch
- `psql` 14.x instalado y conexión verificada al clúster
- Repositorio del curso clonado localmente (`git clone <repo-url>`)
- Usuario IAM con políticas para `rds:ModifyDBClusterParameterGroup`, `rds:DescribeDBClusters` y `cloudwatch:GetMetricData`

> ⚠️ **Advertencia de costos:** Este laboratorio utiliza instancias `db.r6g.large`. Recuerda ejecutar `terraform destroy` al finalizar para evitar cargos innecesarios.

---

## Entorno de Laboratorio

### Hardware recomendado

| Recurso       | Mínimo                  |
|---------------|-------------------------|
| RAM           | 8 GB                    |
| CPU           | 4 núcleos               |
| Almacenamiento| 10 GB libres            |
| Red           | 10 Mbps estable         |

### Software requerido

| Herramienta         | Versión          | Uso en este lab                        |
|---------------------|------------------|----------------------------------------|
| AWS CLI             | 2.x              | Modificar Parameter Groups             |
| psql                | 14.x o superior  | Herramienta principal de SQL           |
| DBeaver Community   | 23.x o superior  | Visualización gráfica de planes        |
| Python              | 3.9 o superior   | Script de carga de datos de prueba     |
| jq                  | 1.6 o superior   | Parsear salidas JSON de AWS CLI        |

### Variables de entorno iniciales

Ejecuta los siguientes comandos en tu terminal antes de comenzar. Reemplaza los valores entre `< >` con los datos de tu entorno:

```bash
# Endpoint del clúster Aurora (writer)
export AURORA_ENDPOINT="<tu-cluster-endpoint>.cluster-xxxx.us-east-1.rds.amazonaws.com"
export AURORA_PORT="5432"
export AURORA_DBNAME="postgres"
export AURORA_MASTER_USER="postgres"
export AURORA_CLUSTER_ID="<tu-cluster-id>"
export AURORA_PARAM_GROUP="<nombre-del-parameter-group-del-cluster>"
export AWS_REGION="us-east-1"

# Verificar conectividad
psql -h "$AURORA_ENDPOINT" -p "$AURORA_PORT" -U "$AURORA_MASTER_USER" \
     -d "$AURORA_DBNAME" -c "SELECT version();"
```

**Salida esperada de verificación:**

```
                                                 version
---------------------------------------------------------------------------------------------------------
 PostgreSQL 14.x on aarch64-unknown-linux-gnu, compiled by aarch64-unknown-linux-gnu-gcc ..., 64-bit
(1 row)
```

---

## Pasos del Laboratorio

---

### Paso 1: Configurar `log_min_duration_statement` en el Parameter Group

**Objetivo:** Habilitar el registro de consultas lentas en Aurora PostgreSQL para identificar ineficiencias en tiempo real.

#### Instrucciones

1. Identifica el Parameter Group de clúster asociado a tu instancia:

```bash
aws rds describe-db-clusters \
    --db-cluster-identifier "$AURORA_CLUSTER_ID" \
    --region "$AWS_REGION" \
    --query 'DBClusters[0].DBClusterParameterGroup' \
    --output text
```

2. Modifica el parámetro `log_min_duration_statement` para registrar consultas que tarden más de 100 ms:

```bash
aws rds modify-db-cluster-parameter-group \
    --db-cluster-parameter-group-name "$AURORA_PARAM_GROUP" \
    --region "$AWS_REGION" \
    --parameters \
      "ParameterName=log_min_duration_statement,ParameterValue=100,ApplyMethod=immediate" \
      "ParameterName=log_statement,ParameterValue=none,ApplyMethod=immediate" \
      "ParameterName=log_line_prefix,ParameterValue='%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h ',ApplyMethod=immediate"
```

3. Verifica que los parámetros se aplicaron correctamente:

```bash
aws rds describe-db-cluster-parameters \
    --db-cluster-parameter-group-name "$AURORA_PARAM_GROUP" \
    --region "$AWS_REGION" \
    --query "Parameters[?ParameterName=='log_min_duration_statement']" \
    | jq '.[0] | {ParameterName, ParameterValue, ApplyType}'
```

4. Confirma la configuración desde psql:

```sql
-- Conectarse a Aurora
\c postgres

SHOW log_min_duration_statement;
SHOW log_line_prefix;
```

#### Salida esperada

```
 log_min_duration_statement
----------------------------
 100ms
(1 row)

                   log_line_prefix
------------------------------------------------------
 %t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h
(1 row)
```

#### Verificación

```bash
# Confirmar con AWS CLI que el parámetro tiene el valor correcto
aws rds describe-db-cluster-parameters \
    --db-cluster-parameter-group-name "$AURORA_PARAM_GROUP" \
    --region "$AWS_REGION" \
    --query "Parameters[?ParameterName=='log_min_duration_statement'].{Nombre:ParameterName,Valor:ParameterValue}" \
    --output table
```

> 📝 **Nota:** En Aurora PostgreSQL, `log_min_duration_statement` es un parámetro dinámico (`ApplyMethod=immediate`). No es necesario reiniciar la instancia.

---

### Paso 2: Crear la base de datos y el esquema de prueba

**Objetivo:** Provisionar la base de datos `labdb` con tablas representativas de una carga OLTP para los ejercicios de tuning del laboratorio.

#### Instrucciones

1. Crea la base de datos de laboratorio:

```sql
-- Conectado como usuario master
CREATE DATABASE labdb
    ENCODING 'UTF8'
    LC_COLLATE 'en_US.UTF-8'
    LC_CTYPE 'en_US.UTF-8'
    TEMPLATE template0;

\c labdb
```

2. Crea el esquema `app` con un rol propietario dedicado (sin LOGIN):

```sql
-- Rol propietario sin capacidad de login (separación de ownership)
CREATE ROLE app_owner NOLOGIN;

-- Delegar la creación del esquema al rol propietario
SET ROLE app_owner;

CREATE SCHEMA app AUTHORIZATION app_owner;

-- Tablas OLTP representativas
CREATE TABLE app.customers (
    customer_id  bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    email        text   NOT NULL UNIQUE,
    full_name    text   NOT NULL,
    country_code char(2) NOT NULL DEFAULT 'US',
    created_at   timestamptz NOT NULL DEFAULT now(),
    is_active    boolean NOT NULL DEFAULT true
);

CREATE TABLE app.products (
    product_id   bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    sku          text   NOT NULL UNIQUE,
    category     text   NOT NULL,
    price        numeric(10,2) NOT NULL,
    stock        int    NOT NULL DEFAULT 0,
    created_at   timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE app.orders (
    order_id     bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    customer_id  bigint NOT NULL REFERENCES app.customers(customer_id),
    status       text   NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending','processing','shipped','delivered','cancelled')),
    total_amount numeric(12,2) NOT NULL,
    created_at   timestamptz NOT NULL DEFAULT now(),
    updated_at   timestamptz NOT NULL DEFAULT now(),
    created_by   name   NOT NULL DEFAULT current_user
);

CREATE TABLE app.order_items (
    item_id      bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    order_id     bigint NOT NULL REFERENCES app.orders(order_id),
    product_id   bigint NOT NULL REFERENCES app.products(product_id),
    quantity     int    NOT NULL CHECK (quantity > 0),
    unit_price   numeric(10,2) NOT NULL
);

-- Devolver al rol original
RESET ROLE;
```

3. Carga datos de prueba con el script Python incluido en el repositorio, o ejecuta el siguiente bloque SQL de carga rápida:

```sql
\c labdb

-- Insertar 50,000 clientes
INSERT INTO app.customers (email, full_name, country_code, is_active)
SELECT
    'user_' || i || '@example.com',
    'Customer ' || i,
    (ARRAY['US','MX','BR','AR','CO'])[1 + (i % 5)],
    (i % 10 != 0)   -- 10% inactivos
FROM generate_series(1, 50000) AS s(i);

-- Insertar 1,000 productos
INSERT INTO app.products (sku, category, price, stock)
SELECT
    'SKU-' || LPAD(i::text, 6, '0'),
    (ARRAY['Electronics','Clothing','Books','Food','Sports'])[1 + (i % 5)],
    (random() * 500 + 5)::numeric(10,2),
    (random() * 1000)::int
FROM generate_series(1, 1000) AS s(i);

-- Insertar 200,000 órdenes
INSERT INTO app.orders (customer_id, status, total_amount)
SELECT
    1 + (random() * 49999)::int,
    (ARRAY['pending','processing','shipped','delivered','cancelled'])[1 + (random()*4)::int],
    (random() * 2000 + 10)::numeric(12,2)
FROM generate_series(1, 200000) AS s(i);

-- Insertar ~600,000 order_items (3 por orden en promedio)
INSERT INTO app.order_items (order_id, product_id, quantity, unit_price)
SELECT
    o.order_id,
    1 + (random() * 999)::int,
    1 + (random() * 5)::int,
    (random() * 500 + 5)::numeric(10,2)
FROM app.orders o,
     generate_series(1, 3) AS s(i);

-- Actualizar estadísticas
ANALYZE app.customers;
ANALYZE app.products;
ANALYZE app.orders;
ANALYZE app.order_items;
```

#### Salida esperada

```
INSERT 0 50000
INSERT 0 1000
INSERT 0 200000
INSERT 0 600000
ANALYZE
```

#### Verificación

```sql
SELECT
    schemaname,
    tablename,
    n_live_tup AS filas_vivas
FROM pg_stat_user_tables
WHERE schemaname = 'app'
ORDER BY tablename;
```

```
 schemaname |  tablename   | filas_vivas
------------+--------------+-------------
 app        | customers    |       50000
 app        | order_items  |      600000
 app        | orders       |      200000
 app        | products     |        1000
```

---

### Paso 3: Analizar planes de ejecución — línea base sin índices

**Objetivo:** Establecer la línea base de rendimiento de consultas representativas antes de crear índices, identificando escaneos secuenciales (Seq Scan) costosos.

#### Instrucciones

1. Analiza una consulta de búsqueda por email (columna sin índice aún):

```sql
\c labdb

-- Desactivar caché de buffers para resultados reproducibles
SET enable_bitmapscan = on;
SET enable_indexscan = on;

EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT customer_id, email, full_name
FROM app.customers
WHERE email = 'user_12345@example.com';
```

2. Analiza una consulta de órdenes por estado y rango de fechas:

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT order_id, customer_id, status, total_amount
FROM app.orders
WHERE status = 'pending'
  AND created_at >= now() - interval '30 days';
```

3. Analiza un JOIN entre órdenes y clientes:

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT
    c.full_name,
    c.email,
    COUNT(o.order_id)   AS total_orders,
    SUM(o.total_amount) AS revenue
FROM app.orders o
JOIN app.customers c ON o.customer_id = c.customer_id
WHERE o.status = 'delivered'
GROUP BY c.customer_id, c.full_name, c.email
ORDER BY revenue DESC
LIMIT 20;
```

#### Salida esperada (fragmento ilustrativo — Seq Scan esperado)

```
Seq Scan on customers  (cost=0.00..1541.00 rows=1 width=42)
                       (actual time=8.231..18.452 rows=1 loops=1)
  Filter: (email = 'user_12345@example.com'::text)
  Rows Removed by Filter: 49999
  Buffers: shared hit=791
Planning Time: 0.312 ms
Execution Time: 18.521 ms
```

> 📌 **Punto clave:** Observa `Seq Scan`, el alto `Rows Removed by Filter` y el `Execution Time`. Estos valores serán tu referencia para medir la mejora tras crear índices.

#### Verificación

```sql
-- Guardar tiempos de referencia en una tabla de control
CREATE TABLE IF NOT EXISTS public.lab_baseline (
    consulta_id   serial PRIMARY KEY,
    descripcion   text,
    execution_ms  numeric,
    plan_type     text,
    registrado_en timestamptz DEFAULT now()
);

INSERT INTO public.lab_baseline (descripcion, execution_ms, plan_type) VALUES
    ('email lookup sin índice',    18.5, 'Seq Scan'),
    ('orders por status+date sin índice', NULL, 'Seq Scan'),
    ('JOIN orders-customers sin índice',  NULL, 'Seq Scan');
```

---

### Paso 4: Crear índices y comparar planes de ejecución

**Objetivo:** Crear índices B-tree simples, compuestos, parciales y de cobertura; verificar el cambio en los planes de ejecución y cuantificar la mejora de rendimiento.

#### Instrucciones

**4.1 — Índice B-tree simple (email)**

```sql
\c labdb

-- Crear índice simple sobre email
CREATE INDEX CONCURRENTLY idx_customers_email
    ON app.customers (email);

-- Re-ejecutar la misma consulta del Paso 3
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT customer_id, email, full_name
FROM app.customers
WHERE email = 'user_12345@example.com';
```

**4.2 — Índice B-tree compuesto (status + created_at)**

```sql
-- Índice compuesto para la consulta de órdenes por estado y fecha
CREATE INDEX CONCURRENTLY idx_orders_status_created
    ON app.orders (status, created_at DESC);

EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT order_id, customer_id, status, total_amount
FROM app.orders
WHERE status = 'pending'
  AND created_at >= now() - interval '30 days';
```

**4.3 — Índice parcial (solo órdenes activas)**

```sql
-- Índice parcial: solo órdenes no canceladas (subconjunto más consultado)
CREATE INDEX CONCURRENTLY idx_orders_active
    ON app.orders (customer_id, created_at DESC)
    WHERE status != 'cancelled';

-- Consulta que se beneficia del índice parcial
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT order_id, total_amount, created_at
FROM app.orders
WHERE customer_id = 1000
  AND status != 'cancelled'
ORDER BY created_at DESC
LIMIT 10;
```

**4.4 — Índice de cobertura (covering index)**

```sql
-- Índice de cobertura: incluye todas las columnas necesarias para evitar heap fetch
CREATE INDEX CONCURRENTLY idx_orders_covering
    ON app.orders (status, created_at DESC)
    INCLUDE (order_id, customer_id, total_amount);

EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT order_id, customer_id, total_amount
FROM app.orders
WHERE status = 'shipped'
  AND created_at >= now() - interval '7 days';
```

**4.5 — Verificar uso de índices con pg_stat_user_indexes**

```sql
-- Estadísticas de uso de índices (ejecutar después de las consultas anteriores)
SELECT
    indexrelname  AS nombre_indice,
    relname       AS tabla,
    idx_scan      AS veces_usado,
    idx_tup_read  AS tuplas_leidas,
    idx_tup_fetch AS tuplas_obtenidas
FROM pg_stat_user_indexes
WHERE schemaname = 'app'
ORDER BY idx_scan DESC;
```

#### Salida esperada (fragmento — Index Scan esperado)

```
Index Scan using idx_customers_email on customers
  (cost=0.42..8.44 rows=1 width=42)
  (actual time=0.041..0.043 rows=1 loops=1)
  Index Cond: (email = 'user_12345@example.com'::text)
  Buffers: shared hit=4
Planning Time: 0.198 ms
Execution Time: 0.067 ms
```

> 📌 **Observa:** El tiempo bajó de ~18.5 ms a ~0.07 ms. El `Seq Scan` fue reemplazado por `Index Scan`. Los `Buffers` pasaron de 791 a 4.

#### Verificación

```sql
-- Confirmar existencia de todos los índices creados
SELECT
    indexname,
    indexdef
FROM pg_indexes
WHERE schemaname = 'app'
ORDER BY tablename, indexname;

-- Actualizar tabla de baseline con los nuevos tiempos
UPDATE public.lab_baseline
SET execution_ms = 0.067, plan_type = 'Index Scan'
WHERE descripcion = 'email lookup sin índice';
```

---

### Paso 5: Gestionar estadísticas del optimizador con ANALYZE y pg_statistic

**Objetivo:** Comprender cómo las estadísticas del optimizador afectan la selección de planes y cómo forzar su actualización para obtener estimaciones precisas.

#### Instrucciones

1. Inspecciona las estadísticas actuales de la columna `status` en `orders`:

```sql
\c labdb

-- Ver estadísticas de la columna status (histograma y MCVs)
SELECT
    attname         AS columna,
    n_distinct,
    correlation,
    most_common_vals,
    most_common_freqs
FROM pg_stats
WHERE schemaname = 'app'
  AND tablename  = 'orders'
  AND attname    = 'status';
```

2. Aumenta el `statistics target` para columnas con alta cardinalidad:

```sql
-- Incrementar target para mejor estimación en total_amount
ALTER TABLE app.orders
    ALTER COLUMN total_amount SET STATISTICS 500;

-- Re-ejecutar ANALYZE para recalcular con el nuevo target
ANALYZE VERBOSE app.orders;
```

3. Verifica el impacto en las estimaciones del planificador:

```sql
-- Comparar estimación vs. filas reales
EXPLAIN (ANALYZE, FORMAT TEXT)
SELECT *
FROM app.orders
WHERE total_amount BETWEEN 100 AND 200;
```

4. Examina la frecuencia de nulos y valores más comunes:

```sql
SELECT
    attname            AS columna,
    null_frac          AS fraccion_nulos,
    avg_width          AS ancho_promedio_bytes,
    n_distinct         AS valores_distintos,
    most_common_vals   AS valores_frecuentes,
    most_common_freqs  AS frecuencias
FROM pg_stats
WHERE schemaname = 'app'
  AND tablename  = 'customers'
ORDER BY attname;
```

#### Salida esperada (fragmento)

```
 columna  | fraccion_nulos | ancho_promedio_bytes | valores_distintos
----------+----------------+----------------------+------------------
 country_code |       0    |                    2 |                5
 email        |       0    |                   17 |           -0.9998
 is_active    |       0    |                    1 |                2
```

#### Verificación

```sql
-- Confirmar que el statistics target fue actualizado
SELECT
    attname,
    attstattarget
FROM pg_attribute
WHERE attrelid = 'app.orders'::regclass
  AND attname  = 'total_amount';
```

```
   attname    | attstattarget
--------------+---------------
 total_amount |           500
```

---

### Paso 6: Implementar roles y privilegios con mínimo privilegio

**Objetivo:** Crear la jerarquía de roles definida en la lección 1.1, aplicar `ALTER DEFAULT PRIVILEGES`, habilitar RLS básico y verificar los accesos con las vistas del catálogo.

#### Instrucciones

**6.1 — Crear roles de grupo y usuarios**

```sql
\c labdb

-- Roles de grupo (sin LOGIN): definen perfiles de acceso
CREATE ROLE app_ro NOLOGIN;
CREATE ROLE app_rw NOLOGIN;

-- Usuarios operativos (con LOGIN)
CREATE ROLE api_user  LOGIN PASSWORD 'Api_S3guro_2024!'  VALID UNTIL 'infinity';
CREATE ROLE analyst   LOGIN PASSWORD 'An4l!t1cs_2024!'   VALID UNTIL 'infinity';
CREATE ROLE etl_user  LOGIN PASSWORD 'Etl_S3guro_2024!'  VALID UNTIL 'infinity';

-- Asignar membresías
GRANT app_ro TO analyst;         -- analyst hereda solo lectura
GRANT app_rw TO api_user;        -- api_user hereda lectura + escritura
GRANT app_ro TO etl_user;        -- etl_user: lectura base, escritura puntual
```

**6.2 — Otorgar privilegios sobre esquema y objetos existentes**

```sql
-- Acceso al esquema
GRANT USAGE ON SCHEMA app TO app_ro, app_rw;

-- Privilegios sobre objetos existentes
GRANT SELECT ON ALL TABLES IN SCHEMA app TO app_ro, app_rw;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_rw;

-- Secuencias (necesarias para INSERT con columnas IDENTITY)
GRANT USAGE, SELECT, UPDATE ON ALL SEQUENCES IN SCHEMA app TO app_rw;
```

**6.3 — Configurar DEFAULT PRIVILEGES para objetos futuros**

```sql
-- Asegurar que nuevas tablas y secuencias creadas por app_owner
-- reciban automáticamente los privilegios correctos
ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA app
    GRANT SELECT ON TABLES TO app_ro, app_rw;

ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA app
    GRANT INSERT, UPDATE, DELETE ON TABLES TO app_rw;

ALTER DEFAULT PRIVILEGES FOR ROLE app_owner IN SCHEMA app
    GRANT USAGE, SELECT, UPDATE ON SEQUENCES TO app_rw;
```

**6.4 — Habilitar Row Level Security en la tabla orders**

```sql
-- Habilitar RLS
ALTER TABLE app.orders ENABLE ROW LEVEL SECURITY;

-- Política de lectura: cada usuario ve solo sus propias órdenes
CREATE POLICY orders_select_policy ON app.orders
    FOR SELECT
    TO app_ro, app_rw
    USING (created_by = current_user);

-- Política de escritura: solo puede modificar sus propias órdenes
CREATE POLICY orders_modify_policy ON app.orders
    FOR ALL
    TO app_rw
    USING (created_by = current_user)
    WITH CHECK (created_by = current_user);

-- El usuario master (rds_superuser) necesita BYPASSRLS implícito
-- Para pruebas de administración, se puede deshabilitar temporalmente:
-- ALTER TABLE app.orders FORCE ROW LEVEL SECURITY; -- activa para el owner también
```

**6.5 — Verificar la estructura de roles y privilegios**

```sql
-- Atributos de todos los roles del laboratorio
SELECT
    rolname,
    rolcanlogin   AS puede_login,
    rolinherit    AS hereda_privilegios,
    rolcreaterole AS puede_crear_roles,
    rolcreatedb   AS puede_crear_bd
FROM pg_roles
WHERE rolname IN ('app_ro','app_rw','api_user','analyst','etl_user','app_owner')
ORDER BY rolname;

-- Membresías (qué roles pertenecen a qué grupos)
SELECT
    r.rolname  AS grupo,
    m.rolname  AS miembro
FROM pg_auth_members am
JOIN pg_roles r ON r.oid = am.roleid
JOIN pg_roles m ON m.oid = am.member
WHERE r.rolname IN ('app_ro','app_rw')
ORDER BY grupo, miembro;

-- Privilegios efectivos por tabla
SELECT
    table_schema,
    table_name,
    grantee,
    privilege_type,
    is_grantable
FROM information_schema.table_privileges
WHERE table_schema = 'app'
ORDER BY table_name, grantee, privilege_type;
```

#### Salida esperada (pg_roles)

```
  rolname   | puede_login | hereda_privilegios | puede_crear_roles | puede_crear_bd
------------+-------------+--------------------+-------------------+----------------
 analyst    | t           | t                  | f                 | f
 api_user   | t           | t                  | f                 | f
 app_owner  | f           | t                  | f                 | f
 app_ro     | f           | t                  | f                 | f
 app_rw     | f           | t                  | f                 | f
 etl_user   | t           | t                  | f                 | f
```

#### Verificación

```sql
-- Probar acceso de analyst (solo lectura)
SET ROLE analyst;
SELECT COUNT(*) FROM app.customers;   -- debe funcionar
-- INSERT INTO app.customers ... -- debe fallar con "permission denied"

RESET ROLE;

-- Probar que api_user puede insertar
SET ROLE api_user;
INSERT INTO app.products (sku, category, price, stock)
VALUES ('SKU-TEST-001', 'Test', 9.99, 10);   -- debe funcionar

RESET ROLE;

-- Limpiar registro de prueba
DELETE FROM app.products WHERE sku = 'SKU-TEST-001';
```

---

### Paso 7: Simular y detectar slow queries en los logs

**Objetivo:** Generar consultas deliberadamente lentas para confirmar que `log_min_duration_statement` las registra correctamente, y revisar los logs desde la consola AWS.

#### Instrucciones

1. Genera una consulta lenta de forma controlada:

```sql
\c labdb

-- Consulta sin índice en columna full_name (fuerza Seq Scan lento)
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id, total_amount
FROM app.orders o
JOIN app.customers c ON o.customer_id = c.customer_id
WHERE c.full_name LIKE '%Customer 4999%'
  AND o.status IN ('pending', 'processing');
```

2. Fuerza una espera artificial para garantizar que supera el umbral de 100 ms:

```sql
-- Consulta con pg_sleep para garantizar registro en logs
SELECT pg_sleep(0.2), COUNT(*)
FROM app.order_items
WHERE unit_price > 400;
```

3. Revisa los logs de Aurora desde AWS CLI:

```bash
# Listar archivos de log disponibles
aws rds describe-db-log-files \
    --db-instance-identifier "${AURORA_CLUSTER_ID}-instance-1" \
    --region "$AWS_REGION" \
    --query 'DescribeDBLogFiles[*].{Archivo:LogFileName,Tamaño:Size}' \
    --output table

# Descargar las últimas líneas del log de PostgreSQL
aws rds download-db-log-file-portion \
    --db-instance-identifier "${AURORA_CLUSTER_ID}-instance-1" \
    --region "$AWS_REGION" \
    --log-file-name "error/postgresql.log" \
    --starting-token 0 \
    --output text | grep "duration:" | tail -20
```

#### Salida esperada en logs

```
2024-01-15 10:23:45 UTC [1234]: [1-1] user=postgres,db=labdb,app=psql,client=10.0.1.5
  LOG:  duration: 234.567 ms  statement: SELECT pg_sleep(0.2), COUNT(*)
        FROM app.order_items WHERE unit_price > 400;
```

#### Verificación

```bash
# Confirmar que el parámetro está activo y registrando
aws rds download-db-log-file-portion \
    --db-instance-identifier "${AURORA_CLUSTER_ID}-instance-1" \
    --region "$AWS_REGION" \
    --log-file-name "error/postgresql.log" \
    --output text | grep -c "duration:" | \
    xargs -I{} echo "Slow queries registradas en log: {}"
```

---

## Validación y Pruebas

Ejecuta el siguiente bloque de validación completo para confirmar que todos los objetivos del laboratorio se cumplieron:

```sql
\c labdb

-- ============================================================
-- VALIDACIÓN COMPLETA DEL LABORATORIO 01-00-01
-- ============================================================

-- 1. Verificar tablas creadas con datos
SELECT 'TABLAS' AS categoria,
       tablename,
       n_live_tup AS filas
FROM pg_stat_user_tables
WHERE schemaname = 'app'
ORDER BY tablename;

-- 2. Verificar índices creados
SELECT 'ÍNDICES' AS categoria,
       indexname,
       tablename
FROM pg_indexes
WHERE schemaname = 'app'
  AND indexname LIKE 'idx_%'
ORDER BY tablename, indexname;

-- 3. Verificar roles creados
SELECT 'ROLES' AS categoria,
       rolname,
       CASE WHEN rolcanlogin THEN 'LOGIN' ELSE 'GRUPO' END AS tipo
FROM pg_roles
WHERE rolname IN ('app_ro','app_rw','api_user','analyst','etl_user','app_owner')
ORDER BY rolname;

-- 4. Verificar RLS habilitado
SELECT 'RLS' AS categoria,
       tablename,
       rowsecurity AS rls_habilitado
FROM pg_tables
WHERE schemaname = 'app'
  AND tablename  = 'orders';

-- 5. Verificar políticas RLS
SELECT 'POLÍTICAS RLS' AS categoria,
       policyname,
       tablename,
       cmd
FROM pg_policies
WHERE schemaname = 'app'
ORDER BY policyname;

-- 6. Verificar statistics target actualizado
SELECT 'STATISTICS TARGET' AS categoria,
       attname AS columna,
       attstattarget AS target
FROM pg_attribute
WHERE attrelid    = 'app.orders'::regclass
  AND attstattarget > 0;
```

**Resultado esperado de validación:**

| Categoría         | Elemento                  | Estado        |
|-------------------|---------------------------|---------------|
| TABLAS            | customers (50,000 filas)  | ✅ OK         |
| TABLAS            | orders (200,000 filas)    | ✅ OK         |
| TABLAS            | order_items (~600,000)    | ✅ OK         |
| TABLAS            | products (1,000 filas)    | ✅ OK         |
| ÍNDICES           | idx_customers_email       | ✅ OK         |
| ÍNDICES           | idx_orders_status_created | ✅ OK         |
| ÍNDICES           | idx_orders_active         | ✅ OK         |
| ÍNDICES           | idx_orders_covering       | ✅ OK         |
| ROLES             | app_ro, app_rw (grupos)   | ✅ OK         |
| ROLES             | api_user, analyst         | ✅ OK         |
| RLS               | orders (habilitado)       | ✅ OK         |
| POLÍTICAS RLS     | orders_select_policy      | ✅ OK         |
| STATISTICS TARGET | total_amount = 500        | ✅ OK         |

---

## Resolución de Problemas

### Problema 1: El índice no es utilizado por el planificador (Seq Scan persiste)

**Síntoma:** Después de crear un índice con `CREATE INDEX`, `EXPLAIN ANALYZE` sigue mostrando `Seq Scan` en lugar de `Index Scan`, incluso para consultas que deberían beneficiarse del índice.

**Causa:** El planificador de PostgreSQL decide no usar el índice cuando estima que el costo del escaneo secuencial es menor. Esto ocurre principalmente en tres situaciones: (a) las estadísticas de la tabla están desactualizadas y el planificador subestima la selectividad de la consulta, (b) la tabla es pequeña y el Seq Scan es genuinamente más eficiente, o (c) el parámetro `random_page_cost` está configurado muy alto para el almacenamiento SSD/NVMe de Aurora.

**Solución:**

```sql
-- Paso 1: Actualizar estadísticas
ANALYZE VERBOSE app.orders;

-- Paso 2: Verificar que el índice existe y no está en construcción
SELECT indexname, indexdef
FROM pg_indexes
WHERE schemaname = 'app' AND tablename = 'orders';

-- Paso 3: Ajustar random_page_cost para Aurora (almacenamiento de baja latencia)
-- Aurora usa almacenamiento distribuido similar a SSD, valor recomendado: 1.1
SET random_page_cost = 1.1;
SET effective_cache_size = '6GB';  -- ajustar según RAM de la instancia

-- Paso 4: Re-ejecutar EXPLAIN para confirmar uso del índice
EXPLAIN (ANALYZE, BUFFERS)
SELECT order_id FROM app.orders WHERE status = 'pending';

-- Paso 5 (diagnóstico adicional): Forzar uso del índice para confirmar que funciona
SET enable_seqscan = off;
EXPLAIN (ANALYZE) SELECT order_id FROM app.orders WHERE status = 'pending';
SET enable_seqscan = on;  -- SIEMPRE restaurar en producción
```

> ⚠️ **Importante:** `SET enable_seqscan = off` es solo para diagnóstico. Nunca lo dejes desactivado en producción. Si el índice tampoco mejora el rendimiento con `enable_seqscan = off`, el problema puede ser que la consulta retorna demasiadas filas (baja selectividad) y el Seq Scan es realmente más eficiente.

---

### Problema 2: Error "permission denied for table" al conectar con usuario de aplicación

**Síntoma:** Al conectarse con `api_user` o `analyst` y ejecutar `SELECT * FROM app.customers`, se obtiene el error: `ERROR: permission denied for table customers` o `ERROR: permission denied for schema app`.

**Causa:** Los `GRANT` ejecutados en el Paso 6 solo aplican a objetos existentes en el momento de la ejecución. Si las tablas se crearon después de los `GRANT`, o si el `ALTER DEFAULT PRIVILEGES` no se configuró correctamente para el rol `app_owner`, los nuevos objetos no tendrán los privilegios necesarios. También puede ocurrir si el `GRANT USAGE ON SCHEMA app` no se ejecutó antes de los `GRANT` sobre tablas.

**Solución:**

```sql
-- Conectarse como usuario master (postgres)
\c labdb postgres

-- Paso 1: Verificar si el esquema tiene USAGE otorgado
SELECT grantee, privilege_type
FROM information_schema.role_usage_grants
WHERE object_schema = 'app';

-- Paso 2: Si falta, otorgar USAGE en el esquema
GRANT USAGE ON SCHEMA app TO app_ro, app_rw;

-- Paso 3: Verificar privilegios sobre las tablas específicas
SELECT table_name, grantee, privilege_type
FROM information_schema.table_privileges
WHERE table_schema = 'app' AND grantee IN ('app_ro', 'app_rw')
ORDER BY table_name, grantee;

-- Paso 4: Re-aplicar GRANTs sobre todos los objetos existentes
GRANT SELECT ON ALL TABLES IN SCHEMA app TO app_ro, app_rw;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_rw;
GRANT USAGE, SELECT, UPDATE ON ALL SEQUENCES IN SCHEMA app TO app_rw;

-- Paso 5: Verificar DEFAULT PRIVILEGES configurados
SELECT pg_get_userbyid(defaclrole) AS rol_propietario,
       defaclobjtype               AS tipo_objeto,
       defaclacl                   AS acl_por_defecto
FROM pg_default_acl
WHERE defaclnamespace = 'app'::regnamespace;

-- Paso 6: Probar el acceso nuevamente
SET ROLE analyst;
SELECT COUNT(*) FROM app.customers;
RESET ROLE;
```

> 📝 **Nota:** El orden importa: siempre otorga `USAGE ON SCHEMA` antes de `GRANT` sobre tablas. Sin acceso al esquema, el usuario no puede resolver los nombres de los objetos, independientemente de los privilegios sobre las tablas.

---

## Limpieza de Recursos

Ejecuta los siguientes comandos para limpiar los recursos creados en este laboratorio y evitar costos innecesarios.

### Limpieza de objetos de base de datos

```sql
-- Conectarse como usuario master
\c labdb postgres

-- Revocar privilegios antes de eliminar roles
REVOKE ALL ON ALL TABLES IN SCHEMA app FROM app_ro, app_rw;
REVOKE ALL ON ALL SEQUENCES IN SCHEMA app FROM app_ro, app_rw;
REVOKE USAGE ON SCHEMA app FROM app_ro, app_rw;

-- Eliminar políticas RLS
DROP POLICY IF EXISTS orders_select_policy ON app.orders;
DROP POLICY IF EXISTS orders_modify_policy ON app.orders;

-- Eliminar roles de usuario (primero los miembros, luego los grupos)
DROP ROLE IF EXISTS analyst;
DROP ROLE IF EXISTS api_user;
DROP ROLE IF EXISTS etl_user;
DROP ROLE IF EXISTS app_ro;
DROP ROLE IF EXISTS app_rw;

-- Eliminar esquema completo (incluye todas las tablas e índices)
DROP SCHEMA app CASCADE;

-- Conectarse a postgres para eliminar labdb
\c postgres

DROP DATABASE IF EXISTS labdb;

-- Eliminar tabla de baseline
DROP TABLE IF EXISTS public.lab_baseline;
```

### Restaurar parámetros del Parameter Group

```bash
# Restaurar log_min_duration_statement al valor por defecto (-1 = deshabilitado)
aws rds modify-db-cluster-parameter-group \
    --db-cluster-parameter-group-name "$AURORA_PARAM_GROUP" \
    --region "$AWS_REGION" \
    --parameters \
      "ParameterName=log_min_duration_statement,ParameterValue=-1,ApplyMethod=immediate"

# Verificar restauración
aws rds describe-db-cluster-parameters \
    --db-cluster-parameter-group-name "$AURORA_PARAM_GROUP" \
    --region "$AWS_REGION" \
    --query "Parameters[?ParameterName=='log_min_duration_statement'].{Nombre:ParameterName,Valor:ParameterValue}" \
    --output table
```

### Limpieza con Terraform (si se usó para provisionar)

```bash
cd <directorio-terraform-modulo-1>
terraform destroy -auto-approve
```

> ✅ **Confirma** que la instancia Aurora aparece como "deleted" en la consola AWS antes de cerrar la sesión para evitar cargos adicionales.

---

## Resumen

En este laboratorio aplicaste los conceptos fundamentales de rendimiento y seguridad en Aurora PostgreSQL:

| Objetivo                             | Técnica Aplicada                                      | Resultado                              |
|--------------------------------------|-------------------------------------------------------|----------------------------------------|
| Captura de slow queries              | `log_min_duration_statement = 100ms`                  | Consultas >100ms registradas en logs   |
| Línea base de rendimiento            | `EXPLAIN ANALYZE` sin índices                         | Seq Scan identificado como cuello      |
| Optimización con índices B-tree      | Simple, compuesto, parcial y de cobertura             | Index Scan: reducción de ~18ms a ~0.07ms |
| Estadísticas del optimizador         | `ANALYZE` + `SET STATISTICS 500`                      | Estimaciones más precisas del planificador |
| Modelo de roles y mínimo privilegio  | `CREATE ROLE`, `GRANT`, `DEFAULT PRIVILEGES`          | Separación owner/operativo verificada  |
| Row Level Security                   | `ENABLE ROW LEVEL SECURITY` + `CREATE POLICY`         | Aislamiento a nivel de fila por usuario |

### Conceptos clave consolidados

- **`log_min_duration_statement`** es el punto de partida para cualquier análisis de rendimiento; establece el umbral de captura sin saturar los logs.
- **El orden de `EXPLAIN ANALYZE`** para diagnosticar es: identificar el nodo más costoso → verificar estimación vs. realidad → crear índice si hay alta selectividad → re-analizar.
- **La separación de ownership** (`app_owner` sin LOGIN) de los roles operativos es una práctica de seguridad crítica en entornos Aurora.
- **`ALTER DEFAULT PRIVILEGES`** cierra el "agujero" de privilegios para objetos futuros; sin él, cada nueva tabla requiere GRANT manual.
- **RLS** añade una capa de seguridad que el código de aplicación no puede eludir, ideal para entornos multi-tenant.

### Recursos adicionales

- [PostgreSQL 14 — EXPLAIN](https://www.postgresql.org/docs/14/sql-explain.html)
- [PostgreSQL 14 — Tipos de índices](https://www.postgresql.org/docs/14/indexes-types.html)
- [PostgreSQL 14 — Roles y privilegios](https://www.postgresql.org/docs/14/ddl-priv.html)
- [Aurora PostgreSQL — Mejores prácticas de rendimiento](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.BestPractices.html)
- [Aurora PostgreSQL — Parameter Groups](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_WorkingWithParamGroups.html)
- [pg_stat_user_indexes — Referencia](https://www.postgresql.org/docs/14/monitoring-stats.html#MONITORING-PG-STAT-ALL-INDEXES-VIEW)

---
*Lab 01-00-01 — Módulo 1: Fundamentos de rendimiento en Aurora PostgreSQL*
