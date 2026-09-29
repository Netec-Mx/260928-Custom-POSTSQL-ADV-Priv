# 6 Diseño de Controles de Seguridad y Cumplimiento

## Metadatos

| Métrica | Detalle |
| :--- | :--- |
| **Duración** | 60 minutos |
| **Complejidad** | Alta (Hard) |
| **Nivel Bloom** | Crear |

## Descripción General

En este laboratorio, asumirás el rol de Arquitecto de Seguridad y Base de Datos (DBA Security Architect) para una entidad financiera regulada. Tu misión es diseñar, implementar y validar un entorno de base de datos PostgreSQL 16.2 altamente endurecido (hardened), alineado con los controles de las normativas internacionales **ISO 27001** (Control de Accesos y Criptografía) y **NIST SP 800-53** (Auditoría y Rendición de Cuentas, Protección de Medios y de la Información).

Para lograrlo, configurarás conexiones cifradas forzosas mediante certificados TLS/SSL digitales autofirmados con OpenSSL, instalarás y parametrizarás la extensión de auditoría detallada `pgAudit 16.0` para capturar acciones administrativas y DDLs críticas, y finalmente estructurarás políticas de control de acceso granular a nivel de fila (Row-Level Security - RLS) complementadas con cifrado de nivel de columna utilizando la extensión nativa `pgcrypto`.

## Objetivos de Aprendizaje

Al finalizar este laboratorio, serás capaz de:
- [ ] Configurar un canal seguro de comunicación extremo a extremo implementando TLS/SSL forzoso (`hostssl`) mediante certificados digitales autofirmados generados con OpenSSL.
- [ ] Instalar, cargar y parametrizar la extensión oficial `pgAudit 16.0` en PostgreSQL para el registro estricto de sentencias DDL, escrituras críticas y operaciones de control de accesos.
- [ ] Diseñar y desplegar políticas complejas de Seguridad a Nivel de Fila (Row-Level Security - RLS) para restringir el acceso a datos financieros sensibles de acuerdo al rol del usuario del motor.
- [ ] Aplicar funciones criptográficas de la extensión `pgcrypto` para asegurar la confidencialidad de columnas de identidad críticas en reposo de manera complementaria a los esquemas físicos.

## Prerrequisitos

Para completar este laboratorio de manera exitosa, requieres:
1. **Conocimientos teóricos y prácticos:**
   - Administración intermedia de PostgreSQL (modificación de archivos `postgresql.conf` y `pg_hba.conf`).
   - Conceptos de infraestructura de clave pública (PKI), certificados X.509 y cifrado asimétrico/simétrico.
   - Sintaxis SQL estándar y de control de accesos (roles, usuarios, permisos `GRANT` / `REVOKE`).
2. **Acceso y herramientas de software:**
   - Terminal de comandos con privilegios administrativos (capacidad de ejecutar comandos `sudo` o Docker).
   - Docker Engine (versión recomendada 26.0.0 o superior) y Docker Compose.
   - Cliente de base de datos SQL como `psql` (nativo en consola) o DBeaver Community Edition (v24.0.0).

## Entorno de Laboratorio

El laboratorio se construirá utilizando contenedores Docker para garantizar la reproducibilidad. Crearemos una imagen personalizada de PostgreSQL 16.2 montada sobre Debian Bookworm para incorporar las dependencias del compilador y la extensión `pgAudit`.

### Especificaciones de Software y Fuentes Oficiales

