# 5 Diseño de una Base de Datos Empresarial

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

# 6 Grandes Volúmenes de Información

## Metadatos

| Atributo | Detalle |
| :--- | :--- |
| **Duración** | 30 minutos |
| **Complejidad** | Alta (Hard) |
| **Nivel Bloom** | Aplicar (Apply) |

---

## Descripción General

En este laboratorio, aplicarás técnicas avanzadas de optimización física y lógica en PostgreSQL Community Edition (16.2) utilizando un conjunto de datos de alta densidad que supera los 5,000,000 de registros. Partiendo del modelo relacional estructurado en la práctica anterior, simularás un entorno de producción masivo para evaluar empíricamente el impacto de **Partition Pruning** y compararás el rendimiento, eficiencia y coste de almacenamiento de índices **B-Tree**, **BRIN (Block Range Index)** e **Índices Parciales**. Aprenderás a interpretar planes de ejecución complejos con `EXPLAIN (ANALYZE, BUFFERS)` para diagnosticar cuellos de botella de I/O en disco y memoria RAM.

---

## Objetivos de Aprendizaje

- [ ] **Poblar** esquemas relacionales a gran escala de forma eficiente utilizando la función generadora `generate_series` y transacciones optimizadas.
- [ ] **Demostrar** empíricamente el mecanismo de *Partition Pruning* (poda de particiones) en consultas analíticas complejas mediante planes de ejecución detallados.
- [ ] **Diseñar, implementar y comparar** el tamaño físico en disco y los tiempos de acceso de índices B-Tree tradicionales frente a índices BRIN y Parciales sobre millones de filas.
- [ ] **Diagnosticar** problemas de planificación de consultas en PostgreSQL causados por funciones no inmutables o discrepancias de tipos de datos que bloquean la optimización del planificador.

---

## Prerrequisitos

- **Conocimientos teóricos:** Entendimiento sólido del particionamiento declarativo en PostgreSQL (por rango), del funcionamiento interno de los índices B-Tree y BRIN (basados en rangos de páginas físicas de 8KB), y de la lectura de salidas de la herramienta `EXPLAIN`.
- **Acceso técnico:**
  - Contenedor de base de datos `pg-primary` activo y accesible en el puerto local `5432` o mediante la red Docker interna `pg_enterprise_net`.
  - Herramienta cliente de base de datos como **DBeaver Community Edition (24.0.0)** o acceso interactivo vía terminal con `psql`.
  - Usuario de conexión: `postgres` con contraseña `PostgresAdminPass123!` en la base de datos `enterprise_db`.

---

## Entorno de Laboratorio

Este laboratorio se ejecuta en la infraestructura de contenedores preconfigurada. Asegúrate de que los servicios básicos cumplan con las especificaciones técnicas requeridas:

### Software Utilizado

