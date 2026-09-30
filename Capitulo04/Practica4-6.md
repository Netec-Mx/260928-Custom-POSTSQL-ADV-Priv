# 4.6 Implementación de PgBouncer

## Metadatos

| Atributo | Detalle |
| :--- | :--- |
| **Duración** | 60 minutos |
| **Complejidad** | Media |
| **Nivel de Bloom** | Aplicar (Apply) |

---

## Descripción General

En este laboratorio, integrarás **PgBouncer 1.22.0** como intermediario (*connection pooler*) de alto rendimiento frente a una base de datos PostgreSQL 16.2 activa. El objetivo principal es resolver la problemática de saturación de recursos (procesos backend) causada por conexiones de aplicaciones concurrentes e ineficientes.

Diseñarás e implementarás archivos de configuración clave (`pgbouncer.ini` y `userlist.txt`), construirás un contenedor personalizado bajo la red empresarial común `pg_enterprise_net` y validarás de forma práctica las diferencias entre los modos de pooling **Transaction** y **Session**. Mediante el uso de la herramienta de estrés de base de datos `pgbench`, simularás escenarios de alta concurrencia con más de 500 hilos de ejecución para contrastar el comportamiento, la latencia y la denegación de servicios del motor nativo frente a la optimización con PgBouncer.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Configurar y desplegar un contenedor Docker de PgBouncer 1.22.0 enlazado al servidor `pg-primary`.
- [ ] Generar un archivo de autenticación seguro `userlist.txt` utilizando hashes de contraseñas de PostgreSQL (SCRAM-SHA-256).
- [ ] Configurar y diferenciar de manera práctica los modos de pooling **Transaction** y **Session**.
- [ ] Administrar el pooler mediante la consola de comandos interna de PgBouncer.
- [ ] Ejecutar y analizar pruebas de carga con `pgbench` para evaluar la mitigación de la saturación de conexiones.

---

## Prerrequisitos

Para completar este laboratorio con éxito, se requiere:
- **Conocimientos Teóricos**:
  - Arquitectura de procesos de PostgreSQL (proceso *fork-per-connection*).
  - Protocolo de conexión TCP/IP y sockets de sistema.
  - Conocimiento intermedio de comandos SQL y shell de Linux.
- **Acceso y Software**:
  - Docker Engine instalado y configurado (versión 26.0.0 o superior).
  - Cliente `psql` local o dentro de contenedores.
  - Acceso a internet para la descarga de imágenes base oficiales.
  - Cuenta con permisos de superusuario (`sudo`) en el sistema anfitrión.

---

## Entorno de Laboratorio

El entorno se implementará en base a las siguientes tecnologías y variables globales:

### Software Utilizado

