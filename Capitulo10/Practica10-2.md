# 10.2 Automatización de Migraciones con Liquibase

## Metadatos

| Métrica | Detalle |
| :--- | :--- |
| **Duración** | 90 minutos |
| **Complejidad** | Media |
| **Nivel de Bloom** | Aplicar |

## Descripción General

Esta práctica de laboratorio proporciona un escenario de aprendizaje guiado para la implementación práctica del ciclo de vida de bases de datos utilizando el enfoque basado en migraciones. El estudiante aprenderá a configurar, instrumentar y ejecutar un pipeline local de evolución de esquemas relacionales sobre **PostgreSQL 16.2** utilizando la herramienta líder de la industria **Liquibase Community Edition 4.26.0**. Durante el ejercicio, se implementará un esquema híbrido de cambios utilizando formatos estructurados (XML y YAML) y sentencias imperativas de control (SQL con metadatos de rollback), validando el comportamiento transaccional del catálogo y simulando escenarios de falla para verificar la resiliencia y el control ante la deriva del esquema (*schema drift*).

## Objetivos de Aprendizaje

Al finalizar esta práctica de laboratorio, serás capaz de:
* [ ] Instalar, enlazar y configurar la herramienta **Liquibase CLI 4.26.0** con el controlador JDBC oficial compatible con un contenedor Docker de **PostgreSQL 16.2**.
* [ ] Diseñar y estructurar un archivo maestro de cambios (*master changelog*) jerárquico que encapsule definiciones multi-formato (XML, YAML y SQL).
* [ ] Aplicar de manera exitosa migraciones hacia adelante (*forward migrations / updates*) para actualizar el esquema físico de una base de datos corporativa.
* [ ] Inspeccionar y auditar las tablas de control internas de Liquibase (`databasechangelog` y `databasechangeloglock`) para comprobar la sincronía del despliegue.
* [ ] Ejecutar de forma segura estrategias de reversión parcial (*rollback*) verificando la resiliencia e integridad de los datos en PostgreSQL.

## Prerrequisitos

Para la correcta ejecución de esta práctica, el estudiante requiere poseer los siguientes conocimientos y accesos previos:
1. **Conocimientos teóricos y prácticos:**
   * Conceptos de DevOps de Base de Datos expuestos en la Lección 6.1 (Diferencias fundamentales entre esquemas *state-based* y *migration-based*).
   * Administración básica de contenedores con Docker CLI.
   * Dominio básico del lenguaje SQL estándar y comandos del Lenguaje de Definición de Datos (DDL).
2. **Acceso al sistema y privilegios:**
   * Terminal local de Linux (Debian GNU/Linux 12.5 o compatible) con privilegios de ejecución administrativa (`sudo`).
   * Acceso de salida a Internet a través de puertos TCP 80 y 443 para la descarga de dependencias, imágenes Docker y el motor CLI de Liquibase.

## Entorno de Laboratorio

El laboratorio se desarrollará utilizando una arquitectura homogénea y herramientas de versiones específicas para garantizar la total reproducibilidad de los resultados.

### Especificaciones de Software y Fuentes Oficiales

