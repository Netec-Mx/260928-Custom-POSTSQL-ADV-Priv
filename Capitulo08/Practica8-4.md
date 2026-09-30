# 8.4 Diseño de un Data Warehouse en PostgreSQL

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