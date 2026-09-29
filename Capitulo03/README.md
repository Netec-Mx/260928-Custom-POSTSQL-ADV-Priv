# 6 Implementación de PgBouncer

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

# 7 Configuración de HAProxy

## Metadatos

| Métrica | Detalle |
| :--- | :--- |
| **Duración** | 60 minutos |
| **Complejidad** | Alta (Hard) |
| **Nivel de Bloom** | Aplicar (Apply) |

---

## Descripción General

En este laboratorio, los estudiantes diseñarán e implementarán un balanceador de carga TCP de alta disponibilidad utilizando **HAProxy 2.9.5 (Community Edition)**. El objetivo principal es desplegar un proxy reverso de Capa 4 (TCP) que actúe como punto de entrada único para el tráfico corporativo en el puerto `5000`, redirigiendo de manera óptima las conexiones entrantes hacia el pooler de conexiones **PgBouncer 1.22.0**, el cual a su vez gestiona las conexiones al motor backend de **PostgreSQL 16.2**. Asimismo, se implementarán mecanismos de monitoreo activo de salud mediante chequeos avanzados nativos de PostgreSQL a nivel de red TCP (`option pgsql-check`), y se simularán eventos de indisponibilidad y recuperación del servicio de base de datos para evaluar la resiliencia del balanceador.

---

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Instalar, estructurar y desplegar un balanceador de carga de red TCP utilizando **HAProxy 2.9.5** en un entorno contenedorizado.
- [ ] Diseñar y estructurar reglas de ruteo de bajo nivel TCP/IP para el reenvío de paquetes transparentes al pooler de conexiones.
- [ ] Configurar y validar mecanismos de monitorización activa de base de datos utilizando el módulo avanzado `option pgsql-check` de HAProxy.
- [ ] Simular y analizar el comportamiento de HAProxy frente a fallos de infraestructura, validando el tiempo de respuesta y la reconexión automática del backend.

---

## Prerrequisitos

Para completar este laboratorio con éxito, se requiere:
1. **Conocimientos teóricos**:
   - Comprensión del protocolo TCP/IP, direccionamiento de puertos, sockets y el modelo de proxy reverso.
   - Entendimiento del funcionamiento básico de un Connection Pooler (PgBouncer) y el protocolo de red nativo de PostgreSQL (Frontend/Backend Protocol v3.0).
2. **Acceso y software**:
   - Un motor Docker Engine/Desktop versión 26.0.0 o superior instalado en el sistema operativo anfitrión.
   - Acceso de red sin restricciones locales (firewall) para mapear los puertos `5000`, `6432` y `5432`.
   - Cliente SQL interactivo `psql` (versión 16) instalado localmente o accesible dentro de los contenedores para interactuar con la infraestructura.

---

## Entorno de Laboratorio

La infraestructura de este laboratorio está basada en contenedores Docker organizados bajo una única red de tipo Bridge.

### Especificaciones de Hardware (Mínimo recomendado)
- **Procesador**: Arquitectura x86_64, mínimo 4 núcleos físicos.
- **Memoria RAM**: 16 GB instalados.
- **Almacenamiento**: SSD NVMe con al menos 10 GB de espacio libre para imágenes y volúmenes temporales de logs.

### Especificaciones de Software y Licencias

