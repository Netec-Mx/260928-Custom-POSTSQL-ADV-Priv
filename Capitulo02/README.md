# 6 Administración de Respaldos con Barman

## Metadatos

| Métrica | Detalle |
| :--- | :--- |
| **Duración** | 60 minutos |
| **Dificultad** | Alta (Hard) |
| **Nivel Cognitivo (Bloom)** | Aplicar (Apply) |

---

## Descripción General

En este laboratorio, el estudiante diseñará y desplegará una solución avanzada de recuperación ante desastres y respaldo físico continuo utilizando **Barman (Backup and Recovery Manager) versión 3.10.0** en una arquitectura cliente-servidor dedicada. 

El estudiante levantará un contenedor independiente basado en **Debian GNU/Linux 12.5 (Bookworm)** como servidor de respaldos (`barman-server`). Implementará una arquitectura de alta seguridad configurando el intercambio de llaves SSH (autenticación sin contraseña) entre el servidor de base de datos (`pg-primary`) y el servidor de respaldos. Posteriormente, configurará el streaming de registros de transacciones (WALs) en tiempo real mediante `pg_receivewal` y ranuras de replicación (replication slots). Finalmente, ejecutará un respaldo físico completo, simulará transacciones controladas para modificar el estado del motor y ejecutará un respaldo incremental, verificando la consistencia e integridad de las copias generadas mediante herramientas de diagnóstico de Barman.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Desplegar y aprovisionar un servidor de respaldos independiente utilizando **Barman 3.10.0** sobre **Debian GNU/Linux 12.5 (Bookworm)**.
- [ ] Configurar canales de comunicación seguros mediante intercambio de llaves públicas SSH entre el servidor de base de datos y el servidor de respaldos.
- [ ] Establecer un mecanismo de archivado continuo de WALs utilizando streaming nativo con `pg_receivewal` y ranuras de replicación (replication slots) en **PostgreSQL 16.2**.
- [ ] Ejecutar, programar y diagnosticar respaldos físicos completos e incrementales para garantizar un RPO (Recovery Point Objective) cercano a cero.

---

## Prerrequisitos

Para completar este laboratorio de manera exitosa, es indispensable cumplir con:
1. **Conocimientos teóricos y prácticos previos**:
   - Comprensión del funcionamiento del Write-Ahead Logging (WAL) en PostgreSQL.
   - Administración básica de sistemas operativos Linux (gestión de permisos de archivos, usuarios y SSH).
   - Ejecución básica de comandos de administración de contenedores Docker.
2. **Infraestructura y conectividad**:
   - Contenedor de base de datos `pg-primary` activo y configurado en la red Bridge de Docker denominada `pg_enterprise_net` (creado en prácticas previas con **PostgreSQL 16.2**).
   - Acceso a internet para la descarga e instalación de paquetes Debian oficiales.

---

## Entorno de Laboratorio

El laboratorio utiliza las siguientes versiones oficiales de software e infraestructura:

### Componentes de Software y Versiones Exactas

