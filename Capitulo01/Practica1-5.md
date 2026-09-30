# 1.5 Diseño de una Base de Datos Empresarial

## Metadatos

| Parámetro | Detalle |
| :--- | :--- |
| **Duración** | 45 minutos |
| **Complejidad** | Media |
| **Nivel de Taxonomía de Bloom** | Crear |

## Descripción General

En este laboratorio, diseñará e implementará un esquema de base de datos relacional de grado empresarial para el procesamiento de transacciones financieras utilizando **PostgreSQL 16.2**. Configurará almacenamiento físico segregado simulando discos duros reales mediante el uso de *Tablespaces* de PostgreSQL mapeados a volúmenes Docker específicos del host, separando físicamente datos calientes de alta transaccionalidad de datos fríos de archivo histórico. Posteriormente, implementará estrategias avanzadas de particionamiento declarativo nativo, utilizando particionamiento por **rango** para la tabla histórica de transacciones y particionamiento por **hash** para la distribución de perfiles de usuario, garantizando la integridad referencial, llaves primarias compuestas y la propagación automática de índices B-Tree en las particiones hijas.

## Objetivos de Aprendizaje

Al finalizar este laboratorio, usted será capaz de:
- [ ] Configurar y asignar *Tablespaces* lógicos de PostgreSQL a rutas físicas del host utilizando contenedores Docker.
- [ ] Implementar esquemas de particionamiento declarativo por rango (*Range Partitioning*) para series temporales y optimizar el almacenamiento según la antigüedad de los datos.
- [ ] Implementar esquemas de particionamiento declarativo por Hash (*Hash Partitioning*) para distribuir de manera uniforme cargas de trabajo pesadas de lectura y escritura.
- [ ] Resolver restricciones de diseño físico en PostgreSQL, incluyendo la correcta estructuración de llaves primarias compuestas e índices B-Tree en tablas particionadas.

## Prerrequisitos

Para realizar con éxito esta práctica, requiere:
1. **Conocimientos teóricos:** Comprensión básica de sentencias DDL (SQL), conceptos de normalización, llaves primarias, llaves foráneas y el funcionamiento general de almacenamiento físico en PostgreSQL.
2. **Acceso y software local:**
   - Un motor de contenedores Docker Engine instalado y en ejecución.
   - Cliente SQL compatible (como psql integrado en consola, DBeaver, o pgAdmin).
   - Acceso de administración (sudo / root) en su máquina local para manipular directorios locales y permisos del sistema de archivos.

## Entorno de Laboratorio

### Especificaciones de Software y Hardware

