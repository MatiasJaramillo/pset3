# Proyecto P3 – Ingesta de Datos con PySpark y Snowflake

## Integrantes
Leandro (00325644),
Esteban Silva (00329204),
Matías Jaramillo (00326063)

---

## Descripción del Proyecto

En este proyecto se implementa una pipeline básica de ingesta de datos utilizando datasets de taxis de NYC (Yellow y Green).

El objetivo es leer los datos en formato Parquet, estandarizar su esquema y cargarlos en Snowflake como capa RAW (Bronze Layer).

---

## Tecnologías Utilizadas

- Python 3.11
- Pandas
- PySpark
- Snowflake
- Docker / Docker Compose
- Jupyter Notebook

---


## Cómo Ejecutar el Proyecto

1. Levantar el entorno

docker compose up -d

2. Abrir Jupyter

http://localhost:8888

3. Ejecutar el notebook

Abrir:
notebooks/01_ingesta_parquet.ipynb

Ejecutar todas las celdas en orden.

---

## Flujo del Pipeline

1. Lectura de datos en formato Parquet desde NYC TLC
2. Estandarización de columnas entre Yellow y Green taxis
3. Creación de nuevas columnas:
   - run_id
   - source_year
   - source_month
   - service_type
   - ingested_at_utc
4. Carga de datos en Snowflake (schema RAW)
5. Validación mediante queries SQL

---

## Pruebas Realizadas

- Verificación de conexión a Snowflake
- Inserción de datos usando write_pandas
- Consultas de validación (COUNT, SELECT LIMIT)
- Prueba de conexión con PySpark hacia Snowflake

---

## Notas

- Se utilizó una muestra (head(1000)) para pruebas iniciales.
- El pipeline está preparado para escalar al dataset completo sin cambios estructurales.