| Software / Componente | Versión Exacta | Enlace de Referencia Oficial |
| :--- | :--- | :--- |
| **PostgreSQL Community Edition** | 16.2 | [https://www.postgresql.org/docs/16/manual/](https://www.postgresql.org/docs/16/manual/) |
| **Barman (Backup and Recovery)** | 3.10.0 | [https://pgbarman.org/](https://pgbarman.org/) |
| **Debian GNU/Linux** | 12.5 (Bookworm) | [https://www.debian.org/releases/bookworm/](https://www.debian.org/releases/bookworm/) |
| **Docker Engine** | 26.0.0 | [https://docs.docker.com/engine/release-notes/26.0/](https://docs.docker.com/engine/release-notes/26.0/) |

### Parámetros Globales del Entorno

* **Red Docker Bridge**: `pg_enterprise_net`
* **Nombre de Base de Datos**: `enterprise_db`
* **Usuario Maestro PostgreSQL**: `postgres` (Contraseña: `PostgresAdminPass123!`)
* **Contenedor Primario de DB**: `pg-primary` (IP interna asignada dinámicamente en la red)
* **Contenedor del Servidor Barman**: `barman-server` (IP interna asignada dinámicamente en la red)

---

## Instrucciones Paso a Paso

### Paso 1: Creación y Despliegue del Contenedor de Barman

En este paso, se creará el contenedor dedicado `barman-server` basado en la imagen de Debian GNU/Linux 12.5. Se conectará a la red común de Docker y se aprovisionarán las dependencias del sistema y los paquetes de Barman 3.10.0.

**Instrucciones**:

1. Crea el contenedor `barman-server` dentro de la red corporativa `pg_enterprise_net`. Ejecuta el siguiente comando en la terminal del host:
   ```bash
   docker run -d \
     --name barman-server \
     --network pg_enterprise_net \
     -h barman-server \
     -it debian:12.5-slim /bin/bash
   ```

2. Accede a la terminal interactiva del contenedor `barman-server` recién creado como usuario `root`:
   ```bash
   docker exec -it -u root barman-server /bin/bash
   ```

3. Actualiza los repositorios e instala las dependencias de red, el servidor SSH, el cliente de PostgreSQL 16 y el paquete oficial de Barman:
   ```bash
   # Actualizar índices e instalar dependencias básicas
   apt-get update && apt-get install -y \
     curl \
     gnupg \
     lsb-release \
     openssh-server \
     openssh-client \
     procps \
     sudo

   # Agregar el repositorio oficial de PostgreSQL para Debian para asegurar Barman 3.10.0 y postgresql-client-16
   sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
   curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg

   # Actualizar e instalar barman y el cliente postgresql-16
   apt-get update && apt-get install -y \
     barman=3.10.0-1.pgdg120+1 \
     postgresql-client-16
   ```

4. Habilita y arranca el servicio de SSH en el contenedor `barman-server`:
   ```bash
   mkdir -p /var/run/sshd
   echo 'root:RootBarmanSecurePass123!' | chpasswd
   sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
   /usr/sbin/sshd
   ```

**Resultado esperado**:
El contenedor `barman-server` debe estar ejecutándose en segundo plano y los binarios de `barman` y `pg_receivewal` deben estar disponibles en las rutas globales del sistema.

**Verificación**:
Ejecuta el siguiente comando dentro del contenedor para comprobar la versión exacta instalada de Barman:
```bash
barman --version
```
*Salida esperada:*
```text
3.10.0
```

---

### Paso 2: Configuración del Intercambio de Llaves SSH y Acceso Seguro

Para que Barman ejecute respaldos consistentes a nivel de sistema de archivos, necesita comunicarse bidireccionalmente sin contraseñas a través de SSH con el contenedor de base de datos `pg-primary`.

**Instrucciones**:

1. En el contenedor `barman-server`, asigna una shell válida para el usuario del sistema `barman` y establece una contraseña temporal:
   ```bash
   usermod -s /bin/bash barman
   echo 'barman:BarmanSystemPass123!' | chpasswd
   ```

2. Entra en el contenedor `pg-primary` en otra ventana de terminal del host para instalar y configurar el servidor SSH. (Se asume que `pg-primary` es un contenedor Debian/Ubuntu con PostgreSQL 16.2):
   ```bash
   docker exec -it -u root pg-primary /bin/bash
   ```
   Dentro de `pg-primary`, ejecuta:
   ```bash
   apt-get update && apt-get install -y openssh-server openssh-client
   mkdir -p /var/run/sshd
   # Habilitar login del usuario postgres
   usermod -s /bin/bash postgres
   echo 'postgres:PostgresAdminPass123!' | chpasswd
   /usr/sbin/sshd
   ```

3. Regresa a la terminal del contenedor `barman-server`. Cambia a la identidad del usuario del sistema `barman`, genera un par de llaves SSH (RSA de 4096 bits) y envíala al usuario `postgres` en `pg-primary`:
   ```bash
   # Cambiar al usuario barman
   su - barman

   # Generar par de llaves sin frase de contraseña
   ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa

   # Copiar la llave pública a pg-primary (el contenedor resolverá por su nombre de red de Docker)
   ssh-copy-id postgres@pg-primary
   ```
   *(Nota: Escribe `yes` cuando te pregunte si deseas continuar con la conexión e ingresa la contraseña `PostgresAdminPass123!` cuando se te solicite).*

4. Realiza el proceso inverso. Desde el contenedor `pg-primary`, ingresa como usuario `postgres`, genera su llave SSH y cópiala hacia el usuario `barman` en `barman-server`:
   ```bash
   # Cambiar al usuario postgres en pg-primary
   su - postgres

   # Generar llave SSH
   ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/id_rsa

   # Copiar llave pública a barman-server
   ssh-copy-id barman@barman-server
   ```
   *(Nota: Ingresa la contraseña del usuario barman del sistema: `BarmanSystemPass123!`)*.

**Resultado esperado**:
Se habrán creado los archivos `authorized_keys` correspondientes en ambos servidores, permitiendo una comunicación remota confiable bajo los estándares de seguridad ISO 27001 (sin almacenamiento de credenciales explícitas en texto plano).

**Verificación**:
Valida la conexión SSH bidireccional sin contraseña de la siguiente manera:
- Desde `barman-server` (como usuario `barman`):
  ```bash
  ssh postgres@pg-primary "id"
  ```
  *Salida esperada:* `uid=102(postgres) gid=102(postgres) grupos=102(postgres)...`
- Desde `pg-primary` (como usuario `postgres`):
  ```bash
  ssh barman@barman-server "id"
  ```
  *Salida esperada:* `uid=101(barman) gid=101(barman) grupos=101(barman)...`

---

### Paso 3: Configuración de PostgreSQL en `pg-primary` para Streaming de WALs y Replicación

Para que el streaming de WALs funcione mediante `pg_receivewal` y se mantenga una replicación estable, se debe configurar una ranura de replicación física y un usuario de base de datos con permisos dedicados.

**Instrucciones**:

1. En el contenedor `pg-primary` (como usuario `postgres`), inicia sesión en la consola interactiva `psql`:
   ```bash
   psql -d enterprise_db
   ```

2. Crea el usuario con privilegios de replicación y superusuario para garantizar las operaciones de lectura de metadatos de Barman, y crea la ranura de replicación física de manera explícita:
   ```sql
   -- Crear usuario dedicado de replicación
   CREATE USER barman_user WITH SUPERUSER REPLICATION PASSWORD 'PostgresAdminPass123!';

   -- Crear un replication slot físico dedicado para Barman
   SELECT pg_create_physical_replication_slot('barman_slot');
   ```

3. Modifica los parámetros de archivado en el archivo de configuración `postgresql.conf` de tu servidor primario para habilitar la generación de WALs detallados. (La ubicación estándar suele ser `/var/lib/postgresql/data/postgresql.conf` o `/etc/postgresql/postgresql.conf` dependiendo del empaquetamiento):
   ```ini
   wal_level = replica
   max_wal_senders = 10
   max_replication_slots = 10
   archive_mode = on
   archive_command = 'test ! -f /var/lib/postgresql/data/archive/%f && cp %p /var/lib/postgresql/data/archive/%f'
   ```
   *Nota:* Asegura que el directorio `/var/lib/postgresql/data/archive/` exista y pertenezca al usuario `postgres`:
   ```bash
   mkdir -p /var/lib/postgresql/data/archive/
   chown -R postgres:postgres /var/lib/postgresql/data/archive/
   ```

4. Asegura la autenticación en el archivo de control `/var/lib/postgresql/data/pg_hba.conf` para permitir conexiones de replicación y estándar de Barman desde la red interna de Docker:
   Añade las siguientes líneas al final de `pg_hba.conf`:
   ```text
   # Permitir conexiones de consulta estándar desde Barman
   host    enterprise_db   barman_user     172.18.0.0/16           scram-sha-256
   # Permitir conexiones de streaming replication desde Barman
   host    replication     barman_user     172.18.0.0/16           scram-sha-256
   ```
   *(Nota: Ajusta la subred `172.18.0.0/16` si tu red Docker utiliza un direccionamiento diferente).*

5. Reinicia el clúster de PostgreSQL en `pg-primary` para aplicar las variables de red y configuraciones de replicación:
   ```bash
   # Comando de reinicio del motor (dependiendo de la distribución del contenedor)
   pg_ctl -D /var/lib/postgresql/data -m fast restart
   ```

**Resultado esperado**:
El motor de base de datos se encuentra listo para emitir flujos de replicación lógica/física y reconoce la ranura `barman_slot`.

**Verificación**:
Desde el contenedor `pg-primary` ejecuta:
```bash
psql -d enterprise_db -c "SELECT slot_name, slot_type, active FROM pg_replication_slots WHERE slot_name = 'barman_slot';"
```
*Salida esperada:*
```text
  slot_name  | slot_type | active 
-------------+-----------+--------
 barman_slot | physical  | f
(1 row)
```
*(Nota: El estado `active` será `f` (falso) temporalmente hasta que Barman se conecte al canal de replicación).*

---

### Paso 4: Configuración Global y Específica de Barman

En este paso configuraremos las directivas del archivo global de Barman en `barman-server` y se creará la definición de nuestro clúster `pg-primary`.

**Instrucciones**:

1. En el contenedor `barman-server` como usuario `root`, edita el archivo de configuración global `/etc/barman.conf` para asegurar la correcta asignación de rutas y control de logs:
   ```bash
   nano /etc/barman.conf
   ```
   Asegúrate de que las directivas globales queden configuradas de la siguiente manera:
   ```ini
   [barman]
   barman_home = /var/lib/barman
   barman_user = barman
   log_file = /var/log/barman/barman.log
   log_level = INFO
   compression = gzip
   ```

2. Crea la carpeta de configuración específica de servidores de Barman si no existe:
   ```bash
   mkdir -p /etc/barman.d/
   chown -R barman:barman /etc/barman.d/
   ```

3. Crea el archivo de configuración específico para nuestro nodo de base de datos en `/etc/barman.d/pg-primary.conf`:
   ```bash
   nano /etc/barman.d/pg-primary.conf
   ```
   Agrega la siguiente especificación del servidor (reemplaza las IPs si es necesario, o usa la resolución DNS nativa de Docker `pg-primary`):
   ```ini
   [pg-primary]
   description = "Clúster de Base de Datos Empresarial en Producción"
   
   # Conexión SSH para ejecución de scripts locales y lectura de catálogos
   ssh_command = ssh postgres@pg-primary
   
   # Cadena de conexión estándar de PostgreSQL
   conninfo = host=pg-primary user=barman_user dbname=enterprise_db password=PostgresAdminPass123!
   
   # Cadena de conexión dedicada para streaming de replicación (pg_receivewal)
   streaming_conninfo = host=pg-primary user=barman_user dbname=enterprise_db password=PostgresAdminPass123!
   
   # Modo de respaldo y archivado de logs de transacciones
   backup_method = rsync
   archiver = on
   
   # Configuración de Streaming de WALs y Replication Slot
   streaming_archiver = on
   slot_name = barman_slot
   
   # Políticas de retención: mantener un historial de respaldos de 30 días
   retention_policy = RECOVERY WINDOW OF 30 DAYS
   wal_retention_policy = main
   ```

4. Asegura la propiedad de los archivos y asigna los permisos correctos:
   ```bash
   chown -R barman:barman /etc/barman.d/pg-primary.conf
   chmod 600 /etc/barman.d/pg-primary.conf
   ```

5. Inicializa la ranura de replicación activa desde el lado de Barman. Para ello, como usuario `barman` en `barman-server`, ejecuta:
   ```bash
   su - barman
   barman receive-wal --create-slot pg-primary
   ```
   *(Nota: Como la ranura ya fue creada por SQL en el Paso 3, Barman se asociará directamente a ella o te notificará que ya se encuentra disponible para su uso).*

6. Inicia el proceso de recepción de logs transaccionales en tiempo real (`pg_receivewal`) en segundo plano mediante la ejecución de:
   ```bash
   barman receive-wal pg-primary &
   ```

**Resultado esperado**:
El servidor de Barman ha establecido enlace por consola con el clúster remoto y está listo para recibir segmentos WAL mediante streaming directo.

**Verificación**:
Ejecuta la verificación general del clúster con Barman:
```bash
barman check pg-primary
```
*Salida esperada (Todos los checks deben indicar `OK`):*
```text
Server pg-primary:
	PostgreSQL: OK
	superuser or replication role: OK
	PostgreSQL streaming: OK
	wal_level: OK
	directories: OK
	write permission: OK
	recovery.conf: OK (not needed)
	replication slot: OK
	pg_receivewal: OK
	pg_receivewal active: OK
	system checks: OK
```

---

### Paso 5: Ejecución del Respaldo Físico Completo e Incremental

Se ejecutará el ciclo de vida operativo del respaldo, simulando transacciones reales en el motor de base de datos para registrar respaldos diferenciales e incrementales.

**Instrucciones**:

1. En el contenedor `barman-server` (como usuario `barman`), ejecuta el primer respaldo físico completo del sistema:
   ```bash
   barman backup pg-primary
   ```
   *Nota:* Este comando invocará una copia consistente de archivos vía `rsync` y detendrá la fase de copia de forma limpia.

2. Visualiza el estado actual del catálogo de respaldos creados:
   ```bash
   barman list-backup pg-primary
   ```

3. Modifica el estado del motor simulando la creación de transacciones en la base de datos empresarial. Desde `pg-primary` (como usuario `postgres`), accede a la consola de base de datos e inserta nuevos registros de transacciones en una tabla de prueba:
   ```bash
   psql -d enterprise_db
   ```
   Ejecuta las siguientes sentencias SQL:
   ```sql
   CREATE TABLE IF NOT EXISTS audit_log (
       id SERIAL PRIMARY KEY,
       event_description VARCHAR(255),
       created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
   );

   INSERT INTO audit_log (event_description) VALUES 
   ('Iniciando ciclo de respaldo incremental en Barman - Transacción 1'),
   ('Ejecución de cambios críticos de datos - Transacción 2');
   ```

4. Fuerza la rotación del segmento de registro de transacciones (WAL) en el motor para asegurar que los cambios se transmitan al archivador secundario de Barman de inmediato:
   ```sql
   SELECT pg_switch_wal();
   ```
   Cierra la sesión de `psql` (`\q`).

5. Ejecuta un respaldo incremental de Barman para capturar únicamente los bloques de datos modificados desde el respaldo completo anterior:
   Desde el contenedor `barman-server` como usuario `barman`:
   ```bash
   barman backup --reuse-backup=link pg-primary
   ```
   *(Nota: El parámetro `--reuse-backup=link` habilita la copia incremental por rsync utilizando enlaces físicos de archivos sin alteración, reduciendo drásticamente el uso de almacenamiento físico en disco y disminuyendo el tiempo de transferencia).*

**Resultado esperado**:
Se habrán generado dos respaldos diferenciados en el repositorio de Barman local (un backup base completo "Full" y un respaldo incremental estructurado).

**Verificación**:
Inspecciona el catálogo de respaldos de Barman y su información detallada para validar el correcto almacenamiento físico de la estructura de datos:
```bash
barman info pg-primary
```
*Salida esperada:* Debe mostrar información del clúster, detallando que se cuentan con al menos 2 respaldos válidos guardados en disco, la fecha de creación de cada uno y el tamaño del conjunto de datos.

---

## Validación y Pruebas

Para comprobar que la implementación cumple estrictamente con los objetivos de consistencia de datos e integridad del laboratorio, realiza el siguiente plan de pruebas detallado.

### 1. Validación de Estado de Comunicación y Canales Activos

Como usuario `barman` dentro del servidor de respaldos `barman-server`, ejecuta el siguiente diagnóstico de conectividad con PostgreSQL:
```bash
barman status pg-primary
```
*Criterio de Aceptación*: La salida en consola debe reflejar el estado actual del motor, indicando la versión de base de datos (`16.2`), el modo de streaming activo (`streaming_archiver: active`) y el estado actual de la ranura de replicación (`replication_slot: active`).

### 2. Prueba de Caso Adversario (Simulación de Falla de Red / Autenticación Fallida)

Para probar la resiliencia y el manejo de excepciones de Barman, simularemos una desincronización en el archivo de permisos `pg_hba.conf` para el rol de replicación.

1. Accede a `pg-primary` (como root) y edita temporalmente `/var/lib/postgresql/data/pg_hba.conf` modificando el método de autenticación del usuario `barman_user` a un método inválido o bloqueado (por ejemplo, `reject`):
   ```text
   # Cambiar temporalmente de scram-sha-256 a reject para simular error de red/credencial
   host    replication     barman_user     172.18.0.0/16           reject
   ```
2. Aplica los cambios en el motor PostgreSQL:
   ```bash
   pg_ctl -D /var/lib/postgresql/data -m fast reload
   ```
3. Ejecuta la validación del estado en `barman-server` (como usuario `barman`):
   ```bash
   barman check pg-primary
   ```
4. *Análisis Adversario*: Observa la respuesta del sistema. Barman debe detectar de inmediato la anomalía de seguridad y marcar con error explícito las secciones críticas:
   ```text
   PostgreSQL streaming: FAILED (FATAL: password authentication failed for user "barman_user")
   pg_receivewal active: FAILED
   ```
5. *Restauración de Servicio*: Regresa el archivo `pg_hba.conf` a su configuración original de confianza (`scram-sha-256`), recarga el motor con `pg_ctl -D /var/lib/postgresql/data -m fast reload` y vuelve a verificar con `barman check pg-primary`. El estado debe retornar a `OK` de manera automática.

---

## Solución de Problemas

A continuación se describen los dos fallos técnicos más comunes al implementar esta arquitectura de respaldos físicos, detallando sus síntomas, causas internas y soluciones paso a paso.

### Caso 1: Error `PG_RECEIVEWAL: FAILED` al ejecutar `barman check`

- **Síntomas en Consola**:
  ```text
  pg_receivewal: FAILED (pg_receivewal is not running)
  ```
- **Causa Raíz**: El binario secundario `pg_receivewal` no se está ejecutando en segundo plano en el servidor de Barman, o la ranura de replicación en el origen (`pg-primary`) fue eliminada o está deshabilitada.
- **Solución y Mitigación**:
  1. Asegúrate de que la ranura de replicación existe en PostgreSQL ejecutando en `pg-primary`:
     ```sql
     SELECT * FROM pg_replication_slots WHERE slot_name = 'barman_slot';
     ```
     Si no existe, créala: `SELECT pg_create_physical_replication_slot('barman_slot');`.
  2. En `barman-server`, con el usuario `barman`, inicia el proceso de manera persistente en segundo plano:
     ```bash
     nohup barman receive-wal pg-primary > /var/log/barman/receive-wal.log 2>&1 &
     ```
  3. Ejecuta nuevamente `barman check pg-primary` para verificar el estado de conexión directa.

### Caso 2: Error de Permiso Denegado en SSH (`Permission denied (publickey)`)

- **Síntomas en Consola**:
  ```text
  ssh: connect to host pg-primary port 22: Connection refused
  # O bien:
  Permission denied (publickey,password).
  ```
- **Causa Raíz**: El daemon SSH (`sshd`) no está iniciado en alguno de los contenedores Docker, o los permisos de seguridad de las llaves en los directorios `~/.ssh` son incorrectos (SSH rechaza llaves que tienen permisos de lectura/escritura muy abiertos para otros usuarios).
- **Solución y Mitigación**:
  1. Comprueba y levanta el servicio SSH en ambos contenedores de manera explícita:
     ```bash
     /usr/sbin/sshd
     ```
  2. Restringe los permisos de seguridad de las llaves en ambos servidores. Ejecuta en el directorio personal de los usuarios `barman` y `postgres`:
     ```bash
     chmod 700 ~/.ssh
     chmod 600 ~/.ssh/id_rsa
     chmod 600 ~/.ssh/authorized_keys
     ```

---

## Limpieza

Para restaurar el estado inicial de tu infraestructura local de contenedores tras completar este laboratorio, ejecuta secuencialmente los siguientes comandos de desaprovisionamiento en tu máquina host:

1. Sal de todas las sesiones de terminal activas dentro de los contenedores ejecutando `exit` o usando la combinación de teclas `Ctrl + D`.

2. Detén y remueve el contenedor del servidor de respaldos `barman-server` creado para esta práctica:
   ```bash
   docker stop barman-server
   docker rm barman-server
   ```

3. Accede al contenedor `pg-primary` para eliminar el slot de replicación y el rol de base de datos creados:
   ```bash
   docker exec -it pg-primary psql -U postgres -d enterprise_db -c "SELECT pg_drop_replication_slot('barman_slot');"
   docker exec -it pg-primary psql -U postgres -d enterprise_db -c "DROP USER IF EXISTS barman_user;"
   ```

4. Elimina la tabla temporal de auditoría creada para validar los incrementales:
   ```bash
   docker exec -it pg-primary psql -U postgres -d enterprise_db -c "DROP TABLE IF EXISTS audit_log;"
   ```

---

## Resumen

En esta práctica de laboratorio has implementado con éxito una solución de respaldos físicos altamente robusta bajo el estándar de Barman 3.10.0 en un modelo cliente-servidor aislado:

- **Infraestructura Dedicada**: Desplegaste un servidor independiente (`barman-server`) basado en Debian 12.5 con Barman 3.10.0 e integraste comunicación segura mediante intercambio de llaves públicas SSH.
- **Transmisión de WALs**: Configuraste de forma nativa el motor **PostgreSQL 16.2** para que emitiera registros transaccionales continuos hacia una ranura de replicación física activa (`barman_slot`).
- **Control del Ciclo de Vida**: Ejecutaste con éxito una estrategia híbrida de recuperación mediante respaldos físicos completos (`Full`) y respaldos de bloque modificado (`Incremental`) reduciendo el consumo de espacio a través de enlaces directos de archivos (`--reuse-backup=link`).
- **Validación y Resiliencia**: Diagnosticaste la consistencia y comprobaste la tolerancia a fallos ante variaciones inesperadas de seguridad en el archivo `pg_hba.conf`.