| Tecnología | Versión Exacta | Origen / Enlace Oficial |
| :--- | :--- | :--- |
| **PostgreSQL Community Edition** | 16.2 (Debian Bookworm) | [https://hub.docker.com/_/postgres](https://hub.docker.com/_/postgres) |
| **pgAudit** | 16.0 | [https://github.com/pgaudit/pgaudit](https://github.com/pgaudit/pgaudit) |
| **OpenSSL** | 3.0.11 (o superior compatible) | [https://www.openssl.org/](https://www.openssl.org/) |
| **DBeaver Community Edition** | 24.0.0 | [https://dbeaver.io/](https://dbeaver.io/) |

### Configuración de Red y Docker Global
- **Nombre de la Red Docker:** `pg_enterprise_net` (tipo Bridge).
- **Nombre del Contenedor:** `pg-primary`.
- **Base de Datos Global:** `enterprise_db`.
- **Usuario Administrador:** `postgres` con contraseña `PostgresAdminPass123!`.
- **Puerto de Enlace Local:** `5432`.

### Preparación del Directorio de Trabajo y Construcción

Ejecuta los siguientes comandos en tu máquina local o servidor de desarrollo para inicializar el laboratorio:

```bash
## 1. Crear directorios de trabajo locales
mkdir -p ~/secure_pg_lab/{config,ssl,data}
cd ~/secure_pg_lab

## 2. Crear la red Docker de tipo Bridge
docker network create pg_enterprise_net || true
```

Crea un archivo llamado `Dockerfile` dentro de `~/secure_pg_lab` para compilar o instalar directamente la versión exacta de pgAudit adaptada a la distribución de PostgreSQL utilizada:

```dockerfile
## ~/secure_pg_lab/Dockerfile
FROM postgres:16.2-bookworm

## Instalar dependencias necesarias para pgAudit y utilidades de red/seguridad
RUN apt-get update && apt-get install -y --no-install-recommends \
    postgresql-16-pgaudit \
    openssl \
    ca-certificates \
    procps \
    && rm -rf /var/lib/apt/lists/*
```

Construye la imagen personalizada utilizando la instrucción `docker build`:

```bash
docker build -t postgres-secure:16.2 .
```

---

## Instrucciones Paso a Paso

### Paso 1: Configuración de TLS/SSL con Certificados Digitales

**Objetivo:** Crear una infraestructura de clave pública (PKI) minimalista y configurar PostgreSQL para que acepte única y estrictamente conexiones cifradas utilizando autenticación TLS/SSL desde clientes externos.

#### Instrucciones

1. **Generación de los Certificados con OpenSSL:**
   Genera una Autoridad Certificadora (CA) propia, la clave privada del servidor y el certificado digital del servidor firmado por la CA. Ejecutaremos estos comandos localmente en el directorio `~/secure_pg_lab/ssl`.

```bash
cd ~/secure_pg_lab/ssl

## Generar la clave y el certificado de la CA (Autoridad Certificadora)
openssl req -new -x509 -days 365 -nodes \
  -out root.crt -keyout root.key \
  -subj "/C=CL/ST=RM/L=Santiago/O=Enterprise/CN=Enterprise-Root-CA"

## Generar la clave privada del Servidor de Base de Datos
openssl req -new -nodes \
  -out server.csr -keyout server.key \
  -subj "/C=CL/ST=RM/L=Santiago/O=Enterprise/CN=pg-primary"

## Firmar la clave del servidor con la CA generada
openssl x509 -req -in server.csr -CA root.crt -CAkey root.key \
  -CAcreateserial -out server.crt -days 365
```

2. **Ajuste Estricto de Permisos de Archivo:**
   PostgreSQL requiere por seguridad que las claves privadas no tengan permisos de lectura abiertos para el grupo o para otros usuarios.

```bash
## Cambiar propiedad y permisos a los certificados generados para evitar fallos de arranque
chmod 0600 server.key
chmod 0644 server.crt root.crt
```

3. **Creación del Archivo de Configuración de PostgreSQL (`postgresql.conf`):**
   Escribe el archivo básico de configuración en `~/secure_pg_lab/config/postgresql.conf` habilitando SSL y definiendo las rutas correctas hacia las llaves.

```ini
## ~/secure_pg_lab/config/postgresql.conf
## Parámetros de Red y Enlace
listen_addresses = '*'

## Configuración de Cifrado TLS/SSL
ssl = on
ssl_ca_file = '/etc/postgresql/ssl/root.crt'
ssl_cert_file = '/etc/postgresql/ssl/server.crt'
ssl_key_file = '/etc/postgresql/ssl/server.key'
ssl_ciphers = 'HIGH:!aNULL:!3DES' # Restringir suites de cifrado débiles
ssl_prefer_server_ciphers = on

## Configuración por defecto de almacenamiento
shared_buffers = 128MB
dynamic_shared_memory_type = posix
log_destination = 'stderr'
logging_collector = on
log_directory = 'log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_statement = 'none' # Deshabilitamos el log estándar de sentencias para dar paso a pgAudit
```

4. **Creación de la Directiva de Acceso de Red (`pg_hba.conf`):**
   Modifica las reglas de autenticación en `~/secure_pg_lab/config/pg_hba.conf` para exigir conexiones SSL cifradas (`hostssl`) a todos los clientes que accedan remotamente a la red corporativa.

```text
## ~/secure_pg_lab/config/pg_hba.conf
## TYPE  DATABASE        USER            ADDRESS                 METHOD

## Conexiones locales por socket UNIX (necesario para mantenimiento interno)
local   all             all                                     trust

## Conexiones locales TCP desde el host o contenedores locales de confianza
host    all             all             127.0.0.1/32            trust

## Conexión remota FORZOSA bajo SSL y método de autenticación moderno SCRAM-SHA-256
hostssl all             all             0.0.0.0/0               scram-sha-256
```

5. **Lanzamiento del Contenedor de PostgreSQL Seguro:**
   Inicia el contenedor Docker mapeando las carpetas de configuración, certificados SSL y persistencia de datos.

```bash
docker run -d \
  --name pg-primary \
  --network pg_enterprise_net \
  -p 5432:5432 \
  -e POSTGRES_DB=enterprise_db \
  -e POSTGRES_PASSWORD='PostgresAdminPass123!' \
  -v ~/secure_pg_lab/config/postgresql.conf:/etc/postgresql/postgresql.conf \
  -v ~/secure_pg_lab/config/pg_hba.conf:/etc/postgresql/pg_hba.conf \
  -v ~/secure_pg_lab/ssl:/etc/postgresql/ssl \
  -v ~/secure_pg_lab/data:/var/lib/postgresql/data \
  postgres-secure:16.2 \
  -c config_file=/etc/postgresql/postgresql.conf \
  -c hba_file=/etc/postgresql/pg_hba.conf
```

Ajusta los permisos de los archivos de certificados *dentro* del contenedor para que pertenezcan al usuario `postgres`:

```bash
docker exec -u 0 -it pg-primary chown -R postgres:postgres /etc/postgresql/ssl
```

Reinicia el contenedor para asegurar que PostgreSQL lea la configuración con los privilegios e identidades correctas sobre las llaves:

```bash
docker restart pg-primary
```

#### Resultado Esperado
El contenedor debe iniciar correctamente y la base de datos `enterprise_db` debe estar activa con el puerto 5432 expuesto. El log interno de arranque de PostgreSQL no debe presentar excepciones de permisos sobre `server.key`.

#### Verificación
Ejecuta el comando de consulta de estados de conexión SSL localmente utilizando el cliente `psql` integrado en la imagen:

```bash
docker exec -it pg-primary psql -U postgres -d enterprise_db -c "SELECT ssl, version, cipher FROM pg_stat_ssl WHERE pid = pg_backend_pid();"
```

*Salida Esperada:*
```text
 ssl | version |             cipher             
-----+---------+--------------------------------
 f   |         | 
(1 row)
```
*(Nota: Devuelve 'f' o nulo debido a que el comando interactivo ejecutado directamente en el contenedor utiliza el socket de dominio de UNIX local bypassando la interfaz TCP. Esta es la condición esperada para conexiones locales de socket UNIX).*

Ahora verifica que el acceso externo sin SSL sea explícitamente denegado. Intenta conectarte simulando un cliente externo (usando la interfaz de loopback TCP `127.0.0.1` que es evaluada por las reglas de IP del HBA):

```bash
docker exec -it pg-primary psql -h 127.0.0.1 -U postgres -d enterprise_db -c "SELECT 'Conexión Exitosa';"
```
*(Ingresa la contraseña `PostgresAdminPass123!` si te es solicitada. Debería conectar automáticamente con SSL activo).*

---

### Paso 2: Compilación y Activación de pgAudit para Auditoría Estricta

**Objetivo:** Configurar e implementar la auditoría selectiva de sentencias DDL y escrituras críticas utilizando la extensión oficial pgAudit versión 16.0 para cumplir con los lineamientos de rendición de cuentas (NIST SP 800-53 AU-2).

#### Instrucciones

1. **Modificación de la configuración de PostgreSQL para cargar pgAudit:**
   Abre el archivo `~/secure_pg_lab/config/postgresql.conf` y añade las directivas de pre-carga de librerías dinámicas y parámetros detallados de pgAudit al final de este:

```ini
## Añadir al final de ~/secure_pg_lab/config/postgresql.conf

## Carga de pgAudit en Memoria Compartida
shared_preload_libraries = 'pgaudit'

## Parametrización de Auditoría de pgAudit
pgaudit.log = 'write, ddl, role'       # Registrar escrituras (INSERT, UPDATE, DELETE), DDLs y cambios de roles
pgaudit.log_relation = on              # Detallar las tablas afectadas en las consultas auditadas
pgaudit.log_catalog = off              # No saturar logs con consultas internas del catálogo de sistema
pgaudit.log_parameter = on             # Registrar los parámetros en consultas parametrizadas
pgaudit.log_client = off               # No retornar los logs de auditoría al cliente (evitar fugas de información)
```

2. **Aplicar la configuración reiniciando el servicio:**
   Dado que `shared_preload_libraries` requiere una inicialización de la memoria compartida del motor, debemos reiniciar por completo el contenedor Docker.

```bash
docker restart pg-primary
```

3. **Creación de la Extensión en la Base de Datos:**
   Conéctate a la base de datos corporativa y crea formalmente el objeto de la extensión utilizando `psql`:

```bash
docker exec -it pg-primary psql -U postgres -d enterprise_db -c "CREATE EXTENSION pgaudit;"
```

#### Resultado Esperado
La creación de la extensión debe reportar un mensaje de confirmación exitosa en la salida de consola.

*Salida Esperada:*
```text
CREATE EXTENSION
```

#### Verificación
Comprueba que la extensión se encuentra correctamente registrada y cargada en el catálogo interno del motor:

```bash
docker exec -it pg-primary psql -U postgres -d enterprise_db -c "SELECT extname, extversion FROM pg_extension WHERE extname = 'pgaudit';"
```

*Salida Esperada:*
```text
 extname | extversion 
---------+------------
 pgaudit | 16.0
(1 row)
```

---

### Paso 3: Diseño de Políticas de Row-Level Security (RLS)

**Objetivo:** Configurar políticas de control de accesos lógicos y físicos basadas en Row-Level Security (RLS) para segregar el acceso de lectura y escritura sobre tablas críticas de nóminas y sueldos financieros alineado a ISO 27001 (A.9.4 Control de acceso a las aplicaciones e información).

#### Instrucciones

1. **Creación del Esquema de Base de Datos y Usuarios:**
   Crea una tabla sensible llamada `nomina_empleados` con información financiera, e introduce tres perfiles/roles diferenciados dentro del motor de base de datos.

```sql
-- Ejecutar estas instrucciones como superusuario en enterprise_db
CREATE TABLE IF NOT EXISTS public.nomina_empleados (
    id SERIAL PRIMARY KEY,
    empleado_nombre VARCHAR(100) NOT NULL,
    departamento VARCHAR(50) NOT NULL,
    salario_mensual NUMERIC(12, 2) NOT NULL,
    db_user VARCHAR(50) NOT NULL
);

-- Inserción de registros iniciales de prueba
INSERT INTO public.nomina_empleados (empleado_nombre, departamento, salario_mensual, db_user) VALUES
('Carlos Mendoza', 'Ventas', 1800.00, 'usr_ventas'),
('Ana Silva', 'Ventas', 2100.00, 'usr_ventas_senior'),
('Roberto Gómez', 'Recursos Humanos', 3500.00, 'usr_rrhh'),
('Patricia Torres', 'Tecnología', 4200.00, 'usr_it');
```

2. **Creación de Roles sin privilegios de superusuario:**
   Genera los usuarios correspondientes a cada perfil en el motor y concédeles permisos de selección y manipulación sobre la tabla:

```sql
-- Creación de usuarios con contraseñas seguras
CREATE ROLE usr_ventas WITH LOGIN PASSWORD 'VentasPassSecure99!';
CREATE ROLE usr_rrhh WITH LOGIN PASSWORD 'RrhhPassSecure99!';
CREATE ROLE usr_it WITH LOGIN PASSWORD 'ItPassSecure99!';

-- Otorgar privilegios de uso básicos
GRANT SELECT, INSERT, UPDATE, DELETE ON public.nomina_empleados TO usr_ventas;
GRANT SELECT, INSERT, UPDATE, DELETE ON public.nomina_empleados TO usr_rrhh;
GRANT SELECT, INSERT, UPDATE, DELETE ON public.nomina_empleados TO usr_it;
```

3. **Habilitación de Row-Level Security (RLS):**
   Activa RLS de forma explícita en la tabla. De forma predeterminada, esto bloquea todas las lecturas de los usuarios que no sean dueños (owner) de la tabla o superusuarios.

```sql
ALTER TABLE public.nomina_empleados ENABLE ROW LEVEL SECURITY;
```

4. **Definición de Políticas de Acceso Granulares:**
   - **Regla 1 (Recursos Humanos):** El usuario perteneciente al rol `usr_rrhh` tiene visibilidad completa sobre todos los registros de la organización sin importar su departamento.
   - **Regla 2 (Filtro por Usuario):** Los usuarios genéricos como `usr_ventas` o `usr_it` solo pueden leer y manipular sus propios registros correspondientes a su nombre de sesión asignada en el campo `db_user`.

```sql
-- Política para permitir que Recursos Humanos visualice todos los datos
CREATE POLICY policy_rrhh_full_access 
ON public.nomina_empleados
AS PERMISSIVE
FOR ALL
TO usr_rrhh
USING (true)
WITH CHECK (true);

-- Política restrictiva para usuarios generales basada en su identidad de sesión de base de datos
CREATE POLICY policy_personal_access 
ON public.nomina_empleados
AS PERMISSIVE
FOR ALL
TO PUBLIC
USING (CURRENT_USER = db_user)
WITH CHECK (CURRENT_USER = db_user);
```

#### Resultado Esperado
La tabla `nomina_empleados` tiene ahora habilitada la seguridad a nivel de filas con dos políticas activas que evalúan la función del motor `CURRENT_USER`.

#### Verificación
Ejecuta sentencias simulando un cambio de contexto de usuario dentro de la sesión actual de la consola SQL:

```sql
-- Cambiar el contexto de sesión al rol usr_ventas
SET ROLE usr_ventas;

-- Consultar toda la tabla de nómina
SELECT * FROM public.nomina_empleados;
```

*Salida Esperada:*
Solo debe retornar el registro asociado al usuario `usr_ventas` de forma automática:
```text
 id | empleado_nombre | departamento | salario_mensual |  db_user   
----+-----------------+--------------+-----------------+------------
  1 | Carlos Mendoza  | Ventas       |         1800.00 | usr_ventas
(1 row)
```

Prueba ahora con el rol de Recursos Humanos:

```sql
-- Cambiar el contexto de sesión al rol usr_rrhh
SET ROLE usr_rrhh;

-- Consultar toda la tabla de nómina
SELECT * FROM public.nomina_empleados;
```

*Salida Esperada:*
Debe retornar la totalidad de las filas:
```text
 id | empleado_nombre |   departamento   | salario_mensual |  db_user   
----+-----------------+------------------+-----------------+------------
  1 | Carlos Mendoza  | Ventas           |         1800.00 | usr_ventas
  2 | Ana Silva       | Ventas           |         2100.00 | usr_ventas_senior
  3 | Roberto Gómez   | Recursos Humanos |         3500.00 | usr_rrhh
  4 | Patricia Torres | Tecnología       |         4200.00 | usr_it
(4 rows)
```

Regresa el control de sesión al superusuario para las operaciones siguientes:
```sql
RESET ROLE;
```

---

### Paso 4: Cifrado de Datos a Nivel de Columna con pgcrypto

**Objetivo:** Implementar la extensión criptográfica `pgcrypto` para proteger datos de alta sensibilidad (por ejemplo, identificaciones tributarias o RUT de empleados) cifrándolos de forma simétrica antes de almacenarse en los bloques físicos del disco, reforzando el control de defensa en profundidad.

#### Instrucciones

1. **Creación e instalación de pgcrypto:**
   Habilita la extensión dentro del espacio de nombres de la base de datos:

```bash
docker exec -it pg-primary psql -U postgres -d enterprise_db -c "CREATE EXTENSION IF NOT EXISTS pgcrypto;"
```

2. **Alterar la Tabla para Incorporar la Columna de Identidad Cifrada:**
   Dado que los datos cifrados con funciones simétricas retornan matrices de bytes, agregaremos la columna con tipo de dato `BYTEA` (binary array).

```sql
-- Ejecutar como superusuario
ALTER TABLE public.nomina_empleados ADD COLUMN identificacion_cifrada BYTEA;
```

3. **Cifrado de los Datos Existentes:**
   Utiliza la función `pgp_sym_encrypt` para almacenar el valor real cifrado con un algoritmo robusto (por ejemplo, `aes256`). Utilizaremos un hash secreto ficticio corporativo `KeyCorporateShield2026!`.

```sql
-- Cifrar la información de los empleados existentes
UPDATE public.nomina_empleados 
SET identificacion_cifrada = pgp_sym_encrypt('11.111.111-1', 'KeyCorporateShield2026!', 'cipher-algo=aes256') 
WHERE id = 1;

UPDATE public.nomina_empleados 
SET identificacion_cifrada = pgp_sym_encrypt('22.222.222-2', 'KeyCorporateShield2026!', 'cipher-algo=aes256') 
WHERE id = 2;

UPDATE public.nomina_empleados 
SET identificacion_cifrada = pgp_sym_encrypt('33.333.333-3', 'KeyCorporateShield2026!', 'cipher-algo=aes256') 
WHERE id = 3;

UPDATE public.nomina_empleados 
SET identificacion_cifrada = pgp_sym_encrypt('44.444.444-4', 'KeyCorporateShield2026!', 'cipher-algo=aes256') 
WHERE id = 4;
```

4. **Recuperación Segura de los Datos:**
   Diseña una consulta estructurada que permita descifrar en caliente la columna solo si se tiene la contraseña corporativa correspondiente.

```sql
-- Consultar datos descifrados en texto plano
SELECT 
    id, 
    empleado_nombre, 
    pgp_sym_decrypt(identificacion_cifrada, 'KeyCorporateShield2026!') AS identificacion_real
FROM public.nomina_empleados;
```

#### Resultado Esperado
Los datos deben mostrarse en texto plano únicamente si se proporciona el secreto correcto. En caso contrario, el motor arrojará un error de descifrado y bloqueará el resultado para proteger la integridad de la fila.

#### Verificación
Para validar que el almacenamiento en el disco es seguro y completamente ilegible para atacantes con acceso al dump físico, ejecuta una consulta simple del campo binario directo sin descifrar:

```sql
SELECT empleado_nombre, identificacion_cifrada FROM public.nomina_empleados WHERE id = 1;
```

*Salida Esperada (Valores binarios cifrados en formato hexadecimal):*
```text
 empleado_nombre |                              identificacion_cifrada                               
-----------------+-----------------------------------------------------------------------------------
 Carlos Mendoza  | \xc30d040703029fbf9727dfec96cdd4303f019f945763ee51e9be1e2850ff0f37072e5057bd63...
(1 row)
```

---

## Validación y Pruebas

Para garantizar que el diseño de controles satisface estrictamente los estándares de cumplimiento exigidos en auditorías reales, realizaremos una batería de validaciones agresivas y pruebas de penetración lógica.

### 1. Validación de Exclusión de Canales inseguros (No-TLS)
Desde el exterior del sistema operativo del contenedor, usaremos una herramienta como `curl` o la librería básica de conexión TCP (usando `nc` o psql local) apuntando al puerto expuesto de PostgreSQL para validar que se rechazan intentos de comunicación en texto claro desde fuera de la red local.

```bash
## Simular un inicio de conexión no cifrada desde la terminal física del host hacia el contenedor
docker run --net=host --rm postgres:16.2-bookworm psql "postgresql://postgres:PostgresAdminPass123!@127.0.0.1:5432/enterprise_db?sslmode=disable"
```

*Resultado Esperado:*
El motor de base de datos debe rechazar la conexión de manera contundente debido a las restricciones de `pg_hba.conf`:
```text
psql: error: connection to server at "127.0.0.1", port 5432 failed: FATAL: no pg_hba.conf entry for host "127.0.0.1", user "postgres", database "enterprise_db", no SSL
```

---

### 2. Validación de pgAudit (Evidencia de Auditoría)
Realizaremos una operación DDL de alta sensibilidad bajo el contexto de un rol y posteriormente validaremos en los registros físicos de salida (logs de Docker) la huella y la trazabilidad del evento según el estándar NIST SP 800-53 AU-3.

```sql
-- Ejecutar como superusuario: simular intento de borrado o alteración de la tabla financiera
DROP TABLE IF EXISTS public.nomina_empleados_historica;
CREATE TABLE public.nomina_empleados_historica (id INT);
DROP TABLE public.nomina_empleados_historica;
```

Ahora, extrae las líneas de log del contenedor Docker para capturar la auditoría estructurada generada por pgAudit:

```bash
docker logs pg-primary 2>&1 | grep "AUDIT" | tail -n 5
```

*Salida Esperada (Campos estructurados de pgAudit):*
```text
2026-03-31 15:42:10.123 UTC [12345] LOG:  AUDIT: SESSION,3,1,DDL,DROP TABLE,,,DROP TABLE IF EXISTS public.nomina_empleados_historica;,<not logged>
2026-03-31 15:42:10.125 UTC [12345] LOG:  AUDIT: SESSION,4,1,DDL,CREATE TABLE,TABLE,public.nomina_empleados_historica,CREATE TABLE public.nomina_empleados_historica (id INT);,<not logged>
2026-03-31 15:42:10.130 UTC [12345] LOG:  AUDIT: SESSION,5,1,DDL,DROP TABLE,TABLE,public.nomina_empleados_historica,DROP TABLE public.nomina_empleados_historica;,<not logged>
```

*(Nota: Los campos del CSV de pgAudit estructurado corresponden a: `AUDIT: TYPE, ENTRY_ID, SUB_ENTRY_ID, CLASS, COMMAND, OBJECT_TYPE, OBJECT_NAME, STATEMENT, PARAMETERS`).*

---

### 3. Prueba Adversaria: Intento de evasión de RLS (SQL Injection / Escalación de Privilegios)
Simularemos un escenario donde un desarrollador malicioso con acceso a la cuenta `usr_ventas` intenta inyectar código o ejecutar funciones definidas con los privilegios de su creador (`SECURITY DEFINER`) para bypassear la regla lógica de RLS y robar los salarios de otros departamentos.

Creamos una función maliciosa como superusuario que corre con permisos de seguridad elevados (`SECURITY DEFINER` de postgres):

```sql
-- Ejecutar como superusuario
CREATE OR REPLACE FUNCTION public.funcion_vulneradora() 
RETURNS TABLE(nombre VARCHAR, salario NUMERIC) 
SECURITY DEFINER
AS $$
BEGIN
    -- Esta función se ejecuta bajo el contexto del dueño de la función (postgres), lo que omite RLS si no está configurada para ignorarlo
    RETURN QUERY SELECT empleado_nombre, salario_mensual FROM public.nomina_empleados;
END;
$$ LANGUAGE plpgsql;

-- Otorgamos acceso de ejecución al usuario común usr_ventas
GRANT EXECUTE ON FUNCTION public.funcion_vulneradora() TO usr_ventas;
```

Ahora, iniciamos sesión simulando al atacante `usr_ventas` e intentamos evadir RLS:

```sql
SET ROLE usr_ventas;

-- Escenario A: Consulta directa de la tabla con RLS activo
SELECT empleado_nombre, salario_mensual FROM public.nomina_empleados;
```
*(Resultado A: Retorna únicamente el registro propio por RLS)*

```sql
-- Escenario B: Consulta a través del vector de bypass (función SECURITY DEFINER del administrador)
SELECT * FROM public.funcion_vulneradora();
```

*Resultado Esperado de la Mitigación:*
Dado que la función se definió como `SECURITY DEFINER` por el superusuario y este no tiene RLS aplicada por defecto, el usuario con menor privilegio logra sortear la regla de aislamiento.

**Medida de Remediación Obligatoria:**
Para mitigar esta vulnerabilidad, el arquitecto de seguridad debe forzar a que las políticas de RLS apliquen inclusive sobre el usuario propietario de la tabla utilizando la directiva `FORCE ROW LEVEL SECURITY`.

```sql
-- Regresar a superusuario
RESET ROLE;

-- Forzar la aplicación de políticas RLS al propietario de la tabla y a contextos security definer vinculados
ALTER TABLE public.nomina_empleados FORCE ROW LEVEL SECURITY;
```

Vuelve a probar el intento de ataque:
```sql
SET ROLE usr_ventas;
SELECT * FROM public.funcion_vulneradora();
```

*Salida Esperada post-Remediación (Aislamiento Total):*
```text
     nombre     | salario 
----------------+---------
 Carlos Mendoza | 1800.00
(1 row)
```
El ataque ha sido mitigado exitosamente. Las políticas de seguridad a nivel de fila se aplican ahora de forma contundente en todos los niveles de ejecución lógica.

---

## Solución de Problemas

A continuación, se listan dos fallos de configuración comunes identificados durante el despliegue de estos controles avanzados de seguridad, con sus causas raíz y resoluciones.

### Problema 1: El contenedor Docker falla al iniciar y muestra el error "FATAL: private key file 'server.key' has group or world access" en los logs

- **Síntoma:** El contenedor de PostgreSQL entra en bucle de reinicio infinito o se apaga inmediatamente tras el arranque. Al inspeccionar con `docker logs pg-primary` se lee la traza descrita.
- **Causa Raíz:** El archivo `server.key` generado en el directorio compartido (`~/secure_pg_lab/ssl`) tiene permisos excesivos (usualmente `0644` o `0777` heredados del host de desarrollo), lo cual es rechazado estrictamente por el motor de PostgreSQL por violar principios básicos de seguridad física.
- **Solución:** Ajustar los permisos del archivo directamente en el host para limitar el acceso exclusivamente al usuario dueño. Ejecuta:
  ```bash
  chmod 0600 ~/secure_pg_lab/ssl/server.key
  docker restart pg-primary
  ```

### Problema 2: Error "ERROR: pgAudit not loaded via shared_preload_libraries" al ejecutar `CREATE EXTENSION pgaudit;`

- **Síntoma:** La sentencia de base de datos falla reportando que el motor de base de datos no tiene pre-cargada la extensión en memoria compartida.
- **Causa Raíz:** Se omitió agregar la instrucción `shared_preload_libraries = 'pgaudit'` en el archivo de configuración activo, o bien, no se reinició la instancia física del contenedor tras editar el archivo `postgresql.conf`.
- **Solución:**
  1. Abre `~/secure_pg_lab/config/postgresql.conf`.
  2. Confirma que la directiva `shared_preload_libraries = 'pgaudit'` esté presente y descomentada.
  3. Reinicia por completo el contenedor:
     ```bash
     docker restart pg-primary
     ```
  4. Vuelve a ejecutar la sentencia de creación en psql.

---

## Limpieza

Para evitar el consumo persistente de recursos de almacenamiento y memoria ram dentro del entorno de desarrollo, sigue estos pasos para apagar y remover la infraestructura montada:

```bash
## 1. Detener y eliminar el contenedor de base de datos seguro
docker stop pg-primary
docker rm pg-primary

## 2. Remover la red de prueba creada para el laboratorio
docker network rm pg_enterprise_net

## 3. Eliminar los directorios temporales de configuración y datos generados
## ADVERTENCIA: Asegúrate de estar en la ruta correcta antes de ejecutar
rm -rf ~/secure_pg_lab
```

---

## Resumen

En este laboratorio, has implementado controles robustos de seguridad a nivel de infraestructura, motor de base de datos y diseño lógico, alineados con estándares internacionales de ciberseguridad:

1. **Cifrado de Capa de Transporte (NIST SP 800-53 SC-8 / ISO 27001 A.10):** A través del uso estricto de OpenSSL y la directiva `hostssl` en `pg_hba.conf`, cerraste la puerta a ataques de intercepción en tránsito (Man-in-the-Middle) forzando el cifrado criptográfico robusto en todas las conexiones remotas.
2. **Auditoría Detallada (NIST SP 800-53 AU-2 / ISO 27001 A.12.4):** Implementando la extensión de grado empresarial `pgAudit`, estableciste una pista de auditoría inmutable e inalterable que registra con precisión quirúrgica el qué, cuándo y quién de cada operación DDL y de modificación sobre la data regulada.
3. **Control de Acceso de Mínimo Privilegio (ISO 27001 A.9.4):** Empleando Row-Level Security (RLS) y forzando su aplicación incluso sobre el propietario de la tabla con `FORCE ROW LEVEL SECURITY`, garantizaste que los datos confidenciales de nómina queden segregados lógicamente de manera automática a nivel de motor.
4. **Cifrado en Profundidad (ISO 27001 A.18.1.5):** Reforzaste el resguardo de la identidad corporativa mediante cifrado simétrico en disco (`pgcrypto`), protegiendo el dato sensible contra robos físicos de volcados de disco.

### Recursos Adicionales
- [Documentación oficial de PostgreSQL 16 sobre Seguridad](https://www.postgresql.org/docs/16/security.html)
- [Guía de Implementación y Parametrización de pgAudit](https://github.com/pgaudit/pgaudit)
- [Estándar NIST SP 800-53: Security and Privacy Controls](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)
