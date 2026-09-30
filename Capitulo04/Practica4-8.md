# 4.8 Diseño de una Arquitectura HA Empresarial

## Metadatos

| Métrica | Detalle |
| :--- | :--- |
| **Duración** | 50 minutos |
| **Complejidad** | Alta (Hard) |
| **Nivel de Bloom** | Crear |

---

## Descripción General

En esta práctica de laboratorio, diseñarás e implementarás una topología completa de base de datos de Alta Disponibilidad (HA - *High Availability*) diseñada para entornos corporativos con tolerancia a fallos. El flujo de trabajo involucra la creación de un entorno multi-contenedor utilizando Docker Compose, compuesto por una base de datos PostgreSQL primaria y una réplica física asíncrona mediante *Streaming Replication*.

Para optimizar el rendimiento y evitar la saturación de conexiones físicas, integrarás instancias intermedias de PgBouncer configuradas en modo transacción. En la capa frontal, implementarás un balanceador HAProxy que actuará como el único punto de entrada de la aplicación, discriminando de forma inteligente el tráfico de escritura (puerto `5000`) y de solo lectura (puerto `5001`) mediante la monitorización en caliente del estado de replicación de los nodos backend.

[VISUAL: 03-01-0002 - Diagrama de arquitectura de alta disponibilidad del laboratorio. Los clientes envían peticiones a HAProxy (puertos 5000/5001). HAProxy monitoriza y enruta el tráfico hacia las instancias de PgBouncer, las cuales gestionan los pools de conexiones hacia los nodos de PostgreSQL Primario y Réplica respectivamente.]

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Diseñar y desplegar una arquitectura de red aislada multi-contenedor utilizando Docker Compose para simular entornos de alta disponibilidad.
- [ ] Configurar una replicación física por streaming asíncrona robusta y automatizar la provisión del nodo réplica mediante `pg_basebackup`.
- [ ] Implementar pools de conexiones eficientes con PgBouncer en modo de transacción (`transaction`) utilizando autenticación cifrada SCRAM-SHA-256.
- [ ] Configurar HAProxy para realizar el enrutamiento inteligente (R/W Splitting) mediante scripts de comprobación externa (*external checks*) basados en el rol dinámico de la base de datos.
- [ ] Ejecutar y validar un proceso de conmutación por error (*failover*) manual, asegurando la continuidad operativa del clúster sin pérdida de acceso al servicio.

---

## Prerrequisitos

Para completar este laboratorio con éxito, se requiere:
1. **Conocimientos Teóricos**:
   - Replicación física asíncrona, estados de recuperación (*recovery mode*) y el funcionamiento de la herramienta `pg_basebackup`.
   - Modos de pooling de conexiones de PgBouncer (específicamente la diferencia entre el modo sesión y transacción).
   - Configuración básica de balanceadores de carga TCP de Capa 4 (HAProxy).
2. **Acceso y Herramientas**:
   - Una terminal de comandos con acceso de administrador (`sudo`).
   - Conexión estable a Internet para la descarga de imágenes oficiales de contenedores.
   - Herramientas auxiliares de inteligencia artificial: En caso de utilizar asistentes como Microsoft 365 Copilot Chat (bajo licencias empresariales que garanticen la privacidad de datos corporativos), recuerda que actúan únicamente como soporte para plantillas sintácticas. La validación arquitectónica, control de seguridad física y mitigación de fallos de red recaen exclusivamente bajo tu supervisión humana directa.

---

## Entorno de Laboratorio

Este laboratorio está basado en la suite de software oficial de nivel empresarial detallada a continuación:

### Especificaciones de Software

