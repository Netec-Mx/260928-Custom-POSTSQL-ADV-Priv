<p align="center">
  <img src="https://raw.githubusercontent.com/Netec-Mx/260928-Custom-POSTSQL-ADV-Priv/main/assets/LogoNetec.png" alt="NETEC" width="180" />
</p>

# PostgreSQL Avanzado MS

El curso Administración Avanzada de PostgreSQL, Alta Disponibilidad, Seguridad y Automatización proporciona a los participantes los conocimientos y habilidades necesarias para administrar, optimizar, proteger y operar plataformas PostgreSQL en entornos empresariales de misión crítica. A lo largo del curso se profundiza en la arquitectura interna del motor, estrategias avanzadas de administración y mantenimiento, optimización del rendimiento, análisis y ajuste de consultas, programación avanzada mediante PL/pgSQL, mecanismos de alta disponibilidad y recuperación ante desastres, así como técnicas de monitoreo, auditoría y seguridad orientadas al cumplimiento normativo. Adicionalmente, se incorporan temas especializados como particionamiento, extensiones avanzadas, integración de fuentes externas, diseño de arquitecturas analíticas, Data Warehouse, automatización de despliegues y prácticas DevOps para la gestión controlada de cambios. Mediante laboratorios prácticos y casos de estudio, los participantes implementarán soluciones completas utilizando herramientas del ecosistema PostgreSQL como pgBackRest, Barman, Patroni, PgBouncer, HAProxy, pgBadger, Liquibase y Flyway, permitiéndoles diseñar e implementar plataformas robustas, escalables, seguras y altamente disponibles para soportar cargas transaccionales, analíticas y procesos críticos de negocio.

## Accesos rápidos

