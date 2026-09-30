# 4.7 Configuración de HAProxy

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