| Software / Componente | Edición / Arquitectura | Proveedor / Enlace de Origen | Licencia |
| :--- | :--- | :--- | :--- |
| **PostgreSQL** | Community Edition v16.2 x86_64 | [PostgreSQL Downloads](https://www.postgresql.org/download/) | PostgreSQL License |
| **HAProxy** | Community Edition v2.9.5 | [HAProxy Official](https://www.haproxy.org/) | GPLv2 / LGPLv2.1 |
| **PgBouncer** | Connection Pooler v1.22.0 | [PgBouncer Downloads](https://www.pgbouncer.org/) | BSD License |
| **Docker Compose** | Engine v26.0.0 (Compose v2.26.0) | [Docker Engine Docs](https://docs.docker.com/engine/) | Apache License 2.0 |
| **Debian GNU/Linux** | Bookworm v12.5 | [Debian CD Images](https://www.debian.org/) | GPL/Free Software |

### Especificaciones de Hardware Recomendadas

*   **Procesador**: CPU x86_64 de 8 núcleos físicos (Intel Core i7/i9 o AMD Ryzen 7/9).
*   **Memoria RAM**: Mínimo 16 GB (Recomendado 32 GB para prevenir latencias en la virtualización).
*   **Almacenamiento**: 100 GB de espacio libre en unidad de estado sólido (SSD NVMe recomendado).
*   **Red**: Interfaz Bridge Docker local sobre la subred del contenedor `pg_enterprise_net`.

---

## Instrucciones Paso a Paso

### Paso 1: Creación de la Estructura de Directorios del Proyecto

**Objetivo**: Establecer la estructura física de directorios y los archivos de configuración iniciales para garantizar el aislamiento de cada servicio y la persistencia de datos.

**Instrucciones**:

1. Abre tu terminal y crea un directorio raíz para el proyecto de alta disponibilidad:
   ```bash
   mkdir -p ~/ha_enterprise_lab/config/{postgres,pgbouncer,haproxy}
   mkdir -p ~/ha_enterprise_lab/scripts
   cd ~/ha_enterprise_lab
   ```

2. Verifica que la jerarquía de carpetas se haya generado correctamente utilizando el comando `find`:
   ```bash
   find . -type d
   ```

**Resultado esperado**:
```text
.
./scripts
./config
./config/postgres
./config/pgbouncer
./config/haproxy
```

---

### Paso 2: Configuración de los Scripts de Inicialización de PostgreSQL (Nodo Primario)

**Objetivo**: Crear los scripts de inicio automatizados para el nodo primario que configuran los permisos del archivo `pg_hba.conf` para la replicación y crean el usuario replicador con privilegios SCRAM-SHA-256.

**Instrucciones**:

1. Crea un script de configuración de autenticación de red para el nodo primario en `config/postgres/00-config-hba.sh`:
   ```bash
   cat << 'EOF' > config/postgres/00-config-hba.sh
   #!/bin/bash
   echo "Modificando pg_hba.conf para habilitar replicación segura..."
   echo "host replication replicator 0.0.0.0/0 scram-sha-256" >> "$PGDATA/pg_hba.conf"
   echo "host all all 0.0.0.0/0 scram-sha-256" >> "$PGDATA/pg_hba.conf"
   EOF
   chmod +x config/postgres/00-config-hba.sh
   ```

2. Crea el script SQL para inicializar los roles de réplica y el esquema inicial en `config/postgres/01-setup-replication.sql`:
   ```bash
   cat << 'EOF' > config/postgres/01-setup-replication.sql
   -- Creación del rol de replicación física asíncrona
   CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'PostgresAdminPass123!';

   -- Base de datos de prueba global del laboratorio
   CREATE DATABASE enterprise_db;
   \c enterprise_db;

   -- Tabla para pruebas de carga y failover
   CREATE TABLE cluster_heartbeat (
       id SERIAL PRIMARY KEY,
       inserted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
       inserted_by VARCHAR(100)
   );

   INSERT INTO cluster_heartbeat (inserted_by) VALUES ('initial_provision');
   EOF
   ```

**Verificación**: Asegúrate de que los archivos se crearon con sus respectivos permisos de ejecución donde corresponda.

---

### Paso 3: Configuración del Script de Inicialización de la Réplica (pg_replica)

**Objetivo**: Diseñar un script de inicio dinámico para el contenedor de la réplica que realice un `pg_basebackup` contra el nodo primario antes de levantar el motor PostgreSQL. Esto previene condiciones de carrera (*race conditions*) en el despliegue automático de Docker Compose.

**Instrucciones**:

1. Crea el script de arranque para la réplica en `scripts/init-replica.sh`:
   ```bash
   cat << 'EOF' > scripts/init-replica.sh
   #!/bin/bash
   set -e

   # Limpiar el directorio de datos de la réplica si no está inicializado para evitar colisiones
   if [ ! -s "$PGDATA/PG_VERSION" ]; then
       echo "Directorio de datos vacío. Iniciando pg_basebackup desde el nodo primario..."
       
       # Esperar a que el nodo primario esté respondiendo conexiones en su puerto nativo
       until pg_isready -h pg-primary -p 5432 -U postgres; do
           echo "Esperando a que pg-primary esté en línea..."
           sleep 2
       done

       # Ejecutar copia física del sistema de archivos de la base de datos
       PGPASSWORD="PostgresAdminPass123!" pg_basebackup \
           -h pg-primary \
           -D "$PGDATA" \
           -U replicator \
           -v -P -R -X stream

       echo "Sincronización física completada exitosamente."
       chmod 700 "$PGDATA"
   else
       echo "El directorio de datos ya contiene información. Omitiendo pg_basebackup."
   fi

   # Cargar el punto de entrada oficial de la imagen de PostgreSQL
   exec docker-entrypoint.sh postgres
   EOF
   chmod +x scripts/init-replica.sh
   ```

---

### Paso 4: Configuración de las Instancias de PgBouncer

**Objetivo**: Configurar PgBouncer para optimizar el pooling en modo transacción y autenticar de manera segura a los clientes usando hashes SCRAM-SHA-256.

**Instrucciones**:

1. Crea el archivo de contraseñas de PgBouncer en `config/pgbouncer/userlist.txt`. Este archivo permite autenticar de manera local las solicitudes entrantes sin sobrecargar al motor PostgreSQL con negociaciones iniciales de handshake:
   ```bash
   cat << 'EOF' > config/pgbouncer/userlist.txt
   "postgres" "PostgresAdminPass123!"
   "replicator" "PostgresAdminPass123!"
   EOF
   ```

2. Modifica los permisos de `userlist.txt` para emular políticas de mínimo privilegio requeridas por normativas como ISO 27001/NIST (solo lectura para el dueño):
   ```bash
   chmod 600 config/pgbouncer/userlist.txt
   ```

3. Crea el archivo de configuración global `config/pgbouncer/pgbouncer.ini`. Configura un pool óptimo en modo `transaction`:
   ```bash
   cat << 'EOF' > config/pgbouncer/pgbouncer.ini
   [databases]
   * = host=127.0.0.1 port=5432 auth_user=postgres

   [pgbouncer]
   logfile = /var/log/postgresql/pgbouncer.log
   pidfile = /var/run/postgresql/pgbouncer.pid
   listen_addr = *
   listen_port = 6432
   auth_type = scram-sha-256
   auth_file = /etc/pgbouncer/userlist.txt
   admin_users = postgres

   # Gestión Avanzada del Pool de Conexiones
   pool_mode = transaction
   max_client_conn = 10000
   default_pool_size = 50
   min_pool_size = 10
   reserve_pool_size = 5
   reserve_pool_timeout = 5

   # Timeouts de seguridad
   query_timeout = 0
   query_wait_timeout = 120
   client_idle_timeout = 0
   idle_transaction_timeout = 60
   EOF
   ```

---

### Paso 5: Configuración de HAProxy para Enrutamiento R/W (Read/Write Splitting)

**Objetivo**: Configurar HAProxy para dirigir el tráfico de escritura al puerto `5000` (dirigiéndose únicamente al nodo primario activo) y el tráfico de lectura al puerto `5001` (balanceado entre primario y réplica) utilizando scripts de comprobación física en bash.

**Instrucciones**:

1. Crea el archivo de configuración principal de HAProxy en `config/haproxy/haproxy.cfg`:
   ```bash
   cat << 'EOF' > config/haproxy/haproxy.cfg
   global
       log stdout format raw local0
       # Habilitar el uso de scripts de comprobación de salud externos
       external-check
       insecure-fork-wanted

   defaults
       log     global
       mode    tcp
       timeout connect 4s
       timeout client  30m
       timeout server  30m

   # FRONTEND DE ESCRITURA (Lectura/Escritura - Puerto 5000)
   frontend fe_write
       bind *:5000
       default_backend be_write

   backend be_write
       mode tcp
       option external-check
       external-check command /usr/local/bin/check_pg_primary.sh
       # Enrutar a través de los pools de PgBouncer correspondientes
       server pgbouncer-primary pgbouncer-primary:6432 maxconn 10000 check inter 3s fall 2 rise 2
       server pgbouncer-replica pgbouncer-replica:6432 maxconn 10000 check inter 3s fall 2 rise 2

   # FRONTEND DE SOLO LECTURA (Balanceado - Puerto 5001)
   frontend fe_read
       bind *:5001
       default_backend be_read

   backend be_read
       mode tcp
       balance round-robin
       option external-check
       external-check command /usr/local/bin/check_pg_read.sh
       server pgbouncer-primary pgbouncer-primary:6432 maxconn 10000 check inter 3s fall 2 rise 2
       server pgbouncer-replica pgbouncer-replica:6432 maxconn 10000 check inter 3s fall 2 rise 2
   EOF
   ```

2. Crea el script de verificación para el nodo de escritura en `config/haproxy/check_pg_primary.sh`. Este script evalúa si el nodo está en modo de recuperación (Read-Only):
   ```bash
   cat << 'EOF' > config/haproxy/check_pg_primary.sh
   #!/bin/bash
   # HAProxy pasa automáticamente argumentos: $1=IP_Origen, $2=Port_Origen, $3=IP_Destino, $4=Port_Destino
   TARGET_IP=$3

   # Determinar si el destino final es la base de datos primaria consultando a través de PgBouncer
   RESULT=$(PGPASSWORD="PostgresAdminPass123!" psql -h "$TARGET_IP" -p 6432 -U postgres -d enterprise_db -t -A -c "SELECT pg_is_in_recovery();" 2>/dev/null)

   if [ "$RESULT" = "f" ]; then
       # No está en recovery (Es el Primario R/W)
       exit 0
   else
       # Está en recovery (Réplica) o caído
       exit 1
   fi
   EOF
   chmod +x config/haproxy/check_pg_primary.sh
   ```

3. Crea el script de verificación para nodos de solo lectura en `config/haproxy/check_pg_read.sh`:
   ```bash
   cat << 'EOF' > config/haproxy/check_pg_read.sh
   #!/bin/bash
   TARGET_IP=$3

   # Cualquier nodo en línea (primario o réplica) es apto para lecturas
   RESULT=$(PGPASSWORD="PostgresAdminPass123!" psql -h "$TARGET_IP" -p 6432 -U postgres -d enterprise_db -t -A -c "SELECT pg_is_in_recovery();" 2>/dev/null)

   if [ "$RESULT" = "f" ] || [ "$RESULT" = "t" ]; then
       exit 0
   else
       exit 1
   fi
   EOF
   chmod +x config/haproxy/check_pg_read.sh
   ```

---

### Paso 6: Orquestación General con Docker Compose

**Objetivo**: Integrar todas las piezas en un archivo descriptivo único de Docker Compose utilizando una red de puente aislada (`pg_enterprise_net`).

**Instrucciones**:

1. Crea el archivo `docker-compose.yml` en el directorio raíz de tu proyecto (`~/ha_enterprise_lab/docker-compose.yml`):
   ```yaml
   version: '3.8'

   networks:
     pg_enterprise_net:
       name: pg_enterprise_net
       driver: bridge

   services:
     pg-primary:
       image: postgres:16.2-bookworm
       container_name: pg-primary
       environment:
         POSTGRES_PASSWORD: PostgresAdminPass123!
         POSTGRES_INITDB_ARGS: "--auth-host=scram-sha-256 --auth-local=scram-sha-256"
       volumes:
         - ./config/postgres/00-config-hba.sh:/docker-entrypoint-initdb.d/00-config-hba.sh
         - ./config/postgres/01-setup-replication.sql:/docker-entrypoint-initdb.d/01-setup-replication.sql
         - pg_primary_data:/var/lib/postgresql/data
       command: >
         postgres 
         -c wal_level=replica 
         -c max_wal_senders=10 
         -c max_replication_slots=10 
         -c hot_standby=on
       networks:
         - pg_enterprise_net
       healthcheck:
         test: ["CMD-SHELL", "pg_isready -U postgres -d postgres"]
         interval: 5s
         timeout: 5s
         retries: 5

     pg-replica:
       image: postgres:16.2-bookworm
       container_name: pg-replica
       environment:
         POSTGRES_PASSWORD: PostgresAdminPass123!
       volumes:
         - ./scripts/init-replica.sh:/usr/local/bin/init-replica.sh
         - pg_replica_data:/var/lib/postgresql/data
       entrypoint: ["/usr/local/bin/init-replica.sh"]
       networks:
         - pg_enterprise_net
       depends_on:
         pg-primary:
           condition: service_healthy

     pgbouncer-primary:
       image: edoburu/pgbouncer:1.22.0
       container_name: pgbouncer-primary
       environment:
         - DB_HOST=pg-primary
         - DB_PORT=5432
         - DB_USER=postgres
         - DB_PASSWORD=PostgresAdminPass123!
       volumes:
         - ./config/pgbouncer/pgbouncer.ini:/etc/pgbouncer/pgbouncer.ini
         - ./config/pgbouncer/userlist.txt:/etc/pgbouncer/userlist.txt
       networks:
         - pg_enterprise_net
       depends_on:
         - pg-primary

     pgbouncer-replica:
       image: edoburu/pgbouncer:1.22.0
       container_name: pgbouncer-replica
       environment:
         - DB_HOST=pg-replica
         - DB_PORT=5432
         - DB_USER=postgres
         - DB_PASSWORD=PostgresAdminPass123!
       volumes:
         - ./config/pgbouncer/pgbouncer.ini:/etc/pgbouncer/pgbouncer.ini
         - ./config/pgbouncer/userlist.txt:/etc/pgbouncer/userlist.txt
       networks:
         - pg_enterprise_net
       depends_on:
         - pg-replica

     haproxy-lb:
       image: haproxy:2.9.5-alpine
       container_name: haproxy-lb
       user: root
       ports:
         - "5000:5000"
         - "5001:5001"
       volumes:
         - ./config/haproxy/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
         - ./config/haproxy/check_pg_primary.sh:/usr/local/bin/check_pg_primary.sh:ro
         - ./config/haproxy/check_pg_read.sh:/usr/local/bin/check_pg_read.sh:ro
       # Instalar cliente de postgresql en HAProxy para habilitar comandos de comprobación física
       entrypoint: >
         /bin/sh -c "
         apk add --no-cache postgresql-client bash &&
         exec haproxy -f /usr/local/etc/haproxy/haproxy.cfg
         "
       networks:
         - pg_enterprise_net
       depends_on:
         - pgbouncer-primary
         - pgbouncer-replica

   volumes:
     pg_primary_data:
     pg_replica_data:
   ```

2. Levanta la infraestructura ejecutando el siguiente comando:
   ```bash
   docker compose up -d
   ```

3. Espera 15 segundos para que los servicios se estabilicen y comprueba que todos los contenedores estén en estado activo:
   ```bash
   docker compose ps
   ```

**Resultado esperado**:
```text
NAME                IMAGE                    COMMAND                  SERVICE             CREATED             STATUS              PORTS
haproxy-lb          haproxy:2.9.5-alpine     "/bin/sh -c 'apk add…"   haproxy-lb          10 seconds ago      Up 9 seconds        0.0.0.0:5000-5001->5000-5001/tcp
pg-primary          postgres:16.2-bookworm   "docker-entrypoint.s…"   pg-primary          10 seconds ago      Up 9 seconds (healthy)  5432/tcp
pg-replica          postgres:16.2-bookworm   "/usr/local/bin/init…"   pg-replica          10 seconds ago      Up 9 seconds        5432/tcp
pgbouncer-primary   edoburu/pgbouncer:1.22.0 "/usr/bin/pgbouncer …"   pgbouncer-primary   10 seconds ago      Up 9 seconds        6432/tcp
pgbouncer-replica   edoburu/pgbouncer:1.22.0 "/usr/bin/pgbouncer …"   pgbouncer-replica   10 seconds ago      Up 9 seconds        6432/tcp
```

---

## Validación y Pruebas

En esta sección, someterás tu clúster HA a una serie de pruebas de esfuerzo y fallos inyectados para comprobar su robustez frente a caídas imprevistas y ataques vectoriales de datos.

### 1. Validación de la Replicación Física Asíncrona

Conéctate al nodo primario y verifica si el remitente de replicación física está activo y enviando segmentos WAL al nodo réplica:

```bash
docker exec -it pg-primary psql -U postgres -d enterprise_db -c "SELECT * FROM pg_stat_replication;"
```

**Resultado esperado**: Debe listar una fila donde el valor de `application_name` sea `walreceiver`, el `state` esté en `streaming` y el `sync_state` sea `async`.

---

### 2. Validación de Lectura y Escritura mediante HAProxy

1. **Prueba de Escritura (Puerto 5000)**: Inserta un nuevo registro apuntando tu cliente psql al balanceador de carga en el puerto `5000`:
   ```bash
   PGPASSWORD="PostgresAdminPass123!" psql -h 127.0.0.1 -p 5000 -U postgres -d enterprise_db -c \
   "INSERT INTO cluster_heartbeat (inserted_by) VALUES ('test_write_via_haproxy');"
   ```

2. **Prueba de Lectura (Puerto 5001)**: Ejecuta una consulta para obtener el registro recién insertado, apuntando al puerto de lectura:
   ```bash
   PGPASSWORD="PostgresAdminPass123!" psql -h 127.0.0.1 -p 5001 -U postgres -d enterprise_db -c \
   "SELECT * FROM cluster_heartbeat;"
   ```

**Resultado esperado**:
```text
 id |         inserted_at        |        inserted_by        
----+----------------------------+---------------------------
  1 | 2026-03-31 10:00:00.000000 | initial_provision
  2 | 2026-03-31 10:05:00.000000 | test_write_via_haproxy
(2 rows)
```

---

### 3. Prueba de Caso Adversario (Inyección de Sentencias SQL y Transmisión Inválida)

Para probar la resiliencia y el comportamiento ante entradas inesperadas, simularemos un comportamiento donde un cliente inyecta sentencias potencialmente destructivas intentando evadir el enrutamiento a través del puerto de solo lectura (Puerto `5001`).

1. Intenta realizar una escritura sobre el puerto de lectura balanceado `5001`:
   ```bash
   PGPASSWORD="PostgresAdminPass123!" psql -h 127.0.0.1 -p 5001 -U postgres -d enterprise_db -c \
   "INSERT INTO cluster_heartbeat (inserted_by) VALUES ('malicious_write_bypass');"
   ```

**Resultado esperado**: Si la conexión de lectura es dirigida al nodo réplica (`pg-replica`), la transacción debe ser cancelada inmediatamente con un mensaje explícito:
```text
ERROR: cannot execute INSERT in a read-only transaction
```
*(Nota: Si ocasionalmente es dirigida al primario, la transacción se completará ya que el puerto 5001 balancea entre ambos. Esto valida que la regla del clúster de solo lectura en aplicaciones debe reforzarse estrictamente a nivel de configuración).*

---

### 4. Simulación de Conmutación por Error (Failover Manual)

Simula un desastre físico en el centro de datos apagando abruptamente el nodo primario. Luego, promueve la réplica en caliente y verifica que HAProxy adapte sus rutas dinámicamente sin intervención humana en su archivo de configuración.

1. Apaga el nodo primario:
   ```bash
   docker compose stop pg-primary
   ```

2. Realiza un intento de escritura en el puerto `5000`:
   ```bash
   PGPASSWORD="PostgresAdminPass123!" psql -h 127.0.0.1 -p 5000 -U postgres -d enterprise_db -c \
   "INSERT INTO cluster_heartbeat (inserted_by) VALUES ('after_fail_write');"
   ```

**Resultado esperado**: El comando fallará con un error de conexión agotada o rechazada (`500 Internal Error` o similar en la capa TCP), debido a que no existe un nodo activo en modo Lectura/Escritura.

3. Promueve el nodo `pg-replica` para que asuma el rol de nuevo nodo primario:
   ```bash
   docker exec -it pg-replica pg_ctl -D /var/lib/postgresql/data promote
   ```

4. Espera 6 segundos para dar tiempo a que el script de monitoreo interno de HAProxy verifique el cambio de rol en el backend y ejecuta nuevamente la prueba de escritura:
   ```bash
   PGPASSWORD="PostgresAdminPass123!" psql -h 127.0.0.1 -p 5000 -U postgres -d enterprise_db -c \
   "INSERT INTO cluster_heartbeat (inserted_by) VALUES ('failover_recovery_success');"
   ```

5. Realiza la comprobación de los registros insertados:
   ```bash
   PGPASSWORD="PostgresAdminPass123!" psql -h 127.0.0.1 -p 5001 -U postgres -d enterprise_db -c \
   "SELECT * FROM cluster_heartbeat;"
   ```

**Resultado esperado**:
```text
 id |         inserted_at        |        inserted_by        
----+----------------------------+---------------------------
  1 | 2026-03-31 10:00:00.000000 | initial_provision
  2 | 2026-03-31 10:05:00.000000 | test_write_via_haproxy
  3 | 2026-03-31 10:10:00.000000 | failover_recovery_success
(3 rows)
```
¡El clúster ha redirigido automáticamente la escritura hacia la réplica promovida de forma transparente!

---

## Solución de Problemas

A continuación, se describen dos escenarios comunes de fallas en la implementación de esta topología junto con su diagnóstico y mitigación de nivel empresarial.

### Escenario 1: HAProxy marca de forma persistente los backends como "DOWN" en las comprobaciones externas

*   **Síntoma**: El clúster se despliega correctamente en Docker Compose, pero intentar realizar cualquier consulta en los puertos `5000` o `5001` resulta en el error: `Connection refused` o `No route to host`. Al revisar los logs de HAProxy, se observan mensajes como:
    ```text
    External check '/usr/local/bin/check_pg_primary.sh' failed with exit code 1
    ```
*   **Causa Raíz**: El contenedor de HAProxy no cuenta con los paquetes ejecutables cliente de PostgreSQL (`psql`) para evaluar las sentencias, o las credenciales especificadas en los scripts `/usr/local/bin/check_pg_*.sh` difieren de las configuradas en `userlist.txt`.
*   **Solución**: 
    1. Asegúrate de que el bloque `entrypoint` de HAProxy en `docker-compose.yml` ejecute de forma efectiva `apk add --no-cache postgresql-client bash` antes de inicializar el binario de HAProxy.
    2. Comprueba la conexión manual desde el contenedor HAProxy:
       ```bash
       docker exec -it haproxy-lb psql -h pgbouncer-primary -p 6432 -U postgres -d enterprise_db -c "SELECT 1;"
       ```
    3. Si falla, valida que el archivo `config/pgbouncer/userlist.txt` contenga exactamente las credenciales mapeadas y cuente con permisos restrictivos `600` para evitar fallos de lectura de PgBouncer.

### Escenario 2: El nodo réplica entra en un bucle de reinicio infinito informando desajuste de línea de tiempo (timeline mismatch)

*   **Síntoma**: Al iniciar la réplica mediante Docker Compose, este falla reportando:
    ```text
    FATAL: recovery aborted because of mismatch in replication timeline
    ```
*   **Causa Raíz**: El volumen del contenedor de la réplica tiene datos residuales de una ejecución anterior de este laboratorio o de una base de datos local pre-existente. Al intentar correr `pg_basebackup`, el script `init-replica.sh` detecta la existencia de archivos y no sincroniza, heredando un estado corrupto del clúster anterior.
*   **Solución**: Es necesario vaciar de raíz los volúmenes de Docker persistentes y realizar un despliegue limpio:
    ```bash
    docker compose down -v
    docker volume rm ha_enterprise_lab_pg_primary_data ha_enterprise_lab_pg_replica_data
    docker compose up -d
    ```

---

## Limpieza

Para restaurar tu entorno de desarrollo local eliminando todos los contenedores y recursos creados para este laboratorio:

1. Detén los contenedores de forma definitiva y elimina los volúmenes compartidos:
   ```bash
   docker compose down -v
   ```

2. Elimina la estructura física de directorios de laboratorio de tu disco duro:
   ```bash
   rm -rf ~/ha_enterprise_lab
   ```

---

## Resumen

En esta sesión práctica de arquitectura e infraestructura avanzada, has logrado:

*   **Consolidar** una arquitectura empresarial de Alta Disponibilidad de punta a punta, enrutando dinámicamente el tráfico sobre redes aisladas de Docker.
*   **Acelerar** el rendimiento global del motor de base de datos implementando pools de conexiones intermedios con **PgBouncer** configurados en modo transacción, lo que reduce la creación de subprocesos costosos y optimiza de manera drástica el uso de memoria RAM del servidor.
*   **Integrar** un proxy inteligente mediante **HAProxy**, configurando scripts de comprobación externa que interactúan dinámicamente con los catálogos del motor (`pg_is_in_recovery()`), permitiendo la separación completa de flujos de trabajo de Lectura y Escritura (*Read/Write Splitting*).
*   **Validar** las capacidades de resiliencia ante desastres físicos promoviendo de manera exitosa nodos secundarios en vivo, preservando la continuidad del negocio bajo políticas de recuperación de desastres alineadas con normativas **ISO 27001** y **NIST**.