- [**Setup Guide del curso**](https://github.com/Netec-Mx/260928-Custom-POSTSQL-ADV-Priv/blob/main/SETUP_GUIDE.md)
- [Laboratorios por capítulo](#lista-de-laboratorios)

## Estructura

- `SETUP_GUIDE.md`: guía de instalación y preparación del entorno.
- `CapituloXX/README.md`: guía de laboratorio por capítulo.

## Lista de laboratorios

### Capítulo 1

- [5 Diseño de una Base de Datos Empresarial](Capitulo01/README.md#5-diseño-de-una-base-de-datos-empresarial)
  - Descripción: Diseñar e implementar una base de datos relacional aplicando principios de modelado y normalización para utilizarla durante las prácticas del curso.
  - Duración estimada: 45 min
- [6 Grandes Volúmenes de Información](Capitulo01/README.md#6-grandes-volúmenes-de-información)
  - Descripción: Generar e incorporar grandes volúmenes de información para analizar el comportamiento de PostgreSQL durante operaciones de consulta, mantenimiento y administración.
  - Duración estimada: 30 min
  - [Ver capítulo completo](Capitulo01/README.md)

### Capítulo 2

- [6 Administración de Respaldos con Barman](Capitulo02/README.md#6-administración-de-respaldos-con-barman)
  - Descripción: Implementar Barman para administrar respaldos de PostgreSQL, realizando operaciones de respaldo, validación y restauración.
  - Duración estimada: 60 min
  - [Ver capítulo completo](Capitulo02/README.md)

### Capítulo 3

- [6 Implmentación de PgBouncer](Capitulo03/README.md#6-implmentación-de-pgbouncer)
  - Descripción: Configurar PgBouncer para administrar conexiones a PostgreSQL, evaluando su efecto sobre el número de conexiones concurrentes.
  - Duración estimada: 60 min
- [7 Configuración de HAProxy](Capitulo03/README.md#7-configuración-de-haproxy)
  - Descripción: Configurar HAProxy para distribuir las conexiones hacia los servidores PostgreSQL disponibles, verificando el enrutamiento del tráfico entre nodos.
  - Duración estimada: 60 min
- [8 Diseño de una Arquitectura HA Empresarial](Capitulo03/README.md#8-diseño-de-una-arquitectura-ha-empresarial)
  - Descripción: Diseñar una arquitectura de alta disponibilidad para PostgreSQL integrando los componentes de replicación, balanceo de carga y administración de conexiones estudiados durante el capítulo.
  - Duración estimada: 50 min
  - [Ver capítulo completo](Capitulo03/README.md)

### Capítulo 4

- [6 Diseño de Controles de Seguridad y Cumplimiento](Capitulo04/README.md#6-diseño-de-controles-de-seguridad-y-cumplimiento)
  - Descripción: Relacionar los mecanismos de seguridad de PostgreSQL con los controles de acceso, auditoría y protección de datos definidos por los estándares ISO 27001 y el marco de ciberseguridad NIST.
  - Duración estimada: 60 min
  - [Ver capítulo completo](Capitulo04/README.md)

### Capítulo 5

- [4 Diseño de un Data Warehouse en PostgreSQL](Capitulo05/README.md#4-diseño-de-un-data-warehouse-en-postgresql)
  - Descripción: Diseñar e implementar un modelo básico de Data Warehouse utilizando PostgreSQL, organizando la información mediante tablas de hechos y dimensiones para realizar consultas analíticas.
  - Duración estimada: 60 min
- [5 Optimización de Consultas Analíticas e Indicadores](Capitulo05/README.md#5-optimización-de-consultas-analíticas-e-indicadores)
  - Descripción: Desarrollar consultas analíticas para generar indicadores y reportes históricos utilizando grandes volúmenes de información.
  - Duración estimada: 60 min
  - [Ver capítulo completo](Capitulo05/README.md)

### Capítulo 6

- [1 Versionamiento de Objetos PostgreSQL con Git](Capitulo06/README.md#1-versionamiento-de-objetos-postgresql-con-git)
  - Descripción: Utilizar Git para administrar el versionamiento de scripts SQL, funciones, procedimientos y otros objetos de PostgreSQL, identificando los cambios realizados sobre el esquema de la base de datos.
  - Duración estimada: 60 min
- [2 Automatización de Migraciones con Liquibase](Capitulo06/README.md#2-automatización-de-migraciones-con-liquibase)
  - Descripción: Implementar migraciones de esquemas mediante Liquibase, ejecutando cambios controlados sobre una base de datos PostgreSQL y verificando el registro de las modificaciones realizadas.
  - Duración estimada: 90 min
- [3 Gestión de Migraciones con Flyway](Capitulo06/README.md#3-gestión-de-migraciones-con-flyway)
  - Descripción: Administrar la evolución del esquema de una base de datos utilizando Flyway, aplicando migraciones versionadas y verificando el estado de los cambios ejecutados.
  - Duración estimada: 90 min
- [4 Diseño de una Tubería CI/CD para PostgreSQL](Capitulo06/README.md#4-diseño-de-una-tubería-cicd-para-postgresql)
  - Descripción: Diseñar un flujo básico de integración y despliegue continuo para administrar cambios sobre una base de datos PostgreSQL utilizando herramientas de control de versiones y migración.
  - Duración estimada: 90 min
- [5 Simulación de Despliegue Controlado entre DEV, QA y PROD](Capitulo06/README.md#5-simulación-de-despliegue-controlado-entre-dev-qa-y-prod)
  - Descripción: Aplicar un proceso de promoción de cambios entre los ambientes de Desarrollo (DEV), Pruebas (QA) y Producción (PROD), verificando la secuencia de despliegue de modificaciones sobre una base de datos PostgreSQL.
  - Duración estimada: 90 min
  - [Ver capítulo completo](Capitulo06/README.md)

## Flujo de colaboración

- Trabajar en `changes_course`.
- Crear Pull Request hacia `main`.
- Merge por `Squash and merge`.

---

*Material didáctico preparado por Global K, S.A. de C.V.*
