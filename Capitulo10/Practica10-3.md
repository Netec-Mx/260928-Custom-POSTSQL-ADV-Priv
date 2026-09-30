# 10.3 Gestión de Migraciones con Flyway

## Metadatos

| Campo | Detalle |
| :--- | :--- |
| **Duración** | 90 minutos |
| **Complejidad** | Media |
| **Nivel de Bloom** | Aplicar (Apply) |

## Descripción General

En este laboratorio práctico, aprenderás a gestionar el ciclo de vida del esquema de una base de datos relacional PostgreSQL utilizando **Flyway Community Edition 10.10.0**. Configurarás Flyway CLI para interactuar de forma segura con un esquema aislado (`inventory_schema`) dentro de la base de datos empresarial preexistente (`enterprise_db`). 

A lo largo de la práctica, estructurarás scripts de migración basados exclusivamente en SQL usando convenciones de nomenclatura estrictas para migraciones versionadas (V__) y repetibles (R__). Adicionalmente, experimentarás un escenario realista de falla por desviación del esquema (*schema drift*), alterando intencionalmente un archivo de migración ya aplicado para provocar una incompatibilidad de suma de verificación (*checksum validation error*), y resolverás este conflicto de manera determinista utilizando el comando `flyway repair`.

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
* [ ] Configurar el archivo de propiedades `flyway.conf` para interactuar con un esquema aislado en un contenedor PostgreSQL 16.2.
* [ ] Desarrollar y ordenar cronológicamente scripts de migración SQL versionados (`V__`) siguiendo un estándar de nomenclatura estricto.
* [ ] Implementar migraciones repetibles (`R__`) para gestionar objetos sin estado tales como vistas y funciones en PostgreSQL.
* [ ] Diagnosticar, depurar y resolver conflictos de validación de sumas de verificación (*checksums*) utilizando las utilidades `flyway info` y `flyway repair`.

## Prerrequisitos

Para completar este laboratorio con éxito, es indispensable contar con:
1. **Conocimientos teóricos**: Comprensión clara de la diferencia entre el enfoque basado en estado (*state-based*) y el basado en migraciones (*migration-based*), así como de las transacciones DDL en PostgreSQL.
2. **Acceso al sistema**: Consola de comandos (*bash* o *zsh*) en un sistema operativo basado en Unix/Linux (o emulación compatible en Windows como WSL2).
3. **Contenedor PostgreSQL activo**: Haber completado el laboratorio previo y tener levantado el contenedor Docker `pg-primary` (PostgreSQL 16.2) en la red bridge `pg_enterprise_net`, con la base de datos `enterprise_db` disponible.
4. **DBeaver Community Edition 24.0.0**: Instalado localmente en la máquina del estudiante para la inspección física de los objetos creados.

## Entorno de Laboratorio

El entorno de ejecución consta de los siguientes componentes tecnológicos con sus respectivas versiones y licencias:

