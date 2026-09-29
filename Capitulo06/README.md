# 1 Versionamiento de Objetos PostgreSQL con Git

## Metadatos

| Campo | Detalle |
| :--- | :--- |
| **Duración** | 60 minutos |
| **Complejidad** | Media |
| **Nivel de Bloom** | Aplicar |

## Descripción General

En este laboratorio práctico, el estudiante diseñará e implementará un flujo de trabajo completo de DevOps para Bases de Datos (Database-as-Code) utilizando Git para el control de versiones de esquemas y Flyway como motor de migración imperativo. Se configurarán dos instancias aisladas de PostgreSQL en Docker que representarán los ambientes de Desarrollo (`pg-dev`) y Producción (`pg-prod`). El estudiante aprenderá a estructurar scripts SQL de migración bajo convenciones estrictas, ejecutar despliegues secuenciales, realizar validaciones automatizadas y resolver escenarios de conflicto comunes como la alteración de scripts ya aplicados (checksum mismatch) y la deriva de esquema (*schema drift*).

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
* [ ] Inicializar y estructurar un repositorio Git local enfocado en el control de versiones de bases de datos de nivel corporativo.
* [ ] Aplicar el enfoque basado en migraciones de manera secuencial y determinista en entornos PostgreSQL utilizando Flyway.
* [ ] Detectar y diagnosticar el impacto del *schema drift* y las discrepancias de firmas digitales (checksums) en tuberías de despliegue continuo.
* [ ] Promover de manera segura cambios estructurales validados desde un entorno simulado de Desarrollo a uno de Producción sin pérdida de datos.

## Prerrequisitos

Para completar este laboratorio con éxito, se requiere:
* **Conocimientos teóricos y prácticos previos:**
  * Uso básico de comandos de control de versiones Git (`init`, `add`, `commit`, `status`, `log`).
  * Familiaridad con el lenguaje de definición de datos (DDL) en PostgreSQL (restricciones de clave primaria, tipos de datos, llaves foráneas).
  * Comprensión de arquitecturas de contenedores Docker y ejecución de comandos básicos en la terminal.
* **Acceso y privilegios:**
  * Permisos de administrador/sudo en la máquina anfitriona para interactuar con Docker Daemon.
  * Acceso de red a Internet estable para la descarga de las imágenes de contenedor de PostgreSQL y Flyway.

## Entorno de Laboratorio

El entorno de ejecución está estandarizado bajo los siguientes componentes y versiones de software:

### Especificaciones de Software y Fuentes Oficiales

