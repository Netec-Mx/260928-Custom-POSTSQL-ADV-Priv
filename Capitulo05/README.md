# 4 Diseño de un Data Warehouse en PostgreSQL

## Metadatos

| Métrica | Valor |
| :--- | :--- |
| **Duración** | 60 minutos |
| **Complejidad** | Media |
| **Nivel Bloom** | Crear |

---

## Descripción General

En este laboratorio, diseñarás e implementarás un Data Warehouse analítico completo llamado `dwh_enterprise` utilizando PostgreSQL 16.2. Desarrollarás un modelo multidimensional clásico (Esquema de Estrella / Star Schema) compuesto por una tabla de hechos de ventas masivas (`fact_sales`) y tres tablas de dimensiones (`dim_products`, `dim_customers`, `dim_time`). 

Para optimizar el rendimiento de este entorno frente a millones de filas, aplicarás técnicas avanzadas de infraestructura física y lógica de base de datos: particionamiento declarativo mensual por rangos de fecha, separación física de datos calientes y fríos mediante el uso de Tablespaces simulados dentro del contenedor Docker, e indexación especializada utilizando índices de rango de bloques (BRIN). Finalmente, realizarás una carga masiva sintética de 1,000,000 de registros para analizar y validar de forma cuantificable la eficiencia de la poda de particiones (partition pruning).

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Diseñar e implementar un esquema de estrella optimizado para análisis OLAP en PostgreSQL.
- [ ] Configurar el particionamiento declarativo por rango en tablas de hechos analíticas de gran volumen.
- [ ] Implementar y simular Tablespaces en contenedores Docker para separar físicamente los datos por temperatura (caliente/frío).
- [ ] Crear y evaluar el rendimiento de índices BRIN (*Block Range Indexes*) en datos correlacionados temporalmente.
- [ ] Auditar y comprobar la activación de la poda de particiones (*partition pruning*) mediante planes de ejecución (`EXPLAIN ANALYZE`).

---

## Prerrequisitos

Para completar con éxito este laboratorio, requieres:
1. **Conocimientos teóricos**: Comprensión del modelo dimensional (hechos, dimensiones, claves subrogadas) y conceptos básicos de optimización física en PostgreSQL.
2. **Acceso al sistema**: Permisos de administrador en tu estación de trabajo para ejecutar comandos de terminal y administrar contenedores Docker.
3. **Herramientas de red**: Asegurar que no existan restricciones de firewall para levantar puertos en el rango `5000-6500` de la máquina anfitriona.

---

## Entorno de Laboratorio

### Requisitos de Hardware

| Componente | Requisito Mínimo | Requisito Recomendado |
| :--- | :--- | :--- |
| **Procesador (CPU)** | Arquitectura x86_64 (mínimo 8 núcleos físicos) | Intel Core i7/i9 o AMD Ryzen 7/9 |
| **Memoria RAM** | 16 GB libres | 32 GB libres |
| **Almacenamiento** | 100 GB en disco SSD (> 500 MB/s de lectura) | 100 GB en disco SSD NVMe dedicado |

### Requisitos de Software