| Software | Versión Exacta / Arquitectura | Origen / Enlace Oficial de Descarga | Licencia aplicable |
| :--- | :--- | :--- | :--- |
| **PostgreSQL Community Edition** | v16.2 (x86_64) | [PostgreSQL Docker Hub](https://hub.docker.com/_/postgres) | PostgreSQL License |
| **Liquibase Community Edition** | v4.26.0 (x86_64) | [GitHub Releases Liquibase](https://github.com/liquibase/liquibase/releases/tag/v4.26.0) | Apache License 2.0 |
| **PostgreSQL JDBC Driver** | v42.7.2 (Platform Independent) | [Maven Central Repository](https://repo1.maven.org/maven2/org/postgresql/postgresql/42.7.2/) | BSD-2-Clause |
| **Docker Engine / Desktop** | v26.0.0 (x86_64) | [Docker Docs Installation Guide](https://docs.docker.com/engine/install/) | Apache License 2.0 |

> **Nota sobre el uso de herramientas de IA:** Si utilizas asistentes de desarrollo como *Microsoft 365 Copilot* o *Copilot Chat* para la asistencia en la escritura de scripts de migración, se requiere configurar las sugerencias en formato estricto bajo el estándar de transacciones seguras de PostgreSQL, bajo la supervisión directa y la validación de un instructor humano en este laboratorio.

### Configuración Inicial de Red y Entorno

Ejecute los siguientes comandos en su terminal para establecer la red aislada de Docker que albergará el contenedor de base de datos de desarrollo.

```bash
## Crear la red de tipo bridge requerida para el ciclo empresarial
docker network create pg_enterprise_net

## Crear la estructura jerárquica de directorios del proyecto
mkdir -p ~/liquibase-lab/changelogs
mkdir -p ~/liquibase-lab/sql
cd ~/liquibase-lab
```

---

## Instrucciones Paso a Paso

### Paso 1: Configurar el Contenedor PostgreSQL y Descargar Liquibase CLI con su Driver JDBC

**Objetivo:** Inicializar la instancia de base de datos aislada de desarrollo e instalar localmente la herramienta Liquibase CLI conectada con su respectivo driver JDBC para habilitar la interacción.

**Instrucciones:**

1. Inicie el contenedor Docker de PostgreSQL versión 16.2 bajo el nombre de host `pg-dev` mapeando el puerto de desarrollo local 5432. Configure las variables de entorno de acuerdo a los estándares globales de seguridad empresarial del curso.

```bash
docker run -d \
  --name pg-dev \
  --network pg_enterprise_net \
  -p 5432:5432 \
  -e POSTGRES_DB=enterprise_db \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=PostgresAdminPass123! \
  postgres:16.2
```

2. Descargue e instale el binario oficial de **Liquibase Community Edition v4.26.0** en una carpeta local de utilidad.

```bash
## Descargar el archivo comprimido oficial
wget https://github.com/liquibase/liquibase/releases/download/v4.26.0/liquibase-4.26.0.tar.gz

## Crear directorio de extracción
mkdir -p ~/liquibase-lab/liquibase-cli

## Extraer el contenido
tar -xzf liquibase-4.26.0.tar.gz -C ~/liquibase-lab/liquibase-cli/

## Limpiar archivo temporal
rm liquibase-4.26.0.tar.gz
```

3. Descargue el controlador **PostgreSQL JDBC v42.7.2** requerido por la Java Virtual Machine de Liquibase para establecer la comunicación por sockets con la base de datos PostgreSQL.

```bash
## Descargar el driver de Maven Central a la raíz de trabajo del proyecto
wget https://repo1.maven.org/maven2/org/postgresql/postgresql/42.7.2/postgresql-42.7.2.jar -O ~/liquibase-lab/postgresql-42.7.2.jar
```

4. Agregue temporalmente la ruta del ejecutable de Liquibase a la variable del sistema `$PATH` para facilitar la invocación directa del comando.

```bash
export PATH=$PATH:~/liquibase-lab/liquibase-cli
```

**Resultado esperado:**
Un contenedor de base de datos PostgreSQL 16.2 corriendo en segundo plano, la carpeta `liquibase-cli` creada y cargada con el binario ejecutable, y el archivo del driver JDBC `.jar` guardado en la carpeta base `~/liquibase-lab`.

**Verificación:**
Ejecute la verificación de versiones desde la terminal para confirmar que ambos sistemas están operativos e instalados correctamente:

```bash
## Verificar estado del contenedor Docker
docker ps --filter "name=pg-dev"

## Verificar versión de la CLI de Liquibase
liquibase --version
```

Salida esperada de la CLI (extracto similar):
```text
Liquibase Version: 4.26.0
Liquibase Community 4.26.0 by LIQUIBASE
```

---

### Paso 2: Crear el Archivo de Propiedades de Configuración (`liquibase.properties`)

**Objetivo:** Configurar de forma persistente los parámetros de conexión JDBC de Liquibase que le indicarán a la herramienta a qué base de datos, con qué credenciales de acceso y bajo qué controlador realizar la sincronización.

**Instrucciones:**

1. Cree el archivo de propiedades en la raíz de su espacio de trabajo `~/liquibase-lab/liquibase.properties`.

```bash
nano ~/liquibase-lab/liquibase.properties
```

2. Introduzca el siguiente bloque de parámetros de configuración exactos. Guarde el archivo (`Ctrl+O`, `Enter`, `Ctrl+X`).

```properties
## Archivo de Propiedades para Liquibase
changeLogFile=changelogs/db.changelog-master.xml
url=jdbc:postgresql://localhost:5432/enterprise_db
username=postgres
password=PostgresAdminPass123!
driver=org.postgresql.Driver
classpath=postgresql-42.7.2.jar
```

**Resultado esperado:**
Se ha generado un archivo de configuración válido apuntando al archivo central que contendrá la referencia de los scripts e indicará el driver compatible de conexión.

**Verificación:**
Valide la existencia del archivo de propiedades y su lectura exitosa intentando ejecutar una prueba previa de conexión utilizando el comando de validación `status`:

```bash
liquibase status
```

Salida de error esperada temporalmente:
```text
Unexpected error running Liquibase: changelogs/db.changelog-master.xml does not exist
```
*Nota: Este error es correcto en esta fase e indica que Liquibase leyó exitosamente el archivo de propiedades de conexión, pero aún no encuentra el archivo maestro de cambios.*

---

### Paso 3: Diseñar el Archivo de Migración Maestro y los Cambios Multi-Formato (XML, YAML y SQL)

**Objetivo:** Configurar una tubería de cambios que combine la definición declarativa y controlada de múltiples lenguajes de definición soportados por Liquibase. Esto representa un entorno real de integración donde diferentes equipos de desarrollo (Platform, Frontend y Analytics) usan distintas convenciones.

```
Estructura de archivos esperada:
~/liquibase-lab/
├── changelogs/
│   ├── db.changelog-master.xml
│   ├── v1.0.0_create_catalogs.xml
│   └── v1.1.0_add_employees_and_fk.yaml
├── sql/
│   └── v1.2.0_seed_data_and_function.sql
```

**Instrucciones:**

1. Cree el archivo maestro XML del proyecto en `changelogs/db.changelog-master.xml`:

```bash
nano ~/liquibase-lab/changelogs/db.changelog-master.xml
```

Inserte la siguiente estructura XML estándar (que mapea los esquemas de validación de la versión 4.26.x e incluye los tres scripts que crearemos a continuación):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
    http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.26.xsd">

    <include file="changelogs/v1.0.0_create_catalogs.xml"/>
    <include file="changelogs/v1.1.0_add_employees_and_fk.yaml"/>
    <include file="sql/v1.2.0_seed_data_and_function.sql"/>
</databaseChangeLog>
```

2. Cree la migración de Catálogos (XML) en `changelogs/v1.0.0_create_catalogs.xml`. Esta migración define la tabla `departamentos` y un rollback explícito seguro.

```bash
nano ~/liquibase-lab/changelogs/v1.0.0_create_catalogs.xml
```

Inserte el siguiente contenido:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
    http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.26.xsd">

    <changeSet id="1.0.0" author="platform-team">
        <createTable tableName="departamentos">
            <column name="id" type="int">
                <constraints primaryKey="true" nullable="false" primaryKeyName="pk_departamentos"/>
            </column>
            <column name="nombre" type="varchar(100)">
                <constraints nullable="false"/>
            </column>
            <column name="presupuesto" type="numeric(12,2)">
                <constraints nullable="false"/>
            </column>
        </createTable>
        <rollback>
            <dropTable tableName="departamentos"/>
        </rollback>
    </changeSet>
</databaseChangeLog>
```

3. Cree la migración de Empleados y Restricciones (YAML) en `changelogs/v1.1.0_add_employees_and_fk.yaml`. Este bloque define la tabla `empleados` y asocia de forma segura su restricción de clave foránea.

```bash
nano ~/liquibase-lab/changelogs/v1.1.0_add_employees_and_fk.yaml
```

Inserte la sintaxis YAML exacta:

```yaml
databaseChangeLog:
  - changeSet:
      id: "1.1.0"
      author: "dev-team"
      changes:
        - createTable:
            tableName: "empleados"
            columns:
              - column:
                  name: "id"
                  type: "int"
                  constraints:
                    primaryKey: true
                    nullable: false
                    primaryKeyName: "pk_empleados"
              - column:
                  name: "nombre"
                  type: "varchar(150)"
                  constraints:
                    nullable: false
              - column:
                  name: "departamento_id"
                  type: "int"
                  constraints:
                    nullable: false
        - addForeignKeyConstraint:
            baseTableName: "empleados"
            baseColumnNames: "departamento_id"
            constraintName: "fk_empleados_departamento"
            referencedTableName: "departamentos"
            referencedColumnNames: "id"
            onDelete: "RESTRICT"
      rollback:
        - dropForeignKeyConstraint:
            baseTableName: "empleados"
            constraintName: "fk_empleados_departamento"
        - dropTable:
            tableName: "empleados"
```

4. Cree el archivo de cargas iniciales y funciones de procesamiento en formato SQL puro compatible con Liquibase (`--liquibase formatted sql`) en `sql/v1.2.0_seed_data_and_function.sql`.

```bash
nano ~/liquibase-lab/sql/v1.2.0_seed_data_and_function.sql
```

Inserte el siguiente script SQL enriquecido con metadatos para rollback:

```sql
--liquibase formatted sql

--changeset analytics-team:1.2.0
--comment: Poblar datos iniciales corporativos y crear funcion helper de calculo
INSERT INTO departamentos (id, nombre, presupuesto) VALUES (1, 'Tecnología', 250000.00);
INSERT INTO departamentos (id, nombre, presupuesto) VALUES (2, 'Operaciones', 120000.00);

INSERT INTO empleados (id, nombre, departamento_id) VALUES (101, 'Ana Gomez', 1);
INSERT INTO empleados (id, nombre, departamento_id) VALUES (102, 'Carlos Ruiz', 2);

CREATE OR REPLACE FUNCTION get_presupuesto_departamento(dep_id INT)
RETURNS NUMERIC AS $$
DECLARE
    v_presupuesto NUMERIC;
BEGIN
    SELECT presupuesto INTO v_presupuesto FROM departamentos WHERE id = dep_id;
    RETURN COALESCE(v_presupuesto, 0.00);
END;
$$ LANGUAGE plpgsql;

--rollback DROP FUNCTION IF EXISTS get_presupuesto_departamento(INT);
--rollback DELETE FROM empleados WHERE id IN (101, 102);
--rollback DELETE FROM departamentos WHERE id IN (1, 2);
```

**Resultado esperado:**
Los archivos estructurados del changeset creados correctamente en sus carpetas `changelogs/` y `sql/` con sintaxis válida para Liquibase 4.26.0.

**Verificación:**
Valide la sintaxis de las migraciones sin aplicarlas aún a la base de datos física mediante el comando `status --verbose`:

```bash
liquibase status --verbose
```

Salida esperada (resumen):
```text
3 change sets have not been applied to postgres@jdbc:postgresql://localhost:5432/enterprise_db
     changelogs/v1.0.0_create_catalogs.xml::1.0.0::platform-team
     changelogs/v1.1.0_add_employees_and_fk.yaml::1.1.0::dev-team
     sql/v1.2.0_seed_data_and_function.sql::1.2.0::analytics-team
```

---

### Paso 4: Ejecutar la Migración hacia Adelante (Forward Migration / Update)

**Objetivo:** Aplicar las migraciones planificadas hacia el servidor de desarrollo PostgreSQL ejecutando de manera ordenada la transacción e inspeccionando las tablas internas de metadatos.

**Instrucciones:**

1. Ejecute el comando de actualización para aplicar de forma secuencial todos los cambios pendientes registrados en el archivo maestro:

```bash
liquibase update
```

**Resultado esperado:**
Salida informativa en consola que indica que Liquibase se conectó y aplicó exitosamente las tres migraciones creadas en el Paso 3.

Salida esperada (extracto de éxito):
```text
Running Queue...
Running Changeset: changelogs/v1.0.0_create_catalogs.xml::1.0.0::platform-team
Running Changeset: changelogs/v1.1.0_add_employees_and_fk.yaml::1.1.0::dev-team
Running Changeset: sql/v1.2.0_seed_data_and_function.sql::1.2.0::analytics-team
Update Command Completed Successfully.
```

**Verificación:**

Para comprobar que las tablas internas del sistema creadas automáticamente por Liquibase y las tablas del negocio existen físicamente en PostgreSQL, se utilizará la herramienta interactiva de línea de comandos de PostgreSQL (`psql`) embebida dentro de nuestro contenedor:

1. Ingrese a la base de datos dentro del contenedor:

```bash
docker exec -it pg-dev psql -U postgres -d enterprise_db
```

2. Liste las tablas del esquema público para validar que existan tanto las del negocio como las de metadatos de Liquibase:

```sql
\dt
```

Salida esperada en consola:
```text
                List of relations
 Schema |         Name          | Type  |  Owner   
--------+-----------------------+-------+----------
 public | databasechangelog     | table | postgres
 public | databasechangeloglock | table | postgres
 public | departamentos         | table | postgres
 public | empleados             | table | postgres
(4 rows)
```

3. Consulte el historial de auditoría de despliegues dentro de la tabla de metadatos `databasechangelog` para verificar los identificadores, autores e historiales de suma de comprobación MD5 (*hashes*):

```sql
SELECT id, author, filename, dateexecuted, md5sum FROM databasechangelog ORDER BY dateexecuted ASC;
```

Salida esperada en consola:
```text
  id   |     author     |                    filename                    |        dateexecuted        |                md5sum                
-------+----------------+------------------------------------------------+----------------------------+--------------------------------------
 1.0.0 | platform-team  | changelogs/v1.0.0_create_catalogs.xml          | 2024-xx-xx 12:00:00.123456 | 9:fcae217c91c2f9e4210cf49987f2e1a3
 1.1.0 | dev-team       | changelogs/v1.1.0_add_employees_and_fk.yaml    | 2024-xx-xx 12:00:01.456789 | 9:c79bc1982b61f253ccfa7c7fa918b9de
 1.2.0 | analytics-team | sql/v1.2.0_seed_data_and_function.sql          | 2024-xx-xx 12:00:02.890123 | 9:aa7fb7833fbb1cd0897f26f22fa102ec
(3 rows)
```

4. Verifique la existencia y funcionamiento de la función de base de datos creada en el Paso 3, llamándola mediante una instrucción de selección:

```sql
SELECT get_presupuesto_departamento(1) AS presupuesto_ti;
```

Salida esperada en consola:
```text
 presupuesto_ti 
----------------
      250000.00
(1 row)
```

5. Salga del CLI de `psql`:

```sql
\q
```

---

### Paso 5: Ejecutar una Estrategia de Reversión de Cambios (Rollback Parcial)

**Objetivo:** Evaluar la resiliencia y el comportamiento dinámico del control de versiones ante un escenario de reversión (*rollback*). Retornaremos la base de datos un paso atrás para verificar la correcta desinstalación de los scripts de carga de datos y de la función creada por el equipo de Analytics sin alterar la estructura básica del negocio.

**Instrucciones:**

1. Ejecute un rollback controlado para revertir exactamente el último changeset aplicado (`rollback-count 1` que corresponde a las instrucciones SQL escritas por `analytics-team` en el id `1.2.0`):

```bash
liquibase rollback-count 1
```

**Resultado esperado:**
Salida en consola confirmando la remoción de los datos iniciales y de la función PostgreSQL aplicando de forma reversa las instrucciones marcadas por la etiqueta `--rollback` del archivo del Paso 3.

Salida esperada (extracto):
```text
Rolling Back Changeset: sql/v1.2.0_seed_data_and_function.sql::1.2.0::analytics-team
Rollback Command Completed Successfully.
```

**Verificación:**

1. Vuelva a ingresar a la consola interactiva de base de datos:

```bash
docker exec -it pg-dev psql -U postgres -d enterprise_db
```

2. Verifique si los datos de la tabla `departamentos` fueron borrados conforme a la regla del rollback:

```sql
SELECT * FROM departamentos;
```

Salida esperada:
```text
 id | nombre | presupuesto 
----+--------+-------------
(0 rows)
```
*(Nota: Las tablas `departamentos` y `empleados` continúan existiendo, pero la carga inicial ha sido removida).*

3. Intente ejecutar la función para confirmar que ha sido desinstalada del catálogo:

```sql
SELECT get_presupuesto_departamento(1);
```

Salida esperada:
```text
ERROR: function get_presupuesto_departamento(integer) does not exist
LINE 1: SELECT get_presupuesto_departamento(1);
               ^
HINT: No function matches the given name and argument types. You might need to add explicit type casts.
```

4. Verifique los registros del log en la tabla `databasechangelog` para comprobar que la fila `1.2.0` se ha eliminado del registro histórico de forma limpia:

```sql
SELECT id, author, filename FROM databasechangelog ORDER BY dateexecuted ASC;
```

Salida esperada:
```text
  id   |    author     |                  filename                   
-------+---------------+---------------------------------------------
 1.0.0 | platform-team | changelogs/v1.0.0_create_catalogs.xml
 1.1.0 | dev-team      | changelogs/v1.1.0_add_employees_and_fk.yaml
(2 rows)
```

5. Cierre la sesión de PostgreSQL:

```sql
\q
```

6. Vuelva a aplicar el comando `update` para dejar el entorno de desarrollo en su estado actualizado final:

```bash
liquibase update
```

---

## Validación y Pruebas

Para garantizar que el laboratorio se ha ejecutado bajo rigurosos estándares profesionales de control y consistencia, se han definido pruebas de conformidad de infraestructura que el estudiante debe superar.

### Caso de Prueba Adversario: Simulación de Falla y Resiliencia en Transacciones DDL

En un entorno real de despliegue continuo, un script con errores de sintaxis o referencias inválidas puede comprometer la base de datos si no es gestionado bajo transacciones DDL seguras. PostgreSQL soporta de forma nativa DDL transaccional (lo que significa que si una sentencia DDL falla, se deshace todo el bloque en vez de dejar el esquema "a medias"). Comprobemos esta propiedad usando Liquibase.

**Instrucciones:**

1. Cree un nuevo archivo de migración con fallas intencionadas en `sql/v1.3.0_bad_dml.sql`:

```bash
nano ~/liquibase-lab/sql/v1.3.0_bad_dml.sql
```

Inserte el siguiente contenido. Note el error explícito (la tabla `tabla_fantasma_inexistente` no existe, por lo tanto la alteración fallará):

```sql
--liquibase formatted sql

--changeset security-team:1.3.0
--comment: Intentar agregar una columna de auditoria pero con error de sintaxis intencionado
ALTER TABLE departamentos ADD COLUMN fecha_actualizacion TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP;

-- ESTO CAUSARÁ UN ERROR DE BASE DE DATOS E INTERRUMPIRÁ LA TRANSACCIÓN
ALTER TABLE tabla_fantasma_inexistente ADD COLUMN error_columna INT;

--rollback ALTER TABLE departamentos DROP COLUMN fecha_actualizacion;
```

2. Actualice el archivo maestro `changelogs/db.changelog-master.xml` para incluir este cambio inválido:

```bash
nano ~/liquibase-lab/changelogs/db.changelog-master.xml
```

Modifique el archivo para que contenga las 4 referencias:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
    http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.26.xsd">

    <include file="changelogs/v1.0.0_create_catalogs.xml"/>
    <include file="changelogs/v1.1.0_add_employees_and_fk.yaml"/>
    <include file="sql/v1.2.0_seed_data_and_function.sql"/>
    <include file="sql/v1.3.0_bad_dml.sql"/>
</databaseChangeLog>
```

3. Intente realizar la migración utilizando `update`:

```bash
liquibase update
```

**Análisis de Resultado:**
La ejecución debe fallar de manera inmediata indicando el error de sintaxis o la falta de relación de la tabla fantasma.

Salida esperada (extracto del error):
```text
Unexpected error running Liquibase: Migration failed for changeset sql/v1.3.0_bad_dml.sql::1.3.0::security-team:
     Reason: liquibase.exception.DatabaseException: ERROR: relation "tabla_fantasma_inexistente" does not exist [Failed SQL: ...]
```

4. Valide mediante `psql` que la tabla `departamentos` **no contiene** la columna `fecha_actualizacion`. Dado que PostgreSQL ejecuta los cambiosets dentro de una única transacción por defecto, todo el bloque `changeset 1.3.0` debió revertirse al fallar la segunda instrucción, evitando inconsistencias físicas (el temido estado "parcialmente desplegado").

```bash
docker exec -it pg-dev psql -U postgres -d enterprise_db -c "\d departamentos"
```

Salida esperada:
```text
                       Table "public.departamentos"
   Column    |         Type          | Collation | Nullable | Default 
-------------+-----------------------+-----------+----------+---------
 id          | integer               |           | not null | 
 nombre      | character varying(100)|           | not null | 
 presupuesto | numeric(12,2)         |           | not null | 
Indexes:
    "pk_departamentos" PRIMARY KEY, btree (id)
Referenced by:
    TABLE "empleados" CONSTRAINT "fk_empleados_departamento" FOREIGN KEY (departamento_id) REFERENCES departamentos(id) ON DELETE RESTRICT
```
*(Efectivamente, la columna `fecha_actualizacion` no existe, demostrando que la base de datos se mantuvo limpia y consistente).*

---

## Solución de Problemas

A continuación se detallan dos de las incidencias más comunes que ocurren en el despliegue práctico de esta configuración tecnológica y cómo resolverlas de manera precisa.

### Problema 1: Error de Validación de Checksum (`Validation Failed: Change set ... has changed since it was run`)

* **Síntomas:** Al intentar ejecutar `liquibase update` o `liquibase status`, el comando finaliza de manera abrupta con un mensaje de alerta similar al siguiente:
  ```text
  Validation Failed:
  1 change sets had changed since they were ran against the database
  changelogs/v1.0.0_create_catalogs.xml::1.0.0::platform-team is now: 9:ab2bc... (was: 9:fcae2...)
  ```
* **Causa:** Un desarrollador modificó el código DDL/DML de un archivo de migración que **ya había sido ejecutado** previamente en la base de datos de producción o desarrollo. Liquibase detecta que la firma MD5 actual no coincide con la guardada en `databasechangelog` para prevenir cambios descontrolados y deriva en el esquema (*schema drift*).
* **Solución:**
  1. Si el cambio en el archivo fue un error de edición y se desea restablecer el estado original, devuelva el contenido del archivo a su estado original.
  2. Si el cambio es menor y se está completamente seguro de que el esquema actual en PostgreSQL coincide con las modificaciones aplicadas, fuerce a Liquibase a recalcular los hashes en la tabla de metadatos utilizando el comando:
     ```bash
     liquibase clear-checksums
     ```
  3. Ejecute nuevamente `liquibase update` para reanudar el flujo.

### Problema 2: El sistema arroja un bloqueo eterno o un mensaje de Lock (`Waiting for changelog lock....`)

* **Síntomas:** La consola se congela indefinidamente mostrando de manera repetida el texto:
  ```text
  Waiting for changelog lock....
  Waiting for changelog lock....
  ```
* **Causa:** Liquibase utiliza una fila en la tabla `databasechangeloglock` para garantizar que un solo proceso esté modificando la base de datos a la vez (previniendo concurrencias conflictivas de pipelines paralelos). Si un pipeline previo fue interrumpido violentamente (por ejemplo, cancelando con `Ctrl+C` en medio de una migración), la base de datos nunca liberó el candado dejando la columna `locked` configurada como verdadera (`true`).
* **Solución:**
  1. Asegúrese de que no haya otra tarea de migración ejecutándose en segundo plano.
  2. Fuerza la liberación manual del lock ejecutando la instrucción CLI provista por Liquibase para este fin:
     ```bash
     liquibase release-locks
     ```
  3. (Alternativa desde SQL en caso de fallar el comando anterior): ingrese al contenedor e interactúe con psql para actualizar el candado manualmente:
     ```sql
     UPDATE databasechangeloglock SET locked = FALSE, lockgranted = NULL, lockedby = NULL WHERE id = 1;
     ```

---

## Limpieza

Para restaurar los recursos de su máquina anfitriona al estado inicial libre de elementos residuales una vez terminada la sesión, ejecute los siguientes comandos:

```bash
## Detener y remover el contenedor Docker pg-dev
docker stop pg-dev
docker rm pg-dev

## Remover la red de prueba creada para el laboratorio
docker network rm pg_enterprise_net

## Remover el directorio de trabajo del laboratorio y sus binarios
rm -rf ~/liquibase-lab
```

---

## Resumen

En esta práctica de laboratorio, implementamos de manera práctica las metodologías clave de DevOps de Bases de Datos explicadas en la teoría del curso, utilizando el enfoque de gestión **basado en migraciones** con la suite **Liquibase Community Edition 4.26.0** conectada a un servidor de datos **PostgreSQL 16.2**. 

### Aprendizajes clave consolidados:
1. **Control de Cambios Descentralizado:** Diseñamos un despliegue híbrido utilizando múltiples especificaciones (XML, YAML y SQL) coordinadas bajo un único archivo de cambios maestro (`db.changelog-master.xml`).
2. **Determinismo y Trazabilidad:** Validamos el funcionamiento de las tablas internas `databasechangelog` y `databasechangeloglock`, comprendiendo cómo se previene la modificación fuera de orden y se audita el historial de despliegues mediante firmas hash MD5.
3. **Resiliencia ante Fallos (Rollback):** Comprobamos la capacidad de reversión parcial del esquema y experimentamos cómo la propiedad transaccional de PostgreSQL impide que fallas parciales de ejecución en un changeset dejen las tablas en estados inconsistentes.

### Recursos Adicionales:
* [Documentación Oficial de Liquibase CLI](https://docs.liquibase.com/)
* [PostgreSQL 16 Reference Manual on Transactional DDL](https://www.postgresql.org/docs/16/sql-begin.html)

---