| Herramienta / Tecnología | Versión | Arquitectura | Enlace Oficial / Fuente |
| :--- | :--- | :--- | :--- |
| **PostgreSQL Community Edition** | 16.2 | x86_64 | [PostgreSQL Downloads](https://www.postgresql.org/download/) |
| **Debian GNU/Linux** | 12.5 (Bookworm) | x86_64 | [Debian Official Release](https://www.debian.org/releases/bookworm/) |
| **Docker Engine** | 26.0.0 | x86_64 | [Docker Engine Install](https://docs.docker.com/engine/install/) |
| **DBeaver Community Edition** | 24.0.0 | x86_64 | [DBeaver Release Archive](https://dbeaver.io/download/) |

### Parámetros de Conexión y Red

- **Red Docker Bridge:** `pg_enterprise_net`
- **Contenedor Principal:** `pg-primary`
- **Base de Datos Global:** `enterprise_db`
- **Usuario Administrador:** `postgres`
- **Contraseña:** `PostgresAdminPass123!`
- **Puerto de Acceso:** `5432`

### Inicialización del Contenedor (Si es requerido)

Si el contenedor no está activo, puedes iniciarlo con la siguiente instrucción de Docker:

```bash
docker start pg-primary
```

---

## Instrucciones Paso a Paso

### Paso 1: Preparación del Esquema de Pruebas y Particionamiento

**Objetivo:** Crear una tabla particionada por rangos temporales para alojar transacciones financieras masivas, estableciendo un esquema limpio y estructurado de forma idéntica a un entorno empresarial real.

#### Instrucciones

1. Conéctate a la base de datos `enterprise_db` utilizando el cliente `psql` o **DBeaver** con las credenciales provistas.
2. Ejecuta el script DDL de inicialización para crear la tabla maestra particionada `transacciones_rentas` y sus respectivas particiones trimestrales para el año 2025.
3. Asegúrate de incluir una partición por defecto para mitigar errores de inserción imprevistos.

```sql
-- Conectar a la base de datos de la empresa
-- \c enterprise_db

-- Eliminar tablas de pruebas previas si existen
DROP TABLE IF EXISTS transacciones_rentas CASCADE;

-- Crear la tabla maestra particionada por rango sobre la columna fecha_pago
CREATE TABLE transacciones_rentas (
    transaccion_id BIGINT GENERATED ALWAYS AS IDENTITY,
    cliente_id UUID NOT NULL,
    propiedad_id BIGINT NOT NULL,
    monto NUMERIC(12, 2) NOT NULL,
    fecha_pago TIMESTAMP WITHOUT TIME ZONE NOT NULL,
    estado VARCHAR(20) NOT NULL,
    PRIMARY KEY (transaccion_id, fecha_pago)
) PARTITION BY RANGE (fecha_pago);

-- Crear las particiones trimestrales para el año 2025
CREATE TABLE transacciones_2025_q1 PARTITION OF transacciones_rentas
    FOR VALUES FROM ('2025-01-01 00:00:00') TO ('2025-04-01 00:00:00');

CREATE TABLE transacciones_2025_q2 PARTITION OF transacciones_rentas
    FOR VALUES FROM ('2025-04-01 00:00:00') TO ('2025-07-01 00:00:00');

CREATE TABLE transacciones_2025_q3 PARTITION OF transacciones_rentas
    FOR VALUES FROM ('2025-07-01 00:00:00') TO ('2025-10-01 00:00:00');

CREATE TABLE transacciones_2025_q4 PARTITION OF transacciones_rentas
    FOR VALUES FROM ('2025-10-01 00:00:00') TO ('2026-01-01 00:00:00');

-- Partición por defecto para mitigar desbordes de rango
CREATE TABLE transacciones_defecto PARTITION OF transacciones_rentas DEFAULT;
```

#### Resultado esperado

El motor procesará las sentencias SQL secuencialmente y retornará confirmaciones de creación de tablas individuales para cada partición lógica:

```text
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
CREATE TABLE
```

#### Verificación

Verifica que la jerarquía de particiones sea reconocida correctamente por PostgreSQL consultando las vistas del catálogo del sistema:

```sql
SELECT nmsp_parent.nspname AS schema_padre,
       tbl_parent.relname  AS tabla_padre,
       nmsp_child.nspname  AS schema_hijo,
       tbl_child.relname   AS tabla_hijo
FROM pg_inherits
JOIN pg_class tbl_parent ON pg_inherits.inhparent = tbl_parent.oid
JOIN pg_class tbl_child  ON pg_inherits.inhrelid = tbl_child.oid
JOIN pg_namespace nmsp_parent ON tbl_parent.relnamespace = nmsp_parent.oid
JOIN pg_namespace nmsp_child  ON tbl_child.relnamespace = nmsp_child.oid
WHERE tbl_parent.relname = 'transacciones_rentas';
```

Debe retornar un listado con las 5 particiones (`q1`, `q2`, `q3`, `q4`, `defecto`) asociadas a la tabla padre `transacciones_rentas`.

---

### Paso 2: Generación Ultrarrápida e Inserción de 5,000,000 de Registros

**Objetivo:** Poblar de forma uniforme las particiones lógicas mediante un script de inserción masiva optimizado con `generate_series`, simulando un histórico de transacciones transaccional real.

#### Instrucciones

1. Ejecuta la siguiente sentencia `INSERT INTO ... SELECT` diseñada para generar exactamente 5,000,000 de registros distribuidos en el tiempo a lo largo de las particiones del año 2025.
2. Nota cómo usamos operaciones aritméticas y de residuo (`%`) para garantizar que el identificador del cliente, la propiedad, los montos y los estados estén distribuidos uniformemente de forma determinista.
3. El estado de la transacción será `'fallido'` exactamente en un 1% de los datos (cuando el residuo de la serie sea divisible por 100).
4. **IMPORTANTE:** Inmediatamente después de la inserción, ejecuta un `VACUUM ANALYZE` para calcular estadísticas de distribución actualizadas para el Optimizador de Consultas.

```sql
-- Ejecución del cargador de datos masivos en una sola transacción para mejorar rendimiento
BEGIN;

INSERT INTO transacciones_rentas (cliente_id, propiedad_id, monto, fecha_pago, estado)
SELECT 
    'a1b2c3d4-e5f6-7a8b-9c0d-1e2f3a4b5c6d'::uuid AS cliente_id,
    (1 + (g % 10000))::bigint AS propiedad_id,
    (50.00 + (g % 1000) * 1.50)::numeric(12,2) AS monto,
    '2025-01-01 00:00:00'::timestamp + (g * interval '6.3072 seconds') AS fecha_pago,
    CASE WHEN g % 100 = 0 THEN 'fallido' ELSE 'completado' END AS estado
FROM generate_series(1, 5000000) g;

COMMIT;

-- Recolección inmediata de estadísticas físicas
VACUUM ANALYZE transacciones_rentas;
```

#### Resultado esperado

La inserción masiva tardará entre 15 y 30 segundos dependiendo de la velocidad de escritura de tu disco SSD NVMe / SATA. La consola psql retornará:

```text
BEGIN
INSERT 0 5000000
COMMIT
VACUUM
```

#### Verificación

Valida la distribución física uniforme de las filas consultando cuántos registros terminaron en cada partición física concreta:

```sql
SELECT tableoid::regclass AS particion_fisica, COUNT(*) AS total_filas
FROM transacciones_rentas
GROUP BY tableoid::regclass
ORDER BY particion_fisica;
```

Debe retornar aproximadamente 1,250,000 registros por cada una de las 4 particiones principales (Q1 a Q4) del año 2025, y 0 registros en la partición de defecto.

---

### Paso 3: Evaluación y Demostración Práctica de Partition Pruning

**Objetivo:** Comprobar que PostgreSQL descarta por completo la lectura de archivos de disco correspondientes a particiones que no contienen datos relevantes para los filtros especificados en la consulta (Poda de Particiones o *Partition Pruning*).

#### Instrucciones

1. Ejecuta una consulta con `EXPLAIN (ANALYZE, BUFFERS, COSTS OFF)` que recupere el total y promedio de transacciones correspondientes exclusivamente a la primera mitad del año (de enero a marzo, Q1).
2. Analiza el plan de ejecución resultante y busca las particiones físicas involucradas.
3. Desactiva temporalmente el mecanismo de poda de particiones mediante la variable de configuración de sesión `enable_partition_pruning = off`, vuelve a ejecutar el plan y compara la diferencia en bloques leídos (`shared hit/read`) y tiempo de respuesta.

```sql
-- Asegurar que la poda de particiones esté activa (Valor por defecto de producción)
SET enable_partition_pruning = on;

-- Consultar con EXPLAIN ANALYZE el comportamiento del planificador
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF)
SELECT COUNT(*), AVG(monto)
FROM transacciones_rentas
WHERE fecha_pago >= '2025-01-15 00:00:00' AND fecha_pago < '2025-02-15 00:00:00';
```

Ahora desactiva la directiva para comparar:

```sql
-- Desactivar el comportamiento nativo
SET enable_partition_pruning = off;

EXPLAIN (ANALYZE, BUFFERS, COSTS OFF)
SELECT COUNT(*), AVG(monto)
FROM transacciones_rentas
WHERE fecha_pago >= '2025-01-15 00:00:00' AND fecha_pago < '2025-02-15 00:00:00';

-- Restaurar el valor por defecto para las pruebas subsiguientes
SET enable_partition_pruning = on;
```

#### Resultado esperado

- **Con `enable_partition_pruning = on`**: El plan de ejecución debe mostrar únicamente un escaneo secuencial (`Seq Scan`) en la tabla física de la partición `transacciones_2025_q1`. El resto de las tablas/particiones (`q2`, `q3`, `q4`, `defecto`) ni siquiera figuran en la salida del plan de ejecución.
- **Con `enable_partition_pruning = off`**: PostgreSQL se ve obligado a mapear un plan del tipo `Append` sobre **todas y cada una** de las particiones creadas, realizando escaneos secuenciales o condicionales redundantes que multiplican la carga en memoria compartida (`shared buffers`), incluso si devuelven 0 filas en las demás particiones.

#### Verificación

Verifica que el bloque `Buffers` con la directiva activada muestre lecturas únicamente para `transacciones_2025_q1`:

```text
-- Fragmento de salida esperada con Pruning ON:
Aggregate (actual time=X.XX..Y.YY rows=1 loops=1)
  Buffers: shared hit=NNN
  ->  Seq Scan on transacciones_2025_q1 (actual time=X..Y rows=428571 loops=1)
        Filter: (fecha_pago >= '2025-01-15 00:00:00'::timestamp...)
```

---

### Paso 4: Implementación de Escenario Plano de Comparación y Creación de Índices

**Objetivo:** Crear una tabla "plana" (no particionada) con el mismo set de 5M de datos para comparar científicamente y sin ruido el comportamiento físico de almacenamiento y búsqueda entre un índice B-Tree, un índice BRIN y un índice Parcial.

#### Instrucciones

1. Crea la tabla `transacciones_flat` e inserta el mismo set de datos de forma lineal. Esto nos permitirá aislar el impacto de los índices sin la interferencia lógica de las uniones y subdivisiones del particionamiento.
2. Ejecuta un `VACUUM ANALYZE` sobre esta nueva tabla para estabilizar el diseño físico.

```sql
-- Crear tabla de comparación plana
CREATE TABLE transacciones_flat (
    transaccion_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    cliente_id UUID NOT NULL,
    propiedad_id BIGINT NOT NULL,
    monto NUMERIC(12, 2) NOT NULL,
    fecha_pago TIMESTAMP WITHOUT TIME ZONE NOT NULL,
    estado VARCHAR(20) NOT NULL
);

-- Copiar los datos directamente para mantener equivalencia exacta
INSERT INTO transacciones_flat (cliente_id, propiedad_id, monto, fecha_pago, estado)
SELECT cliente_id, propiedad_id, monto, fecha_pago, estado
FROM transacciones_rentas;

-- Actualizar estadísticas de sistema para asegurar optimización correcta
VACUUM ANALYZE transacciones_flat;
```

A continuación, crearemos tres tipos de índices diferenciados:

1. **Índice B-Tree Estándar:** Sobre la columna temporal secuencial `fecha_pago`.
2. **Índice BRIN (Block Range Index):** Sobre la misma columna `fecha_pago`, configurando `pages_per_range = 64` (un bloque de indexación por cada 64 páginas del disco duro).
3. **Índice Parcial B-Tree:** Sobre la columna de estado únicamente filtrando por filas con valor `'fallido'`.

```sql
-- 1. Crear Índice B-Tree Estándar
CREATE INDEX idx_flat_btree_fecha ON transacciones_flat USING btree (fecha_pago);

-- 2. Crear Índice BRIN optimizando el tamaño físico por rangos correlacionados de páginas
CREATE INDEX idx_flat_brin_fecha ON transacciones_flat USING brin (fecha_pago) WITH (pages_per_range = 64);

-- 3. Crear Índice Parcial B-Tree enfocado en casos anómalos o de bajo volumen (1% del total)
CREATE INDEX idx_flat_partial_fallidos ON transacciones_flat USING btree (estado) WHERE estado = 'fallido';
```

#### Resultado esperado

La creación del índice B-Tree tardará unos segundos más que la creación del índice BRIN. El índice BRIN se genera de forma casi instantánea debido a que no tiene que ordenar elementos secuencialmente, sino evaluar valores mínimos y máximos por rango físico de página de datos.

```text
CREATE TABLE
INSERT 0 5000000
VACUUM
CREATE INDEX (B-Tree en ~5s)
CREATE INDEX (BRIN en ~0.5s)
CREATE INDEX (Parcial en ~0.2s)
```

#### Verificación

Consulta el tamaño en disco ocupado por cada índice utilizando las funciones integradas de PostgreSQL:

```sql
SELECT relname AS nombre_objeto,
       pg_size_pretty(pg_relation_size(oid)) AS tamano_legible,
       pg_relation_size(oid) AS bytes_absolutos
FROM pg_class
WHERE relname IN ('transacciones_flat', 'idx_flat_btree_fecha', 'idx_flat_brin_fecha', 'idx_flat_partial_fallidos')
ORDER BY bytes_absolutos DESC;
```

Registra el tamaño en Megabytes (MB) o Kilobytes (KB) devuelto por esta consulta para tu posterior análisis en la sección de validación.

---

### Paso 5: Pruebas de Rendimiento y Análisis Comparativo

**Objetivo:** Analizar cómo responde el optimizador frente a diferentes búsquedas utilizando los índices creados, demostrando el beneficio y coste de cada enfoque arquitectónico.

#### Instrucciones

1. Ejecuta una consulta de rango de tiempo usando el índice B-Tree tradicional. Fuerza el uso de B-Tree desactivando el índice BRIN de forma temporal en la transacción (o examinando el planificador).
2. Ejecuta una consulta similar optimizada para usar el índice BRIN.
3. Evalúa la diferencia entre un `Index Scan` de B-Tree y un `Bitmap Index Scan` de BRIN.
4. Consulta el estado de las transacciones fallidas para verificar el comportamiento de alta velocidad del Índice Parcial.

```sql
-- Escenario 1: Consulta de rango acotado usando B-Tree
-- Buscamos transacciones de un único día (muy selectiva)
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF)
SELECT COUNT(*) 
FROM transacciones_flat 
WHERE fecha_pago >= '2025-05-10 00:00:00' AND fecha_pago <= '2025-05-10 23:59:59';

-- Escenario 2: Consulta analítica de rango amplio usando BRIN
-- Nota: Para que el optimizador elija BRIN frente a B-Tree o SeqScan,
-- forzamos una búsqueda de rango más amplia donde la lectura secuencial de páginas agrupadas destaque.
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF)
SELECT COUNT(*) 
FROM transacciones_flat 
WHERE fecha_pago >= '2025-03-01 00:00:00' AND fecha_pago <= '2025-08-31 23:59:59';

-- Escenario 3: Búsqueda selectiva de registros específicos con el índice parcial
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF)
SELECT COUNT(*) 
FROM transacciones_flat 
WHERE estado = 'fallido';
```

#### Resultado esperado

- El escenario 1 utilizará un **Index Scan** rápido de B-Tree en milisegundos de un solo dígito.
- El escenario 2 utilizará un **Bitmap Index Scan** usando el índice BRIN si la selectividad y la correlación física son ideales.
- El escenario 3 utilizará el índice parcial `idx_flat_partial_fallidos` de manera instantánea, requiriendo muy pocos `shared hit` de buffers ya que solo contiene los punteros al 1% de la tabla.

#### Verificación

En la salida del escenario 3, verifica que la consulta ejecute un **Index Only Scan** o **Index Scan** utilizando específicamente la relación `idx_flat_partial_fallidos`:

```text
->  Bitmap Index Scan on idx_flat_partial_fallidos (actual time=X.XX..Y.YY rows=NNNNN loops=1)
      Index Cond: (estado = 'fallido'::text)
```

---

## Validación y Pruebas

En esta sección, evaluarás el resultado del laboratorio bajo escenarios de estrés controlados y casos conflictivos.

### Cuadro de Eficiencia de Índices (Métricas Reales)

Ejecuta la siguiente plantilla SQL para recuperar un informe consolidado del espacio de almacenamiento para tu reporte:

```sql
SELECT 
    indrelid::regclass AS tabla_asociada,
    indexrelid::regclass AS nombre_indice,
    pg_size_pretty(pg_relation_size(indexrelid)) AS tamano_indexado,
    pg_relation_size(indexrelid) AS bytes_totales,
    CASE 
        WHEN pg_relation_size(indexrelid) < 500000 THEN 'Ultra-Compacto (BRIN/Parcial)'
        WHEN pg_relation_size(indexrelid) BETWEEN 500000 AND 20000000 THEN 'Moderado'
        ELSE 'Grande (B-Tree Estándar)'
    END AS tipo_escala
FROM pg_index
WHERE indrelid = 'transacciones_flat'::regclass;
```

**Análisis de Resultados Esperados:**
*   El índice **B-Tree** (`idx_flat_btree_fecha`) ocupará entre **100 MB y 115 MB** en disco ya que almacena un puntero por cada una de las 5,000,000 de filas.
*   El índice **BRIN** (`idx_flat_brin_fecha`) ocupará típicamente menos de **100 KB** (un ahorro superior al 99.9% de almacenamiento físico) debido a que solo guarda los rangos de fechas por cada bloque de 64 páginas físicas.
*   El índice **Parcial** (`idx_flat_partial_fallidos`) se mantendrá extremadamente compacto (aproximadamente **1.1 MB**) ya que solo almacena punteros de 50,000 filas con el estado `'fallido'`.

---

### Caso de Prueba Adversario: Bloqueo de Partition Pruning por Funciones No Inmutables

**Propósito:** Probar y comprender una limitación crítica del optimizador de consultas de PostgreSQL que suele degradar el rendimiento en producción de forma imprevista.

Cuando utilizas funciones dinámicas no estables ni inmutables como `now()`, `statement_timestamp()`, o haces conversiones implícitas de tipos de datos incomparables dentro de la cláusula `WHERE`, el optimizador **no puede evaluar estáticamente las particiones en tiempo de planificación** y se ve obligado a escanear **todas** las particiones, anulando por completo el Partition Pruning.

#### Ejecución de la prueba conflictiva

Ejecuta la siguiente consulta con `EXPLAIN ANALYZE` simulando un sistema que intenta filtrar de manera dinámica con conversión a tipo de dato de zona horaria (`timestamptz`) cuando la columna fue declarada como `timestamp` (sin zona horaria):

```sql
-- Intento de consulta que rompe la optimización estática debido a conversión implícita de tipos
EXPLAIN (ANALYZE, BUFFERS, COSTS OFF)
SELECT COUNT(*) 
FROM transacciones_rentas 
-- Se compara un TIMESTAMP WITHOUT TIME ZONE con una expresión variable con zona horaria (TIMESTAMPTZ)
WHERE fecha_pago >= ('2025-01-01 00:00:00'::timestamptz AT TIME ZONE 'UTC');
```

#### Análisis del Comportamiento Adversario

1. Al inspeccionar el plan de ejecución obtenido, notarás que PostgreSQL **no realiza Partition Pruning estático de forma inmediata**.
2. Dependiendo de la versión del planificador, resolverá la coacción de tipos mediante un nodo `Append` ejecutando filtros en todas las particiones hijas en tiempo de ejecución (`Subquery Scan` o escaneos secuenciales múltiples).
3. **Lección de ingeniería de datos:** Para asegurar que el motor descarte particiones en la primera fase de análisis, las condiciones del filtro `WHERE` deben comparar columnas indexadas/particionadas directamente con constantes literales del mismo tipo de datos exacto o expresiones explícitamente inmutables.

---

## Solución de Problemas

Aquí se presentan dos de los problemas más comunes identificados durante pruebas masivas con `generate_series` e indexación de alta densidad en PostgreSQL.

### Problema 1: Inserción Masiva Abortada con Error "out of shared memory"

- **Síntomas:** Durante la inserción de los 5,000,000 de registros en la tabla particionada, la transacción se interrumpe abruptamente y retorna un mensaje de error:
  `ERROR: out of shared memory. HINT: You might need to increase max_locks_per_transaction.`
- **Causa:** Cada partición en PostgreSQL actúa técnicamente como una tabla física independiente. Al ejecutar consultas masivas complejas o modificaciones masivas dentro de una transacción única que involucre muchas particiones y restricciones, PostgreSQL debe adquirir bloqueos exclusivos en cada objeto. Si el número de particiones es alto y se supera el límite asignado en la memoria interna de la tabla de bloqueos (`shared buffers` de control), la transacción falla.
- **Resolución:** 
  1. Incrementa el parámetro `max_locks_per_transaction` en el archivo de configuración `postgresql.conf` de tu servidor a un valor mayor (ej. `128` o `256`).
  2. Alternativamente, puedes segmentar la inserción de datos en bloques más pequeños ejecutando cargas parciales por trimestre o mes, reduciendo la cantidad de tablas concurrentes bloqueadas dentro de la misma transacción.

---

### Problema 2: El Índice BRIN No es Utilizado (El planificador prefiere Seq Scan)

- **Síntomas:** Al ejecutar la consulta analítica del paso 5 (Escenario 2), el planificador de PostgreSQL ignora el índice `idx_flat_brin_fecha` y opta por un escaneo secuencial completo (`Seq Scan`) de la tabla de 5M de registros, tardando más tiempo de lo esperado.
- **Causa:** Los índices BRIN son eficientes **únicamente** si los datos físicos están altamente correlacionados u ordenados en el disco duro respecto a la columna indexada (en este caso, la fecha de pago). Si los datos fueron insertados de forma aleatoria o se ha realizado un proceso de reordenamiento desordenado (`UPDATEs` masivos aleatorios), el rango de valores (mínimo y máximo) de cada bloque de páginas físicas se solapará, haciendo que el índice BRIN sea inútil a ojos del optimizador.
- **Resolución:**
  1. Reordena físicamente la tabla conforme a la columna del índice utilizando el comando `CLUSTER`:
     ```sql
     -- Reordenar la tabla físicamente conforme al índice B-Tree temporal para reordenar las páginas
     CLUSTER transacciones_flat USING idx_flat_btree_fecha;
     -- Recalcular estadísticas físicas inmediatamente
     ANALYZE transacciones_flat;
     ```
  2. Al volver a ejecutar la consulta analítica amplia, verás que el planificador selecciona de inmediato el índice BRIN mediante un plan `Bitmap Index Scan`.

---

## Limpieza

Para liberar el almacenamiento físico en disco utilizado por las estructuras de prueba masivas, ejecuta las siguientes instrucciones de limpieza en tu consola de base de datos. Esto es fundamental para evitar la saturación del almacenamiento persistente del contenedor:

```sql
-- Desconectarse de cualquier sesión activa y ejecutar la limpieza de objetos
DROP TABLE IF EXISTS transacciones_flat CASCADE;
DROP TABLE IF EXISTS transacciones_rentas CASCADE;

-- Limpieza física del almacenamiento (liberar espacio en disco a nivel de sistema operativo)
VACUUM FULL;
```

---

## Resumen

En este laboratorio práctico has implementado técnicas clave de ingeniería de datos y administración de bases de datos PostgreSQL a gran escala:

1. **Generación Masiva Determinista:** Aprendiste a poblar tablas de manera rápida con `generate_series` de forma transaccional directa, evitando bloqueos por iteraciones ineficientes a nivel de software de aplicación.
2. **Partition Pruning:** Comprobaste la enorme diferencia de rendimiento que ofrece el descarte inteligente de particiones en consultas de rango temporal, ahorrando cientos de miles de accesos de lectura a disco de manera transparente.
3. **Análisis de Tipos de Índices:** 
   - El índice **B-Tree** es rápido para búsquedas de punto y alta selectividad, pero a costa de un enorme tamaño en disco (más de 100 MB).
   - El índice **BRIN** demostró ser una alternativa extremadamente compacta (menos de 100 KB, ahorro del 99.9%) y óptima para consultas de reportes analíticos e históricos de gran tamaño con datos ordenados temporalmente en disco.
   - El índice **Parcial** demostró que restringir el alcance de indexación a filtros de negocio específicos (como estados de falla) ahorra espacio masivo y acelera consultas transaccionales de control.

### Recursos Adicionales

*   [Documentación Oficial de PostgreSQL sobre Particionamiento de Tablas](https://www.postgresql.org/docs/16/ddl-partitioning.html)
*   [Guía de PostgreSQL sobre Índices BRIN (Block Range Indexes)](https://www.postgresql.org/docs/16/brin.html)
*   [Explicación de Planes de Consulta con EXPLAIN ANALYZE](https://www.postgresql.org/docs/16/using-explain.html)
