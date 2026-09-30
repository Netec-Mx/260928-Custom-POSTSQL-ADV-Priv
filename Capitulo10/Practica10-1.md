# 10.1 Versionamiento de Objetos PostgreSQL con Git

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