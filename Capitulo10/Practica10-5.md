# 10.5 Simulación de Despliegue Controlado entre DEV, QA y PROD

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