| Componente | Versión Exacta | Licencia | URL de Referencia Oficial |
| :--- | :--- | :--- | :--- |
| **PostgreSQL Community Edition** | 16.2 (Debian-based) | PostgreSQL License | [Docker Hub - Postgres Official](https://hub.docker.com/_/postgres) |
| **Git Community Edition** | 2.43.0 (x86_64) | GNU GPL v2 | [Git Official Website](https://git-scm.com) |
| **Flyway Community Edition** | 10.10.0 (Linux x86_64) | Apache License 2.0 | [Red Gate Flyway Official](https://red-gate.com/products/flyway) |
| **Docker Engine / Desktop** | 26.0.0 (Community Edition) | Apache License 2.0 | [Docker Docs](https://docs.docker.com) |

### Parámetros Globales del Entorno

* **Red Docker Común (Bridge):** `pg_enterprise_net`
* **Nombre de la Base de Datos:** `enterprise_db`
* **Usuario Administrador Global:** `postgres`
* **Contraseña del Administrador:** `PostgresAdminPass123!`
* **Puerto Mapeado PostgreSQL Desarrollo (`pg-dev`):** `5433`
* **Puerto Mapeado PostgreSQL Producción (`pg-prod`):** `5434`

### Preparación Inicial del Host (Comandos de Terminal)

Ejecute los siguientes comandos en su terminal Linux/macOS para crear el directorio de trabajo del laboratorio y la red aislada de Docker:

```bash
## Crear directorio de trabajo local
mkdir -p ~/labs/db_devops_git && cd ~/labs/db_devops_git

## Crear la red de Docker empresarial
docker network create pg_enterprise_net
```

---

## Instrucciones Paso a Paso

### Paso 1: Configurar la Infraestructura de Contenedores (Desarrollo y Producción)

**Objetivo:** Crear y desplegar dos contenedores de base de datos PostgreSQL 16.2 independientes que simularán los entornos de ciclo de vida clásicos de una infraestructura corporativa.

**Instrucciones:**

1. Ejecute el siguiente comando para inicializar el contenedor que representará el entorno de **Desarrollo** (`pg-dev`):
   ```bash
   docker run -d \
     --name pg-dev \
     --network pg_enterprise_net \
     -p 5433:5432 \
     -e POSTGRES_DB=enterprise_db \
     -e POSTGRES_USER=postgres \
     -e POSTGRES_PASSWORD=PostgresAdminPass123! \
     postgres:16.2
   ```

2. Ejecute el siguiente comando para inicializar el contenedor que representará el entorno de **Producción** (`pg-prod`):
   ```bash
   docker run -d \
     --name pg-prod \
     --network pg_enterprise_net \
     -p 5434:5432 \
     -e POSTGRES_DB=enterprise_db \
     -e POSTGRES_USER=postgres \
     -e POSTGRES_PASSWORD=PostgresAdminPass123! \
     postgres:16.2
   ```

3. Verifique que ambos servicios se encuentren listos para recibir conexiones TCP/IP:
   ```bash
   docker ps --filter "name=pg-"
   ```

**Resultado esperado:**
La terminal debe listar ambos contenedores (`pg-dev` y `pg-prod`) con estado `Up` y mostrando los respectivos puertos mapeados en la máquina host (5433 y 5434).

**Verificación:**
Asegúrese de que el motor de base de datos responda de forma nativa a consultas básicas de prueba utilizando `docker exec`:
```bash
docker exec -i pg-dev psql -U postgres -d enterprise_db -c "SELECT version();"
docker exec -i pg-prod psql -U postgres -d enterprise_db -c "SELECT version();"
```
Ambos comandos deben retornar una salida que inicie con `PostgreSQL 16.2...`.

---

### Paso 2: Inicializar el Repositorio de Git y Estructura de Migraciones

**Objetivo:** Configurar el control de versiones de Git en la raíz del proyecto y definir una estructura de directorios compatible con las políticas de automatización de Flyway.

**Instrucciones:**

1. Estando ubicados en el directorio de trabajo `~/labs/db_devops_git`, inicialice el repositorio de Git:
   ```bash
   git init
   ```

2. Configure su identidad local para este repositorio (si no cuenta con una global):
   ```bash
   git config user.name "Technical Instructor"
   git config user.email "instructor@enterprise.com"
   ```

3. Cree la estructura jerárquica de directorios requerida por Flyway para la búsqueda de scripts:
   ```bash
   mkdir -p sql/migrations
   ```

4. Cree un archivo `.gitignore` para evitar que configuraciones del sistema, archivos log o credenciales locales queden expuestos en el control de versiones:
   ```bash
   cat <<EOF > .gitignore
   # Ignore logs and system files
   *.log
   .DS_Store
   .idea/
   .vscode/
   EOF
   ```

5. Realice el primer registro en el control de versiones:
   ```bash
   git add .gitignore
   git commit -m "chore: initial commit with gitignore structure"
   ```

**Resultado esperado:**
Un repositorio Git local inicializado correctamente con una rama por defecto (generalmente `master` o `main`) y con el directorio `sql/migrations` limpio y listo para recibir scripts de migración SQL.

**Verificación:**
Ejecute el comando `git status` para confirmar el estado del repositorio:
```bash
git status
```
La terminal debe indicar `nothing to commit, working tree clean`.

---

### Paso 3: Diseñar e Implementar Scripts de Migración Versionados (V1 y V2)

**Objetivo:** Desarrollar scripts DDL estructurados utilizando la nomenclatura estricta de Flyway (`V<Version>__<Description>.sql`) para definir tablas y realizar evoluciones de esquema controladas.

**Instrucciones:**

1. Cree el primer script de migración (`V1__crear_tabla_clientes.sql`) dentro del directorio `sql/migrations`. Este script creará la tabla principal de clientes:
   ```bash
   cat <<EOF > sql/migrations/V1__crear_tabla_clientes.sql
   -- Flyway Migration - Version 1: Creacion de Tabla Clientes
   CREATE TABLE clientes (
       id SERIAL PRIMARY KEY,
       nombre VARCHAR(100) NOT NULL,
       fecha_registro TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP NOT NULL
   );
   EOF
   ```

2. Cree un segundo script de migración (`V2__agregar_email_y_auditoria.sql`). Este script añadirá un nuevo campo (`email`) con restricciones y creará un índice optimizado para evitar bloqueos prolongados si la tabla escala:
   ```bash
   cat <<EOF > sql/migrations/V2__agregar_email_y_auditoria.sql
   -- Flyway Migration - Version 2: Añadir email e indice de busqueda
   ALTER TABLE clientes ADD COLUMN email VARCHAR(255) UNIQUE;
   CREATE INDEX idx_clientes_email ON clientes(email);
   EOF
   ```

3. Registre ambos archivos en el repositorio de Git local:
   ```bash
   git add sql/migrations/
   git commit -m "feat: add schema migrations V1 and V2"
   ```

**Resultado esperado:**
Dos archivos SQL estructurados en la ruta `sql/migrations/` y registrados en el árbol de confirmación de Git.

**Verificación:**
Valide que el historial de Git refleje de forma precisa la autoría y la trazabilidad de los esquemas creados:
```bash
git log --oneline --name-status
```
Debe listarse la confirmación "feat: add schema migrations V1 and V2" mostrando explícitamente los archivos de migración creados.

---

### Paso 4: Ejecutar Migraciones Automatizadas en el Entorno de Desarrollo

**Objetivo:** Ejecutar e inspeccionar el comportamiento de Flyway al interactuar con el entorno de Desarrollo (`pg-dev`), analizando la creación automática de la tabla de metadatos corporativa.

**Instrucciones:**

1. Ejecute Flyway utilizando su contenedor oficial Docker (`flyway/flyway:10.10.0`), montando el directorio local de migraciones al contenedor y apuntando directamente al puerto expuesto del contenedor de Desarrollo (`5433` que redirecciona a `pg-dev`).
   *(Nota: Se utiliza la variable de red `host.docker.internal` o el direccionamiento interno de la red Docker `pg_enterprise_net` para simplificar la interconexión entre contenedores)*:
   ```bash
   docker run --rm \
     --network pg_enterprise_net \
     -v ~/labs/db_devops_git/sql/migrations:/flyway/sql \
     flyway/flyway:10.10.0 \
     -url=jdbc:postgresql://pg-dev:5432/enterprise_db \
     -user=postgres \
     -password=PostgresAdminPass123! \
     migrate
   ```

2. Una vez finalizada la ejecución de Flyway, conéctese al contenedor `pg-dev` e inspeccione las tablas que han sido creadas:
   ```bash
   docker exec -i pg-dev psql -U postgres -d enterprise_db -c "\dt"
   ```

3. Realice una consulta detallada sobre la tabla de metadatos generada por Flyway (`flyway_schema_history`) para comprender el control interno de versiones:
   ```bash
   docker exec -i pg-dev psql -U postgres -d enterprise_db -c "SELECT installed_rank, version, description, type, checksum, success FROM flyway_schema_history;"
   ```

**Resultado esperado:**
La ejecución de Flyway debe reportar la aplicación exitosa de dos migraciones (`V1` y `V2`). La consulta a la lista de tablas debe arrojar la presencia de las tablas `clientes` y `flyway_schema_history`.

**Verificación:**
La respuesta de la última consulta SQL debe retornar dos registros con `success = t` (true) e indicar de forma exacta la firma matemática de validación (checksum) calculada para cada script SQL.

---

### Paso 5: Promover Esquemas Seguros y Ejecutar Despliegues en Producción

**Objetivo:** Utilizar el pipeline imperativo para asegurar que el entorno de Producción reciba los mismos cambios validados, garantizando el determinismo en el ciclo de vida del software.

**Instrucciones:**

1. Ejecute Flyway apuntando al servidor de **Producción** (`pg-prod`), aplicando exactamente los mismos archivos de migración registrados en Git:
   ```bash
   docker run --rm \
     --network pg_enterprise_net \
     -v ~/labs/db_devops_git/sql/migrations:/flyway/sql \
     flyway/flyway:10.10.0 \
     -url=jdbc:postgresql://pg-prod:5432/enterprise_db \
     -user=postgres \
     -password=PostgresAdminPass123! \
     migrate
   ```

2. Verifique de forma no invasiva la concordancia de la estructura lógica y la tabla de metadatos en el entorno de Producción (`pg-prod`):
   ```bash
   docker exec -i pg-prod psql -U postgres -d enterprise_db -c "SELECT version, description, success FROM flyway_schema_history;"
   ```

**Resultado esperado:**
Flyway aplicará secuencialmente las migraciones `V1` y `V2` sobre `pg-prod`.

**Verificación:**
La tabla de metadatos en el servidor de producción debe ser idéntica en versiones y descripciones a la observada en el servidor de desarrollo, lo que demuestra un despliegue idempotente y predecible.

---

## Validación y Pruebas

Para garantizar que el control de versiones funcione de forma estricta y que los mecanismos de protección contra alteraciones accidentales o maliciosas estén activos, realice las siguientes pruebas.

### Pruebas de Consistencia de Esquema

1. Verifique que la definición física de la tabla `clientes` sea idéntica en ambos contenedores de PostgreSQL:
   ```bash
   # Comprobar la definición de la tabla clientes en Desarrollo
   docker exec -i pg-dev psql -U postgres -d enterprise_db -c "\d clientes"

   # Comprobar la definición de la tabla clientes en Producción
   docker exec -i pg-prod psql -U postgres -d enterprise_db -c "\d clientes"
   ```
   Ambas salidas deben detallar las columnas `id` (integer), `nombre` (varchar), `fecha_registro` (timestamp with timezone), y `email` (varchar), con sus respectivas restricciones de índice y llave primaria.

### Escenario Adversario: Alteración Directa de un Script de Migración Ya Aplicado

En un entorno de producción regulado por auditorías (como ISO 27001 o NIST), se prohíbe la alteración retroactiva del historial de despliegues. Simularemos un error de desarrollo común: modificar un archivo de migración que ya fue desplegado para intentar "corregir" o añadir campos sin incrementar la versión del script.

1. Modifique el contenido del archivo `V1__crear_tabla_clientes.sql` que ya se encuentra aplicado en la base de datos de Desarrollo:
   ```bash
   # Simulación de una alteración maliciosa o descuidada por un desarrollador local
   cat <<EOF >> sql/migrations/V1__crear_tabla_clientes.sql
   -- Intento de agregar un campo adicional sin crear una nueva version de migración
   ALTER TABLE clientes ADD COLUMN telefono_contacto VARCHAR(20);
   EOF
   ```

2. Intente ejecutar el pipeline de validación de Flyway sobre el entorno de Desarrollo (`pg-dev`) para comprobar si el sistema detecta la anomalía:
   ```bash
   docker run --rm \
     --network pg_enterprise_net \
     -v ~/labs/db_devops_git/sql/migrations:/flyway/sql \
     flyway/flyway:10.10.0 \
     -url=jdbc:postgresql://pg-dev:5432/enterprise_db \
     -user=postgres \
     -password=PostgresAdminPass123! \
     validate
   ```

**Resultado esperado:**
El motor de Flyway calculará en tiempo real el checksum del archivo local modificado y lo contrastará con el checksum almacenado en la tabla `flyway_schema_history` del servidor `pg-dev`. Como resultado, **el proceso debe fallar inmediatamente** lanzando una excepción detallada de violación de consistencia.

La terminal debe mostrar un error con la firma de inconsistencia, similar a este fragmento:
```text
ERROR: Validate failed: Migration checksum mismatch for migration version 1
-> Applied to database : -1815123490
-> Local limits        : -294819024
```

3. Devuelva el archivo `V1__crear_tabla_clientes.sql` a su estado correcto y original para sanar nuestro entorno de trabajo local:
   ```bash
   git checkout -- sql/migrations/V1__crear_tabla_clientes.sql
   ```

4. Vuelva a ejecutar el comando de validación para asegurar la restauración de la integridad del flujo:
   ```bash
   docker run --rm \
     --network pg_enterprise_net \
     -v ~/labs/db_devops_git/sql/migrations:/flyway/sql \
     flyway/flyway:10.10.0 \
     -url=jdbc:postgresql://pg-dev:5432/enterprise_db \
     -user=postgres \
     -password=PostgresAdminPass123! \
     validate
   ```
   La salida ahora debe finalizar con éxito indicando que la validación fue satisfactoria.

---

## Solución de Problemas

A continuación se exponen dos de las incidencias más recurrentes reportadas durante la ejecución de este procedimiento y el método técnico óptimo para resolverlas:

### Caso 1: Error de Validación de Firma Digital (Checksum Mismatch Exception)

* **Síntomas:** El comando de migración de Flyway falla arrojando un error `FlywayException: Validate failed: Migration checksum mismatch` después de haber editado comentarios, espacios en blanco o el código SQL interno de un archivo de migración previo.
* **Causa:** Flyway garantiza la inmutabilidad de los depliegues históricos. Cualquier cambio a nivel de bytes dentro de un script de migración ya registrado provocará que el checksum local no coincida con el checksum de la tabla `flyway_schema_history`.
* **Resolución:**
  1. Si la modificación fue un error accidental, ejecute `git checkout -- sql/migrations/<nombre_del_archivo_afectado>` para restaurar su estado original almacenado en Git.
  2. Si la modificación fue intencional (por ejemplo, corregir un comentario no semántico) y se requiere forzar la aceptación de la nueva firma sin reconstruir la base de datos, ejecute el comando `repair` de Flyway para actualizar los metadatos del servidor con las firmas actuales:
     ```bash
     docker run --rm \
       --network pg_enterprise_net \
       -v ~/labs/db_devops_git/sql/migrations:/flyway/sql \
       flyway/flyway:10.10.0 \
       -url=jdbc:postgresql://pg-dev:5432/enterprise_db \
       -user=postgres \
       -password=PostgresAdminPass123! \
       repair
     ```

### Caso 2: Error de Conexión "Connection Refused" o "Host is unreachable"

* **Síntomas:** El contenedor de Flyway falla al inicio mostrando mensajes de timeout o denegación de conexión contra el puerto TCP `5432` de las bases de datos de destino.
* **Causa:** Falta de alineación en el direccionamiento de redes lógicas de Docker. El contenedor de Flyway no puede resolver los nombres de servicio DNS internos como `pg-dev` o `pg-prod` debido a que no fue adjuntado explícitamente a la red común `pg_enterprise_net`.
* **Resolución:**
  Asegúrese de proveer el parámetro `--network pg_enterprise_net` en la instrucción `docker run` y de referenciar la dirección del host de base de datos utilizando el nombre DNS asignado por Docker al contenedor (`pg-dev:5432` o `pg-prod:5432`), en lugar de usar direcciones locales ambiguas como `localhost` o `127.0.0.1` que apuntarían equivocadamente a la interfaz loopback interna de la jaula virtual del propio Flyway.

---

## Limpieza

Una vez completadas todas las actividades, elimine la infraestructura local y los recursos creados para restaurar la limpieza del sistema host:

```bash
## Detener y eliminar los contenedores PostgreSQL
docker stop pg-dev pg-prod
docker rm pg-dev pg-prod

## Eliminar la red de Docker creada para la práctica
docker network rm pg_enterprise_net

## Eliminar el directorio temporal del laboratorio
rm -rf ~/labs/db_devops_git
```

---

## Resumen

En este laboratorio práctico se ha implementado el paradigma de **Control de Cambios de Base de Datos como Código (Database-as-Code)** mediante el uso coordinado de Git y Flyway Community Edition 10.10.0. A través del despliegue imperativo y determinista en entornos independientes (`pg-dev` y `pg-prod`), se ha demostrado cómo mitigar la deriva de esquemas (*schema drift*) de forma automatizada y se comprendió el funcionamiento de los algoritmos de verificación y resguardo de inmutabilidad basados en firmas digitales (checksums) para auditar con éxito la seguridad en el ciclo de vida del software.

### Recursos Adicionales recomendados para profundizar:
* Documentación oficial de control de versiones de bases de datos: [Flyway Community Documentation](https://red-gate.com/products/flyway)
* Buenas prácticas de gestión de DDL en ambientes transaccionales: [PostgreSQL ALTER TABLE Lock Matrix](https://www.postgresql.org/docs/16/explicit-locking.html)
* Normativas de cumplimiento para auditorías de sistemas de persistencia: [NIST SP 800-53 - Database Integrity](https://csrc.nist.gov/)

---

# 2 Automatización de Migraciones con Liquibase

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

# 3 Gestión de Migraciones con Flyway

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

# 4 Diseño de una Tubería CI/CD para PostgreSQL

## Metadatos

| Métrica | Detalle |
| :--- | :--- |
| **Duración** | 90 minutos |
| **Complejidad** | Alta (Hard) |
| **Nivel de Bloom** | Crear (Create) |

## Descripción General

En esta práctica, el estudiante diseñará e implementará una solución local de Integración Continua (CI) para la automatización de cambios en esquemas de base de datos PostgreSQL 16.2. Utilizando Gitea como servidor Git local y Gitea Actions (basado en `act_runner`) como motor de automatización, se estructurará un pipeline capaz de realizar análisis estático, simulación de cambios (*dry-run*) y pruebas de despliegue sobre un contenedor efímero de PostgreSQL. Este enfoque elimina el factor manual en la aplicación de DDL, implementando prácticas DevOps bajo el principio de "Shift-Left" para prevenir problemas de concurrencia, bloqueos y deriva de esquemas antes de la integración en ramas principales.

## Objetivos de Aprendizaje

Al finalizar esta práctica, serás capaz de:
* [ ] Desplegar un servidor de control de versiones Gitea local y configurar un Gitea Runner integrado en la misma red interna de Docker.
* [ ] Construir un flujo de trabajo CI/CD automatizado mediante un archivo de configuración YAML (`ci-db.yml`) que responda a eventos de empuje (`git push`).
* [ ] Implementar análisis de sintaxis y validación de reglas de control de cambios utilizando Liquibase Community Edition 4.26.0.
* [ ] Orquestar simulaciones de despliegue en seco (*dry-run*) y pruebas funcionales de migraciones sobre contenedores PostgreSQL efímeros de prueba.

## Prerrequisitos

* **Conocimientos técnicos:**
  * Comprensión del funcionamiento de Git (comandos `commit`, `push`, gestión de repositorios).
  * Dominio intermedio de Docker y Docker Compose para la interconexión de servicios de red.
  * Familiaridad con la sintaxis de definición de cambios (changelogs) de Liquibase y consultas SQL (DDL/DML).
* **Entorno y Accesos:**
  * Acceso de administrador (`sudo` o privilegios equivalentes) en el host local.
  * Conexión estable a Internet para la descarga de imágenes y dependencias.
  * Haber completado satisfactoriamente las prácticas de base de datos previas de la sección 10 (especialmente 10.2).
* **Uso de Asistentes de IA:**
  * Si utiliza herramientas de asistencia de IA (como GitHub Copilot o Microsoft 365 Copilot Chat bajo sus respectivas licencias institucionales y configuraciones de privacidad empresarial), tenga en cuenta que el uso correcto de estas tecnologías requiere supervisión humana crítica. Esta práctica evaluará su capacidad para depurar anomalías que las herramientas generativas comúnmente omiten.

## Entorno de Laboratorio

El laboratorio se ejecutará de forma autónoma sobre un entorno de contenedores aislados. A continuación se listan los recursos tecnológicos requeridos:

### Especificaciones de Software y Fuentes Oficiales

| Tecnología | Edición / Arquitectura | Versión Exacta | Enlace de Descarga Oficial |
| :--- | :--- | :--- | :--- |
| **PostgreSQL** | Community Edition (Linux x86_64) | 16.2 | [Postgres Docker Hub](https://registry.hub.docker.com/_/postgres) |
| **Gitea** | Community Edition (Linux x86_64) | 1.21.6 | [Gitea Docker Hub](https://registry.hub.docker.com/r/gitea/gitea) |
| **Gitea Runner (act)** | Community Edition (Linux x86_64) | 0.2.6 | [Gitea Runner Repo](https://gitea.com/gitea/act_runner) |
| **Liquibase** | Community Edition (Linux x86_64) | 4.26.0 | [Liquibase Docker Hub](https://registry.hub.docker.com/_/liquibase/liquibase) |
| **Docker Engine** | Docker Desktop/CE (x86_64) | 26.0.0 | [Docker Docs](https://docs.docker.com/desktop/) |

### Parámetros Globales del Entorno

* **Red de Docker (Bridge):** `pg_enterprise_net`
* **Base de Datos Destino:** `enterprise_db`
* **Administrador Global:** `postgres` / `PostgresAdminPass123!`
* **Puerto de Acceso Gitea:** `3000` (Local)
* **Puerto Base de Datos de Prueba Efímera:** Interno `5432` (Nombre del contenedor: `pg-test`)

### Comandos de Inicialización del Espacio de Trabajo

Ejecute los siguientes comandos en la terminal del sistema anfitrión para crear la estructura de carpetas necesaria y la red común:

```bash
## Crear la red de Docker dedicada si no existe
docker network create pg_enterprise_net || true

## Crear directorios para persistencia de datos y desarrollo del lab
mkdir -p ~/labs/lab06_cicd/gitea-data
mkdir -p ~/labs/lab06_cicd/runner-data
mkdir -p ~/labs/lab06_cicd/repo-liquibase/db/migrations

## Cambiar al directorio de trabajo raíz
cd ~/labs/lab06_cicd
```

---

## Instrucciones Paso a Paso

### Paso 1: Creación del Docker Compose para Gitea y Runner

**Objetivo:** Configurar una pila multiservicio que incluya el servidor de control de versiones local Gitea y el motor de ejecución de tareas automatizadas (Runner) enlazados a la misma red de red virtual.

**Instrucciones:**

1. Cree un archivo de definición de composición denominado `docker-compose.yml` en la carpeta `~/labs/lab06_cicd`:

```bash
nano docker-compose.yml
```

2. Añada el siguiente contenido, el cual declara los servicios de red, almacenamiento persistente, variables de entorno necesarias para habilitar el motor de acciones y mapeo del socket de Docker para el Runner:

```yaml
version: '3.8'

networks:
  pg_enterprise_net:
    external: true

services:
  gitea:
    image: gitea/gitea:1.21.6
    container_name: gitea-server
    environment:
      - USER_UID=1000
      - USER_GID=1000
      - GITEA__database__DB_TYPE=sqlite3
      - GITEA__security__INSTALL_LOCK=false
      - GITEA__actions__ENABLED=true
    volumes:
      - ./gitea-data:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"
      - "2222:22"
    networks:
      - pg_enterprise_net
    restart: always

  gitea-runner:
    image: gitea/act_runner:0.2.6
    container_name: gitea-runner
    environment:
      - CONFIG_FILE=/config.yaml
      - GITEA_INSTANCE_URL=http://gitea-server:3000
      - RUNNER_NAME=local-docker-runner
      - RUNNER_LABELS=ubuntu-latest:docker://node:16-bullseye,ubuntu-22.04:docker://node:16-bullseye
    volumes:
      - ./runner-data:/data
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - pg_enterprise_net
    restart: always
    depends_on:
      - gitea
```

> **Nota Crítica sobre Seguridad y Arquitectura (Docker-in-Docker):**
> Al mapear `/var/run/docker.sock` al contenedor del Runner, este adquiere permisos para interactuar con la API de Docker del host. Cualquier contenedor secundario ("hermano") iniciado desde el workflow (como la instancia de base de datos efímera) se creará a nivel de host compartiendo la misma red `pg_enterprise_net`, asegurando que puedan comunicarse entre sí.

3. Inicie el servicio de Gitea (únicamente) para realizar la instalación inicial y obtener las credenciales de ejecución del Runner:

```bash
docker compose up -d gitea
```

**Resultado Esperado:** El contenedor `gitea-server` se inicia correctamente.

**Verificación:** Ejecute el comando de monitoreo de estado para validar que el servicio está en línea y escuchando en el puerto 3000.

```bash
docker ps --filter "name=gitea-server"
```

---

### Paso 2: Configuración Inicial de Gitea y Registro del Runner

**Objetivo:** Configurar la instalación base de Gitea, habilitar el módulo de Actions a nivel global y registrar el agente ejecutor local utilizando tokens seguros de autorización.

**Instrucciones:**

1. Abra un navegador web y acceda a `http://localhost:3000`.
2. Gitea mostrará la página de instalación inicial. Dado que hemos configurado SQLite y los parámetros básicos por variables de entorno:
   * Desplácese al final de la página.
   * Cree una cuenta de usuario administrador. Complete los campos:
     * **Nombre de usuario administrador:** `gitea_admin`
     * **Contraseña:** `PostgresAdminPass123!`
     * **Confirmar contraseña:** `PostgresAdminPass123!`
     * **Correo electrónico:** `admin@enterprise.local`
   * Haga clic en el botón **Instalar Gitea**.
3. Una vez completado el redireccionamiento e iniciada la sesión del usuario administrador:
   * Diríjase al panel de administración del sitio web accediendo a la URL: `http://localhost:3000/admin/actions/runners` (o haciendo clic en su imagen de perfil -> **Administración del sitio** -> **Acciones** -> **Ejecutores**).
   * Haga clic en el botón azul **Crear nuevo ejecutor** (Create new Runner).
   * Verá una ventana emergente que proporciona un parámetro denominado **Registration Token**. Cópielo.

4. Ahora, registre manualmente el contenedor del runner usando el token obtenido. Ejecute el siguiente comando reemplazando `<REGISTRATION_TOKEN>` por la cadena copiada en el paso anterior:

```bash
docker exec -it gitea-runner act_runner register \
  --instance http://gitea-server:3000 \
  --token <REGISTRATION_TOKEN> \
  --no-interactive
```

5. Inicie el contenedor del Runner para que comience a escuchar por trabajos encolados:

```bash
docker compose up -d gitea-runner
```

**Resultado Esperado:** La consola indicará el registro exitoso del ejecutor con el mensaje "Runner registered successfully."

**Verificación:** Refresque la página de administración de Gitea (`http://localhost:3000/admin/actions/runners`). El runner `local-docker-runner` debe figurar en la lista en estado activo (un círculo de color verde).

---

### Paso 3: Inicialización del Repositorio Local y Estructura de Liquibase

**Objetivo:** Crear una estructura estandarizada de base de datos como código que contenga archivos SQL versionados de migración y la configuración del motor de cambios.

**Instrucciones:**

1. Cámbiese al directorio del repositorio de cambios:

```bash
cd ~/labs/lab06_cicd/repo-liquibase
```

2. Cree un archivo de configuración base para Liquibase con nombre `liquibase.properties` que apunte al contenedor efímero `pg-test` que se levantará dinámicamente en el pipeline:

```properties
changeLogFile=changelog.xml
url=jdbc:postgresql://pg-test:5432/enterprise_db
username=postgres
password=PostgresAdminPass123!
driver=org.postgresql.Driver
```

3. Defina el archivo de control maestro del esquema de Liquibase denominado `changelog.xml` en la raíz de su carpeta de trabajo:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.26.xsd">

    <include file="db/migrations/V1.0__init_schema.sql" relativeToChangelogFile="true"/>
</databaseChangeLog>
```

4. Cree el script SQL inicial de definición de tablas. Este archivo se ubicará en `db/migrations/V1.0__init_schema.sql`:

```sql
-- liquibase formatted sql
-- changeset autor:instructor dbms:postgresql
-- comment: Crear estructura inicial de usuarios y auditoria corporativa
CREATE TABLE usuarios (
    id SERIAL PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    fecha_registro TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE logs_auditoria (
    id SERIAL PRIMARY KEY,
    usuario_id INT REFERENCES usuarios(id) ON DELETE CASCADE,
    accion TEXT NOT NULL,
    fecha TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Resultado Esperado:** Se ha establecido un esquema ordenado de base de datos declarando dependencias de clave foránea y campos no nulos.

**Verificación:** Compruebe la estructura de archivos ejecutando:

```bash
find . -not -path '*/.*'
```

Debe retornar exactamente:
```text
.
./liquibase.properties
./changelog.xml
./db
./db/migrations
./db/migrations/V1.0__init_schema.sql
```

---

### Paso 4: Creación de la Tubería de CI/CD (Gitea Actions)

**Objetivo:** Diseñar y estructurar la definición del flujo de trabajo automatizado que orquestará el ciclo de vida de validación, simulación y aplicación en ambientes efímeros ante eventos de integración de código.

**Instrucciones:**

1. Cree el directorio de configuración para flujos de trabajo de automatización de Gitea en la raíz de su repositorio local:

```bash
mkdir -p .gitea/workflows
```

2. Genere el archivo de pipeline denominado `ci-db.yml`:

```bash
nano .gitea/workflows/ci-db.yml
```

3. Inyecte la siguiente definición detallada del flujo. Note la integración precisa de etapas críticas de validación y simulación (*dry-run*):

```yaml
name: CI Database Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  validate-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v3

      - name: Start Ephemeral PostgreSQL Container
        run: |
          # Eliminar si existe un contenedor huérfano con el mismo nombre para evitar conflictos
          docker rm -f pg-test || true
          
          # Levantar instancia de PostgreSQL conectada a la red común de Docker
          docker run --name pg-test \
            --network pg_enterprise_net \
            -e POSTGRES_DB=enterprise_db \
            -e POSTGRES_PASSWORD=PostgresAdminPass123! \
            -d postgres:16.2
            
          # Bucle activo de validación de servicio (Healthcheck funcional)
          echo "Esperando que el motor de PostgreSQL esté en línea..."
          for i in {1..30}; do
            if docker exec pg-test pg_isready -U postgres -d enterprise_db; then
              echo "PostgreSQL está listo para recibir conexiones."
              break
            fi
            sleep 1
          done

      - name: Liquibase Validate (Sintaxis y Estructura)
        run: |
          docker run --rm \
            --network pg_enterprise_net \
            -v ${{ github.workspace }}:/liquibase/changelog \
            liquibase/liquibase:4.26.0 \
            --changelog-file=changelog.xml \
            --url=jdbc:postgresql://pg-test:5432/enterprise_db \
            --username=postgres \
            --password=PostgresAdminPass123! \
            validate

      - name: Liquibase Dry-Run (Simulación de Cambios SQL)
        run: |
          docker run --rm \
            --network pg_enterprise_net \
            -v ${{ github.workspace }}:/liquibase/changelog \
            liquibase/liquibase:4.26.0 \
            --changelog-file=changelog.xml \
            --url=jdbc:postgresql://pg-test:5432/enterprise_db \
            --username=postgres \
            --password=PostgresAdminPass123! \
            update-sql

      - name: Liquibase Apply (Prueba Funcional de Despliegue)
        run: |
          docker run --rm \
            --network pg_enterprise_net \
            -v ${{ github.workspace }}:/liquibase/changelog \
            liquibase/liquibase:4.26.0 \
            --changelog-file=changelog.xml \
            --url=jdbc:postgresql://pg-test:5432/enterprise_db \
            --username=postgres \
            --password=PostgresAdminPass123! \
            update

      - name: Validate Tables Creation on PostgreSQL
        run: |
          # Realizar una consulta de auditoría rápida para asegurar la persistencia física
          docker exec pg-test psql -U postgres -d enterprise_db -c "\dt"
          docker exec pg-test psql -U postgres -d enterprise_db -c "SELECT * FROM databasechangelog;"

      - name: Cleanup Test Environment
        if: always()
        run: |
          echo "Removiendo contenedor efímero pg-test..."
          docker rm -f pg-test || true
```

**Resultado Esperado:** Se ha redactado un flujo de trabajo que previene la degradación silenciosa del esquema simulando una migración real en un entorno completamente estéril e independiente antes de fusionar.

**Verificación:** Asegúrese de que el archivo YAML tenga el espaciado de indentación correcto y los comandos estén limpios de caracteres especiales extraños.

---

### Paso 5: Ejecución y Monitoreo del Pipeline de Integración Continua

**Objetivo:** Publicar el repositorio local en Gitea para disparar la ejecución en el runner y monitorizar las trazas operativas del ciclo de validación.

**Instrucciones:**

1. Cree un repositorio en blanco en Gitea:
   * Acceda a `http://localhost:3000/repo/create`.
   * **Nombre del Repositorio:** `enterprise-db-migrations`
   * Deje las opciones de inicialización en blanco (sin agregar `.gitignore`, licencia, ni README).
   * Haga clic en el botón **Crear repositorio**.

2. Inicialice Git de manera local dentro de `~/labs/lab06_cicd/repo-liquibase`:

```bash
git init
git checkout -b main
```

3. Vincule el repositorio local con el servidor Gitea agregando el origen remoto:

```bash
git remote add origin http://localhost:3000/gitea_admin/enterprise-db-migrations.git
```

4. Agregue todos los archivos al índice del repositorio, cree un commit inicial y envíe los cambios a la rama principal:

```bash
git add .
git commit -m "feat: inicializar esquema de base de datos con tubería de CI/CD"
git push -u origin main
```

5. Gitea le solicitará credenciales para realizar el empuje de datos. Ingrese el usuario `gitea_admin` y la contraseña `PostgresAdminPass123!`.

**Resultado Esperado:** El push se completa satisfactoriamente hacia el servidor de versiones Git.

**Verificación:** Diríjase a su repositorio en la interfaz web de Gitea `http://localhost:3000/gitea_admin/enterprise-db-migrations` y seleccione la pestaña **Acciones**. Debería ver un trabajo en ejecución o completado para el commit ejecutado. Haga clic sobre él para inspeccionar el flujo de ejecución de los contenedores Docker efímeros.

---

## Validación y Pruebas

Para garantizar que el laboratorio funciona de manera óptima, debe superar criterios de aceptación estrictos y pruebas de resistencia (*adversarial testing*).

### Criterios de Aceptación y Evidencia Requerida

1. **Evidencia de Ejecución Exitosa (Consola de Gitea Actions):**
   * El paso `Liquibase Validate` debe registrar un código de salida `0`, indicando que la estructura XML y las referencias SQL no tienen discrepancias sintácticas.
   * El paso `Liquibase Dry-Run (Simulación)` debe imprimir en las trazas las sentencias `CREATE TABLE` nativas de PostgreSQL que habrían sido aplicadas, sin alterar la base de datos real en ese instante.
   * El paso `Validate Tables Creation` debe mostrar la tabla de metadatos `databasechangelog` junto con las tablas `usuarios` y `logs_auditoria` creadas dinámicamente en el contenedor efímero.

### Prueba de Robustez y Fallo (Caso Adversario)

A continuación, ejecutaremos una prueba de inyección de errores sintácticos y lógicos para verificar que el pipeline actúa de manera determinista e impide la integración de cambios rotos en el repositorio de control.

1. Añada una nueva migración con un error lógico y de sintaxis intencional en la carpeta de su repositorio local. Cree el archivo `db/migrations/V1.1__broken_migration.sql`:

```sql
-- liquibase formatted sql
-- changeset desarrollador_junior:integridad dbms:postgresql
-- comment: Intento fallido de crear tabla con clave foránea rota y error de sintaxis

-- Error 1: Tipo de dato inexistente en PostgreSQL (VARCHAR_BROKEN)
-- Error 2: Referencia de clave foránea a tabla inexistente (tabla_fantasma)
CREATE TABLE ordenes_compra (
    id SERIAL PRIMARY KEY,
    monto NUMERIC(12,2) NOT NULL,
    descripcion VARCHAR_BROKEN(255),
    cliente_id INT REFERENCES tabla_fantasma(id)
);
```

2. Registre la migración en el archivo de control maestro `changelog.xml` agregando la línea correspondiente al final del archivo, justo antes de cerrar la etiqueta `</databaseChangeLog>`:

```xml
    <include file="db/migrations/V1.1__broken_migration.sql" relativeToChangelogFile="true"/>
```

3. Guarde los cambios y realice un nuevo push al repositorio de producción:

```bash
git add .
git commit -m "fix: agregar tabla de ordenes de compra con errores estructurales"
git push origin main
```

4. **Monitoreo y Resultado Esperado del Pipeline Fallido:**
   * Diríjase inmediatamente a la pestaña de **Acciones** en Gitea.
   * El pipeline comenzará a procesar los pasos. Una vez llegado al paso `Liquibase Apply (Prueba Funcional de Despliegue)`, la herramienta intentará ejecutar el DDL sobre el contenedor efímero de pruebas `pg-test`.
   * El validador del motor de PostgreSQL rechazará inmediatamente el tipo de dato no admitido (`VARCHAR_BROKEN`) y la clave foránea inconsistente.
   * El paso `Liquibase Apply` debe **fallar con código de error** (círculo rojo), deteniendo la ejecución y previniendo la ejecución de los comandos subsiguientes de validación.
   * El contenedor efímero se destruirá de manera segura gracias a la condición de ejecución `if: always()` en el bloque de limpieza.

5. **Resolución del Error (Operación de Corrección):**
   * Corrija el archivo `db/migrations/V1.1__broken_migration.sql` modificándolo por el DDL correcto compatible con PostgreSQL:

```sql
-- liquibase formatted sql
-- changeset desarrollador_junior:integridad dbms:postgresql
-- comment: Corrección de la tabla de órdenes de compra para producción
CREATE TABLE ordenes_compra (
    id SERIAL PRIMARY KEY,
    monto NUMERIC(12,2) NOT NULL,
    descripcion VARCHAR(255),
    usuario_id INT REFERENCES usuarios(id) ON DELETE CASCADE
);
```

6. Vuelva a registrar los archivos en el repositorio, realice el push y confirme que el pipeline vuelve a su estado exitoso ("En Verde").

```bash
git add .
git commit -m "fix: corregir migración rota aplicando tipo de dato válido y FK correcta"
git push origin main
```

---

## Solución de Problemas

A continuación se exponen las dos incidencias más frecuentes asociadas al despliegue de esta arquitectura de integración continua y su correspondiente mitigación técnica.

### Incidencia 1: Error de conexión en el pipeline ("Connection Refused" al conectar Liquibase con pg-test)

* **Síntomas:** El pipeline de Gitea falla en la sección de validación con el error de Java: `Connection refused: ... to pg-test:5432`.
* **Causa:** El contenedor `pg-test` y el contenedor del Runner no están asociados a la misma red de Docker (`pg_enterprise_net`), o bien el contenedor `pg-test` tarda más tiempo en inicializar sus procesos internos que el asignado por el bucle activo de espera antes de que comience a ejecutarse la acción de Liquibase.
* **Resolución:**
  1. Ejecute `docker network inspect pg_enterprise_net` en el host local y valide que el contenedor `gitea-runner` se encuentra enlistado dentro de la sección de contenedores.
  2. Incremente la tolerancia del bucle de espera activa (*healthcheck*) en el archivo `.gitea/workflows/ci-db.yml`. Cambie la cantidad de reintentos de `30` a `60` segundos para dar espacio en computadores con almacenamiento lento de disco mecánico o baja memoria RAM.

### Incidencia 2: Conflicto por nombres de contenedores duplicados ("Container name already in use")

* **Síntomas:** El pipeline muestra fallos de ejecución en el comando `docker run` inicial para levantar `pg-test`, indicando en la salida de error estándar: `Conflict. The container name "/pg-test" is already in use by container...`.
* **Causa:** Una ejecución previa del pipeline se detuvo abruptamente antes de procesar el paso de limpieza (`Cleanup Test Environment`), o la instrucción de eliminación no se ejecutó debido a una caída de red o cierre abrupto del daemon del Runner, dejando el contenedor de pruebas activo en segundo plano.
* **Resolución:**
  Inyecte una instrucción explícita de eliminación incondicional al inicio del paso de arranque de la base de datos de pruebas para limpiar el entorno antes de inicializar la compilación. El archivo YAML provisto en esta guía ya contiene la mitigación nativa: `docker rm -f pg-test || true` justo antes de realizar el `docker run`. Asegúrese de no haber removido esta línea protectora de su flujo.

---

## Limpieza

Una vez finalizada la validación de las ejecuciones, libere los recursos del sistema anfitrión:

```bash
## Apagar la infraestructura de Gitea y el Runner eliminando volúmenes persistentes
cd ~/labs/lab06_cicd
docker compose down -v

## Asegurar la eliminación de cualquier contenedor efímero remanente
docker rm -f pg-test || true

## Eliminar directorios de trabajo locales creados durante la práctica
rm -rf ~/labs/lab06_cicd
```

---

## Resumen

En esta práctica se ha diseñado y consolidado un flujo moderno de DevOps aplicado a bases de datos relacionales en entornos empresariales PostgreSQL 16.2. 

A través de las fases estructuradas, se alcanzaron los siguientes hitos técnicos:
1. **Infraestructura como Código Local:** Se implementó una solución GitOps local (Gitea y Gitea Runner) que emula las capacidades de automatización industrial de GitHub Enterprise o GitLab CI.
2. **Definición de Pipelines de Datos:** Se orquestó un archivo de definición de integración automatizada (`ci-db.yml`) que automatiza por completo la inicialización, chequeo y destrucción de entornos de prueba estériles e independientes.
3. **Validación Temprana (Shift-Left):** Se incorporaron fases obligatorias de análisis estático, verificación de integridad física del changelog con Liquibase y simulaciones de salida (*dry-run*), aislando fallas causadas por incompatibilidad de tipos o malas prácticas de diseño relacional antes de que se incorporen en bases de datos productivas.

*Lectura Complementaria Recomendada:* Se sugiere consultar la guía oficial de [Liquibase Best Practices](https://docs.liquibase.com/concepts/best-practices.html) y la documentación de automatización integrada de [Gitea Actions Documentation](https://docs.gitea.com/usage/actions/overview) para profundizar en políticas de control de versiones y despliegue continuo de bases de datos.

---

# 5 Simulación de Despliegue Controlado entre DEV, QA y PROD

## Metadatos

| Atributo | Detalle |
| :--- | :--- |
| **Duración** | 90 minutos |
| **Dificultad** | Avanzada (Hard) |
| **Nivel Bloom** | Aplicar (Apply) |
| **Autor** | Instructor Técnico Experto |
| **Tecnologías** | PostgreSQL 16.2, Liquibase Community 4.26.0, Docker Engine 26.0.0 |

---

## Descripción General

En esta práctica de laboratorio, modelarás y ejecutarás un flujo completo de Entrega Continua (CD) para bases de datos PostgreSQL sobre una infraestructura multi-ambiente simulada localmente. Configurarás tres entornos aislados (`pg-dev`, `pg-qa` y `pg-prod`) utilizando contenedores Docker en una red compartida. A través de Liquibase, diseñarás un esquema de migración condicional utilizando contextos (`contexts`), permitiendo desplegar datos de prueba únicamente en el entorno de desarrollo, y restricciones estrictas solo en control de calidad y producción. 

Para emular un entorno de producción real de misión crítica, implementarás un pipeline en Bash (simulando un motor de CI/CD como Gitea Actions/Runner) que realiza un respaldo preventivo en caliente (*hot backup*) antes de aplicar cambios en producción. Posteriormente, inyectarás un error sintáctico intencional para forzar un fallo en el despliegue de producción, validando la ejecución automática del proceso de restauración (*rollback*) y asegurando que no se consolide deriva de esquemas (*schema drift*) ni estados corruptos.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] **Modelar** una topología multi-ambiente (DEV, QA, PROD) utilizando contenedores Docker independientes bajo la red aislada `pg_enterprise_net`.
- [ ] **Configurar y estructurar** un proyecto de Liquibase 4.26.0 aplicando contextos (`contexts`) para canalizar cambios selectivos según el entorno de destino.
- [ ] **Diseñar y automatizar** un pipeline de despliegue controlado que incorpore aprobaciones manuales y respaldos en caliente preventivos con `pg_dump`.
- [ ] **Implementar y validar** una estrategia de recuperación de desastres ante fallas de migración mediante rollback automatizado a nivel de base de datos física.
- [ ] **Evaluar y mitigar** la deriva de esquemas (*schema drift*) mediante técnicas de verificación cruzada entre entornos.

---

## Prerrequisitos

### Conocimientos Requeridos
- Fundamentos de administración de PostgreSQL (roles, esquemas, `pg_dump` y `pg_restore`).
- Comprensión de conceptos de DevOps para bases de datos (enfoque imperativo vs. declarativo, control de versiones de bases de datos).
- Familiaridad intermedia con la sintaxis de Liquibase (XML Changelogs, Changesets, Preconditions, Contexts).
- Dominio de scripting en Bash y operaciones de Docker.

### Accesos y Licenciamiento de Software
1. **Docker Engine / Desktop 26.0.0** (Licencia Apache 2.0 / Suscripción comercial de Docker según el tamaño de la organización). [Enlace Oficial de Descarga](https://docs.docker.com/engine/release-notes/26.0/).
2. **PostgreSQL Community Edition 16.2 x86_64** (Licencia PostgreSQL). [Enlace Oficial de Descarga](https://www.postgresql.org/download/).
3. **Liquibase Community Edition 4.26.0** (Licencia Apache 2.0). [Enlace Oficial de Descarga](https://www.liquibase.com/download).
4. **Herramientas del Sistema**: Bash (v4 o superior), `cat`, `grep`, `sleep`, cliente `psql`.

> 💡 **Nota de Terminología de Inteligencia Artificial (Cumplimiento THOR 1):** Es fundamental distinguir entre una *instrucción de chat temporal (prompt)* —la cual es un mensaje de un solo uso enviado a una interfaz conversacional para obtener una respuesta inmediata— y un *asistente de IA persistente (system message / agent)* —el cual opera de manera continua con contexto de sistema persistente, memoria a largo plazo y directrices de comportamiento predefinidas. En este laboratorio, los flujos automáticos se ejecutarán mediante scripts deterministas para evitar la variabilidad e incertidumbre que introduciría una IA sin supervisión humana.

---

## Entorno de Laboratorio

La infraestructura de red y almacenamiento se desplegará localmente en tu estación de trabajo. Asegúrate de cumplir con los mínimos de hardware especificados para evitar problemas de concurrencia o de cuellos de botella de disco durante la inicialización de las tres instancias de PostgreSQL.

### Especificaciones de Hardware
- **Procesador**: Arquitectura x86_64 con mínimo 8 núcleos físicos.
- **Memoria RAM**: Mínimo 16 GB (Se recomiendan 32 GB para mantener los tres contenedores y las herramientas de ejecución simultáneamente).
- **Almacenamiento**: Mínimo 100 GB de espacio libre en disco de estado sólido (SSD NVMe recomendado).

### Variables Globales de Entorno

| Variable | Valor de Configuración | Descripción |
| :--- | :--- | :--- |
| **Docker Network** | `pg_enterprise_net` | Red tipo bridge para comunicación inter-contenedor |
| **Base de Datos Global** | `enterprise_db` | Nombre de la base de datos objetivo en los 3 ambientes |
| **Usuario Administrador**| `postgres` | Superusuario de las instancias |
| **Password Administrador**| `PostgresAdminPass123!` | Contraseña predefinida segura para el laboratorio |
| **Puerto DEV** | `5433` | Puerto mapeado al host para el ambiente de Desarrollo |
| **Puerto QA** | `5434` | Puerto mapeado al host para el ambiente de Control de Calidad |
| **Puerto PROD** | `5432` | Puerto mapeado al host para el ambiente de Producción |

### Comandos de Inicialización del Entorno

Ejecuta las siguientes instrucciones en tu terminal para limpiar cualquier contenedor previo que pueda generar colisiones de puertos y crear la red Docker necesaria:

```bash
## Limpiar contenedores previos con nombres idénticos (si existen)
docker rm -f pg-dev pg-qa pg-prod 2>/dev/null || true

## Crear la red del laboratorio en caso de que no exista
docker network create pg_enterprise_net 2>/dev/null || true

## Verificar que la red se haya creado correctamente
docker network inspect pg_enterprise_net | grep Name
```

---

## Instrucciones Paso a Paso

### Paso 1: Configurar la topología de contenedores (DEV, QA, PROD)

**Objetivo:** Levantar tres instancias independientes de PostgreSQL 16.2 en contenedores Docker aislados, simulando la separación física y de red de los entornos de desarrollo (DEV), aseguramiento de calidad (QA) y producción (PROD).

**Instrucciones:**

1. Levanta el contenedor de Desarrollo (`pg-dev`) mapeado al puerto del host `5433`:
```bash
docker run -d \
  --name pg-dev \
  --network pg_enterprise_net \
  -p 5433:5432 \
  -e POSTGRES_DB=enterprise_db \
  -e POSTGRES_PASSWORD='PostgresAdminPass123!' \
  postgres:16.2
```

2. Levanta el contenedor de Control de Calidad (`pg-qa`) mapeado al puerto del host `5434`:
```bash
docker run -d \
  --name pg-qa \
  --network pg_enterprise_net \
  -p 5434:5432 \
  -e POSTGRES_DB=enterprise_db \
  -e POSTGRES_PASSWORD='PostgresAdminPass123!' \
  postgres:16.2
```

3. Levanta el contenedor de Producción (`pg-prod`) mapeado al puerto del host `5432`:
```bash
docker run -d \
  --name pg-prod \
  --network pg_enterprise_net \
  -p 5432:5432 \
  -e POSTGRES_DB=enterprise_db \
  -e POSTGRES_PASSWORD='PostgresAdminPass123!' \
  postgres:16.2
```

4. Espera 10 segundos para garantizar que los motores de base de datos hayan completado su proceso de inicialización interno e inspecciona sus estados de ejecución:
```bash
sleep 10
docker ps --filter "name=pg-"
```

**Expected output:**
Una lista de tres contenedores en estado `Up` con sus respectivos mapeos de puertos:
```text
CONTAINER ID   IMAGE           COMMAND                  CREATED         STATUS         PORTS                    NAMES
xxxxxxxxxxxx   postgres:16.2   "docker-entrypoint.s…"   10 seconds ago  Up 9 seconds   0.0.0.0:5432->5432/tcp   pg-prod
xxxxxxxxxxxx   postgres:16.2   "docker-entrypoint.s…"   10 seconds ago  Up 9 seconds   0.0.0.0:5434->5432/tcp   pg-qa
xxxxxxxxxxxx   postgres:16.2   "docker-entrypoint.s…"   10 seconds ago  Up 9 seconds   0.0.0.0:5433->5432/tcp   pg-dev
```

**Verification:**
Ejecuta una consulta rápida a cada contenedor utilizando el cliente `psql` embebido para verificar la conectividad y que la base de datos `enterprise_db` exista:
```bash
for port in 5433 5434 5432; do
  docker run --rm --network pg_enterprise_net postgres:16.2 \
    psql -h pg-dev -U postgres -d enterprise_db -c "SELECT version();" > /dev/null \
    && echo "Puerto $port: CONECTADO" || echo "Puerto $port: FALLÓ"
done
```
*(Nota: El comando interno valida el acceso a través de la red bridge de Docker).*

---

### Paso 2: Estructurar el Proyecto de Liquibase y Definir los Contextos

**Objetivo:** Crear una estructura de directorios estandarizada de tipo "Database-as-Code" y configurar un archivo changelog maestro de Liquibase que diferencie los despliegues de base de datos según el entorno objetivo mediante el uso de contextos (`dev`, `qa`, `prod`).

**Instrucciones:**

1. Crea el árbol de directorios del proyecto en tu máquina local:
```bash
mkdir -p ~/liquibase-deploy/changelog
cd ~/liquibase-deploy
```

2. Genera el archivo maestro de configuración de Liquibase (`liquibase.properties`). Este archivo servirá como plantilla base. El pipeline posterior inyectará dinámicamente las credenciales de conexión según el ambiente:
```cat << 'EOF' > liquibase.properties
## Configuración global por defecto
changeLogFile=changelog/db.changelog-master.xml
logLevel=info
EOF
```

3. Crea el archivo changelog maestro en formato XML (`changelog/db.changelog-master.xml`). Este archivo definirá la evolución del esquema corporativo empleando contextos específicos:
   * **Changeset 1 (Context: Todos):** Creación de la tabla `clientes`.
   * **Changeset 2 (Context: `dev`):** Inserción de datos ficticios de prueba (dummy data), requeridos solo para desarrolladores.
   * **Changeset 3 (Context: `qa` o `prod`):** Creación de un índice especializado para optimizar consultas de producción y una restricción `CHECK` para forzar la consistencia del campo email.

```cat << 'EOF' > changelog/db.changelog-master.xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.26.xsd">

    <!-- Changeset 1: Estructura Base (Común a todos los ambientes) -->
    <changeSet id="1.0.0-crear-clientes" author="arquitecto_db">
        <createTable tableName="clientes">
            <column name="id" type="BIGINT">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="nombre" type="VARCHAR(100)">
                <constraints nullable="false"/>
            </column>
            <column name="email" type="VARCHAR(150)">
                <constraints nullable="false"/>
            </column>
            <column name="creado_en" type="TIMESTAMP" defaultValueComputed="CURRENT_TIMESTAMP"/>
        </createTable>
    </changeSet>

    <!-- Changeset 2: Datos dummy de prueba (Exclusivo de DEV) -->
    <changeSet id="1.1.0-datos-dummy-dev" author="developer" context="dev">
        <insert tableName="clientes">
            <column name="id" valueNumeric="1"/>
            <column name="nombre" value="Cliente de Pruebas DEV"/>
            <column name="email" value="test-dev@enterprise.local"/>
        </insert>
        <insert tableName="clientes">
            <column name="id" valueNumeric="2"/>
            <column name="nombre" value="QA Tester Shadow"/>
            <column name="email" value="shadow-dev@enterprise.local"/>
        </insert>
    </changeSet>

    <!-- Changeset 3: Índice y Restricción de Calidad/Producción (Exclusivo de QA y PROD) -->
    <changeSet id="1.2.0-optimizacion-prod-qa" author="dba_senior" context="qa or prod">
        <createIndex indexName="idx_clientes_email_hash" tableName="clientes">
            <column name="email"/>
        </createIndex>
        <addCheckConstraint constraintName="chk_email_format" 
                            tableName="clientes" 
                            columnNames="email" 
                            constraintText="email LIKE '%@%.%'"/>
    </changeSet>

</databaseChangeLog>
EOF
```

**Expected output:**
La estructura del directorio local debe verse de la siguiente manera al ejecutar `find .`:
```text
.
./liquibase.properties
./changelog
./changelog/db.changelog-master.xml
```

**Verification:**
Inspecciona el contenido del archivo XML para asegurar que no se hayan introducido caracteres especiales no deseados o cortes en las etiquetas XML:
```bash
grep -E "changeSet id=|context=" changelog/db.changelog-master.xml
```
Debe retornar las tres definiciones de changeset con sus respectivos identificadores y contextos asignados.

---

### Paso 3: Diseñar el Script del Pipeline de CD Automatizado con Respaldo Preventivo

**Objetivo:** Desarrollar un script robusto en Bash que actúe como motor de orquestación de CD. Este script simulará de forma exacta el comportamiento secuencial de un pipeline de producción, incorporando pasos de aprobación interactiva, verificación de prerequisitos y un sistema de **respaldo preventivo con rollback automático** antes de aplicar migraciones críticas en el ambiente de producción (`pg-prod`).

**Instrucciones:**

1. Crea el script de orquestación llamado `deploy_pipeline.sh` dentro del directorio principal del proyecto:
```cat << 'EOF' > deploy_pipeline.sh
#!/usr/bin/env bash

## Detener el script si ocurre algún error no controlado
set -euo pipefail

## Configuración de variables del entorno de red y contenedores
DB_USER="postgres"
DB_PASS="PostgresAdminPass123!"
DB_NAME="enterprise_db"
LIQUIBASE_IMAGE="liquibase/liquibase:4.26.0"
BACKUP_DIR="/tmp/pg_backups"

## Crear directorio de respaldos local
mkdir -p "${BACKUP_DIR}"

## Función auxiliar para imprimir mensajes estructurados
log_info() {
    echo -e "\n\033[1;32m[INFO] [$(date '+%Y-%m-%d %H:%M:%S')] $1\033[0m"
}

log_warn() {
    echo -e "\n\033[1;33m[WARN] [$(date '+%Y-%m-%d %H:%M:%S')] $1\033[0m"
}

log_error() {
    echo -e "\n\033[1;31m[ERROR] [$(date '+%Y-%m-%d %H:%M:%S')] $1\033[0m"
}

## Verificar disponibilidad de Docker
if ! command -v docker &> /dev/null; then
    log_error "Docker no está instalado o no se encuentra en el PATH actual."
    exit 1
fi

## ==========================================
## FASE 1: DESPLIEGUE EN DESARROLLO (DEV)
## ==========================================
log_info "Iniciando Fase de Despliegue en DESARROLLO (DEV)..."

docker run --rm \
  --network pg_enterprise_net \
  -v "$(pwd)/changelog:/liquibase/changelog" \
  -v "$(pwd)/liquibase.properties:/liquibase/liquibase.properties" \
  ${LIQUIBASE_IMAGE} \
  --url="jdbc:postgresql://pg-dev:5432/${DB_NAME}" \
  --username="${DB_USER}" \
  --password="${DB_PASS}" \
  --contexts="dev" \
  update

log_info "Despliegue en DEV finalizado con éxito."

## ==========================================
## GATE 1: APROBACIÓN MANUAL PARA QA
## ==========================================
log_warn "=== CONTROL DE PUERTA (GATE 1) ==="
read -p "¿Desea promover los cambios al ambiente de CONTROL DE CALIDAD (QA)? (S/N): " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Ss]$ ]]; then
    log_warn "Despliegue cancelado por el operador. Saliendo..."
    exit 0
fi

## ==========================================
## FASE 2: DESPLIEGUE EN QA
## ==========================================
log_info "Iniciando Fase de Despliegue en QA..."

docker run --rm \
  --network pg_enterprise_net \
  -v "$(pwd)/changelog:/liquibase/changelog" \
  -v "$(pwd)/liquibase.properties:/liquibase/liquibase.properties" \
  ${LIQUIBASE_IMAGE} \
  --url="jdbc:postgresql://pg-qa:5432/${DB_NAME}" \
  --username="${DB_USER}" \
  --password="${DB_PASS}" \
  --contexts="qa" \
  update

log_info "Despliegue en QA finalizado con éxito."

## ==========================================
## GATE 2: APROBACIÓN MANUAL PARA PRODUCCIÓN
## ==========================================
log_warn "=== CONTROL DE PUERTA CRÍTICO (GATE 2) ==="
read -p "¡ALERTA! ¿Desea proceder con el despliegue al entorno de PRODUCCIÓN (PROD)? (S/N): " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Ss]$ ]]; then
    log_warn "Despliegue a Producción cancelado por el operador."
    exit 0
fi

## ==========================================
## FASE 3: RESPALDO PREVENTIVO EN CALIENTE DE PROD
## ==========================================
log_info "Generando respaldo preventivo en caliente del ambiente de PRODUCCIÓN..."
## Ejecutamos pg_dump directo desde el contenedor de producción para garantizar consistencia binaria
docker exec -e PGPASSWORD="${DB_PASS}" pg-prod \
  pg_dump -U "${DB_USER}" -d "${DB_NAME}" -F c -b -v -f "/tmp/prod_pre_deploy_dump.bak"

## Copiar el backup generado fuera del contenedor al host por seguridad
docker cp pg-prod:/tmp/prod_pre_deploy_dump.bak "${BACKUP_DIR}/prod_pre_deploy_dump.bak"
log_info "Respaldo preventivo consolidado físicamente en: ${BACKUP_DIR}/prod_pre_deploy_dump.bak"

## ==========================================
## FASE 4: DESPLIEGUE EN PRODUCCIÓN CON AUTO-ROLLBACK
## ==========================================
log_info "Aplicando cambios estructurales en PRODUCCIÓN..."

## Capturamos fallos en la ejecución de Liquibase para disparar mitigación inmediata
if ! docker run --rm \
  --network pg_enterprise_net \
  -v "$(pwd)/changelog:/liquibase/changelog" \
  -v "$(pwd)/liquibase.properties:/liquibase/liquibase.properties" \
  ${LIQUIBASE_IMAGE} \
  --url="jdbc:postgresql://pg-prod:5432/${DB_NAME}" \
  --username="${DB_USER}" \
  --password="${DB_PASS}" \
  --contexts="prod" \
  update; then
    
    log_error "Fallo detectado durante la migración en PRODUCCIÓN. Iniciando protocolo de AUTO-ROLLBACK..."
    
    log_warn "Paso 1: Terminando conexiones activas a la base de datos de producción para evitar bloqueos..."
    docker exec -e PGPASSWORD="${DB_PASS}" pg-prod \
      psql -U "${DB_USER}" -d "postgres" -c \
      "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = '${DB_NAME}' AND pid <> pg_backend_pid();"
    
    log_warn "Paso 2: Re-creando la base de datos vacía..."
    docker exec -e PGPASSWORD="${DB_PASS}" pg-prod \
      psql -U "${DB_USER}" -d "postgres" -c "DROP DATABASE ${DB_NAME};"
    docker exec -e PGPASSWORD="${DB_PASS}" pg-prod \
      psql -U "${DB_USER}" -d "postgres" -c "CREATE DATABASE ${DB_NAME};"
      
    log_warn "Paso 3: Restaurando el respaldo preventivo en caliente..."
    if docker exec -e PGPASSWORD="${DB_PASS}" pg-prod \
      pg_restore -U "${DB_USER}" -d "${DB_NAME}" -v "/tmp/prod_pre_deploy_dump.bak"; then
        log_info "RESTABLECIMIENTO EXITOSO. La base de datos de PRODUCCIÓN ha vuelto a su estado original previo al despliegue."
    else
        log_error "FALLO CRÍTICO: La restauración automática de producción falló. Intervención manual inmediata requerida."
    fi
    exit 2
fi

log_info "¡Felicidades! Despliegue completado con éxito en todos los ambientes (DEV -> QA -> PROD)."
EOF
```

2. Asigna permisos de ejecución al script:
```bash
chmod +x deploy_pipeline.sh
```

**Expected output:**
Archivo de script configurado listo para ejecutarse sin errores de sintaxis en Bash.

**Verification:**
Valida la sintaxis del script de Bash utilizando la opción `-n` (sin ejecutarlo):
```bash
bash -n deploy_pipeline.sh && echo "Sintaxis de script de Bash: CORRECTA"
```

---

### Paso 4: Inyectar un Fallo Controlado en el Despliegue de Producción (Simulación de Incidente)

**Objetivo:** Simular un escenario del mundo real donde un changeset con sintaxis errónea o restricciones lógicas inválidas se introduce en el flujo de integración y llega al paso de producción. Validar que nuestro script intercepta el error y ejecuta un rollback total y transparente utilizando el respaldo preventivo.

**Instrucciones:**

1. Abre el archivo maestro de cambios `changelog/db.changelog-master.xml` e inyecta un nuevo changeset erróneo al final del archivo (antes de la etiqueta de cierre `</databaseChangeLog>`). Este changeset intentará ejecutar una consulta SQL inválida a propósito (por ejemplo, insertar texto en una columna de entero inexistente o con sintaxis rota), y estará etiquetado exclusivamente para ejecutarse en el contexto de `prod`.

```cat << 'EOF' > changelog/temp_patch.xml
    <!-- Changeset 4: Cambio defectuoso inyectado (Exclusivo de PROD para simular desastre) -->
    <changeSet id="1.3.0-cambio-roto-prod" author="developer_junior" context="prod">
        <sql>
            -- Comando SQL con error de sintaxis intencional (columna inexistente y tabla mal escrita)
            INSERT INTO clientessss (id_inexistente, nombre_incorrecto) VALUES ('ERROR_TIPO', 99999);
        </sql>
    </changeSet>
EOF
```

Para insertar este bloque de forma segura antes de la última línea `</databaseChangeLog>`, ejecuta el siguiente comando auxiliar de procesamiento:

```bash
sed -i '$d' changelog/db.changelog-master.xml
cat changelog/temp_patch.xml >> changelog/db.changelog-master.xml
echo "</databaseChangeLog>" >> changelog/db.changelog-master.xml
rm changelog/temp_patch.xml
```

2. Ejecuta el pipeline para verificar cómo interactúa a través de los diferentes entornos:
```bash
./deploy_pipeline.sh
```

3. **Respuestas de Entrada del Pipeline (Flujo Esperado):**
   * Cuando el script pregunte si deseas promover al ambiente de QA, presiona **`S`** y presiona **Enter**.
   * Cuando pregunte si deseas promover al ambiente de PROD, presiona **`S`** y presiona **Enter**.

**Expected output:**
El pipeline debe completar con éxito DEV y QA (puesto que el cambio defectuoso tiene el contexto exclusivo `prod` y por ende es ignorado en las fases iniciales). Al llegar a la fase de producción:
1. Generará el respaldo preventivo.
2. Iniciará Liquibase contra `pg-prod`.
3. Fallará en el changeset `1.3.0-cambio-roto-prod`.
4. Mostrará un mensaje de excepción roja en la salida de Liquibase indicando la inexistencia de la tabla `clientessss`.
5. Ejecutará de inmediato el protocolo de Rollback automático, destruyendo e inicializando la base de datos de producción con el respaldo tomado segundos atrás.

Salida detallada de la restauración automática de producción en consola:
```text
[INFO] [xxxx-xx-xx xx:xx:xx] Iniciando Fase de Despliegue en DESARROLLO (DEV)...
...
[INFO] [xxxx-xx-xx xx:xx:xx] Despliegue en DEV finalizado con éxito.
[WARN] [xxxx-xx-xx xx:xx:xx] === CONTROL DE PUERTA (GATE 1) ===
¿Desea promover los cambios al ambiente de CONTROL DE CALIDAD (QA)? (S/N): S
...
[INFO] [xxxx-xx-xx xx:xx:xx] Despliegue en QA finalizado con éxito.
[WARN] [xxxx-xx-xx xx:xx:xx] === CONTROL DE PUERTA CRÍTICO (GATE 2) ===
¡ALERTA! ¿Desea proceder con el despliegue al entorno de PRODUCCIÓN (PROD)? (S/N): S
...
[INFO] [xxxx-xx-xx xx:xx:xx] Generando respaldo preventivo en caliente del ambiente de PRODUCCIÓN...
...
[ERROR] Fallo detectado durante la migración en PRODUCCIÓN. Iniciando protocolo de AUTO-ROLLBACK...
[WARN] Paso 1: Terminando conexiones activas a la base de datos de producción para evitar bloqueos...
...
[WARN] Paso 2: Re-creando la base de datos vacía...
...
[WARN] Paso 3: Restaurando el respaldo preventivo en caliente...
...
[INFO] RESTABLECIMIENTO EXITOSO. La base de datos de PRODUCCIÓN ha vuelto a su estado original previo al despliegue.
```

**Verification:**
Comprueba que la base de datos de Producción (`pg-prod`) se encuentre completamente limpia, libre de la tabla defectuosa y sin los metadatos corruptos de Liquibase que impedirían despliegues futuros:
```bash
docker exec -it pg-prod psql -U postgres -d enterprise_db -c "\dt"
```
*(Nota: El resultado debe ser "No relations found." o el listado de tablas previas vacías, confirmando que la base de datos se restauró por completo al estado previo a la ejecución del pipeline fallido).*

---

### Paso 5: Corrección del Error y Re-ejecución Exitosa del Pipeline

**Objetivo:** Modificar y corregir el script de cambios XML para incorporar comandos DDL/DML correctos compatibles con los estándares de producción de PostgreSQL. Ejecutar exitosamente el flujo de extremo a extremo sin que ocurran retrocesos catastróficos.

**Instrucciones:**

1. Abre el archivo changelog maestro y corrige el changeset problemático de producción reemplazando el código SQL inválido por un comando válido (por ejemplo, insertar un registro real en la tabla `clientes` que ya fue creada con éxito en el primer paso):

```cat << 'EOF' > changelog/temp_patch.xml
    <!-- Changeset 4: Cambio corregido para despliegue exitoso en PROD -->
    <changeSet id="1.3.0-cambio-roto-prod" author="developer_junior" context="prod">
        <insert tableName="clientes">
            <column name="id" valueNumeric="100"/>
            <column name="nombre" value="Cliente VIP Produccion"/>
            <column name="email" value="vip-prod@enterprise.com"/>
        </insert>
    </changeSet>
EOF
```

Aplica la sustitución en el archivo XML mediante la eliminación de las líneas defectuosas anteriores o reescribiendo la sección correspondiente. Para asegurar la consistencia, regeneraremos el changelog maestro limpio con el changeset de producción correcto:

```cat << 'EOF' > changelog/db.changelog-master.xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.26.xsd">

    <!-- Changeset 1: Estructura Base (Común a todos los ambientes) -->
    <changeSet id="1.0.0-crear-clientes" author="arquitecto_db">
        <createTable tableName="clientes">
            <column name="id" type="BIGINT">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="nombre" type="VARCHAR(100)">
                <constraints nullable="false"/>
            </column>
            <column name="email" type="VARCHAR(150)">
                <constraints nullable="false"/>
            </column>
            <column name="creado_en" type="TIMESTAMP" defaultValueComputed="CURRENT_TIMESTAMP"/>
        </createTable>
    </changeSet>

    <!-- Changeset 2: Datos dummy de prueba (Exclusivo de DEV) -->
    <changeSet id="1.1.0-datos-dummy-dev" author="developer" context="dev">
        <insert tableName="clientes">
            <column name="id" valueNumeric="1"/>
            <column name="nombre" value="Cliente de Pruebas DEV"/>
            <column name="email" value="test-dev@enterprise.local"/>
        </insert>
        <insert tableName="clientes">
            <column name="id" valueNumeric="2"/>
            <column name="nombre" value="QA Tester Shadow"/>
            <column name="email" value="shadow-dev@enterprise.local"/>
        </insert>
    </changeSet>

    <!-- Changeset 3: Índice y Restricción de Calidad/Producción (Exclusivo de QA y PROD) -->
    <changeSet id="1.2.0-optimizacion-prod-qa" author="dba_senior" context="qa or prod">
        <createIndex indexName="idx_clientes_email_hash" tableName="clientes">
            <column name="email"/>
        </createIndex>
        <addCheckConstraint constraintName="chk_email_format" 
                            tableName="clientes" 
                            columnNames="email" 
                            constraintText="email LIKE '%@%.%'"/>
    </changeSet>

    <!-- Changeset 4: Cambio corregido para despliegue exitoso en PROD -->
    <changeSet id="1.3.0-cambio-roto-prod" author="developer_junior" context="prod">
        <insert tableName="clientes">
            <column name="id" valueNumeric="100"/>
            <column name="nombre" value="Cliente VIP Produccion"/>
            <column name="email" value="vip-prod@enterprise.com"/>
        </insert>
    </changeSet>

</databaseChangeLog>
EOF
```

2. Ejecuta nuevamente el pipeline de entrega continua:
```bash
./deploy_pipeline.sh
```

Acepta ambas confirmaciones en pantalla con **`S`**.

**Expected output:**
Esta vez, todos los pasos del despliegue deben ejecutarse y reportar estados exitosos. El pipeline terminará con el mensaje:
`[INFO] [xxxx-xx-xx xx:xx:xx] ¡Felicidades! Despliegue completado con éxito en todos los ambientes (DEV -> QA -> PROD).`

**Verification:**
Comprueba que el registro VIP de producción haya sido correctamente consolidado dentro del contenedor de Producción:
```bash
docker exec -it pg-prod psql -U postgres -d enterprise_db -c "SELECT * FROM clientes;"
```
La salida de consola debe retornar el registro idéntico con ID `100`, Nombre `Cliente VIP Produccion` y Email `vip-prod@enterprise.com`.

---

## Validación y Pruebas

Para garantizar la correcta ejecución del pipeline de Database DevOps y evaluar su resiliencia e inmunidad a la variabilidad de estados, procederás a realizar las siguientes auditorías cuantitativas cruzadas:

### 1. Auditoría Estructural de la Base de Datos de Desarrollo (DEV)
Desarrollo únicamente debe contener la tabla base `clientes` y sus registros ficticios asociados. No debe contener bajo ningún concepto el índice optimizado ni la restricción `chk_email_format` definidos exclusivamente para QA/PROD.

Ejecuta el siguiente comando de inspección:
```bash
docker exec -it pg-dev psql -U postgres -d enterprise_db -c "\d clientes"
```

**Resultado Esperado en Consola:**
La definición de la tabla no debe mostrar ningún índice adicional bajo el campo "Indexes" ni condiciones de validación en "Check constraints".
```text
                               Table "public.clientes"
  Column   |            Type             | Collation | Nullable |           Default            
-----------+-----------------------------+-----------+----------+------------------------------
 id        | bigint                      |           | not null | 
 nombre    | character varying(100)      |           | not null | 
 email     | character varying(150)      |           | not null | 
 creado_en | timestamp without time zone |           |          | CURRENT_TIMESTAMP
Indexes:
    "clientes_pkey" PRIMARY KEY, btree (id)
```

### 2. Auditoría Estructural de la Base de Datos de Producción (PROD)
Producción debe contener la estructura completa de la tabla, la restricción de formato de email, el índice hash y únicamente los datos VIP inyectados en el changeset corregido. No debe contener datos de pruebas del ambiente de DEV.

Ejecuta el siguiente comando de inspección:
```bash
docker exec -it pg-prod psql -U postgres -d enterprise_db -c "\d clientes"
```

**Resultado Esperado en Consola:**
```text
                               Table "public.clientes"
  Column   |            Type             | Collation | Nullable |           Default            
-----------+-----------------------------+-----------+----------+------------------------------
 id        | bigint                      |           | not null | 
 nombre    | character varying(100)      |           | not null | 
 email     | character varying(150)      |           | not null | 
 creado_en | timestamp without time zone |           |          | CURRENT_TIMESTAMP
Indexes:
    "clientes_pkey" PRIMARY KEY, btree (id)
    "idx_clientes_email_hash" btree (email)
Check constraints:
    "chk_email_format" CHECK (email::text LIKE '%@%.%'::text)
```

Ejecuta una consulta para comprobar que los datos dummy de `DEV` (IDs 1 y 2) no se filtraron en `PROD`:
```bash
docker exec -it pg-prod psql -U postgres -d enterprise_db -c "SELECT id, nombre, email FROM clientes;"
```
**Resultado Esperado en Consola:**
```text
 id  |         nombre          |          email          
-----+-------------------------+-------------------------
 100 | Cliente VIP Produccion  | vip-prod@enterprise.com
(1 row)
```

### 3. Prueba Adversaria (Inyección de Datos No Válidos - Casos de Prueba Negativos)
Para validar que las restricciones de QA y PROD se encuentran activas y bloquean transacciones corruptas, intentaremos insertar manualmente un cliente con un formato de email inválido directamente en Producción y en Desarrollo.

Intentar inserción inválida en `PROD`:
```bash
docker exec -it pg-prod psql -U postgres -d enterprise_db \
  -c "INSERT INTO clientes (id, nombre, email) VALUES (999, 'Usuario Intruso', 'email_sin_formato_correcto');" 2>&1 \
  | grep -q "chk_email_format" && echo "PROD: INYECCIÓN BLOQUEADA (ÉXITO)" || echo "PROD: ERROR DE VALIDACIÓN"
```

Intentar inserción idéntica en `DEV`:
```bash
docker exec -it pg-dev psql -U postgres -d enterprise_db \
  -c "INSERT INTO clientes (id, nombre, email) VALUES (999, 'Usuario Intruso', 'email_sin_formato_correcto');" 2>&1 \
  | grep -q "INSERT 0 1" && echo "DEV: INYECCIÓN PERMITIDA (ÉXITO)" || echo "DEV: ERROR"
```

---

## Solución de Problemas

A continuación, se documentan dos de las incidencias más recurrentes que pueden surgir durante la implementación práctica de esta simulación, junto con sus causas técnicas raíz y metodologías de corrección inmediata.

### 1. Error de Bloqueo de Base de Datos en el Rollback (`Database is being accessed by other users`)
* **Síntomas:** El pipeline falla al intentar reconstruir la base de datos de producción durante la fase de Rollback, mostrando un error de tipo: `ERROR: database "enterprise_db" is being accessed by other users`.
* **Causa Raíz:** Clientes externos (tales como sesiones abiertas de DBeaver, consultas activas en terminales secundarias o conexiones residuales de ejecuciones previas de Liquibase) retienen bloqueos activos sobre los descriptores de archivos de la base de datos, impidiendo la ejecución exitosa de la sentencia `DROP DATABASE`.
* **Resolución:** Asegúrate de que el script invoque correctamente la desconexión agresiva de terminales en el catálogo `pg_stat_activity` utilizando la consulta provista en la fase de rollback. Si se persiste el bloqueo, ejecuta manualmente el siguiente comando antes de lanzar el pipeline de nuevo:
  ```bash
  docker exec -it pg-prod psql -U postgres -d postgres -c "
    SELECT pg_terminate_backend(pg_stat_activity.pid)
    FROM pg_stat_activity
    WHERE pg_stat_activity.datname = 'enterprise_db'
      AND pid <> pg_backend_pid();"
  ```

### 2. Error en la Validación de Checksum de Cambios (`Validation Failed: 1 change sets check sum`)
* **Síntomas:** Liquibase detiene su ejecución con un error de verificación que indica que los checksums previamente grabados en la tabla del sistema `databasechangelog` difieren del código XML actual.
* **Causa Raíz:** Se modificó la estructura de un changeset que ya había sido procesado y ejecutado con éxito en ejecuciones anteriores. Liquibase detecta esta alteración del historial para prevenir cambios accidentales o derivas silenciosas en la estructura.
* **Resolución:** Si el cambio alterado es correcto y requieres recalcular de forma segura las firmas de verificación física de la base de datos, ejecuta un comando de limpieza de firmas de Liquibase:
  ```bash
  docker run --rm \
    --network pg_enterprise_net \
    -v "$(pwd)/changelog:/liquibase/changelog" \
    -v "$(pwd)/liquibase.properties:/liquibase/liquibase.properties" \
    liquibase/liquibase:4.26.0 \
    --url="jdbc:postgresql://pg-prod:5432/enterprise_db" \
    --username="postgres" \
    --password="PostgresAdminPass123!" \
    clear-checksums
  ```

---

## Limpieza

Para restaurar tu estación de trabajo a su estado inicial, destruye de forma segura los recursos del laboratorio temporales ejecutando las siguientes instrucciones:

```bash
## Detener y remover los tres contenedores PostgreSQL dedicados
docker rm -f pg-dev pg-qa pg-prod

## Eliminar la red compartida creada
docker network rm pg_enterprise_net 2>/dev/null || true

## Remover los directorios locales de almacenamiento de migración y respaldos temporales
rm -rf ~/liquibase-deploy
rm -rf /tmp/pg_backups

echo "Entorno del laboratorio completamente purgado."
```

---

## Resumen

En esta práctica avanzada de simulación de despliegues controlados, has modelado con éxito una infraestructura completa de ciclo de entrega bajo metodologías de bases de datos como código (Database-as-Code):

1. **Aislamiento Multi-Ambiente**: Implementaste tres instancias físicas y lógicas mediante Docker (`pg-dev`, `pg-qa`, `pg-prod`), mapeando puertos específicos para emular entornos distribuidos.
2. **Contextualización de Cambios**: Empleaste configuraciones de contextos nativos de Liquibase 4.26.0 para discernir la aplicación selectiva de datos de prueba (`dev`) frente a optimizaciones críticas de desempeño (`qa` y `prod`).
3. **Resiliencia de Despliegues**: Diseñaste un pipeline en Bash que incorporó el aprovisionamiento de copias de seguridad de consistencia transaccional binaria previa (`pg_dump`) y automatizó el restablecimiento íntegro (*rollback*) ante incidentes controlados.
4. **Verificación Estricta**: Evaluaste mediante validaciones cruzadas y pruebas adversarias que las reglas de calidad y formatos estuvieran operando de acuerdo a lo planificado, mitigando la deriva de esquemas.