| Tecnología / Herramienta | Versión Exacta | Licencia | Origen Oficial / Descarga |
| :--- | :--- | :--- | :--- |
| **PostgreSQL Community Edition** | 16.2 (Debian Bookworm) | PostgreSQL License | [Docker Hub - official postgres](https://hub.docker.com/_/postgres) |
| **Flyway Community Edition** | 10.10.0 (Command Line) | Apache License 2.0 | [Redgate Flyway Download](https://flywaydb.org/download/community) |
| **DBeaver Community Edition** | 24.0.0 | GPLv2 | [DBeaver Official Page](https://dbeaver.io/download/) |

### Parámetros de Configuración Globales

* **Red Docker Bridge**: `pg_enterprise_net`
* **Nombre del Contenedor**: `pg-primary`
* **Nombre de la Base de Datos**: `enterprise_db`
* **Usuario Maestro**: `postgres`
* **Contraseña del Usuario**: `PostgresAdminPass123!`
* **Puerto de Conexión Externo**: `5432`

### Inicialización y Verificación del Contenedor

Antes de iniciar con Flyway, asegúrate de que el contenedor de PostgreSQL esté en ejecución y conectado a la red correcta ejecutando el siguiente comando en tu terminal:

```bash
docker ps --filter "name=pg-primary" --format "table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Ports}}"
```

*Nota: Si el contenedor no se encuentra encendido, inícialo con `docker start pg-primary`.*

---

## Instrucciones Paso a Paso

### Paso 1: Preparación de la Estructura de Directorios y Descarga de Flyway

**Objetivo**: Establecer un entorno de trabajo organizado y configurar la versión exacta de Flyway Community Edition CLI en el espacio local.

#### Instrucciones

1. Crea el árbol de directorios para el proyecto en tu máquina local:
   ```bash
   mkdir -p ~/flyway-lab/sql
   mkdir -p ~/flyway-lab/conf
   cd ~/flyway-lab
   ```

2. Descarga la versión específica de **Flyway Community Edition 10.10.0** para tu arquitectura de sistema operativo (ejemplo basado en Linux x86_64). Si utilizas macOS o Windows, descarga el binario equivalente desde el portal de descargas oficial de Redgate.
   ```bash
   # Descarga del tarball oficial
   wget -q https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/10.10.0/flyway-commandline-10.10.0-linux-x64.tar.gz -O flyway.tar.gz
   
   # Descompresión de la herramienta en el directorio actual
   tar -xzf flyway.tar.gz --strip-components=1 -C ~/flyway-lab
   
   # Eliminar el empaquetado descargado para mantener limpio el entorno
   rm flyway.tar.gz
   ```

3. Verifica que la instalación de Flyway es operativa y responde con la versión exacta indicada en los metadatos:
   ```bash
   ./flyway -v
   ```

#### Resultado esperado
Al ejecutar el comando de verificación de versión, la salida en consola debe confirmar la carga de Flyway Community 10.10.0:

```text
Flyway Community Edition 10.10.0 by Redgate
```

#### Verificación
Asegúrate de que la estructura de directorios en tu ruta local `~/flyway-lab` contenga el ejecutable `flyway`, el subdirectorio de configuración `conf` y el directorio de migraciones `sql`.

---

### Paso 2: Creación del Esquema Aislado y Configuración de Conexión

**Objetivo**: Crear un esquema lógico dedicado en la base de datos PostgreSQL para encapsular las migraciones, evitando colisiones con otras aplicaciones de la base de datos global `enterprise_db`, y parametrizar la conexión en Flyway.

#### Instrucciones

1. Conéctate a la base de datos mediante la herramienta `psql` dentro del contenedor Docker para crear el esquema inicial:
   ```bash
   docker exec -it pg-primary psql -U postgres -d enterprise_db -c "CREATE SCHEMA IF NOT EXISTS inventory_schema;"
   ```

2. Crea el archivo de propiedades de Flyway `~/flyway-lab/conf/flyway.conf`. Este archivo indicará a Flyway dónde conectarse, las credenciales del usuario maestro y qué esquemas gestionar. Utiliza tu editor de texto preferido (como `nano` o `vim`) para generar el archivo con el siguiente contenido exacto:

   ```properties
   # ~/flyway-lab/conf/flyway.conf
   flyway.url=jdbc:postgresql://localhost:5432/enterprise_db
   flyway.user=postgres
   flyway.password=PostgresAdminPass123!
   flyway.schemas=inventory_schema
   flyway.defaultSchema=inventory_schema
   flyway.locations=filesystem:./sql
   flyway.connectRetries=10
   ```

3. Valida que Flyway se conecta exitosamente a la base de datos consultando el estado del esquema mediante la instrucción `info`:
   ```bash
   ./flyway -configFiles=conf/flyway.conf info
   ```

#### Resultado esperado
La salida del comando `info` debe mostrar una estructura de tabla limpia, indicando que el esquema está vacío y que aún no existe una tabla de historial de migraciones (`flyway_schema_history`):

```text
Database: jdbc:postgresql://localhost:5432/enterprise_db (PostgreSQL 16.2)
Schema history table "inventory_schema"."flyway_schema_history" does not exist yet; Flyway will create it on demand.

No migrations found
```

#### Verificación
En DBeaver, conéctate a la base de datos `enterprise_db` utilizando el puerto 5432. Despliega el nodo de esquemas y confirma la existencia de `inventory_schema` sin tablas asociadas.

---

### Paso 3: Diseño y Despliegue de Migraciones Versionadas (V1 y V2)

**Objetivo**: Escribir y ejecutar las primeras dos migraciones de base de datos siguiendo la nomenclatura secuencial estricta (`V<Versión>__<Descripción>.sql`) y verificando el estado de la tabla de metadatos históricos.

#### Instrucciones

1. Escribe la primera migración versionada (`V1__init_inventory.sql`) que se encargará de crear la tabla inicial de productos. Crea el archivo dentro del directorio `~/flyway-lab/sql/`:

   ```sql
   -- ~/flyway-lab/sql/V1__init_inventory.sql
   -- Crear tabla de productos de inventario
   CREATE TABLE productos (
       id SERIAL PRIMARY KEY,
       nombre VARCHAR(100) NOT NULL,
       sku VARCHAR(50) UNIQUE NOT NULL,
       precio NUMERIC(10, 2) NOT NULL,
       stock INT DEFAULT 0
   );
   ```

2. Escribe una segunda migración versionada (`V2__add_audit_columns.sql`) que simula una alteración incremental de esquema solicitada por el equipo de auditoría:

   ```sql
   -- ~/flyway-lab/sql/V2__add_audit_columns.sql
   -- Añadir columnas de auditoría a la tabla productos
   ALTER TABLE productos 
   ADD COLUMN creado_por VARCHAR(50) DEFAULT 'system',
   ADD COLUMN fecha_creacion TIMESTAMP DEFAULT CURRENT_TIMESTAMP;
   ```

3. Ejecuta el proceso de migración de Flyway para aplicar los cambios en la base de datos:
   ```bash
   ./flyway -configFiles=conf/flyway.conf migrate
   ```

4. Consulta nuevamente la información del esquema para rastrear las migraciones aplicadas:
   ```bash
   ./flyway -configFiles=conf/flyway.conf info
   ```

#### Resultado esperado
La consola de salida para `migrate` debe confirmar la aplicación exitosa de ambos archivos SQL dentro de transacciones independientes:

```text
Successfully validated 2 migrations (execution time 00:00.045s)
Creating Schema History table "inventory_schema"."flyway_schema_history" ...
Migrating schema "inventory_schema" to version "1 - init inventory"
Migrating schema "inventory_schema" to version "2 - add audit columns"
Successfully applied 2 migrations to schema "inventory_schema" (execution time 00:00.120s)
```

Al consultar `info`, el estado de las migraciones debe registrar el estado `Success`:

```text
+-----------+---------+-------------------+------+---------------------+---------+----------+
| Category  | Version | Description       | Type | Installed On        | State   | Undoable |
+-----------+---------+-------------------+------+---------------------+---------+----------+
| Versioned | 1       | init inventory    | SQL  | 2024-10-24 10:00:00 | Success | No       |
| Versioned | 2       | add audit columns | SQL  | 2024-10-24 10:00:02 | Success | No       |
+-----------+---------+-------------------+------+---------------------+---------+----------+
```

#### Verificación
Usa DBeaver para consultar la estructura de la tabla `productos` dentro del esquema `inventory_schema`. Confirma que las columnas `creado_por` y `fecha_creacion` existen y tienen los valores por defecto configurados en la migración `V2`. Adicionalmente, examina la tabla recién creada `flyway_schema_history` para confirmar que contiene las dos filas correspondientes a las migraciones ejecutadas con sus respectivas sumas de verificación (*checksums*).

---

### Paso 4: Implementación de una Migración Repetible (R__)

**Objetivo**: Diseñar y desplegar una migración repetible (*Repeatable Migration*) para la creación de una vista de auditoría, comprendiendo cómo Flyway reacciona e instala de nuevo estos scripts solo cuando detecta un cambio físico en su contenido.

#### Instrucciones

1. Crea un script de migración repetible denominado `R__create_inventory_view.sql` en el directorio `~/flyway-lab/sql/`. Este tipo de migraciones no lleva versión secuencial numérica (`V`) sino la letra `R` fija, y se ejecuta al final de todas las migraciones versionadas:

   ```sql
   -- ~/flyway-lab/sql/R__create_inventory_view.sql
   -- Vista del estado de inventario valorado para reportes rápidos
   CREATE OR REPLACE VIEW vista_inventario_valorado AS
   SELECT 
       id,
       nombre,
       sku,
       stock,
       precio,
       (stock * precio) AS valor_total_inventario,
       fecha_creacion
   FROM productos;
   ```

2. Aplica la migración repetible usando el comando de migración:
   ```bash
   ./flyway -configFiles=conf/flyway.conf migrate
   ```

3. Ahora, simula un cambio en el requerimiento del negocio. Modifica la vista para incluir también el campo de auditoría `creado_por`. Edita el archivo `~/flyway-lab/sql/R__create_inventory_view.sql` reemplazando su contenido por el siguiente:

   ```sql
   -- ~/flyway-lab/sql/R__create_inventory_view.sql
   -- Vista del estado de inventario valorado con adición del auditor
   CREATE OR REPLACE VIEW vista_inventario_valorado AS
   SELECT 
       id,
       nombre,
       sku,
       stock,
       precio,
       (stock * precio) AS valor_total_inventario,
       creado_por,
       fecha_creacion
   FROM productos;
   ```

4. Ejecuta nuevamente la migración y observa cómo actúa Flyway sobre el script repetible modificado:
   ```bash
   ./flyway -configFiles=conf/flyway.conf migrate
   ```

#### Resultado esperado
En la primera ejecución de `migrate` con el archivo repetible, la salida indicará que la migración ha sido aplicada:

```text
Successfully validated 3 migrations (execution time 00:00.035s)
Current version of schema "inventory_schema": 2
Migrating schema "inventory_schema" with repeatable migration "create inventory view"
Successfully applied 1 migration to schema "inventory_schema" (execution time 00:00.048s)
```

En la segunda ejecución (después de modificar el archivo), Flyway recalcula la firma criptográfica (checksum) del archivo repetible. Detecta que no coincide con el guardado en `flyway_schema_history`, lo que desencadena una actualización automática de la vista:

```text
Successfully validated 3 migrations (execution time 00:00.040s)
Current version of schema "inventory_schema": 2
Migrating schema "inventory_schema" with repeatable migration "create inventory view" (re-applying)
Successfully applied 1 migration to schema "inventory_schema" (execution time 00:00.052s)
```

#### Verificación
Ejecuta la siguiente consulta SQL en DBeaver para certificar que la vista del esquema se reconstruyó correctamente y que contiene el nuevo campo `creado_por`:

```sql
SELECT * FROM inventory_schema.vista_inventario_valorado;
```

Debe retornar un conjunto de columnas idénticas a las definidas en la segunda versión de tu migración repetible.

---

### Paso 5: Simulación de Conflictos por Checksum (Schema Drift) y Reparación del Historial

**Objetivo**: Experimentar de primera mano uno de los fallos más habituales en la integración continua de bases de datos: la alteración ilegal de un script SQL histórico que ya fue desplegado en entornos superiores, y aprender a recuperarse de forma segura utilizando la utilidad `repair`.

#### Instrucciones

1. Simula una edición no autorizada sobre una migración ya aplicada. Imagina que un desarrollador de manera negligente abrió el archivo `~/flyway-lab/sql/V1__init_inventory.sql` y alteró su contenido físico añadiendo, por ejemplo, un comentario o cambiando las mayúsculas de una sentencia. 
   Modifica el archivo `~/flyway-lab/sql/V1__init_inventory.sql` para simular este incidente de la siguiente manera:

   ```sql
   -- ~/flyway-lab/sql/V1__init_inventory.sql
   -- COMENTARIO NO AUTORIZADO AÑADIDO A POSTERIORI
   CREATE TABLE productos (
       id SERIAL PRIMARY KEY,
       nombre VARCHAR(100) NOT NULL,
       sku VARCHAR(50) UNIQUE NOT NULL,
       precio NUMERIC(10, 2) NOT NULL,
       stock INT DEFAULT 0
   );
   ```

2. Intenta validar o realizar una nueva ejecución de migración sobre el esquema:
   ```bash
   ./flyway -configFiles=conf/flyway.conf migrate
   ```

3. Analiza el mensaje de error de suma de verificación (*checksum validation*).
4. Para solventar este bloqueo (tras asegurar que el cambio de código es legítimo y alineado con el equipo), ejecuta el comando `repair` de Flyway. Este comando recalculará los checksums locales y alineará la tabla de historial `flyway_schema_history` con los archivos físicos del directorio `./sql`:
   ```bash
   ./flyway -configFiles=conf/flyway.conf repair
   ```

5. Comprueba de nuevo el estado de las migraciones de Flyway para validar que el conflicto está completamente saneado:
   ```bash
   ./flyway -configFiles=conf/flyway.conf info
   ```

#### Resultado esperado
Al ejecutar `migrate` después de la modificación no autorizada, Flyway detendrá inmediatamente la transacción emitiendo un fallo de integridad:

```text
ERROR: Validate failed: Migrations schema checksum mismatch for migration version 1
-> Applied to database : -1842054118
-> Resolved locally    : 1147820352
Either revert the changes to the migration, or run flyway:repair to update the schema history.
```

Al ejecutar `repair`, la utilidad mostrará que el checksum ha sido actualizado con éxito:

```text
Successfully repaired schema history table "inventory_schema"."flyway_schema_history" (execution time 00:00.060s)
```

Posteriormente, el comando `info` reportará un estado saludable libre de fallos de discrepancia de checksums.

#### Verificación
Confirma ejecutando `./flyway -configFiles=conf/flyway.conf validate` que la salida ya no produce errores y devuelve código de salida `0` en el sistema operativo.

---

## Validación y Pruebas

Para garantizar que todo el proceso se ha llevado a cabo bajo un estándar riguroso de cumplimiento y que se ha comprendido el funcionamiento de la herramienta, realiza las siguientes actividades de evaluación:

### 1. Auditoría del Historial en Base de Datos
Ejecuta la siguiente consulta desde DBeaver para inspeccionar los metadatos registrados por Flyway:

```sql
SELECT version, description, type, checksum, success 
FROM inventory_schema.flyway_schema_history 
ORDER BY installed_rank;
```

**Criterio de Aceptación:** Debes verificar la existencia de tres registros principales:
*   Versión `1` (init inventory) con el nuevo checksum reparado.
*   Versión `2` (add audit columns) con estado exitoso.
*   La migración repetible de la vista de inventario con tipo `SQL` y versión nula (`null`).

### 2. Prueba de Resistencia Adversarial (Caso de Fallo Provocado)
**Escenario de inyección de error**: Intenta simular una falla de ejecución DDL concurrente. Añade una migración fallida en un archivo nuevo `V3__failed_migration.sql` que intente crear una tabla con una columna de tipo de dato inexistente (`TEXT_ERRONEO`):

```sql
-- ~/flyway-lab/sql/V3__failed_migration.sql
CREATE TABLE tabla_error (
    id SERIAL PRIMARY KEY,
    campo_erroneo TEXT_ERRONEO
);
```

Ejecuta `./flyway -configFiles=conf/flyway.conf migrate`.

**Comportamiento esperado**: La migración fallará debido a la sintaxis incorrecta en el motor de base de datos de PostgreSQL. Sin embargo, dado que PostgreSQL soporta DDL transaccional (Transactional DDL), la transacción completa se deshará de forma automática.
Al ejecutar `./flyway -configFiles=conf/flyway.conf info`, el estado de la migración `3` aparecerá como `Failed`.

*Procedimiento de Mitigación del Error:*
1. Elimina el archivo `~/flyway-lab/sql/V3__failed_migration.sql` de tu sistema de archivos.
2. Ejecuta `./flyway -configFiles=conf/flyway.conf repair` para eliminar la referencia del registro fallido de la tabla `flyway_schema_history`.
3. Valida que el estado vuelve a ser limpio y estable con `./flyway -configFiles=conf/flyway.conf info`.

---

## Solución de Problemas

Aquí se presentan las dos situaciones de falla más comunes detectadas durante el despliegue de migraciones con Flyway en entornos Dockerizados de PostgreSQL.

### Caso 1: Error de conexión TCP/IP debido a puertos no expuestos o contenedores apagados

*   **Síntoma**: Al ejecutar cualquier comando de Flyway se obtiene la siguiente excepción:
    ```text
    Connection refused (Connection refused). Ensure that the database server is running and accepting TCP/IP connections on the port.
    ```
*   **Causa**: El contenedor Docker de PostgreSQL está detenido, la variable del puerto no es coincidente en el archivo `flyway.conf` (ej. se está intentando usar el puerto 5432 pero el contenedor no lo tiene expuesto al host), o hay una regla de firewall local bloqueando la conexión loopback.
*   **Solución**: 
    1. Verifica si el contenedor se está ejecutando usando `docker ps`.
    2. Si el contenedor no está activo, ejecútalo: `docker start pg-primary`.
    3. Confirma que el mapeo de puertos local de tu docker-compose o comando `docker run` asocia efectivamente el puerto `5432` de tu sistema anfitrión con el `5432` interno del contenedor.

### Caso 2: Error de inconsistencia debido a operaciones DDL concurrentes o bloqueadas

*   **Síntoma**: Flyway se congela indefinidamente o retorna un error de desconexión de base de datos por tiempo de espera excedido (*query timeout*) al procesar una instrucción `ALTER TABLE`.
*   **Causa**: La tabla sobre la cual estás aplicando la migración está siendo retenida bajo un bloqueo de tipo exclusivo (`AccessExclusiveLock`) por alguna sesión persistente de consulta abierta en DBeaver o por una aplicación simulando tráfico de transacciones concurrentes.
*   **Solución**:
    1. Identifica las transacciones activas de bloqueo en PostgreSQL ejecutando:
       ```sql
       SELECT pid, query, state, age(clock_timestamp(), query_start) 
       FROM pg_stat_activity 
       WHERE state != 'idle';
       ```
    2. Finaliza de manera forzada la sesión que está bloqueando la ejecución DDL ejecutando en DBeaver:
       ```sql
       SELECT pg_terminate_backend(blocking_pid); -- Reemplaza blocking_pid por el PID encontrado
       ```

---

## Limpieza

Para restaurar tu entorno de laboratorio de manera segura y eliminar los recursos que fueron creados de forma temporal durante la práctica, sigue detalladamente los siguientes pasos:

1. Elimina todo el esquema de inventario, incluyendo las tablas y la vista, con una orden limpia desde la consola psql:
   ```bash
   docker exec -it pg-primary psql -U postgres -d enterprise_db -c "DROP SCHEMA IF EXISTS inventory_schema CASCADE;"
   ```

2. Borra los archivos de base de datos locales generados para esta guía:
   ```bash
   rm -rf ~/flyway-lab
   ```

---

## Resumen

En esta práctica de laboratorio has implementado de forma práctica el flujo de **Gestión de Cambios de Esquemas Basado en Migraciones** con **Flyway Community Edition 10.10.0** integrado con PostgreSQL 16.2. 

Aprendiste que el uso de herramientas como Flyway proporciona un control riguroso y determinista sobre cómo evolucionan las bases de datos de entornos corporativos al tratarse el esquema como código fuente versionable. Durante la guía, creaste migraciones de tipo:
*   **Versionadas (V__)**: Ideales para cambios DDL estructurales e irreversibles (creación de tablas, nuevas columnas) que deben conservar un orden riguroso en producción.
*   **Repetibles (R__)**: El mecanismo óptimo para mantener vistas, disparadores o funciones de almacenamiento de manera limpia y sincronizada sin requerir constantes saltos numéricos de versión.

Por último, comprendiste que la integridad referencial y las firmas criptográficas (*checksums*) aplicadas por Flyway protegen la base de datos contra el nocivo efecto de la **deriva del esquema** (*schema drift*), obligando a todo el equipo de ingeniería a canalizar sus contribuciones a través de flujos estructurados de control de versiones.

### Lecturas y recursos recomendados
*   [Documentación Oficial de Flyway por Redgate](https://documentation.redgate.com/fd/flyway-documentation)
*   [PostgreSQL 16 - Gestión de Concurrencia y Bloqueos de Catálogo](https://www.postgresql.org/docs/16/explicit-locking.html)

---