| Software | Versión Exacta | Arquitectura | Licencia | URL de Descarga / Repositorio Oficial |
| :--- | :--- | :--- | :--- | :--- |
| **PostgreSQL** | 16.2 Community Edition | x86_64 (Debian) | PostgreSQL License | [https://www.postgresql.org/](https://www.postgresql.org/) |
| **PgBouncer** | 1.22.0 Community Edition | x86_64 (Debian) | BSD 3-Clause | [https://www.pgbouncer.org/](https://www.pgbouncer.org/) |
| **HAProxy** | 2.9.5 Community Edition | x86_64 (Alpine/Debian) | GPLv2 | [https://www.haproxy.org/](https://www.haproxy.org/) |

### Parámetros Globales de Red y Entorno
- **Red Docker**: `pg_enterprise_net` (tipo Bridge).
- **Base de Datos**: `enterprise_db`.
- **Usuario Administrador**: `postgres` (Contraseña: `PostgresAdminPass123!`).
- **Contenedor Primario PostgreSQL (`pg-primary`)**: Puerto interno `5432`.
- **Contenedor PgBouncer (`pg-bouncer`)**: Puerto interno/externo `6432`.
- **Contenedor HAProxy (`haproxy-lb`)**: Puerto externo corporativo `5000`.

---

## Instrucciones Paso a Paso

### Paso 1: Configurar la estructura de archivos e inicializar la infraestructura base

**Objetivo**: Establecer los archivos de configuración necesarios para PostgreSQL, PgBouncer y HAProxy, y construir la red base de contenedores.

**Instrucciones**:

1. En tu máquina anfitriona, crea una estructura de directorios limpia para el laboratorio:
   ```bash
   mkdir -p ~/haproxy-lab/config
   cd ~/haproxy-lab
   ```

2. Crea el archivo de usuarios autorizados de PgBouncer (`config/userlist.txt`). Como buena práctica de seguridad (alineada a ISO 27001), utilizaremos un hash MD5/SCRAM-SHA-256 pregenerado. Para simplificar el laboratorio, introduciremos las contraseñas autorizadas correspondientes a la contraseña `PostgresAdminPass123!`:
   ```bash
   cat << 'EOF' > config/userlist.txt
   "postgres" "PostgresAdminPass123!"
   "app_user" "AppUserPassword456!"
   EOF
   ```

3. Modifica los permisos del archivo `userlist.txt` para emular entornos Unix reales de alta seguridad:
   ```bash
   chmod 600 config/userlist.txt
   ```

4. Crea el archivo de configuración global para PgBouncer (`config/pgbouncer.ini`):
   ```ini
   cat << 'EOF' > config/pgbouncer.ini
   [databases]
   enterprise_db = host=pg-primary port=5432 dbname=enterprise_db

   [pgbouncer]
   logfile = /tmp/pgbouncer.log
   pidfile = /tmp/pgbouncer.pid
   listen_addr = *
   listen_port = 6432
   auth_type = plain
   auth_file = /etc/pgbouncer/userlist.txt
   admin_users = postgres
   pool_mode = transaction
   max_client_conn = 10000
   default_pool_size = 20
   min_pool_size = 5
   reserve_pool_size = 5
   reserve_pool_timeout = 5
   EOF
   ```

**Resultado esperado**: Los archivos `userlist.txt` y `pgbouncer.ini` deben estar correctamente creados dentro de la ruta `~/haproxy-lab/config/`.

**Verificación**:
Ejecuta el comando `ls -la config/` y verifica la existencia y permisos de los archivos:
```bash
ls -la config/
```
*Salida esperada:*
```text
-rw-r--r-- 1 user group 345 Mar 30 10:00 pgbouncer.ini
-rw------- 1 user group  64 Mar 30 10:00 userlist.txt
```

---

### Paso 2: Diseñar y estructurar la configuración de HAProxy

**Objetivo**: Generar un archivo de configuración para HAProxy que habilite el balanceo TCP en el puerto 5000, implemente chequeos nativos de PostgreSQL y active la interfaz estadística de monitoreo.

**Instrucciones**:

1. Crea el archivo de configuración de HAProxy (`config/haproxy.cfg`) en el directorio local:
   ```bash
   cat << 'EOF' > config/haproxy.cfg
   global
       log stdout format raw local0 info
       maxconn 15000

   defaults
       log     global
       mode    tcp
       timeout connect 10s
       timeout client  30m
       timeout server  30m
       retries 3

   frontend pg_cluster_front
       bind *:5000
       mode tcp
       default_backend pg_cluster_back

   backend pg_cluster_back
       mode tcp
       balance roundrobin
       
       # Configurar chequeo activo nativo usando el protocolo de PostgreSQL.
       # 'option pgsql-check' envía un mensaje inicial de Handshake de PostgreSQL (StartupMessage).
       option pgsql-check user postgres
       
       # Conectores y pesos
       server srv_pgbouncer pg-bouncer:6432 check inter 3s rise 2 fall 3 weight 100
   
   # Interfaz Web de Estadísticas (Monitoreo Corporativo)
   listen stats
       mode http
       bind *:7000
       stats enable
       stats uri /
       stats refresh 5s
       stats auth admin:StatsPass123!
   EOF
   ```

2. Analiza los componentes del archivo de configuración:
   - `mode tcp`: HAProxy actúa en la Capa 4 (Transporte), sin descifrar el tráfico nativo SQL ni incurrir en la latencia de la capa de aplicación (Capa 7).
   - `option pgsql-check user postgres`: Ejecuta un escaneo de salud de base de datos real. El balanceador intentará simular un login con el usuario `postgres` contra `pg-bouncer` en el puerto `6432`. Si la base de datos o el pooler están caídos, no responderán el paquete de protocolo nativo de PostgreSQL, marcando al backend como "DOWN".
   - `stats auth admin:StatsPass123!`: Habilita una consola web segura en el puerto `7000` para monitorear el estado del balanceador.

**Resultado esperado**: Un archivo de configuración robusto e inmune a inyecciones de comandos estructurado en la carpeta `config`.

**Verificación**:
```bash
cat config/haproxy.cfg | grep -E "pgsql-check|bind"
```
*Salida esperada:*
```text
    bind *:5000
    option pgsql-check user postgres
    bind *:7000
```

---

### Paso 3: Orquestar el despliegue multicontenedor con Docker Compose

**Objetivo**: Crear y ejecutar un archivo `docker-compose.yml` que integre el backend PostgreSQL, el pooler PgBouncer y el balanceador HAProxy dentro de la red corporativa `pg_enterprise_net`.

**Instrucciones**:

1. Crea el archivo de orquestación `docker-compose.yml` en la raíz de tu directorio de trabajo:
   ```yaml
   cat << 'EOF' > docker-compose.yml
   version: '3.8'

   networks:
     pg_enterprise_net:
       name: pg_enterprise_net
       driver: bridge

   services:
     pg-primary:
       image: postgres:16.2
       container_name: pg-primary
       environment:
         POSTGRES_DB: enterprise_db
         POSTGRES_USER: postgres
         POSTGRES_PASSWORD: PostgresAdminPass123!
       ports:
         - "5432:5432"
       networks:
         - pg_enterprise_net
       healthcheck:
         test: ["CMD-SHELL", "pg_isready -U postgres -d enterprise_db"]
         interval: 5s
         timeout: 5s
         retries: 5

     pg-bouncer:
       image: edoburu/pgbouncer:1.22.0
       container_name: pg-bouncer
       ports:
         - "6432:6432"
       volumes:
         - ./config/pgbouncer.ini:/etc/pgbouncer/pgbouncer.ini:ro
         - ./config/userlist.txt:/etc/pgbouncer/userlist.txt:ro
       networks:
         - pg_enterprise_net
       depends_on:
         pg-primary:
           condition: service_healthy

     haproxy-lb:
       image: haproxy:2.9.5-alpine
       container_name: haproxy-lb
       ports:
         - "5000:5000"
         - "7000:7000"
       volumes:
         - ./config/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
       networks:
         - pg_enterprise_net
       depends_on:
         - pg-bouncer
   EOF
   ```

2. Levanta toda la topología de contenedores en modo segundo plano (background):
   ```bash
   docker compose up -d
   ```

3. Comprueba el estado físico de inicialización de los tres contenedores:
   ```bash
   docker compose ps
   ```

**Resultado esperado**: Los tres servicios (`pg-primary`, `pg-bouncer`, `haproxy-lb`) deben estar en estado "Up" o "healthy" e integrados a la red común.

**Verificación**:
```bash
docker ps --filter "network=pg_enterprise_net" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```
*Salida esperada:*
```text
NAMES        STATUS                  PORTS
haproxy-lb   Up ...                  0.0.0.0:5000->5000/tcp, 0.0.0.0:7000->7000/tcp
pg-bouncer   Up ...                  0.0.0.0:6432->6432/tcp
pg-primary   Up (healthy)...         0.0.0.0:5432->5432/tcp
```

---

### Paso 4: Validar el enrutamiento de consultas a través del balanceador HAProxy (Puerto 5000)

**Objetivo**: Demostrar que los clientes externos pueden conectarse de manera transparente y realizar transacciones en PostgreSQL accediendo únicamente por el puerto virtual corporativo del balanceador `5000`.

**Instrucciones**:

1. Conéctate a la base de datos a través del puerto de HAProxy (`5000`) utilizando el usuario `postgres`:
   ```bash
   docker run --rm --net pg_enterprise_net postgres:16.2 psql -h haproxy-lb -p 5000 -U postgres -d enterprise_db -c "SELECT version();"
   ```
   *Nota: Cuando te solicite la contraseña, digita:* `PostgresAdminPass123!`

2. Crea una tabla transaccional de prueba para asegurar el flujo completo de datos (Escritura):
   ```bash
   docker run --rm --net pg_enterprise_net postgres:16.2 psql -h haproxy-lb -p 5000 -U postgres -d enterprise_db -c "
   CREATE TABLE IF NOT EXISTS audit_haproxy (
       id SERIAL PRIMARY KEY,
       event_time TIMESTAMP DEFAULT NOW(),
       client_ip VARCHAR(50)
   );"
   ```

3. Inserta un registro y consúltalo para validar la lectura/escritura integrada:
   ```bash
   docker run --rm --net pg_enterprise_net postgres:16.2 psql -h haproxy-lb -p 5000 -U postgres -d enterprise_db -c "
   INSERT INTO audit_haproxy (client_ip) VALUES ('172.20.0.99');"
   
   docker run --rm --net pg_enterprise_net postgres:16.2 psql -h haproxy-lb -p 5000 -U postgres -d enterprise_db -c "
   SELECT * FROM audit_haproxy;"
   ```

**Resultado esperado**: Las sentencias de consulta, creación de tabla e inserción deben completarse exitosamente a través de la IP de HAProxy sin errores de comunicación.

**Verificación**:
La salida del comando `SELECT * FROM audit_haproxy;` debe ser similar a:
```text
 id |         event_time         |  client_ip  
----+----------------------------+-------------
  1 | 2024-03-30 10:15:30.123456 | 172.20.0.99
(1 row)
```

---

### Paso 5: Simular fallos, aislamiento de nodos y recuperación del servicio

**Objetivo**: Comprobar el comportamiento dinámico de HAProxy y sus políticas de reintento (`pgsql-check`) simulando la caída programada de servicios críticos.

**Instrucciones**:

1. Analiza los logs iniciales de HAProxy para comprobar que detectó la presencia y correcto estado de salud de PgBouncer:
   ```bash
   docker logs haproxy-lb | grep -E "Server|health"
   ```
   *(Si el log básico está en silencio, puedes consultar el socket de estadísticas o levantar la página web de administración en http://localhost:7000 en tu navegador)*

2. Simula una interrupción crítica del pooler de conexiones deteniendo el contenedor `pg-bouncer`:
   ```bash
   docker stop pg-bouncer
   ```

3. Inmediatamente después del comando anterior, intenta conectarte a la base de datos a través del puerto corporativo `5000` de HAProxy:
   ```bash
   docker run --rm --net pg_enterprise_net postgres:16.2 psql -h haproxy-lb -p 5000 -U postgres -d enterprise_db -c "SELECT 1;"
   ```

4. Revisa las últimas líneas de logs de HAProxy para comprobar cómo detectó la interrupción basándose en la configuración de `fall 3` (3 fallos seguidos cada 3 segundos):
   ```bash
   docker logs haproxy-lb | tail -n 15
   ```

5. Levanta el servicio nuevamente para demostrar la recuperación automática (*Self-healing*):
   ```bash
   docker start pg-bouncer
   ```
   *Espera unos 6-8 segundos para permitir que HAProxy revalide el estado con la regla `rise 2`.*

6. Ejecuta de nuevo la validación del punto 3.

**Resultado esperado**: 
- Al apagar `pg-bouncer`, el comando `psql` contra el puerto `5000` debe fallar inmediatamente con un error de conexión rechazada.
- Los logs de HAProxy deben reportar de manera visible la deshabilitación del backend `srv_pgbouncer` (marcado como `DOWN`).
- Al levantar de nuevo el contenedor, la conexión debe fluir con total normalidad tras unos segundos de forma automática.

**Verificación**:
Los logs de HAProxy al apagar y prender el contenedor deben reflejar la transición de estados:
```text
[WARNING] ... : Server pg_cluster_back/srv_pgbouncer is DOWN, reason: Layer4 connection problem, info: "Connection refused", check duration: 0ms. 0 active and 0 backup servers left. 0 sessions active.
[ALERT] ... : backend 'pg_cluster_back' has no server available!
[NOTICE] ... : Server pg_cluster_back/srv_pgbouncer is UP, reason: Instance is healthy, check duration: 3ms. 1 active and 0 backup servers left.
```

---

## Validación y Pruebas

Para garantizar que el balanceador de carga está operando bajo niveles de control estricto de ingeniería de bases de datos, aplicaremos pruebas automatizadas utilizando la interfaz administrativa de HAProxy y validando casos hostiles.

### 1. Monitoreo Automatizado de Estadísticas mediante `curl`
Podemos consultar el reporte de estado de salud en formato CSV directamente desde el backend administrativo protegido con credenciales de HAProxy utilizando `curl`:

```bash
docker run --rm --net pg_enterprise_net curlimages/curl:8.6.0 -u admin:StatsPass123! -s "http://haproxy-lb:7000/;csv" | cut -d',' -f1,2,5,18,19
```

**Métricas de Éxito**:
El comando superior procesará los datos de balanceo de HAProxy. Busca las líneas asociadas a `pg_cluster_back`. La columna `status` debe reportar de manera clara:

```text
#pxname,svname,scur,status,weight
pg_cluster_front,FRONTEND,0,OPEN,
pg_cluster_back,srv_pgbouncer,0,UP,100
pg_cluster_back,BACKEND,0,UP,100
```

### 2. Caso Adversario: Inyección de Tráfico e Intentos de bypass en Chequeos de Salud
Como caso de validación defensiva para descartar falsos positivos de salud por parte de atacantes o bloqueos por ataques de denegación de servicio (DoS):
¿Qué ocurre si se satura el puerto `6432` con conexiones inválidas que no envían el protocolo de PostgreSQL? 

Ejecutemos una simulación de ataque inundando el puerto de PgBouncer con datos no estructurados usando `nc` (Netcat):

```bash
## Envío de bytes no válidos para simular payload malicioso o corrupto
docker run --rm --net pg_enterprise_net subfuzion/netcat -w 2 pg-bouncer 6432 <<< "INYECCION_DE_PRUEBA_NO_PROTOCOLO"
```

**Resultado de Validación**:
PgBouncer y el chequeo activo `option pgsql-check` deben ignorar esta sesión espuria y continuar identificando el backend como **UP**. HAProxy no debe corromper sus estados por paquetes corruptos ni entrar en pánico. 

Confirma ejecutando de nuevo la consulta de datos en la base de datos de producción:
```bash
docker run --rm --net pg_enterprise_net postgres:16.2 psql -h haproxy-lb -p 5000 -U postgres -d enterprise_db -c "SELECT NOW();"
```
*(Debe responder exitosamente con la estampa de tiempo actual del sistema)*.

---

## Solución de Problemas

En entornos productivos, pueden presentarse escenarios de desalineación de parámetros o inconsistencias de red. Aquí se documentan dos problemas reales de alta probabilidad:

### Problema 1: Los logs de HAProxy indican "Server is DOWN, reason: Layer7 invalid response" o "Socket error durante pgsql-check"
* **Síntomas**: El contenedor HAProxy inicia correctamente, pero marca de manera indefinida a `srv_pgbouncer` como `DOWN`, incluso cuando pgbouncer se encuentra activo en el puerto `6432`.
* **Causa Raíz**: El mecanismo `option pgsql-check` de HAProxy intenta conectarse al backend enviando un paquete de inicio compatible con el usuario definido (`user postgres` en este caso). Si el usuario indicado no tiene privilegios de login, o si el archivo `userlist.txt` de PgBouncer no cuenta con la entrada del usuario enviado en la validación, el pooler rechazará la conexión inmediatamente a nivel de capa superior, invalidando la prueba de salud.
* **Solución**: 
  1. Abre el archivo `config/userlist.txt` y asegúrate de que el usuario de chequeo de salud (p. ej., `postgres`) esté explícitamente listado con sus credenciales correspondientes.
  2. Verifica que en `pgbouncer.ini` el parámetro `auth_file` apunte correctamente al volumen montado `/etc/pgbouncer/userlist.txt`.
  3. Ejecuta `docker compose restart pg-bouncer` para asegurar la recarga del archivo de usuarios en memoria.

### Problema 2: Error de tipo "bind: Address already in use" al iniciar HAProxy
* **Síntomas**: Al arrancar los contenedores mediante `docker compose up -d`, el contenedor `haproxy-lb` falla repetidamente con códigos de salida `1` o `137` y no se expone en la máquina anfitriona.
* **Causa Raíz**: Otro servicio local ejecutándose en el sistema anfitrión (como otro balanceador, servidor web local o proceso de desarrollo) está ocupando el puerto TCP `5000` o `7000` de forma exclusiva en la interfaz física del host.
* **Solución**:
  1. Identifica el proceso que está acaparando el puerto conflictivo en el host:
     ```bash
     sudo ss -tulpn | grep -E "5000|7000"
     ```
  2. Detén dicho servicio local, o modifica el mapeo de puertos de la máquina anfitriona en el archivo `docker-compose.yml` mapeando un puerto externo diferente (ej. `"5001:5000"` o `"7001:7000"`):
     ```yaml
     ports:
       - "5001:5000"  # Cambiar solo el puerto izquierdo del host
     ```

---

## Limpieza

Para liberar los recursos asignados a este laboratorio en la máquina anfitriona y garantizar un entorno limpio, ejecuta los siguientes comandos de desmantelamiento controlado:

1. Detén y remueve todos los contenedores, redes virtuales creadas y volúmenes locales utilizados:
   ```bash
   docker compose down -v
   ```

2. Remueve de manera selectiva las configuraciones físicas para evitar la acumulación de datos residuales de laboratorio:
   ```bash
   cd ~
   rm -rf ~/haproxy-lab
   ```

3. (Opcional) Remueve las imágenes de Docker descargadas en caso de requerir liberar almacenamiento total en disco:
   ```bash
   docker rmi postgres:16.2 edoburu/pgbouncer:1.22.0 haproxy:2.9.5-alpine
   ```

---

## Resumen

En este laboratorio práctico, has implementado una topología altamente escalable para el tráfico corporativo de bases de datos relacionales:

* **Arquitectura de Capas**: Diseñamos un flujo de entrada que desacopla la carga directa en el motor principal, permitiendo que la aplicación se comunique a través de **HAProxy (Puerto 5000)**, el cual deriva el flujo de manera óptima al multiplexor de conexiones **PgBouncer (Puerto 6432)**, reduciendo así la huella de memoria en el backend **PostgreSQL (Puerto 5432)**.
* **Alta Disponibilidad Práctica**: Se configuraron políticas de monitoreo activo de red en HAProxy (`option pgsql-check`). Esto asegura que, ante cualquier caída o falla del pooler, el balanceador deje de enviar tráfico de inmediato a ese nodo corrupto de manera transparente para el cliente final.
* **Monitoreo Unificado**: Se desplegó con éxito el módulo HTTP de estadísticas de HAProxy (`stats`), brindando un tablero corporativo visual para auditar el rendimiento y el comportamiento del pool de conexiones en tiempo real.

---
### Recursos Adicionales y Enlaces Oficiales
- Documentación de Referencia de HAProxy (Configuración de TCP backend): [https://docs.haproxy.org/](https://docs.haproxy.org/)
- Guía de arquitectura del protocolo PostgreSQL Frontend/Backend: [https://www.postgresql.org/docs/16/protocol.html](https://www.postgresql.org/docs/16/protocol.html)
- Repositorio oficial de PgBouncer (Configuración avanzada de timeouts): [https://www.pgbouncer.org/config.html](https://www.pgbouncer.org/config.html)

---

# 8 Diseño de una Arquitectura HA Empresarial

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
