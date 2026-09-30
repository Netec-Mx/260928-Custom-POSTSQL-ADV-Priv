# 10.4 Diseño de una Tubería CI/CD para PostgreSQL

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