| Componente | Requisito Mínimo | Requisito Recomendado | Enlace de Referencia Oficial |
| :--- | :--- | :--- | :--- |
| **Sistema Operativo** | Debian 12.5 (Bookworm) / Ubuntu 22.04 LTS / macOS 13 | Debian 12.5 (Bookworm) x86_64 | [Debian Official Release](https://www.debian.org/releases/bookworm/) |
| **Motor de Docker** | Docker Engine 26.0.0 (Linux/x86_64) | Docker Engine 26.0.0 (Enterprise/Community) | [Docker Engine Installation](https://docs.docker.com/engine/install/) |
| **Base de Datos** | PostgreSQL 16.2 Community Edition (Docker Image) | PostgreSQL 16.2 (Debian Image) | [PostgreSQL Docker Hub](https://hub.docker.com/_/postgres) |
| **Memoria RAM** | 16 GB RAM | 32 GB RAM | N/A |
| **Procesador** | Arquitectura x86_64 con 8 núcleos | CPU Intel Core i7/i9 o AMD Ryzen 7/9 | N/A |
| **Almacenamiento**| 100 GB SSD SATA (Lectura > 500 MB/s) | 100 GB NVMe M.2 SSD | N/A |

### Comandos de Preparación del Entorno

Ejecute los siguientes comandos en su terminal para crear la estructura de carpetas locales del host donde se simulará el almacenamiento caliente (*hot*) y frío (*cold*) de los *Tablespaces*, y definir la red de contenedores de Docker:

```bash
## 1. Crear directorios físicos para simular discos de almacenamiento diferenciados
mkdir -p /tmp/pg_enterprise/hot_data
mkdir -p /tmp/pg_enterprise/cold_data

## 2. Configurar permisos de escritura máximos para evitar conflictos con el usuario 'postgres' interno del contenedor (UID 999)
chmod -R 777 /tmp/pg_enterprise

## 3. Crear la red Docker bridge unificada para la arquitectura de la base de datos empresarial
docker network create pg_enterprise_net
```

---

## Instrucciones Paso a Paso

### Paso 1: Inicialización del Contenedor de PostgreSQL con Volúmenes para Tablespaces

**Objetivo**: Instanciar el contenedor de base de datos principal `pg-primary` utilizando la versión exacta de **PostgreSQL 16.2**, montando las rutas del host para simular diferentes unidades de almacenamiento de datos.

**Instrucciones**:

1. Despliegue el contenedor `pg-primary` ejecutando el siguiente comando en su terminal. Note que montaremos las carpetas temporales creadas en el paso anterior dentro de `/var/lib/postgresql/tablespaces/` en el contenedor:

```bash
docker run -d \
  --name pg-primary \
  --net pg_enterprise_net \
  -p 5432:5432 \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD='PostgresAdminPass123!' \
  -v /tmp/pg_enterprise/hot_data:/var/lib/postgresql/tablespaces/hot \
  -v /tmp/pg_enterprise/cold_data:/var/lib/postgresql/tablespaces/cold \
  postgres:16.2
```

2. Verifique que el contenedor se encuentre activo, en ejecución y que los montajes se hayan realizado de forma correcta utilizando la inspección de Docker.

**Resultado esperado**: Un contenedor llamado `pg-primary` levantado correctamente en el puerto local 5432. El comando `docker ps` debe mostrar el estado `Up`.

**Verificación**:
```bash
docker ps --filter "name=pg-primary"
```

---

### Paso 2: Creación de la Base de Datos Global y Configuración de Tablespaces

**Objetivo**: Crear la base de datos global de la empresa (`enterprise_db`) y configurar los objetos físicos *Tablespace* dentro de PostgreSQL, mapeados a las rutas montadas en el contenedor.

**Instrucciones**:

1. Ingrese a la consola interactiva `psql` dentro del contenedor utilizando las credenciales globales del administrador:
```bash
docker exec -it pg-primary psql -U postgres -d postgres
```

2. Desde la terminal interactiva de SQL, cree la base de datos `enterprise_db` y conéctese a ella:
```sql
CREATE DATABASE enterprise_db;
\c enterprise_db;
```

3. Cree los objetos lógicos de almacenamiento (*Tablespaces*) especificando las ubicaciones absolutas internas del contenedor que fueron previamente montadas desde el host:
```sql
CREATE TABLESPACE ts_hot_data LOCATION '/var/lib/postgresql/tablespaces/hot';
CREATE TABLESPACE ts_cold_data LOCATION '/var/lib/postgresql/tablespaces/cold';
```

4. Verifique la correcta existencia y registro de los Tablespaces en el catálogo del sistema.

**Resultado esperado**: Las sentencias SQL se completarán con `CREATE TABLESPACE` y la base de datos de destino cambiará a `enterprise_db`.

**Verificación**:
Ejecute la meta-instrucción en la consola psql para listar los tablespaces registrados:
```sql
\db
```
Debe visualizar los tablespaces `ts_hot_data` y `ts_cold_data` asociados a las rutas especificadas.

---

### Paso 3: Modelado de Particionamiento por Rango de la Tabla de Transacciones Financieras

**Objetivo**: Diseñar y crear una tabla de transacciones de alto volumen particionada por rangos de fecha (`Range Partitioning`). Los datos más antiguos (fríos) se almacenarán en el Tablespace de bajo costo, y los datos recientes (calientes) se mantendrán en el Tablespace rápido.

**Instrucciones**:

1. En la consola interactiva conectada a `enterprise_db`, cree la tabla maestra (o tabla padre) `transactions`. **Regla crítica de diseño en PostgreSQL**: Las restricciones de Clave Primaria (`PRIMARY KEY`) en tablas particionadas deben incluir explícitamente la o las columnas que forman la clave de partición (`transaction_date` en este caso).

```sql
CREATE TABLE transactions (
    transaction_id UUID NOT NULL,
    user_id UUID NOT NULL,
    amount NUMERIC(15, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    transaction_date TIMESTAMP WITH TIME ZONE NOT NULL,
    description TEXT,
    PRIMARY KEY (transaction_id, transaction_date)
) PARTITION BY RANGE (transaction_date);
```

2. Defina y cree las particiones concretas de almacenamiento físico de la base de datos.
   - Crearemos una partición para datos históricos de "Archivo" (por ejemplo, el año 2025) que residirá en el Tablespace de almacenamiento lento/frío (`ts_cold_data`).
   - Crearemos dos particiones para el año en curso 2026 (Semestre 1 y Semestre 2) que residirán en el Tablespace rápido/caliente (`ts_hot_data`).

```sql
-- Partición de archivo histórico de transacciones (2025) -> Almacenamiento Frío
CREATE TABLE transactions_y2025 PARTITION OF transactions
    FOR VALUES FROM ('2025-01-01 00:00:00+00') TO ('2026-01-01 00:00:00+00')
    TABLESPACE ts_cold_data;

-- Partición del primer semestre de 2026 -> Almacenamiento Caliente
CREATE TABLE transactions_y2026_h1 PARTITION OF transactions
    FOR VALUES FROM ('2026-01-01 00:00:00+00') TO ('2026-07-01 00:00:00+00')
    TABLESPACE ts_hot_data;

-- Partición del segundo semestre de 2026 -> Almacenamiento Caliente
CREATE TABLE transactions_y2026_h2 PARTITION OF transactions
    FOR VALUES FROM ('2026-07-01 00:00:00+00') TO ('2027-01-01 00:00:00+00')
    TABLESPACE ts_hot_data;
```

**Resultado esperado**: Creación exitosa de la tabla principal y las tres tablas secundarias hijas de partición asignadas a sus respectivos espacios de disco.

**Verificación**:
```sql
-- Consultar la jerarquía de particiones mapeada en el diccionario de datos de PostgreSQL
\d+ transactions
```
Observe cómo se listan las particiones hijas y sus correspondientes *Tablespaces*.

---

### Paso 4: Modelado de Particionamiento por Hash de la Tabla de Perfiles de Usuario

**Objetivo**: Diseñar y construir una tabla de perfiles de usuario (`user_profiles`) particionada mediante el método hash (`Hash Partitioning`) sobre el identificador único del usuario (`user_id`). Esto distribuirá equitativamente los accesos de escritura simultáneos en disco, evitando cuellos de botella en una única estructura física.

**Instrucciones**:

1. Diseñe la tabla principal de perfiles de usuario estableciendo la clave de partición sobre la columna `user_id`:

```sql
CREATE TABLE user_profiles (
    user_id UUID NOT NULL,
    full_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL,
    PRIMARY KEY (user_id)
) PARTITION BY HASH (user_id);
```

2. Cree las particiones hijas balanceadas. En un modelo balanceado dividiremos la carga en 4 particiones físicas uniformes. Para simplificar, distribuiremos estas particiones uniformemente a través de ambos Tablespaces.

```sql
CREATE TABLE user_profiles_part_0 PARTITION OF user_profiles
    FOR VALUES WITH (MODULUS 4, REMAINDER 0) TABLESPACE ts_hot_data;

CREATE TABLE user_profiles_part_1 PARTITION OF user_profiles
    FOR VALUES WITH (MODULUS 4, REMAINDER 1) TABLESPACE ts_hot_data;

CREATE TABLE user_profiles_part_2 PARTITION OF user_profiles
    FOR VALUES WITH (MODULUS 4, REMAINDER 2) TABLESPACE ts_cold_data;

CREATE TABLE user_profiles_part_3 PARTITION OF user_profiles
    FOR VALUES WITH (MODULUS 4, REMAINDER 3) TABLESPACE ts_cold_data;
```

3. Agregue un índice B-Tree sobre la columna de correo electrónico `email`. Note que los índices creados sobre la tabla padre particionada se propagarán automáticamente a todas las particiones hijas existentes y futuras.

```sql
CREATE UNIQUE INDEX idx_user_profiles_email ON user_profiles (user_id, email);
```

**Resultado esperado**: La tabla `user_profiles` y sus particiones asociadas por residuo matemático (*modulus*) son inicializadas con éxito.

**Verificación**:
Consulte las propiedades de la tabla particionada por Hash en consola psql:
```sql
\d+ user_profiles
```

---

### Paso 5: Implementación de Integridad Referencial y Carga Inicial de Datos

**Objetivo**: Probar la consistencia relacional y verificar el enrutamiento inteligente de datos de PostgreSQL, forzando la inserción e inserción cruzada en las tablas que cuentan con particionamiento lógico.

**Instrucciones**:

1. Inserte un perfil de usuario de prueba en la tabla principal `user_profiles`:
```sql
INSERT INTO user_profiles (user_id, full_name, email, created_at)
VALUES ('9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d', 'Alejandro Gaviria', 'alejandro@empresa.com', '2026-01-15 10:00:00+00');
```

2. Inserte dos registros de transacciones para este usuario correspondientes a diferentes periodos de tiempo (una transacción "antigua" de 2025 y otra "reciente" del segundo semestre de 2026):

```sql
-- Transacción 1 (Histórica - 2025) -> Se enrutará automáticamente a transactions_y2025 en ts_cold_data
INSERT INTO transactions (transaction_id, user_id, amount, currency, transaction_date, description)
VALUES (
    '1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d',
    '9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d',
    1500.50,
    'USD',
    '2025-06-15 14:30:00-06',
    'Pago de servicios de consultoría Q2 2025'
);

-- Transacción 2 (Reciente - Julio 2026) -> Se enrutará automáticamente a transactions_y2026_h2 en ts_hot_data
INSERT INTO transactions (transaction_id, user_id, amount, currency, transaction_date, description)
VALUES (
    '8e7d6c5b-4a3f-2e1d-0c9b-8a7f6e5d4c3b',
    '9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d',
    250.00,
    'USD',
    '2026-07-20 09:00:00-06',
    'Suscripción mensual de software Saas'
);
```

**Resultado esperado**: Ambas inserciones se procesan exitosamente desde la perspectiva del cliente, sin necesidad de saber a qué partición específica corresponde el registro.

**Verificación**:
Compruebe la dispersión física de las inserciones ejecutando consultas dirigidas explícitamente a las particiones hijas y no a la tabla padre:
```sql
SELECT count(*) FROM transactions_y2025;     -- Debe retornar 1
SELECT count(*) FROM transactions_y2026_h1;  -- Debe retornar 0
SELECT count(*) FROM transactions_y2026_h2;  -- Debe retornar 1
```

---

## Validación y Pruebas

Para garantizar que el diseño relacional y físico del motor de bases de datos se encuentra alineado con los requerimientos empresariales de alto rendimiento, ejecute el siguiente conjunto de pruebas estructuradas.

### 1. Prueba de Enrutamiento y Poda de Particiones (Partition Pruning)

Esta prueba asegura que el optimizador de PostgreSQL es capaz de excluir del plan de ejecución aquellas particiones que no contienen la información filtrada en la consulta, ahorrando operaciones de lectura en disco.

Ejecute una consulta explicando el plan físico sobre una búsqueda del primer semestre de 2026:

```sql
EXPLAIN ANALYZE 
SELECT * FROM transactions 
WHERE transaction_date >= '2026-01-01 00:00:00+00' 
  AND transaction_date < '2026-06-01 00:00:00+00';
```

**Salida esperada**: 
En la traza de salida devuelta por `EXPLAIN ANALYZE`, el plan de ejecución debe demostrar el escaneo secuencial o de índice **única y exclusivamente** sobre el nodo secundario `transactions_y2026_h1`. Ninguna mención a `transactions_y2025` o `transactions_y2026_h2` debe figurar en el análisis del planificador físico de PostgreSQL:

```text
Seq Scan on transactions_y2026_h1 transactions  (cost=0.00..X.XX rows=X width=XXX) (actual time=0.0XX..0.0XX rows=0 loops=1)
Filter: ...
Rows Removed by Filter: 0
Planning Time: 0.XXX ms
Execution Time: 0.XXX ms
```

### 2. Prueba de Caso Adversario (Validación del Control de Errores)

Como parte del análisis de robustez de la base de datos empresarial, forzaremos intencionalmente un comportamiento erróneo enviando datos huérfanos fuera de la lógica relacional o del rango de particionamiento.

Ejecute la siguiente inserción correspondiente a una fecha futura lejana (Año 2028) para la cual **no** existe ninguna definición de partición declarada en el motor:

```sql
INSERT INTO transactions (transaction_id, user_id, amount, currency, transaction_date, description)
VALUES (
    'f47ac10b-58cc-4372-a567-0e02b2c3d479',
    '9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d',
    99.99,
    'USD',
    '2028-05-01 12:00:00+00',
    'Prueba de inserción fuera de límites'
);
```

**Salida esperada**:
El motor de base de datos debe lanzar de manera inmediata un error con código de estado SQL clásico abortando la transacción por falta de partición de destino asignada.

```text
ERROR: no partition of relation "transactions" found for row
DETAIL: Partition key of the failing row contains (transaction_date) = (2028-05-01 12:00:00+00).
```

Esta prueba demuestra que la base de datos previene de manera activa el ingreso de datos estructurados que no se adapten a la topología física, manteniendo la integridad referencial absoluta a nivel del motor.

---

## Solución de Problemas

A continuación, se documentan dos de los problemas más frecuentes en la inicialización de tablespaces y restricciones en tablas particionadas, junto con sus causas raíz y soluciones paso a paso.

### Caso 1: Error de Permisos Denegados al Crear el Tablespace
*   **Síntoma**: Al ejecutar `CREATE TABLESPACE ...` se obtiene el siguiente mensaje de error:
    ```text
    ERROR: could not set permissions on directory "/var/lib/postgresql/tablespaces/hot": Permission denied
    ```
*   **Causa**: El motor de PostgreSQL se ejecuta dentro del contenedor de Docker con los privilegios del usuario del sistema nativo `postgres` (identificado por el ID de usuario `999`). Si la carpeta montada en el host no posee permisos absolutos de lectura/escritura para usuarios que no pertenecen al propietario raíz (*root*), el motor de base de datos no podrá inicializar la firma de espacio de tablas.
*   **Resolución**: 
    1. Salga de la consola de `psql` escribiendo `\q` o saliendo del contenedor.
    2. Ejecute los comandos de sistema en la consola de su sistema operativo host para otorgar permisos universales temporales sobre el directorio de pruebas:
       ```bash
       sudo chmod -R 777 /tmp/pg_enterprise
       ```
    3. Reingrese a su base de datos corporativa e intente ejecutar el comando DDL de nuevo.

### Caso 2: Error de Restricción Única/Primary Key en el Particionamiento por Rango
*   **Síntoma**: Al intentar definir una clave primaria simple como `PRIMARY KEY (transaction_id)` al declarar la tabla particionada, obtiene el siguiente error:
    ```text
    ERROR: unique constraint on partitioned table must include all partition key columns
    DETAIL: PRIMARY KEY constraint on table "transactions" lacks column "transaction_date" which is part of the partition key.
    ```
*   **Causa**: En el modelo de particionamiento declarativo nativo de PostgreSQL, cada partición funciona bajo una arquitectura física independiente. Para que el motor pueda verificar y asegurar la unicidad total a través de todas las particiones sin realizar un bloqueo de todo el motor, la clave de partición **debe** formar parte obligatoria del índice de la clave primaria estructurada.
*   **Resolución**: Reestructure el script de creación DDL de la tabla padre para implementar una clave primaria compuesta que combine el identificador relacional lógico y la columna de control de rango de fecha, de la siguiente forma:
    ```sql
    PRIMARY KEY (transaction_id, transaction_date)
    ```

---

## Limpieza

Para liberar la memoria asignada, detener la pila de virtualización y limpiar todos los archivos físicos temporales generados durante esta práctica en su máquina local, ejecute la siguiente secuencia de comandos en su terminal del sistema operativo:

```bash
## 1. Detener e inactivar el contenedor pg-primary
docker stop pg-primary

## 2. Remover el contenedor físico del entorno local
docker rm pg-primary

## 3. Eliminar la red Docker creada específicamente para la práctica empresarial
docker network rm pg_enterprise_net

## 4. Eliminar las carpetas temporales con los datos de tablespaces físicos en el Host
sudo rm -rf /tmp/pg_enterprise
```

---

## Resumen

En este laboratorio, ha diseñado e implementado una arquitectura física de base de datos empresarial utilizando **PostgreSQL 16.2** sobre infraestructura de contenedores. 

A través de esta práctica, aprendió a:
1. **Separar Físicamente el Almacenamiento**: Utilizó *Tablespaces* para definir diferentes ubicaciones en disco (`ts_hot_data` y `ts_cold_data`), un paso indispensable en entornos de alta transaccionalidad para segmentar la velocidad de disco disponible entre almacenamiento activo e histórico.
2. **Modelar con Particiones de Rango**: Implementó la tabla de transacciones distribuyendo rangos semestrales hacia discos rápidos y archivos históricos del año anterior hacia almacenamiento de bajo costo. Esto habilita que consultas de procesamiento histórico utilicen de manera eficiente el motor de optimización mediante el enrutamiento inteligente de consultas (*Partition Pruning*).
3. **Optimizar Cargas con Particiones Hash**: Utilizó algoritmos de módulo para segmentar equitativamente los perfiles de usuario, evitando bloqueos por concurrencia a nivel de índices y páginas de datos.

### Recursos Adicionales para Autoestudio
*   [Documentación Oficial de PostgreSQL 16 sobre Tablespaces (Inglés)](https://www.postgresql.org/docs/16/manage-ag-tablespaces.html)
*   [Manual de Particionamiento Declarativo de PostgreSQL (Inglés)](https://www.postgresql.org/docs/16/ddl-partitioning.html)
*   [Mejores Prácticas en el Diseño Físico de Bases de Datos OLTP](https://wiki.postgresql.org/wiki/Performance_Optimization)

---