| Herramienta | Versión Exacta | Licencia | Enlace Oficial |
| :--- | :--- | :--- | :--- |
| **PostgreSQL Community Edition** | 16.2 | PostgreSQL License | [https://www.postgresql.org/](https://www.postgresql.org/) |
| **PgBouncer** | 1.22.0 | MIT License | [https://www.pgbouncer.org/](https://www.pgbouncer.org/) |
| **Docker Engine** | 26.0.0 | Apache 2.0 | [https://www.docker.com/](https://www.docker.com/) |

### Parámetros Globales de Red y Acceso

*   **Nombre de la Red Docker**: `pg_enterprise_net` (tipo Bridge)
*   **Base de Datos Global**: `enterprise_db`
*   **Usuario Administrador**: `postgres` (Contraseña: `PostgresAdminPass123!`)
*   **Puerto PostgreSQL Nativo**: `5432` (Contenedor `pg-primary`)
*   **Puerto PgBouncer**: `6432` (Contenedor `pg-bouncer`)

---

## Instrucciones Paso a Paso

### Paso 1: Levantar y configurar el contenedor de base de datos (`pg-primary`)

En este paso, asegurarás que el contenedor primario de PostgreSQL está activo dentro de la red corporativa especificada y habilitarás una configuración estricta de conexiones máximas para simular un cuello de botella real de hardware.

1. Abre una terminal en tu máquina host y crea la red Docker común si aún no existe:
   ```bash
   docker network create pg_enterprise_net || true
   ```

2. Ejecuta el contenedor `pg-primary` con la imagen de PostgreSQL 16.2:
   ```bash
   docker run -d --name pg-primary \
     --network pg_enterprise_net \
     -p 5432:5432 \
     -e POSTGRES_DB=enterprise_db \
     -e POSTGRES_USER=postgres \
     -e POSTGRES_PASSWORD='PostgresAdminPass123!' \
     postgres:16.2
   ```

3. Espera 5 segundos a que la base de datos esté lista y conéctate para limitar el número máximo de conexiones de PostgreSQL a **100**. Esto emulará un servidor con limitaciones físicas reales de memoria RAM:
   ```bash
   docker exec -it pg-primary psql -U postgres -d enterprise_db -c "ALTER SYSTEM SET max_connections = 100;"
   docker exec -it pg-primary psql -U postgres -d enterprise_db -c "SELECT pg_reload_conf();"
   ```

4. Crea un usuario de aplicación dedicado para las pruebas, asegurando que utiliza cifrado seguro `scram-sha-256`:
   ```bash
   docker exec -it pg-primary psql -U postgres -d enterprise_db -c "CREATE USER app_user WITH PASSWORD 'AppUserSecurePass456!';"
   ```

**Verificación**: Ejecuta la consulta para asegurar que el límite de conexiones se aplicó de forma correcta y que el usuario existe.
```bash
docker exec -it pg-primary psql -U postgres -d enterprise_db -c "SHOW max_connections;"
```
*Salida esperada:*
```text
 max_connections 
-----------------
 100
(1 row)
```

---

### Paso 2: Extraer credenciales y crear el archivo `userlist.txt`

PgBouncer requiere autenticar a los clientes antes de delegar las conexiones a PostgreSQL. Por seguridad, no utilizaremos contraseñas en texto plano, sino los hashes `scram-sha-256` reales almacenados en el catálogo de PostgreSQL.

1. Prepara el directorio temporal en el host donde guardarás los archivos de configuración de PgBouncer:
   ```bash
   mkdir -p /tmp/pgbouncer
   ```

2. Consulta los hashes de autenticación de forma formateada directamente desde el catálogo `pg_shadow` del servidor de base de datos:
   ```bash
   docker exec -it pg-primary psql -U postgres -d enterprise_db -t -A -c \
     "SELECT concat('\"', usename, '\" \"', passwd, '\"') FROM pg_shadow WHERE usename IN ('postgres', 'app_user');"
   ```

   *Ejemplo de salida obtenida (los hashes serán ligeramente distintos en tu entorno):*
   ```text
   "postgres" "SCRAM-SHA-256$4096:hpNq87+D/pSshw==$HhDkR4V..."
   "app_user" "SCRAM-SHA-256$4096:G5dfF5sh84fDdf==$AsFeE9W..."
   ```

3. Escribe estos valores directamente en el archivo `/tmp/pgbouncer/userlist.txt`. Asegúrate de eliminar cualquier retorno de carro (`\r`) si estás en sistemas Windows/WSL:
   ```bash
   # Ejecuta esto en tu host de Linux para escribir el archivo directamente
   cat <<EOF > /tmp/pgbouncer/userlist.txt
   "postgres" "SCRAM-SHA-256$4096:4096/vCis+rN8vW63Xp54Bswg==$xJ/Zq9fHh5U5YdF9h9R1M2mG:Y8W+kY6r8T9xX6u+"
   "app_user" "SCRAM-SHA-256$4096:8shFdfKjhg89gFd==$kJiUhYgTfFrDeEwQaZsXcVbNmMlkjH:YhUjKiLoOpPlKjH"
   EOF
   ```
   > **Nota pedagógica**: Para efectos prácticos de este laboratorio, puedes usar hashes genéricos válidos o, de manera preferencial, copiar exactamente los generados por tu motor PostgreSQL en la consola para garantizar que la autenticación no falle en el paso 4.

---

### Paso 3: Configurar el archivo global `pgbouncer.ini`

En este paso, configurarás los límites del pool de conexiones. Configurarás un modo de pooling inicial de tipo **Transaction** y enlazarás de forma transparente la base de datos lógica `enterprise_db` con el contenedor `pg-primary`.

1. Crea y edita el archivo `/tmp/pgbouncer/pgbouncer.ini` en el host:
   ```bash
   cat <<EOF > /tmp/pgbouncer/pgbouncer.ini
   [databases]
   enterprise_db = host=pg-primary port=5432 dbname=enterprise_db

   [pgbouncer]
   ;; Configuración de puertos de escucha
   listen_addr = *
   listen_port = 6432
   
   ;; Archivos de registro y PID
   logfile = /tmp/pgbouncer.log
   pidfile = /tmp/pgbouncer.pid

   ;; Configuración de Seguridad y Autenticación
   auth_type = scram-sha-256
   auth_file = /etc/pgbouncer/userlist.txt
   admin_users = postgres
   stats_users = app_user

   ;; Gestión de Pool (Optimizado para pruebas de estrés de este laboratorio)
   pool_mode = transaction
   max_client_conn = 1000
   default_pool_size = 20
   min_pool_size = 5
   reserve_pool_size = 5
   reserve_pool_timeout = 3
   EOF
   ```

**Verificación**: Comprueba la existencia y validez visual de los dos archivos creados en el host antes de construir el contenedor:
```bash
ls -la /tmp/pgbouncer/
```

---

### Paso 4: Construir y desplegar el contenedor de PgBouncer (1.22.0)

Para garantizar la estabilidad del laboratorio y cumplir con la directiva de no usar imágenes marcadas como "latest", compilaremos o utilizaremos una imagen basada en la distribución estable Debian 12 con paquetes de PgBouncer 1.22.0 de los repositorios oficiales de PostgreSQL.

1. Crea el `Dockerfile` optimizado en `/tmp/pgbouncer/Dockerfile`:
   ```bash
   cat <<EOF > /tmp/pgbouncer/Dockerfile
   FROM debian:12.5-slim

   # Instalar dependencias necesarias y el repositorio de PostgreSQL
   RUN apt-get update && apt-get install -y --no-install-recommends \\
       gnupg \\
       ca-certificates \\
       curl \\
       && curl -fsSL https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg \\
       && echo "deb http://apt.postgresql.org/pub/repos/apt bookworm-pgdg main" > /etc/apt/sources.list.d/pgdg.list \\
       && apt-get update \\
       && apt-get install -y --no-install-recommends \\
          pgbouncer=1.22.0-1.pgdg120+1 \\
          postgresql-client-16 \\
       && rm -rf /var/lib/apt/lists/*

   # Ejecutar como usuario sin privilegios por seguridad (NIST SP 800-53)
   RUN mkdir -p /etc/pgbouncer /var/log/postgresql /var/run/postgresql && \\
       chown -R pgbouncer:postgres /etc/pgbouncer /var/log/postgresql /var/run/postgresql

   USER pgbouncer
   EXPOSE 6432
   ENTRYPOINT ["pgbouncer"]
   EOF
   ```

2. Construye la imagen localmente:
   ```bash
   docker build -t custom-pgbouncer:1.22.0 /tmp/pgbouncer/
   ```

3. Levanta el contenedor de PgBouncer conectándolo a la misma red de datos y montando los archivos creados:
   ```bash
   docker run -d --name pg-bouncer \
     --network pg_enterprise_net \
     -p 6432:6432 \
     -v /tmp/pgbouncer/pgbouncer.ini:/etc/pgbouncer/pgbouncer.ini \
     -v /tmp/pgbouncer/userlist.txt:/etc/pgbouncer/userlist.txt \
     custom-pgbouncer:1.22.0 /etc/pgbouncer/pgbouncer.ini
   ```

**Verificación**: Comprueba que el contenedor de PgBouncer está corriendo de forma correcta mediante el examen de sus logs internos:
```bash
docker logs pg-bouncer
```
*Salida esperada:*
```text
2024-10-24 12:00:00.123 UTC [1] LOG OS_init: egg: ...
2024-10-24 12:00:00.124 UTC [1] LOG pgbouncer 1.22.0-1.pgdg120+1 startup
2024-10-24 12:00:00.125 UTC [1] LOG Listening on 0.0.0.0:6432
```

---

### Paso 5: Ejecutar pruebas de carga con `pgbench` (Transaction Mode)

Para analizar la ganancia de rendimiento y el manejo de hilos virtuales, inicializaremos un esquema de prueba y lanzaremos ataques controlados de concurrencia de clientes que superen el límite de conexiones del backend real (100 conexiones).

1. Inicializa el esquema de pruebas de `pgbench` con un factor de escala de 5. Ejecutamos el comando directamente desde la base de datos `pg-primary`:
   ```bash
   docker exec -it pg-primary pgbench -i -s 5 -U postgres -d enterprise_db
   ```

2. Ejecuta una prueba de carga apuntando **directamente** a PostgreSQL (`pg-primary` en puerto `5432`) usando una concurrencia superior al límite máximo (`-c 120` conexiones con `-j 4` hilos de CPU):
   ```bash
   docker exec -it pg-primary pgbench -c 120 -j 4 -t 50 -U app_user -h pg-primary -p 5432 enterprise_db
   ```
   *Resultado esperado:* El benchmark fallará casi de inmediato arrojando un error crítico de saturación del pool nativo.
   ```text
   connection to database "enterprise_db" failed:
   FATAL: sorry, too many clients already
   ```

3. Ahora ejecuta la misma prueba de carga de alta concurrencia pero dirigiendo el tráfico a través del puerto de **PgBouncer** (`pg-bouncer` en el puerto `6432`), incrementando el reto a **250 conexiones simultáneas**:
   ```bash
   docker exec -it pg-primary pgbench -c 250 -j 10 -t 50 -U app_user -h pg-bouncer -p 6432 enterprise_db
   ```
   *Resultado esperado:* La prueba se ejecutará de forma completamente exitosa. PgBouncer acepta las 250 conexiones virtuales del frontend y las procesa de manera segura multiplexándolas en un pool persistente de un máximo de 20 conexiones hacia PostgreSQL.

---

### Paso 6: Cambiar a Session Mode y observar el encolamiento

El modo **Session** reserva una conexión física de PostgreSQL para el cliente de manera exclusiva durante todo el ciclo de vida de la conexión. En este paso experimentarás cómo este modo de pooling vuelve a generar ineficiencia bajo condiciones de alta concurrencia.

1. Edita el archivo de configuración en tu host local `/tmp/pgbouncer/pgbouncer.ini` y modifica la línea del modo de pooling para que sea `session`:
   ```bash
   # Modifica la línea del parámetro de pool_mode
   sed -i 's/pool_mode = transaction/pool_mode = session/g' /tmp/pgbouncer/pgbouncer.ini
   ```

2. Conéctate a la consola de administración virtual de PgBouncer para aplicar los cambios sin reiniciar el contenedor (cumpliendo con alta disponibilidad):
   ```bash
   docker exec -it pg-bouncer psql -h 127.0.0.1 -p 6432 -U postgres -d pgbouncer -c "RELOAD;"
   ```

3. Vuelve a ejecutar la prueba de carga usando `pgbench` con una concurrencia de 50 clientes. Nota que la cantidad máxima de conexiones del backend configurada en PgBouncer es `default_pool_size = 20`.
   ```bash
   docker exec -it pg-primary pgbench -c 50 -j 5 -t 10 -U app_user -h pg-bouncer -p 6432 enterprise_db
   ```

**Verificación**: Observa el comportamiento de la terminal. En modo **Session**, los primeros 20 clientes reservarán por completo el backend disponible de PgBouncer. Los otros 30 clientes restantes serán forzados a esperar (*queueing*) hasta que los procesos iniciales terminen sus sesiones de manera secuencial, degradando drásticamente el rendimiento de latencia general.

---

## Validación y Pruebas

Para validar científicamente el funcionamiento robusto de tu implementación de PgBouncer frente a escenarios reales y comportamientos anómalos (casos adversos), realiza los siguientes análisis de verificación.

### 1. Monitoreo en Caliente mediante la Consola Administrativa
Accede a la base de datos virtual de administración interna de PgBouncer desde el contenedor:
```bash
docker exec -it pg-bouncer psql -h 127.0.0.1 -p 6432 -U postgres -d pgbouncer
```

Dentro del prompt administrativo (`pgbouncer=#`), ejecuta el análisis de rendimiento de pools:
```sql
SHOW POOLS;
```
*Salida esperada:*
Identifica las columnas `cl_active` (clientes activos del frontend conectados al pooler) y `sv_active` (conexiones de backend reales activas contra el motor de PostgreSQL). En un entorno optimizado en modo Transaction bajo carga, `cl_active` siempre será significativamente superior a `sv_active` (por ejemplo, 150 vs 20).

```text
  database     |   user   | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait | maxwait_us 
---------------+----------+-----------+------------+-----------+---------+---------+-----------+----------+---------+------------
 enterprise_db | app_user |       150 |          0 |        20 |       0 |       0 |         0 |        0 |       0 |          0
```

### 2. Prueba Adversaria A: Intento de Fuerza Bruta y Bases de Datos Inexistentes
Ejecuta un comando intentando conectar un usuario legítimo a una base de datos maliciosa o inexistente (`fake_enterprise_db`) a través de la capa de PgBouncer:
```bash
docker exec -it pg-primary psql -h pg-bouncer -p 6432 -U app_user -d fake_enterprise_db
```
*Verificación de Seguridad:*
PgBouncer debe interceptar la petición y abortar la conexión en milisegundos con un mensaje de error explícito del proxy, impidiendo que la consulta maliciosa siquiera llegue a consumir recursos del backend nativo `pg-primary`:
```text
psql: error: connection to server at "pg-bouncer" (172.18.0.3), port 6432 failed: ERROR: No such database: fake_enterprise_db
```

### 3. Prueba Adversaria B: Contaminación de Sesión en Modo Transaction
Conéctate mediante `psql` interactivo a PgBouncer configurado en modo **Transaction** y ejecuta comandos de variables de sesión. Esta prueba te demostrará por qué este modo requiere estricto cuidado en el desarrollo de software.

```bash
docker exec -it pg-primary psql -h pg-bouncer -p 6432 -U app_user -d enterprise_db
```
Una vez dentro, ejecuta de manera secuencial:
```sql
-- Cambiar el parámetro de sesión temporal para esta transacción
SET work_mem = '64MB';
SHOW work_mem;
```
*Comportamiento crítico:* En el modo **Transaction**, una vez que finaliza tu transacción actual, la conexión física física regresa al pool para ser usada por otro usuario de la red. Si el pooler no realiza un reset del estado de la sesión, la variable `work_mem = '64MB'` podría heredarse a otros clientes ajenos de forma descontrolada o provocar un comportamiento errático. En producciones reales de alta seguridad se recomienda configurar `track_extra_parameters` y usar comandos de limpieza para evitar fugas de privilegios o fugas de memoria.

---

## Solución de Problemas

A continuación, se describen dos escenarios de fallas comunes descubiertas durante la integración práctica del pooler con sus respectivos diagnósticos y soluciones:

### Problema 1: Error de Autenticación del Cliente (`psql: error: ... Password authentication failed for user`)
*   **Síntomas**: Al intentar realizar la prueba con `pgbench` o `psql` apuntando a PgBouncer (puerto `6432`), se rechaza la conexión de forma sistemática mostrando el mensaje `FATAL: auth failed` o similar, mientras que la conexión directa al puerto `5432` de PostgreSQL funciona sin problemas con el mismo usuario y contraseña.
*   **Causa**: El hash generado y guardado en `/tmp/pgbouncer/userlist.txt` no corresponde al algoritmo configurado en `auth_type` dentro del archivo `pgbouncer.ini` (por ejemplo, se generó un hash en formato MD5 pero el pooler exige obligatoriamente `scram-sha-256`, o existió una copia errónea del hash omitiendo caracteres finales de relleno como el signo `=` ).
*   **Solución**:
    1. Regenera el hash consultando directamente a la base de datos:
       ```bash
       docker exec -it pg-primary psql -U postgres -d enterprise_db -t -A -c "SELECT passwd FROM pg_shadow WHERE usename='app_user';"
       ```
    2. Asegúrate de que el resultado copiado en `/tmp/pgbouncer/userlist.txt` contenga comillas alrededor del usuario y de la clave, y que no tenga caracteres invisibles como finales de línea de formato Windows (`\r`). Aplica permisos seguros al archivo en el host:
       ```bash
       chmod 600 /tmp/pgbouncer/userlist.txt
       ```
    3. Ejecuta un recargo de configuración en PgBouncer:
       ```bash
       docker exec -it pg-bouncer psql -h 127.0.0.1 -p 6432 -U postgres -d pgbouncer -c "RELOAD;"
       ```

### Problema 2: El pooler se detiene abruptamente con error de Socket ocupado (`bind: Address already in use`)
*   **Síntomas**: Al arrancar el contenedor `pg-bouncer`, este falla inmediatamente y su estado en Docker pasa a `Exited (1)`. Al ejecutar `docker logs pg-bouncer` se muestra: `FATAL: TLS/TCP: bind(0.0.0.0, 6432) failed: Address already in use`.
*   **Causa**: Existe un proceso de PgBouncer antiguo corriendo nativamente en la máquina host que ya se encuentra acaparando el puerto TCP `6432`, o se mapeó de forma incorrecta el puerto en el comando `docker run`.
*   **Solución**:
    1. Detecta qué ID de proceso está utilizando el puerto en la máquina anfitriona:
       ```bash
       sudo ss -ltnp | grep 6432
       ```
    2. Detén el servicio competidor del host o destruye contenedores huérfanos de ejecuciones previas de laboratorio:
       ```bash
       docker rm -f pg-bouncer || true
       ```
    3. Vuelve a iniciar el despliegue del paso 4.

---

## Limpieza

Para restaurar tu máquina anfitriona al estado inicial libre de remanentes de almacenamiento o de contenedores de prueba, ejecuta la siguiente secuencia de destrucción ordenada de recursos:

1. Detén y destruye por completo los contenedores generados:
   ```bash
   docker rm -f pg-primary pg-bouncer
   ```

2. Remueve la red Docker empresarial creada para aislar la conectividad:
   ```bash
   docker network rm pg_enterprise_net
   ```

3. Elimina de manera definitiva los archivos temporales y directorios de configuración de PgBouncer creados en la ruta `/tmp`:
   ```bash
   rm -rf /tmp/pgbouncer
   ```

---

## Resumen

En este laboratorio, has implementado exitosamente **PgBouncer 1.22.0** para optimizar un servidor **PostgreSQL 16.2** sujeto a limitaciones realistas de infraestructura. 

### Puntos Clave Aprendidos:
- **Multiplexación de Recursos**: El modelo de procesos pesados de PostgreSQL (un backend por cliente) no escala eficientemente de forma directa bajo altas ráfagas de tráfico, requiriendo un pooler para gestionar la sobrecarga del CPU por cambio de contexto (*context switching*).
- **Modos de Pooling**:
  - **Transaction Mode**: Es el modo más óptimo para microservicios y escalamiento web, permitiendo que 250+ clientes compartan una fracción mínima de hilos persistentes reales (20 conexiones backend). No obstante, invalida el uso seguro de variables y tablas de estado de sesión.
  - **Session Mode**: Mantiene compatibilidad total con características de PostgreSQL, pero reintroduce problemas de encolamiento y latencia severa ante la saturación física de hilos.
- **Seguridad (ISO 27001/NIST)**: Se implementó de manera robusta la autenticación descentralizada mediante hashes cifrados en estándar de seguridad `scram-sha-256`, permitiendo que el proxy maneje la validez de los accesos sin transferir o procesar textos planos en el tráfico de red.

---