| Software | Edición / Versión Exacta | Origen / Enlace Oficial |
| :--- | :--- | :--- |
| **PostgreSQL** | 16.2 Community Edition (Docker) | [PostgreSQL Hub](https://hub.docker.com/_/postgres) |
| **Docker Engine** | v26.0.0 o superior | [Docker Releases](https://docs.docker.com/engine/release-notes/26.0/) |
| **DBeaver Community** | v24.0.0 o superior | [DBeaver Downloads](https://dbeaver.io/download/) |

### Inicialización del Entorno de Red y Contenedor

Ejecuta los siguientes comandos en la terminal de tu sistema operativo para configurar la red compartida y levantar el contenedor PostgreSQL maestro (`pg-primary`) sobre el cual se construirá el Data Warehouse.

```bash
## 1. Crear la red Docker de grado empresarial si no existe
docker network create pg_enterprise_net || true

## 2. Levantar el contenedor de base de datos PostgreSQL 16.2
docker run -d \
  --name pg-primary \
  --net pg_enterprise_net \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD='PostgresAdminPass123!' \
  postgres:16.2-bookworm
```

---

## Instrucciones Paso a Paso

### Paso 1: Configuración Física del Contenedor para Tablespaces

En arquitecturas corporativas reales, los datos calientes (mes actual) se escriben en discos ultra-rápidos (NVMe SSD), mientras que los datos históricos (años anteriores) se relegan a almacenamiento magnético lento de bajo costo. En PostgreSQL, esto se gestiona mediante *Tablespaces*. Para simular este comportamiento de forma local en Docker, debes crear directorios dedicados en el sistema de archivos del contenedor y asociar los permisos de propiedad al usuario del sistema `postgres`.

**Instrucciones**:

1. Abre tu terminal de línea de comandos y ejecuta el comando para crear dos directorios físicos dentro del contenedor `pg-primary`. Uno simulará un almacenamiento de estado sólido ultra-rápido (`ts_hot`) y otro un disco duro convencional de archivado (`ts_cold`):
   ```bash
   docker exec -u 0 -it pg-primary mkdir -p /var/lib/postgresql/data/ts_hot /var/lib/postgresql/data/ts_cold
   ```

2. Asigna la propiedad de dichos directorios al usuario de sistema de PostgreSQL (`postgres:postgres`) dentro del contenedor de Linux:
   ```bash
   docker exec -u 0 -it pg-primary chown -R postgres:postgres /var/lib/postgresql/data/ts_hot /var/lib/postgresql/data/ts_cold
   ```

3. Conéctate a la base de datos maestra inicial por defecto utilizando la herramienta `psql` incorporada dentro del contenedor:
   ```bash
   docker exec -it pg-primary psql -U postgres -d postgres
   ```

4. Ejecuta los siguientes comandos SQL en la consola interactiva para registrar los tablespaces lógicos en el motor y crear la base de datos analítica `dwh_enterprise` asignada por defecto a nuestro tablespace rápido:
   ```sql
   -- Registrar tablespaces en el motor asociando la ubicación física interna
   CREATE TABLESPACE ts_hot LOCATION '/var/lib/postgresql/data/ts_hot';
   CREATE TABLESPACE ts_cold LOCATION '/var/lib/postgresql/data/ts_cold';

   -- Crear la base de datos analítica apuntando al tablespace caliente
   CREATE DATABASE dwh_enterprise TABLESPACE ts_hot;
   ```

5. Sal de la sesión actual de `psql` para prepararte para el siguiente paso:
   ```sql
   \q
   ```

**Resultado esperado**: Los directorios físicos se han creado con los permisos correctos en el contenedor Debian subyacente. Los comandos de creación de tablespaces y base de datos en PostgreSQL devuelven confirmaciones de éxito:
```text
CREATE TABLESPACE
CREATE TABLESPACE
CREATE DATABASE
```

**Verificación**: Ejecuta el siguiente comando rápido desde la terminal para asegurarte de que los tablespaces están correctamente registrados en el catálogo del sistema:
```bash
docker exec -it pg-primary psql -U postgres -d postgres -c "\db"
```

---

### Paso 2: Creación del Modelo de Dimensiones en el DWH

Un modelo dimensional robusto requiere tablas de dimensiones estructuradas con llaves subrogadas de tipo entero (que agilizan los cruces de tablas o *joins*), restricciones explícitas de no nulidad para asegurar consistencia y tipos de datos eficientes.

**Instrucciones**:

1. Conéctate directamente a la nueva base de datos analítica `dwh_enterprise` que creaste en el paso anterior:
   ```bash
   docker exec -it pg-primary psql -U postgres -d dwh_enterprise
   ```

2. Ejecuta el script DDL para construir la dimensión de productos (`dim_products`), la dimensión de clientes (`dim_customers`) y la dimensión temporal (`dim_time`). Esta última es un estándar absoluto en DWH para evitar cálculos matemáticos de extracción de fecha en caliente durante las consultas analíticas de agregación:
   ```sql
   -- Crear Dimensión de Clientes (Soportando demografía básica)
   CREATE TABLE dim_customers (
       customer_key SERIAL PRIMARY KEY,
       customer_id VARCHAR(50) NOT NULL,
       name VARCHAR(150) NOT NULL,
       segment VARCHAR(50) NOT NULL,
       city VARCHAR(100) NOT NULL,
       country VARCHAR(100) NOT NULL,
       registration_date DATE NOT NULL,
       CONSTRAINT uq_customer_id UNIQUE (customer_id)
   );

   -- Crear Dimensión de Productos
   CREATE TABLE dim_products (
       product_key SERIAL PRIMARY KEY,
       product_id VARCHAR(50) NOT NULL,
       name VARCHAR(200) NOT NULL,
       category VARCHAR(100) NOT NULL,
       brand VARCHAR(100) NOT NULL,
       list_price NUMERIC(12,2) NOT NULL,
       CONSTRAINT uq_product_id UNIQUE (product_id)
   );

   -- Crear Dimensión de Tiempo (Clave de negocio int con formato AAAAMMDD)
   CREATE TABLE dim_time (
       time_key INT PRIMARY KEY,
       datum DATE NOT NULL,
       year INT NOT NULL,
       quarter INT NOT NULL,
       month INT NOT NULL,
       month_name VARCHAR(20) NOT NULL,
       day INT NOT NULL,
       day_name VARCHAR(20) NOT NULL,
       is_weekend BOOLEAN NOT NULL,
       CONSTRAINT uq_datum UNIQUE (datum)
   );
   ```

**Resultado esperado**: Las tres tablas dimensionales han sido instanciadas sin errores en la base de datos.

**Verificación**: Confirma la existencia y estructura de las tablas creadas listándolas con el metacomando `\dt`:
```sql
\dt
```
La salida en consola debe listar las tablas `dim_customers`, `dim_products` y `dim_time`.

---

### Paso 3: Diseño de la Tabla de Hechos Particionada y Asignación de Tablespaces

En PostgreSQL, una tabla particionada declarativamente no almacena datos por sí misma; actúa como una plantilla o interfaz lógica. Debes definir un esquema de particionamiento al momento de la creación y, de manera separada, crear las particiones físicas asociadas. En este paso, crearemos la tabla de hechos `fact_sales` particionada por rangos mensuales de la fecha de venta (`sale_date`).

> **Regla de Ingeniería de PostgreSQL**: Cualquier clave primaria (`PRIMARY KEY`) o restricción única (`UNIQUE`) que se defina sobre una tabla particionada **debe incluir obligatoriamente** la columna que sirve como clave de partición. Por tanto, nuestra clave primaria compuesta será `(sale_id, sale_date)`.

**Instrucciones**:

1. En la misma consola interactiva de `psql` sobre la base de datos `dwh_enterprise`, crea la tabla padre de hechos `fact_sales` indicando el criterio de particionamiento por rango:
   ```sql
   CREATE TABLE fact_sales (
       sale_id BIGINT NOT NULL,
       customer_key INT NOT NULL,
       product_key INT NOT NULL,
       time_key INT NOT NULL,
       quantity INT NOT NULL,
       unit_price NUMERIC(12,2) NOT NULL,
       discount NUMERIC(12,2) NOT NULL,
       total_amount NUMERIC(12,2) NOT NULL,
       sale_date TIMESTAMP WITHOUT TIME ZONE NOT NULL,
       PRIMARY KEY (sale_id, sale_date)
   ) PARTITION BY RANGE (sale_date);
   ```

2. Crea las particiones físicas específicas para el primer trimestre de 2024. Para simular el ciclo de vida del almacenamiento empresarial, colocaremos las particiones de Enero y Febrero en nuestro almacenamiento caliente (`ts_hot`) y la partición de Marzo en nuestro almacenamiento frío (`ts_cold`):
   ```sql
   -- Partición Enero 2024 (Enviada a Tablespace Caliente)
   CREATE TABLE fact_sales_2024_m01 PARTITION OF fact_sales
       FOR VALUES FROM ('2024-01-01 00:00:00') TO ('2024-02-01 00:00:00')
       TABLESPACE ts_hot;

   -- Partición Febrero 2024 (Enviada a Tablespace Caliente)
   CREATE TABLE fact_sales_2024_m02 PARTITION OF fact_sales
       FOR VALUES FROM ('2024-02-01 00:00:00') TO ('2024-03-01 00:00:00')
       TABLESPACE ts_hot;

   -- Partición Marzo 2024 (Enviada a Tablespace Frío para emular archivado histórico)
   CREATE TABLE fact_sales_2024_m03 PARTITION OF fact_sales
       FOR VALUES FROM ('2024-03-01 00:00:00') TO ('2024-04-01 00:00:00')
       TABLESPACE ts_cold;
   ```

**Resultado esperado**: La tabla maestra y las tres particiones se configuran de manera satisfactoria.

**Verificación**: Ejecuta el siguiente comando para revisar la jerarquía física de la tabla de hechos, observando qué particiones pertenecen a qué tablespaces:
```sql
\d+ fact_sales
```
La terminal mostrará el listado de particiones asociadas con sus respectivos límites de rangos lógicos y la columna de tablespace mostrando `ts_hot` y `ts_cold` según corresponda.

---

### Paso 4: Implementación de Índices Híbridos para Cargas Masivas (BRIN y B-Tree)

En un Data Warehouse con miles de millones de filas, los índices tradicionales B-Tree sobre la fecha representan un consumo excesivo de memoria RAM y un espacio en disco prohibitivo. En su lugar, utilizaremos un índice BRIN (*Block Range Index*) para la columna temporal `sale_date`. 

Un índice BRIN no almacena una lista exhaustiva de punteros a filas específicas; en cambio, almacena solo el rango mínimo y máximo de valores que existen dentro de un bloque de páginas físicas contiguas en el disco (en este caso definiremos un rango de 128 páginas de disco por bloque). Esto es ideal para fechas de ventas que se escriben en orden cronológico constante.

**Instrucciones**:

1. Crea el índice BRIN sobre la tabla padre `fact_sales` en la columna de partición temporal. El planificador de PostgreSQL aplicará este índice automáticamente a todas las particiones hijas creadas en el Paso 3, así como a las futuras:
   ```sql
   CREATE INDEX idx_fact_sales_date_brin 
   ON fact_sales USING brin (sale_date) 
   WITH (pages_per_range = 128);
   ```

2. Crea índices tradicionales B-Tree en las claves de las dimensiones sobre la tabla de hechos. Dado que estas claves foráneas de dimensiones contienen valores dispersos y aleatorios sin un orden físico en disco, el índice B-Tree es la opción correcta para agilizar los cruces de tablas de dimensiones específicas:
   ```sql
   CREATE INDEX idx_fact_sales_cust_btree ON fact_sales (customer_key);
   CREATE INDEX idx_fact_sales_prod_btree ON fact_sales (product_key);
   ```

**Resultado esperado**: Índices creados exitosamente en la tabla de hechos padre e impactando recursivamente a sus particiones hijas.

**Verificación**: Ejecuta un comando para listar los índices existentes en una de las particiones individuales (por ejemplo, Enero de 2024) y confirma que los índices se heredaron automáticamente de forma local:
```sql
\d fact_sales_2024_m01
```
Debe listarse la clave primaria local y las versiones heredadas de `idx_fact_sales_date_brin`, `idx_fact_sales_cust_btree` y `idx_fact_sales_prod_btree`.

---

### Paso 5: Carga Masiva y Generación de Datos de Prueba Consistentes

Para realizar pruebas analíticas de rendimiento reales, poblaremos la base de datos analítica con **10,000 clientes**, **1,000 productos**, el calendario de tiempo del **primer trimestre de 2024** y un total controlado de **1,000,000 de transacciones de ventas** distribuidas linealmente a lo largo de los tres meses mapeados.

**Instrucciones**:

1. En tu consola activa de `psql`, ejecuta la población estructurada de las dimensiones utilizando la función matemática `generate_series`:
   ```sql
   -- Cargar 10,000 Clientes con datos variados de distribución matemática predefinida
   INSERT INTO dim_customers (customer_id, name, segment, city, country, registration_date)
   SELECT 
       'CUST-' || lpad(i::text, 6, '0'),
       'Cliente Corporativo ' || i,
       (ARRAY['Retail', 'B2B', 'Tech', 'Public'])[mod(i, 4) + 1],
       (ARRAY['Bogotá', 'Santiago', 'Lima', 'Ciudad de México'])[mod(i, 4) + 1],
       (ARRAY['Colombia', 'Chile', 'Perú', 'México'])[mod(i, 4) + 1],
       '2023-01-01'::date + (mod(i, 365) * interval '1 day')
   FROM generate_series(1, 10000) AS i;

   -- Cargar 1,000 Productos con precios simulados dinámicos
   INSERT INTO dim_products (product_id, name, category, brand, list_price)
   SELECT 
       'PROD-' || lpad(i::text, 6, '0'),
       'Producto Inteligente ' || i,
       (ARRAY['Electrónica', 'Hogar', 'Moda', 'Deportes'])[mod(i, 4) + 1],
       (ARRAY['Sony', 'Samsung', 'Nike', 'Adidas'])[mod(i, 4) + 1],
       round((10.0 + mod(i, 100) * 4.5)::numeric, 2)
   FROM generate_series(1, 1000) AS i;

   -- Cargar el calendario para el Q1 2024 completo en la dimensión de Tiempo
   INSERT INTO dim_time (time_key, datum, year, quarter, month, month_name, day, day_name, is_weekend)
   SELECT 
       to_char(d, 'YYYYMMDD')::int,
       d::date,
       extract(year from d)::int,
       extract(quarter from d)::int,
       extract(month from d)::int,
       to_char(d, 'TMMonth'),
       extract(day from d)::int,
       to_char(d, 'TMDia'),
       case when extract(isodow from d) in (6, 7) then true else false end
   FROM generate_series('2024-01-01'::date, '2024-03-31'::date, '1 day'::interval) as d;
   ```

2. Ejecuta la carga masiva de **1,000,000 de registros** de transacciones en la tabla de hechos `fact_sales`. Utilizaremos una lógica matemática determinista basada en el índice incremental para generar una progresión lineal de fechas de venta a lo largo de los 90 días del trimestre, asegurando una distribución perfectamente balanceada entre las tres particiones creadas:
   ```sql
   INSERT INTO fact_sales (sale_id, customer_key, product_key, time_key, quantity, unit_price, discount, total_amount, sale_date)
   SELECT 
       i AS sale_id,
       (mod(i, 10000) + 1) AS customer_key,
       (mod(i, 1000) + 1) AS product_key,
       to_char(dt, 'YYYYMMDD')::int AS time_key,
       (mod(i, 5) + 1) AS quantity,
       (15.00 + mod(i, 100) * 2.50)::numeric AS unit_price,
       round((mod(i, 3) * 0.75)::numeric, 2) AS discount,
       -- Total calculado en línea: (Cantidad * Precio unitario) - Descuento
       (((mod(i, 5) + 1) * (15.00 + mod(i, 100) * 2.50)) - (mod(i, 3) * 0.75))::numeric AS total_amount,
       dt AS sale_date
   FROM generate_series(1, 1000000) AS i
   CROSS JOIN LATERAL (
       -- Distribuye de forma uniforme 1 Millón de filas en 7,776,000 segundos (90 días exactos)
       SELECT '2024-01-01 00:00:00'::timestamp + (mod(i, 7776000) * interval '1 second') AS dt
   ) dts;
   ```

3. Actualiza el analizador estadístico del planificador de PostgreSQL para garantizar que el optimizador de consultas conozca el volumen de datos cargados y use el mejor plan analítico disponible:
   ```sql
   ANALYZE VERBOSE fact_sales;
   ANALYZE VERBOSE dim_customers;
   ANALYZE VERBOSE dim_products;
   ANALYZE VERBOSE dim_time;
   ```

**Resultado esperado**: Las dimensiones se pueblan instantáneamente. La inserción de 1,000,000 de registros en la tabla de hechos puede tardar entre 5 y 15 segundos dependiendo de la potencia del procesador de tu máquina, confirmando el total de filas agregadas. El comando `ANALYZE` reportará los análisis físicos completados con éxito.

**Verificación**: Comprueba cuántas filas se almacenaron en cada una de las particiones físicas consultándolas directamente. Al estar distribuidas proporcionalmente en el tiempo, cada partición debe contener un volumen de datos equivalente aproximado:
```sql
SELECT 'Enero' AS mes, count(*) FROM fact_sales_2024_m01
UNION ALL
SELECT 'Febrero' AS mes, count(*) FROM fact_sales_2024_m02
UNION ALL
SELECT 'Marzo' AS mes, count(*) FROM fact_sales_2024_m03;
```
La suma total de registros de las tres particiones consultadas por separado debe coincidir exactamente con `1,000,000`.

---

## Validación y Pruebas

Para garantizar que el Data Warehouse cumple con las especificaciones de alta eficiencia y escalabilidad del diseño planteado, realizaremos dos pruebas críticas de ejecución del planificador analítico utilizando `EXPLAIN (ANALYZE, COSTS OFF)`.

### Caso de Prueba 1: Validación de Poda de Particiones (Partition Pruning)

**Descripción**: Esta prueba valida que PostgreSQL ignore las particiones físicas irrelevantes durante una consulta con filtro temporal, reduciendo drásticamente la lectura de disco (I/O).

1. Ejecuta la siguiente consulta para revisar el plan de ejecución:
   ```sql
   EXPLAIN (ANALYZE, COSTS OFF)
   SELECT 
       c.country,
       sum(f.total_amount) AS ventas_totales
   FROM fact_sales f
   JOIN dim_customers c ON f.customer_key = c.customer_key
   -- Filtro estricto que limita la búsqueda exclusivamente al mes de Enero 2024
   WHERE f.sale_date >= '2024-01-10 00:00:00' AND f.sale_date < '2024-01-20 00:00:00'
   GROUP BY c.country;
   ```

2. Analiza la salida en la terminal. Debe visualizarse una estructura de plan similar a la siguiente:
   ```text
   HashAggregate (actual time=...)
     Group Key: c.country
     ->  Hash Join (actual time=...)
           Hash Cond: (f.customer_key = c.customer_key)
           ->  Append (actual time=...)
                 ->  Seq Scan on fact_sales_2024_m01 f_1 (actual time=...)
                       Filter: ((sale_date >= '2024-01-10 00:00:00'::timestamp without time zone) AND (sale_date < '2024-01-20 00:00:00'::timestamp without time zone))
           ->  Hash (actual time=...)
                 ->  Seq Scan on dim_customers c (actual time=...)
   ```

> **Resultado del Análisis**: Observa que bajo la directiva `Append` de la consulta de hechos, **solo** se escanea la tabla física `fact_sales_2024_m01` (Enero). El planificador ha descartado de manera proactiva los datos de Febrero (`fact_sales_2024_m02`) y Marzo (`fact_sales_2024_m03`), confirmando la correcta operación de la poda de particiones.

---

### Caso de Prueba 2: Prueba Adversaria de Rendimiento (Fallo de Poda)

**Descripción**: En esta prueba simularemos un escenario de mala práctica de desarrollo analítico donde un filtro no inmutable, una conversión implícita ineficiente o un parámetro inconsistente rompen el algoritmo de optimización del motor PostgreSQL, forzando un escaneo completo de todo el histórico de particiones del Data Warehouse (*Sequential Scan* global).

1. Ejecuta la consulta analítica utilizando una función de cálculo dinámico como parámetro de fecha (en este caso forzando la evaluación mediante `COALESCE` sobre variables externas, lo que impide al planificador de PostgreSQL precalcular los límites en la fase de planeación):
   ```sql
   EXPLAIN (ANALYZE, COSTS OFF)
   SELECT 
       sum(f.total_amount) AS ventas_totales
   FROM fact_sales f
   -- Filtramos usando una expresión dinámica no simplificable en fase de optimización
   WHERE f.sale_date >= coalesce(cast('2024-01-15' as timestamp), now());
   ```

2. Analiza el plan de ejecución resultante:
   ```text
   Aggregate (actual time=...)
     ->  Append (actual time=...)
           ->  Seq Scan on fact_sales_2024_m01 f_1 (actual time=...)
                 Filter: (sale_date >= COALESCE('2024-01-15 00:00:00'::timestamp without time zone, now()))
           ->  Seq Scan on fact_sales_2024_m02 f_2 (actual time=...)
                 Filter: (sale_date >= COALESCE('2024-01-15 00:00:00'::timestamp without time zone, now()))
           ->  Seq Scan on fact_sales_2024_m03 f_3 (actual time=...)
                 Filter: (sale_date >= COALESCE('2024-01-15 00:00:00'::timestamp without time zone, now()))
   ```

> **Resultado del Análisis**: Debido a que `now()` no es una función inmutable (su valor cambia dinámicamente con la hora exacta del reloj de ejecución) encapsulada dentro de un `COALESCE`, el planificador de PostgreSQL se ve obligado a evaluar el filtro de forma dinámica sobre todas y cada una de las filas individuales. Esto deshabilita la poda estática de particiones, resultando en que las tres tablas hijas de hechos (`f_1`, `f_2`, `f_3`) sean escaneadas secuencialmente una por una, incrementando drásticamente el costo operativo y el consumo de I/O.

---

## Solución de Problemas

A continuación, se listan dos problemas habituales documentados durante la configuración e implementación de esta práctica analítica, con sus respectivas causas y soluciones:

### Problema 1: Fallo de creación de Tablespace dentro de un contenedor Docker
* **Síntoma**: Al ejecutar el comando SQL `CREATE TABLESPACE ts_hot LOCATION '/var/lib/postgresql/data/ts_hot';` la consola de PostgreSQL devuelve el siguiente error de base de datos:
  ```text
  ERROR: could not set permissions on directory "/var/lib/postgresql/data/ts_hot": Permission denied
  ```
* **Causa**: El comando de creación de carpetas dentro del contenedor fue ejecutado de forma predeterminada con privilegios del sistema operativo anfitrión que no corresponden al usuario interno de ejecución de PostgreSQL, impidiendo al motor modificar o escribir metadatos en el directorio físico asignado.
* **Solución**: Ejecuta la terminal de comandos de Docker usando la bandera de superusuario de Linux `-u 0` (root) para reasignar la propiedad exclusiva del recurso al identificador de usuario `postgres` mediante `chown`:
  ```bash
  docker exec -u 0 -it pg-primary chown -R postgres:postgres /var/lib/postgresql/data/ts_hot /var/lib/postgresql/data/ts_cold
  ```

### Problema 2: Inconsistencia o rechazo de inserción en tabla de hechos por violación de rango
* **Síntoma**: Durante la inserción masiva o flujos de carga se arroja el siguiente error de base de datos:
  ```text
  ERROR: no partition of relation "fact_sales" found for row
  DETAIL: Partition key of the failing row contains (sale_date) = (2024-04-01 00:00:00).
  ```
* **Causa**: Intentaste cargar una venta que se produjo el primero de Abril del 2024. Al ser PostgreSQL un motor declarativo estricto, si no existe una partición física creada de antemano que cubra el rango de la fecha proporcionada, el registro es rechazado para evitar la desorganización de los datos.
* **Solución**: Diseña una partición por defecto (`DEFAULT`) para contener filas huérfanas fuera del rango establecido o añade explícitamente la partición de Abril de 2024 ejecutando la siguiente instrucción SQL:
  ```sql
  CREATE TABLE fact_sales_2024_m04 PARTITION OF fact_sales
      FOR VALUES FROM ('2024-04-01 00:00:00') TO ('2024-05-01 00:00:00')
      TABLESPACE ts_hot;
  ```

---

## Limpieza

Una vez finalizada la validación, ejecuta las siguientes instrucciones para eliminar todos los recursos y mantener limpio tu entorno local de pruebas:

1. Sal de la consola interactiva `psql` si aún permaneces en ella:
   ```sql
   \q
   ```

2. Detén y destruye el contenedor PostgreSQL `pg-primary` utilizado para el laboratorio:
   ```bash
   docker rm -f pg-primary
   ```

3. Remueve la red Docker empresarial creada:
   ```bash
   docker network rm pg_enterprise_net
   ```

---

## Resumen

En este laboratorio práctico de ingeniería de datos, has diseñado y consolidado con éxito una infraestructura de almacenamiento de Data Warehouse utilizando características nativas avanzadas de PostgreSQL 16.2:

1. **Esquema de Estrella**: Implementaste un modelo dimensional acoplando tablas de dimensiones limpias de contexto con una tabla de hechos de alto rendimiento.
2. **Particionamiento de Base de Datos**: Segmentaste físicamente una tabla de hechos masiva en particiones lógicas por rangos mensuales sobre la marca de tiempo de ventas.
3. **Optimización por Temperatura**: Simulaste la arquitectura de un centro de datos real distribuyendo el almacenamiento de las particiones según su vigencia, asignándolas respectivamente a Tablespaces veloces (`ts_hot`) y de archivado pasivo (`ts_cold`).
4. **Optimización con Índices BRIN**: Introdujiste el concepto de índices basados en rangos de bloques para optimizar el acceso por rangos temporales reduciendo el uso del espacio del índice en disco a una fracción del tamaño clásico de un índice B-Tree convencional.
5. **Poda de Particiones (Partition Pruning)**: Validaste de forma medible por medio de planes de ejecución del optimizador cómo PostgreSQL evita escanear particiones irrelevantes, mejorando drásticamente el rendimiento analítico de tus consultas OLAP empresariales.

---
*Para mayor detalle técnico sobre configuraciones de optimización física, consulta la [Documentación Oficial de PostgreSQL 16 sobre Particionamiento](https://www.postgresql.org/docs/16/ddl-partitioning.html).*

---

# 5 Optimización de Consultas Analíticas e Indicadores

## Metadatos

| Métrica | Valor |
| :--- | :--- |
| **Duración** | 60 minutos |
| **Complejidad** | Difícil |
| **Nivel de Bloom** | Aplicar |

## Descripción General

En este laboratorio práctico, el estudiante se enfrentará al desafío de optimizar el rendimiento de un almacén de datos (*Data Warehouse*) corporativo sobre **PostgreSQL 16.2**. Partiendo de la base de datos `dwh_enterprise` estructurada en un esquema de estrella, se ejecutarán consultas analíticas complejas que calculan indicadores clave de rendimiento (KPIs) mediante expresiones comunes de tabla (CTEs) y funciones de ventana de gran consumo de CPU y memoria.

El estudiante analizará en detalle los planes de ejecución física utilizando `EXPLAIN (ANALYZE, BUFFERS)` para diagnosticar cuellos de botella asociados a escaneos secuenciales masivos y derrames de ordenamiento a disco. Para remediar estos problemas de rendimiento, implementará índices avanzados de rango de bloques (**BRIN**) optimizados para grandes volúmenes de datos ordenados cronológicamente, índices trigrama (**GIN**) para búsquedas de texto desestructurado, y diseñará una estrategia de **Vistas Materializadas** que soporte actualizaciones concurrentes sin interrupción de servicio para herramientas de visualización de datos.

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:

- [ ] Construir y depurar consultas analíticas complejas utilizando Expresiones Comunes de Tabla (CTEs) y Funciones de Ventana para el cálculo de acumulados y comparaciones intermensuales (YoY).
- [ ] Interpretar planes de ejecución complejos mediante `EXPLAIN (ANALYZE, BUFFERS)` identificando operaciones críticas de E/S y costes de CPU.
- [ ] Implementar índices **BRIN (Block Range Indexes)** en tablas de hechos particionadas, evaluando su impacto en la reducción de bloques leídos.
- [ ] Optimizar búsquedas analíticas de texto mediante la extensión `pg_trgm` y la creación de índices **GIN**.
- [ ] Crear vistas materializadas con índices únicos para posibilitar el refresco concurrente y no bloqueante de tableros de control.

## Prerrequisitos

Para completar con éxito este laboratorio, se requiere:

1. **Conocimientos teóricos y prácticos:**
   - Dominio del diseño físico de almacenes de datos y particionamiento declarativo (completado en el laboratorio anterior *05-00-01*).
   - Comprensión básica de los nodos del optimizador de PostgreSQL (Seq Scan, Bitmap Heap Scan, Hash Join, GroupAggregate).
   - Familiaridad con la sintaxis SQL de funciones de ventana analíticas (`LAG`, `LEAD`, `SUM(...) OVER(...)`).

2. **Acceso e Infraestructura:**
   - Acceso de red local o a través del contenedor Docker `pg-primary` en la red `pg_enterprise_net`.
   - Credenciales del usuario maestro administrador global: `postgres` con contraseña `PostgresAdminPass123!`.
   - Cliente SQL con soporte visual (DBeaver Community Edition 24.0.0 o superior recomendado) instalado en el sistema anfitrión.

## Entorno de Laboratorio

Este laboratorio utiliza componentes de nivel empresarial con versiones y orígenes validados para garantizar estabilidad y reproducibilidad:

### Software Requerido

| Software / Herramienta | Versión Exacta / Distribución | Enlace de Descarga Oficial |
| :--- | :--- | :--- |
| **PostgreSQL Community Edition** | 16.2 (Debian Bookworm) | [PostgreSQL Downloads](https://www.postgresql.org/download/) |
| **Docker Engine / Desktop** | 26.0.0 (o superior) | [Docker Docs](https://docs.docker.com/engine/install/) |
| **DBeaver Community Edition** | 24.0.0 | [DBeaver Downloads](https://dbeaver.io/download/) |

### Hardware Mínimo Recomendado

* **Procesador:** Arquitectura x86_64 con mínimo 8 núcleos físicos.
* **Memoria RAM:** 16 GB mínimos (32 GB recomendados para la ejecución concurrente de contenedores).
* **Almacenamiento:** 100 GB de espacio libre en unidad de estado sólido (SSD NVMe recomendado para medir mejoras de E/S con precisión).

### Comandos de Preparación del Entorno

Si no dispones del contenedor del laboratorio previo ejecutándose en la red común `pg_enterprise_net`, inicializa un contenedor PostgreSQL 16.2 limpio con el siguiente comando en tu terminal:

```bash
docker run -d \
  --name pg-primary \
  --net pg_enterprise_net \
  -p 5432:5432 \
  -e POSTGRES_PASSWORD='PostgresAdminPass123!' \
  -e POSTGRES_DB='enterprise_db' \
  postgres:16.2
```

> **Nota de Configuración:** Para simular un entorno analítico realista, configure temporalmente los siguientes parámetros en caliente dentro de su sesión antes de realizar las cargas de datos:
> ```sql
> SET work_mem = '64MB';
> SET max_parallel_workers_per_gather = 4;
> ```

---

## Instrucciones Paso a Paso

### Paso 1: Inicialización y Carga de Datos en 'dwh_enterprise'

**Objetivo:** Crear el esquema analítico de estrella y simular la ingesta de un volumen representativo de registros de transacciones (aproximadamente 500,000 ventas de prueba) para forzar al optimizador a tomar decisiones basadas en costos reales.

**Instrucciones:**

1. Conéctate a tu base de datos utilizando DBeaver u otra herramienta cliente con el usuario `postgres` y la base de datos `enterprise_db`.
2. Ejecuta el script de creación de base de datos para separar los flujos transaccionales de los analíticos:

```sql
-- Crear la base de datos analítica si no existe (ejecutar desde base de datos default)
SELECT 'CREATE DATABASE dwh_enterprise' 
WHERE NOT EXISTS (SELECT FROM pg_database WHERE datname = 'dwh_enterprise') \gexec
```

3. Conéctate a la nueva base de datos `dwh_enterprise` e inicializa las dimensiones y la tabla de hechos particionada ejecutando las siguientes sentencias DDL:

```sql
-- Habilitar extensión para búsquedas avanzadas
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Dimensión de Clientes
CREATE TABLE dim_clientes (
    cliente_key SERIAL PRIMARY KEY,
    cliente_id VARCHAR(50) NOT NULL UNIQUE,
    nombre VARCHAR(150) NOT NULL,
    segmento VARCHAR(50),
    ciudad VARCHAR(100),
    pais VARCHAR(100),
    fecha_registro DATE NOT NULL
);

-- Dimensión de Productos
CREATE TABLE dim_productos (
    producto_key SERIAL PRIMARY KEY,
    producto_id VARCHAR(50) NOT NULL UNIQUE,
    nombre VARCHAR(200) NOT NULL,
    categoria VARCHAR(100),
    subcategoria VARCHAR(100),
    marca VARCHAR(100),
    precio_sugerido NUMERIC(12,2)
);

-- Dimensión de Tiempo
CREATE TABLE dim_tiempo (
    tiempo_key INT PRIMARY KEY, -- Formato AAAAMMDD
    fecha DATE NOT NULL UNIQUE,
    anio INT NOT NULL,
    trimestre INT NOT NULL,
    mes INT NOT NULL,
    nombre_mes VARCHAR(20) NOT NULL,
    dia_mes INT NOT NULL,
    dia_semana VARCHAR(20) NOT NULL,
    es_fin_semana BOOLEAN NOT NULL
);

-- Tabla de Hechos de Ventas (Particionada por Rango sobre el campo fecha_proceso)
CREATE TABLE fact_ventas (
    venta_id BIGINT NOT NULL,
    cliente_key INT NOT NULL,
    producto_key INT NOT NULL,
    tiempo_key INT NOT NULL,
    cantidad INT NOT NULL,
    monto_bruto NUMERIC(15,2) NOT NULL,
    descuento NUMERIC(15,2) NOT NULL,
    monto_neto NUMERIC(15,2) NOT NULL,
    fecha_proceso TIMESTAMP WITHOUT TIME ZONE NOT NULL
) PARTITION BY RANGE (fecha_proceso);

-- Creación de las Particiones Físicas Mensuales para el Primer Trimestre de 2024
CREATE TABLE fact_ventas_y2024m01 PARTITION OF fact_ventas
    FOR VALUES FROM ('2024-01-01 00:00:00') TO ('2024-02-01 00:00:00');

CREATE TABLE fact_ventas_y2024m02 PARTITION OF fact_ventas
    FOR VALUES FROM ('2024-02-01 00:00:00') TO ('2024-03-01 00:00:00');

CREATE TABLE fact_ventas_y2024m03 PARTITION OF fact_ventas
    FOR VALUES FROM ('2024-03-01 00:00:00') TO ('2024-04-01 00:00:00');
```

4. Genera de forma masiva los datos sintéticos para poblar las tablas. Utilizaremos generadores procedurales para crear 5,000 clientes, 500 productos y más de 500,000 transacciones ordenadas temporalmente:

```sql
-- Poblar Clientes
INSERT INTO dim_clientes (cliente_id, nombre, segmento, ciudad, pais, fecha_registro)
SELECT 
    'CLI-' || LPAD(i::text, 6, '0'),
    'Cliente Corporativo Nro ' || i,
    CASE WHEN i % 3 = 0 THEN 'Corporativo' WHEN i % 3 = 1 THEN 'Consumidor Final' ELSE 'Pyme' END,
    CASE WHEN i % 2 = 0 THEN 'Bogotá' ELSE 'Santiago' END,
    CASE WHEN i % 2 = 0 THEN 'Colombia' ELSE 'Chile' END,
    '2023-01-01'::DATE + (i % 365) * INTERVAL '1 day'
FROM generate_series(1, 5000) AS i;

-- Poblar Productos
INSERT INTO dim_productos (producto_id, nombre, categoria, subcategoria, marca, precio_sugerido)
SELECT 
    'PROD-' || LPAD(i::text, 5, '0'),
    'Dispositivo Inteligente Modelo Ultra-' || i,
    CASE WHEN i % 2 = 0 THEN 'Tecnología' ELSE 'Hogar e Innovación' END,
    CASE WHEN i % 2 = 0 THEN 'Celulares y Tablets' ELSE 'Muebles de Oficina' END,
    'Marca Global ' || (i % 10),
    (15.50 + (i % 100) * 12.50)::NUMERIC
FROM generate_series(1, 500) AS i;

-- Poblar Tiempo para el Q1 2024
INSERT INTO dim_tiempo (tiempo_key, fecha, anio, trimestre, mes, nombre_mes, dia_mes, dia_semana, es_fin_semana)
SELECT 
    to_char(d, 'YYYYMMDD')::INT,
    d::DATE,
    EXTRACT(YEAR FROM d)::INT,
    EXTRACT(QUARTER FROM d)::INT,
    EXTRACT(MONTH FROM d)::INT,
    to_char(d, 'TMMonth'),
    EXTRACT(DAY FROM d)::INT,
    to_char(d, 'TMDia'),
    CASE WHEN EXTRACT(ISODOW FROM d) IN (6, 7) THEN TRUE ELSE FALSE END
FROM generate_series('2024-01-01'::timestamp, '2024-03-31'::timestamp, '1 day'::interval) AS d;

-- Ingesta Masiva en la Tabla de Hechos (Ventas Distribuidas Ordenadamente en el Tiempo)
-- Esto garantiza la ordenación física natural que explota el índice BRIN
INSERT INTO fact_ventas (venta_id, cliente_key, producto_key, tiempo_key, cantidad, monto_bruto, descuento, monto_neto, fecha_proceso)
SELECT 
    i AS venta_id,
    1 + (i % 5000) AS cliente_key,
    1 + (i % 500) AS producto_key,
    to_char('2024-01-01'::timestamp + (i % 90) * INTERVAL '1 day' + (i % 24) * INTERVAL '1 hour', 'YYYYMMDD')::INT AS tiempo_key,
    1 + (i % 10) AS cantidad,
    (1 + (i % 10)) * (50 + (i % 50) * 5.0) AS monto_bruto,
    ((1 + (i % 10)) * (50 + (i % 50) * 5.0)) * 0.05 AS descuento,
    ((1 + (i % 10)) * (50 + (i % 50) * 5.0)) * 0.95 AS monto_neto,
    '2024-01-01'::timestamp + (i % 90) * INTERVAL '1 day' + (i % 24) * INTERVAL '1 hour' AS fecha_proceso
FROM generate_series(1, 500000) AS i;
```

5. Recopila estadísticas frescas para que el planificador pueda tomar decisiones correctas de estimación de cardinalidad:

```sql
ANALYZE dim_clientes;
ANALYZE dim_productos;
ANALYZE dim_tiempo;
ANALYZE fact_ventas;
```

**Resultado Esperado:**
El script debe ejecutarse de manera fluida y exitosa. Se insertarán 500,000 registros en `fact_ventas` repartidos equitativamente en tres particiones físicas correspondientes a los meses de enero, febrero y marzo de 2024.

**Verificación:**
Verifica la distribución física y lógica de las particiones ejecutando:
```sql
SELECT tableoid::regclass, count(*), min(fecha_proceso), max(fecha_proceso)
FROM fact_ventas
GROUP BY tableoid;
```

---

### Paso 2: Ejecución y Diagnóstico de la Consulta Analítica Inicial

**Objetivo:** Diseñar y ejecutar una consulta analítica de alta complejidad que evalúe KPIs acumulados, comparativas intermensuales y clasifique clientes. Analizar su plan de ejecución en frío e identificar las ineficiencias de hardware que afectan su desempeño.

**Instrucciones:**

1. Ejecuta la siguiente consulta compleja que utiliza múltiples CTEs, uniones y funciones de ventana analíticas. Antepón la sentencia `EXPLAIN (ANALYZE, BUFFERS, TIMING)` para rastrear el planificador, los accesos a los buffers y el tiempo de ejecución de cada nodo de procesamiento:

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING)
WITH ventas_diarias AS (
    -- CTE 1: Agregación de ventas por día, cliente y categoría
    SELECT 
        f.fecha_proceso::DATE AS fecha_venta,
        c.cliente_id,
        c.segmento,
        p.categoria,
        SUM(f.monto_neto) AS ventas_totales,
        SUM(f.cantidad) AS cantidad_total
    FROM fact_ventas f
    JOIN dim_clientes c ON f.cliente_key = c.cliente_key
    JOIN dim_productos p ON f.producto_key = p.producto_key
    -- Filtro de fecha selectivo que abarca los primeros dos meses
    WHERE f.fecha_proceso >= '2024-01-15 00:00:00' 
      AND f.fecha_proceso <= '2024-02-28 23:59:59'
    GROUP BY f.fecha_proceso::DATE, c.cliente_id, c.segmento, p.categoria
),
acumulado_indicadores AS (
    -- CTE 2: Cálculo de acumulados y ventana móvil analítica
    SELECT 
        fecha_venta,
        segmento,
        categoria,
        ventas_totales,
        -- Suma acumulativa sobre una ventana temporal deslizante
        SUM(ventas_totales) OVER (
            PARTITION BY segmento, categoria 
            ORDER BY fecha_venta 
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS acumulado_7d,
        -- Valor de ventas del mismo día del periodo anterior (Simulación YoY/Mes anterior)
        LAG(ventas_totales, 30) OVER (
            PARTITION BY segmento, categoria 
            ORDER BY fecha_venta
        ) AS ventas_mes_anterior
    FROM ventas_diarias
)
SELECT 
    fecha_venta,
    segmento,
    categoria,
    ventas_totales,
    acumulado_7d,
    ventas_mes_anterior,
    COALESCE(ventas_totales - ventas_mes_anterior, 0) AS variacion_absoluta
FROM acumulado_indicadores
ORDER BY segmento, categoria, fecha_venta DESC;
```

**Resultado Esperado:**
La consulta retornará el plan detallado. El tiempo de ejecución inicial será notable. En la salida de `EXPLAIN ANALYZE` se deben identificar los siguientes puntos de dolor críticos:
*   **Sequential Scan (Seq Scan):** El motor lee secuencialmente todas las particiones del rango (`fact_ventas_y2024m01` y `fact_ventas_y2024m02`) de extremo a extremo debido a la falta de índices específicos para acotar rangos de fecha.
*   **Buffer Reads (Read/Shared Hit):** Se leerá un elevado número de bloques desde el disco o desde la caché (`Buffers: shared read=XXXX`).
*   **Sort / Temp File Spill (si aplica):** Si la memoria disponible (`work_mem`) no es suficiente para acomodar los datos temporales del ordenamiento analítico de la función de ventana, se generarán escrituras en archivos temporales de disco (`Spill to disk`).

```text
-- Fragmento de salida conceptual del EXPLAIN original:
->  Append  (cost=0.00..34345.00 rows=245000 width=72) (actual time=0.034..320.110 rows=251200 loops=1)
      ->  Seq Scan on fact_ventas_y2024m01 f_1  (...) (actual time=0.033..120.450 rows=150000 loops=1)
            Filter: ((fecha_proceso >= '2024-01-15 00:00:00'::timestamp) AND (fecha_proceso <= '2024-02-28 23:59:59'::timestamp))
```

**Verificación:**
Identifica y documenta los valores exactos para tu entorno de:
1. `Planning Time` y `Execution Time`.
2. Número total de bloques leídos (`shared read`) para las tablas de hechos.
3. Tipo de escaneo aplicado en la partición `fact_ventas_y2024m01`.

---

### Paso 3: Optimización con Índices BRIN en la Tabla de Hechos

**Objetivo:** Crear un índice BRIN (Block Range Index) en la tabla padre de hechos para que el planificador aplique poda física y optimice significativamente las lecturas de disco sobre las particiones analizadas, comprendiendo la granularidad de las páginas físicas de almacenamiento.

**Instrucciones:**

1. Compara conceptualmente el tamaño en memoria y la naturaleza de un índice B-Tree tradicional frente a un índice BRIN para esta arquitectura. 
2. Crea el índice BRIN sobre el campo `fecha_proceso` de la tabla heredada `fact_ventas`. Configuraremos el parámetro `pages_per_range` en `32` para que el índice sea altamente selectivo al agrupar rangos reducidos de bloques contiguos en disco físico:

```sql
CREATE INDEX idx_fact_ventas_fecha_brin 
ON fact_ventas USING brin (fecha_proceso) 
WITH (pages_per_range = 32);
```

> **Explicación Técnica:** Un índice BRIN asocia un rango físico de páginas en disco (por ejemplo, 32 páginas consecutivas) con su valor mínimo y máximo de fecha para ese bloque. Como nuestros registros fueron insertados en orden cronológico ascendente (idéntico al tiempo real de las transacciones empresariales), los rangos de fechas no se traslapan en los bloques del archivo físico de disco, permitiendo al optimizador descartar de forma limpia bloques de páginas que quedan fuera del filtro `WHERE`.

3. Para asegurar que los accesos a llaves primarias y foráneas dentro de los cruces de dimensiones no se degraden, crea índices estándar B-Tree tradicionales sobre los identificadores en la tabla base:

```sql
CREATE INDEX idx_fact_ventas_cliente ON fact_ventas(cliente_key);
CREATE INDEX idx_fact_ventas_producto ON fact_ventas(producto_key);
```

4. Vuelve a ejecutar la consulta analítica compleja del **Paso 2** anteponiendo el `EXPLAIN (ANALYZE, BUFFERS)` para medir el impacto de la optimización:

```sql
EXPLAIN (ANALYZE, BUFFERS)
WITH ventas_diarias AS (
    SELECT 
        f.fecha_proceso::DATE AS fecha_venta,
        c.cliente_id,
        c.segmento,
        p.categoria,
        SUM(f.monto_neto) AS ventas_totales,
        SUM(f.cantidad) AS cantidad_total
    FROM fact_ventas f
    JOIN dim_clientes c ON f.cliente_key = c.cliente_key
    JOIN dim_productos p ON f.producto_key = p.producto_key
    WHERE f.fecha_proceso >= '2024-01-15 00:00:00' 
      AND f.fecha_proceso <= '2024-02-28 23:59:59'
    GROUP BY f.fecha_proceso::DATE, c.cliente_id, c.segmento, p.categoria
),
acumulado_indicadores AS (
    SELECT 
        fecha_venta,
        segmento,
        categoria,
        ventas_totales,
        SUM(ventas_totales) OVER (
            PARTITION BY segmento, categoria 
            ORDER BY fecha_venta 
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS acumulado_7d,
        LAG(ventas_totales, 30) OVER (
            PARTITION BY segmento, categoria 
            ORDER BY fecha_venta
        ) AS ventas_mes_anterior
    FROM ventas_diarias
)
SELECT 
    fecha_venta,
    segmento,
    categoria,
    ventas_totales,
    acumulado_7d,
    ventas_mes_anterior,
    COALESCE(ventas_totales - ventas_mes_anterior, 0) AS variacion_absoluta
FROM acumulado_indicadores
ORDER BY segmento, categoria, fecha_venta DESC;
```

**Resultado Esperado:**
El planificador utilizará ahora un **Bitmap Index Scan** sobre el índice BRIN en lugar de un escaneo secuencial puro para acotar la búsqueda en las particiones de enero y febrero. El volumen de páginas de datos leídas (`shared hit / read`) debe descender de forma drástica, impactando positivamente en el `Execution Time`.

```text
-- Ejemplo de nodo optimizado en la salida:
->  Bitmap Heap Scan on fact_ventas_y2024m01 f_1  (...) (actual time=1.120..42.230 rows=150000 loops=1)
      Recheck Cond: ((fecha_proceso >= '2024-01-15 00:00:00'::timestamp) AND (fecha_proceso <= '2024-02-28 23:59:59'::timestamp))
      Rows Removed by Index Recheck: 0
      Heap Blocks: lossy=320
      Buffers: shared hit=335
      ->  Bitmap Index Scan on fact_ventas_y2024m01_fecha_proceso_idx  (...) (actual time=0.850..0.850 rows=10 loops=1)
            Index Cond: ((fecha_proceso >= '2024-01-15 00:00:00'::timestamp) AND (fecha_proceso <= '2024-02-28 23:59:59'::timestamp))
```

**Verificación:**
Compara las métricas de búferes compartidos:
- Anota el valor de `shared read` antes y después de aplicar el índice BRIN.
- Valida que la partición fuera del filtro (`fact_ventas_y2024m03`) **no** haya sido escaneada (demostrando la correcta funcionalidad del *partition pruning*).

---

### Paso 4: Optimización de Búsqueda de Texto con Índices GIN

**Objetivo:** Implementar índices de tipo GIN basados en trigramas (`pg_trgm`) para acelerar dramáticamente las búsquedas analíticas que involucren coincidencias de texto parcial o comodines sobre las dimensiones de clientes o productos, evitando costosos escaneos secuenciales de texto completo.

**Instrucciones:**

1. Ejecuta una consulta analítica típica con un filtro difuso e ineficiente (`LIKE '%Inteligente%'`) analizando su costo inicial:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT p.categoria, p.subcategoria, COUNT(f.venta_id) AS total_ventas, SUM(f.monto_neto) AS total_monto
FROM fact_ventas f
JOIN dim_productos p ON f.producto_key = p.producto_key
WHERE p.nombre LIKE '%Inteligente%'
GROUP BY p.categoria, p.subcategoria;
```

2. Valida en la salida del `EXPLAIN` que el optimizador ejecuta un `Seq Scan` sobre la tabla completa `dim_productos` para evaluar cada cadena de caracteres individualmente debido a que los índices B-Tree estándar no admiten búsquedas comodín frontales (`%texto`).
3. Crea un índice **GIN** de tipo trigrama sobre el atributo `nombre` de la dimensión productos utilizando la extensión del sistema previamente configurada:

```sql
CREATE INDEX idx_dim_productos_nombre_gin 
ON dim_productos 
USING gin (nombre gin_trgm_ops);
```

4. Vuelve a ejecutar la consulta de búsqueda analítica difusa con `EXPLAIN (ANALYZE, BUFFERS)` para cuantificar el cambio de comportamiento:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT p.categoria, p.subcategoria, COUNT(f.venta_id) AS total_ventas, SUM(f.monto_neto) AS total_monto
FROM fact_ventas f
JOIN dim_productos p ON f.producto_key = p.producto_key
WHERE p.nombre LIKE '%Inteligente%'
GROUP BY p.categoria, p.subcategoria;
```

**Resultado Esperado:**
El motor ahora sustituirá el escaneo secuencial en la dimensión de productos por un acceso directo **Bitmap Index Scan** utilizando el índice `idx_dim_productos_nombre_gin`.

```text
-- Fragmento de salida esperado:
->  Bitmap Heap Scan on dim_productos p (...) (actual time=0.210..0.520 rows=500 loops=1)
      Recheck Cond: ((nombre)::text ~~ '%Inteligente%'::text)
      ->  Bitmap Index Scan on idx_dim_productos_nombre_gin (...) (actual time=0.090..0.090 rows=500 loops=1)
            Index Cond: ((nombre)::text ~~ '%Inteligente%'::text)
```

**Verificación:**
Valida que el tiempo de ejecución en la fase de resolución de los nombres de los productos disminuya sustancialmente y que la cantidad de bloques compartidos consultados en la dimensión sea de orden mínimo.

---

### Paso 5: Implementación de Vista Materializada y Refresco Concurrente

**Objetivo:** Crear una Vista Materializada para precalcular y almacenar los KPIs calculados en el Paso 2 de forma física, estructurando un índice único que habilite actualizaciones asíncronas de tipo `CONCURRENT` sin bloquear consultas simultáneas de usuarios o herramientas de visualización de datos (ej. Grafana).

**Instrucciones:**

1. Define la estructura de la Vista Materializada `mv_kpis_ventas_mensuales` que consolide las operaciones analíticas pesadas:

```sql
CREATE MATERIALIZED VIEW mv_kpis_ventas_mensuales AS
WITH ventas_diarias AS (
    SELECT 
        f.fecha_proceso::DATE AS fecha_venta,
        c.cliente_id,
        c.segmento,
        p.categoria,
        SUM(f.monto_neto) AS ventas_totales,
        SUM(f.cantidad) AS cantidad_total
    FROM fact_ventas f
    JOIN dim_clientes c ON f.cliente_key = c.cliente_key
    JOIN dim_productos p ON f.producto_key = p.producto_key
    WHERE f.fecha_proceso >= '2024-01-01 00:00:00' 
      AND f.fecha_proceso <= '2024-03-31 23:59:59'
    GROUP BY f.fecha_proceso::DATE, c.cliente_id, c.segmento, p.categoria
),
acumulado_indicadores AS (
    SELECT 
        fecha_venta,
        segmento,
        categoria,
        ventas_totales,
        SUM(ventas_totales) OVER (
            PARTITION BY segmento, categoria 
            ORDER BY fecha_venta 
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) AS acumulado_7d,
        LAG(ventas_totales, 30) OVER (
            PARTITION BY segmento, categoria 
            ORDER BY fecha_venta
        ) AS ventas_mes_anterior
    FROM ventas_diarias
)
SELECT 
    fecha_venta,
    segmento,
    categoria,
    ventas_totales,
    acumulado_7d,
    ventas_mes_anterior,
    COALESCE(ventas_totales - ventas_mes_anterior, 0) AS variacion_absoluta
FROM acumulado_indicadores
WITH DATA;
```

2. Ejecuta una consulta directa sobre la vista materializada y analiza la velocidad de acceso mediante `EXPLAIN ANALYZE`:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * 
FROM mv_kpis_ventas_mensuales
WHERE segmento = 'Corporativo' AND categoria = 'Tecnología'
ORDER BY fecha_venta DESC;
```

La velocidad de esta consulta es prácticamente instantánea porque los resultados ya están precalculados e indexados de forma nativa.

3. Para posibilitar el refresco concurrente, PostgreSQL requiere de un **índice único restrictivo** en la vista materializada que identifique de forma unívoca cada fila. Define un índice compuesto sobre las claves físicas de agrupamiento de la vista:

```sql
CREATE UNIQUE INDEX idx_mv_kpis_ventas_mensuales_key 
ON mv_kpis_ventas_mensuales (segmento, categoria, fecha_venta);
```

4. Simula la llegada de nuevos datos transaccionales de venta en la tabla de hechos:

```sql
INSERT INTO fact_ventas (venta_id, cliente_key, producto_key, tiempo_key, cantidad, monto_bruto, descuento, monto_neto, fecha_proceso)
VALUES (999999, 10, 20, 20240315, 5, 500.00, 0.00, 500.00, '2024-03-15 10:30:00');
```

5. Ejecuta el comando de refresco concurrente de la vista de forma asíncrona para propagar la inserción de datos de negocio sin adquirir un bloqueo exclusivo (`AccessExclusiveLock`) sobre la tabla física de la vista:

```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_kpis_ventas_mensuales;
```

**Resultado Esperado:**
La vista materializada procesa las consultas de forma inmediata recurriendo a un acceso de índice compuesto o un barrido rápido de la vista precalculada. El proceso de refresco concurrente finaliza con éxito sin interrumpir las consultas lectoras concurrentes activas en el sistema analítico.

**Verificación:**
Compara las diferencias temporales y de coste computacional de acceder a la vista materializada precalculada frente a la consulta analítica inicial que ejecuta joins en tiempo real. La latencia debe reducirse más del 95%.

---

## Validación y Pruebas

Para validar el rendimiento y el cumplimiento de las metas de diseño físico del Data Warehouse, ejecuta las siguientes pruebas diagnósticas:

### Prueba de Impacto del Índice BRIN

Analiza la diferencia en bloques cargados en memoria compartida para un rango selectivo de fechas mediante la ejecución manual de estos comandos:

```sql
-- Ejecutar sin indexación en fecha (Simula caída de índices borrando temporalmente el brin)
BEGIN;
DROP INDEX idx_fact_ventas_fecha_brin;
EXPLAIN (ANALYZE, BUFFERS)
SELECT COUNT(*) FROM fact_ventas WHERE fecha_proceso BETWEEN '2024-02-10 00:00:00' AND '2024-02-15 23:59:59';
ROLLBACK;
```

**Métricas Objetivo:**
- Con índice BRIN activo: `shared hit` o `shared read` < 1,000 bloques.
- Sin índice BRIN activo (Seq Scan): `shared hit/read` > 10,000 bloques.

### Escenario de Prueba Adversaria (Caso Límite de IA / Inyección de Errores)

Uno de los errores más comunes de diseño automatizado en automatizaciones con scripts es intentar refrescar concurrentemente una vista sin un identificador determinista único. Ejecuta intencionadamente el escenario de error para auditar la robustez del sistema:

1. Crea una copia rápida de prueba para inducir el comportamiento incorrecto:
```sql
CREATE MATERIALIZED VIEW mv_kpis_fallida AS SELECT * FROM mv_kpis_ventas_mensuales WITH DATA;
```
2. Ejecuta el refresco concurrente sin haber configurado un índice único en ella:
```sql
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_kpis_fallida;
```

**Resultado Esperado (Código de Error Oficial de PostgreSQL):**
```text
ERROR: cannot refresh materialized view "public.mv_kpis_fallida" concurrently
HINT: Create a unique index with no WHERE clause on one or more columns of the materialized view.
```

Este escenario adversarial demuestra que las optimizaciones concurrentes en entornos reales requieren de políticas estructuradas de restricción física (claves lógicas combinadas) y no de simples declaraciones de actualización de datos. Elimina el objeto fallido:
```sql
DROP MATERIALIZED VIEW mv_kpis_fallida;
```

---

## Solución de Problemas

### Problema 1: El índice BRIN se crea con éxito pero PostgreSQL sigue aplicando un Seq Scan masivo sobre todas las particiones.

- **Síntoma:** Tras la creación de `idx_fact_ventas_fecha_brin`, las ejecuciones con filtros `WHERE` de fechas muestran un rendimiento degradado persistente y la salida de `EXPLAIN` detalla escaneo secuencial en las particiones.
- **Causa Raíz:** Falta de ordenación de los datos físicos al insertarlos en disco, o los datos insertados provienen de transacciones con fechas desordenadas en el tiempo, provocando el solapamiento extremo en los límites mínimos y máximos de la propiedad `pages_per_range` (el índice abarca rangos de fecha que abarcan todo el espectro temporal en casi cada bloque).
- **Resolución:**
  1. Reordena la tabla físicamente utilizando la cláusula `CLUSTER` (para tablas normales) o realiza un volcado ordenado hacia una nueva partición histórica mediante:
     ```sql
     CREATE TABLE fact_ventas_temp AS 
     SELECT * FROM fact_ventas ORDER BY fecha_proceso;
     ```
  2. Reconfigura el índice reduciendo la propiedad `pages_per_range` (por ejemplo, a `16` o `32`) para acotar con mayor resolución espacial los rangos en disco físico:
     ```sql
     DROP INDEX idx_fact_ventas_fecha_brin;
     CREATE INDEX idx_fact_ventas_fecha_brin 
     ON fact_ventas USING brin (fecha_proceso) WITH (pages_per_range = 16);
     ```

### Problema 2: El comando 'REFRESH MATERIALIZED VIEW CONCURRENTLY' falla indicando duplicados en la columna de llave primaria del índice único.

- **Síntoma:** Al refrescar concurrentemente se obtiene el error `ERROR: could not create unique index ...` debido a violaciones de clave única.
- **Causa Raíz:** Los campos elegidos para constituir el índice único compuesto de la vista materializada no garantizan la unicidad. Por ejemplo, si se omite el campo `segmento` o la `fecha_venta` y existen transacciones de distintas categorías el mismo día.
- **Resolución:**
  1. Reevalúa y amplía la clave compuesta agregando todos los campos de agrupación lógica de la vista que garanticen la unicidad del registro agregativo:
     ```sql
     DROP INDEX IF EXISTS idx_mv_kpis_ventas_mensuales_key;
     CREATE UNIQUE INDEX idx_mv_kpis_ventas_mensuales_key 
     ON mv_kpis_ventas_mensuales (segmento, categoria, fecha_venta);
     ```
  2. Ejecuta una consulta de duplicación de control sobre el resultado previo para diagnosticar las filas infractoras en caso de fallar:
     ```sql
     SELECT segmento, categoria, fecha_venta, count(*)
     FROM mv_kpis_ventas_mensuales
     GROUP BY segmento, categoria, fecha_venta
     HAVING count(*) > 1;
     ```

---

## Limpieza

Para restaurar el entorno y liberar los recursos asignados a esta sesión analítica, ejecuta los siguientes comandos de limpieza física en la base de datos:

1. Elimina la base de datos creada en este laboratorio y todos sus objetos de forma definitiva:

```sql
-- Conéctate a otra base de datos (por ejemplo, 'enterprise_db' o 'postgres') para permitir el drop
\c enterprise_db

-- Forzar la desconexión de cualquier sesión persistente activa en 'dwh_enterprise'
SELECT pg_terminate_backend(pg_stat_activity.pid)
FROM pg_stat_activity
WHERE pg_stat_activity.datname = 'dwh_enterprise'
  AND pid <> pg_backend_pid();

-- Eliminar la base de datos completa
DROP DATABASE IF EXISTS dwh_enterprise;
```

2. Si inicializaste un contenedor temporal específico para este laboratorio, detenlo y bórralo utilizando tu shell o terminal del sistema operativo:

```bash
docker stop pg-primary
docker rm pg-primary
```

---

## Resumen

En este laboratorio práctico de alta complejidad técnica, has implementado estrategias críticas para la aceleración y mantenimiento de un almacén de datos empresarial moderno corriendo sobre **PostgreSQL 16.2**:

1. **Uso Avanzado de SQL Analítico:** Estructuraste consultas agregativas mediante el anidamiento estratégico de Expresiones Comunes de Tabla (CTEs) y la asignación de Ventanas Móviles para el cálculo de series de indicadores de negocio.
2. **Índices BRIN:** Demostraste la efectividad del índice de rango de bloques para disminuir el costo de E/S en tablas de hechos particionadas y ordenadas cronológicamente, reduciendo drásticamente el espacio de almacenamiento consumido en comparación con los índices B-Tree estándar.
3. **Optimización con Índices GIN:** Introdujiste la indexación por trigramas para consultas que procesan texto difuso corporativo, previniendo el costoso Seq Scan de textos.
4. **Vistas Materializadas con Refresco Concurrente:** Diseñaste una arquitectura para precalcular métricas complejas, configurando índices únicos necesarios para permitir refrescos no bloqueantes que sustentan flujos ininterrumpidos en aplicaciones BI empresariales.

### Recursos Recomendados para Profundizar:
* [PostgreSQL 16.2 Official Documentation: BRIN Indexes](https://www.postgresql.org/docs/16/brin.html)
* [PostgreSQL 16.2 Official Documentation: pg_trgm Extension](https://www.postgresql.org/docs/16/pgtrgm.html)
* [Using Materialized Views Concurrently in Production Architectures](https://www.postgresql.org/docs/16/rules-materializedviews.html)
