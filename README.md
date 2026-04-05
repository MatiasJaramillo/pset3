# Proyecto P3 – Ingesta de Datos con PySpark y Snowflake

## Integrantes
Leandro (00325644),
Esteban Silva (00329204),
Matías Jaramillo (00326063)

---

## 1. Descripción del Proyecto

En este proyecto se implementa un pipeline de ingesta masiva y enriquecimiento de datos de taxis de NYC (Yellow y Green) abarcando registros desde 2015 hasta 2025. 
El flujo automatiza la descarga de archivos Parquet, los estandariza localmente con PySpark, los ingesta en una capa RAW en Snowflake, implementa validaciones de calidad de datos (Data Quality) y finalmente construye un modelo analítico robusto llamado **"One Big Table" (OBT)** preparado para responder instantáneamente interrogantes de negocio mediante Spark SQL.

---

## 2. Tecnologías Utilizadas

- **Python 3.11**
- **Pandas / PySpark (Spark 3.4)**
- **Snowflake** (Snowpark / Snowflake JDBC Connectors)
- **Docker / Docker Compose** (Despliegue del ecosistema)
- **Jupyter Notebook** (Entorno de desarrollo y Data Analysis)

---

## 3. Guía de Ejecución y Docker Compose

### Requisitos Previos: Archivo `.env`
Antes de ejecutar el contenedor, debes crear un archivo `.env` en la raíz del proyecto. Este archivo contiene las credenciales críticas para conectar PySpark con Snowflake de forma segura:

```env
SNOWFLAKE_USER=tu_usuario
SNOWFLAKE_PASSWORD=tu_contraseña
SNOWFLAKE_ACCOUNT=tu_cuenta_id
SNOWFLAKE_WAREHOUSE=COMPUTE_WH
SNOWFLAKE_DATABASE=NYC_TAXI_DB
SNOWFLAKE_SCHEMA_RAW=RAW
SNOWFLAKE_SCHEMA_ANALYTICS=ANALYTICS
SNOWFLAKE_ROLE=SYSADMIN
```
* **Propósito:** Evitar hardcodear contraseñas en el código, centralizando los apuntadores para que `01_ingesta` guarde la data de inicio en el schema `RAW` y los notebooks posteriores construyan sobre el esquema final de `ANALYTICS`.

### Pasos de Ejecución
1. Levantar el entorno de Jupyter con PySpark configurado en los contenedores:
```bash
docker compose up -d
```
2. Acceder al servidor de Jupyter abriendo en el navegador: `http://localhost:8888`
3. **Orden estricto de ejecución de Notebooks:**
   * **Paso 1:** Ejecutar `01_ingesta_parquet_raw.ipynb` (Descarga masiva y subida a RAW).
   * **Paso 2:** Ejecutar `02_enriquecimiento_y_unificacion.ipynb` (Cruces de lookup zones y armado del STAGE).
   * **Paso 3:** Ejecutar `03_construccion_obt.ipynb` (Filtros duros, limpiezas y carga final en OBT).
   * **Paso 4:** Navegar `04_validaciones_y_exploracion.ipynb` y `05_data_analysis.ipynb` para revisar auditorías, correlaciones y conclusiones de negocio.

---

## 4. Matriz de Cobertura de Datos (2015–2025)

El pipeline está diseñado para iterar dinámicamente en este rango analítico. La disponibilidad del organismo gubernamental (TLC) dicta los resultados de descarga:

| Año | Meses | Estado Servicio YELLOW | Estado Servicio GREEN | Observaciones |
|---|---|---|---|---|
| **2015 – 2023** | 1 al 12 | ✅ OK | ✅ OK | Ingesta histórica completa asimilada satisfactoriamente. |
| **2024** | 1 al 12 | ✅ OK | ✅ OK | Ingesta completada con éxito. |
| **2025** | 1 (Ene) | ✅ OK | ✅ OK | Requiere y efectúa manejo de *Data Drift* procesado con éxito. |
| **2025** | 2 al 12 | ⚠️ Fallido / Faltante | ⚠️ Fallido / Faltante | Arroja `HTTP 403 Forbidden` ya que la TLC aún no los ha hecho públicos. El bloque *Try-Except* del pipeline sortea este error sin crashear. |

---

## 5. Arquitectura del Sistema (ETL -> ELT)

```mermaid
graph TD;
    subgraph Fuentes de Datos Externas
      TLC_S3("AWS Cloudfront: NYC TLC Parquet")
      TAXI_ZONES("Taxi Zones (CSV)")
    end

    subgraph Plataforma Local (Jupyter / PySpark)
      NB01("01_ingesta_parquet_raw.ipynb<br>[Extract API & Cast]")
      NB02("02_enriquecimiento_y_unificacion.ipynb<br>[Join Dimensional]")
      NB03("03_construccion_obt.ipynb<br>[Reglas Matemáticas de Negocio]")
      NB04("04_validaciones_y_exploracion.ipynb<br>[Auditoría Data Quality]")
      NB05("05_data_analysis.ipynb<br>[Análisis Empresarial Spark SQL]")
    end

    subgraph Snowflake Cloud Data Warehouse
      subgraph Schema BASE_RAW
        STG_YEL[("YELLOW_TRIPS_RAW")]
        STG_GRN[("GREEN_TRIPS_RAW")]
      end
      
      subgraph Schema ANALYTICS 
        ENRICHED[("TRIPS_ENRICHED_UNIFIED_STAGE")]
        OBT[("OBT_TRIPS")]
      end
    end

    %% Flujo 01
    TLC_S3 -->|.parquet| NB01
    NB01 -->|Idempotente INSERT| STG_YEL
    NB01 -->|Idempotente INSERT| STG_GRN

    %% Flujo 02
    STG_YEL -.->|Lectura Spark| NB02
    STG_GRN -.->|Lectura Spark| NB02
    TAXI_ZONES -.->|Lectura Spark| NB02
    NB02 -->|Escritura Unificada| ENRICHED

    %% Flujo 03
    ENRICHED -.->|Lectura SQL| NB03
    NB03 -->|SQL Clean (Filtros)| OBT

    %% Flujo 04 y 05
    OBT -.->|Validación Masiva| NB04
    OBT -.->|PySpark Pushdown| NB05
```

---

## 6. Diseño Estructural: Tablas RAW y OBT_TRIPS

### Base RAW (`YELLOW_TRIPS_RAW` / `GREEN_TRIPS_RAW`)
* **Columnas Nativas Directas:** `VendorID`, `tpep_pickup_datetime`, `tpep_dropoff_datetime`, `passenger_count`, `trip_distance`, `PULocationID`, `DOLocationID`, `fare_amount`, `tip_amount`, `total_amount`, etc.
* **Metadatos Inyectados (Trazabilidad):** 
  * `run_id` (UUID único por lote), `ingested_at_utc` (Timestamp del insert), `source_year` / `source_month` (Extraídos de la URL original para control idempotente).

### Tabla Consolidada OBT (`OBT_TRIPS`)
La estructura definitiva (One Big Table) agrupa la información en subcategorías técnicas claras:

* **Tiempo:** `pickup_datetime TIMESTAMP`, `dropoff_datetime TIMESTAMP`, `pickup_date DATE`, `pickup_hour INTEGER`, `dropoff_date DATE`, `dropoff_hour INTEGER`, `day_of_week INTEGER`, `month INTEGER`, `year INTEGER`.
* **Ubicacion:** `pu_location_id INTEGER`, `pu_zone STRING`, `pu_borough STRING`, `do_location_id INTEGER`, `do_zone STRING`, `do_borough STRING`.
* **Servicio y códigos:** `service_type STRING`, `vendor_id INTEGER`, `vendor_name STRING`, `rate_code_id INTEGER`, `rate_code_desc STRING`, `payment_type INTEGER`, `payment_type_desc STRING`, `trip_type INTEGER`.
* **Viajes:** `passenger_count FLOAT`, `trip_distance FLOAT`, `store_and_fwd_flag STRING`.
* **Tarifas:** `fare_amount FLOAT`, `extra FLOAT`, `mta_tax FLOAT`, `tip_amount FLOAT`, `tolls_amount FLOAT`, `improvement_surcharge FLOAT`, `congestion_surcharge FLOAT`, `airport_fee FLOAT`, `ehail_fee FLOAT`, `total_amount FLOAT`.
* **Derivadas:** `trip_duration_min FLOAT`, `avg_speed_mph FLOAT`, `tip_pct FLOAT`.
* **Calidad:** `run_id STRING`, `ingested_at_utc TIMESTAMP`, `source_year INTEGER`, `source_month INTEGER`.

* **Supuestos Requeridos (COALESCE):** Los pasajeros `passenger_count` nulos se asumen bajo el umbral `0`. El `Payment_Type` nulo cambia al código TLC `5` (Desconocido). Zonas e Ids nulos se marcan forzosamente como `'Unknown'` o explícito `99` correspondientemente para robustecer la estadística.

---

## 7. Calidad de Datos (Data Quality y Auditoría)

El proyecto asume una estricta filosofía de limpieza matemática.

### Prevención y Filtros Duros (Ejecutados numéricamente en `03_construccion_obt.ipynb`)
Mediante cláusulas SQL, se introdujeron 8 pilares de validación progresiva inyectando restricciones lógicas para descartar Data Cruda imposible:
* **Validacion 0:** El año y mes geolocalizados coinciden estructuralmente (`source_year = {year}`).
* **Validacion 1:** Fechas existen (no nulas) impidiendo records huérfanos (`pickup_datetime IS NOT NULL...`).
* **Validacion 2:** Sincronización temporal validada donde el `EXTRACT(YEAR/MONTH...)` de la recogida coincide estrictamente con las carpetas descargadas.
* **Validacion 3:** Distancia del viaje de taxímetro debe ser positiva (`trip_distance > 0`).
* **Validacion 4:** "Cargos sin deudas", impidiendo cancelaciones que cobren negativo en la tarifa de inicio (`fare_amount >= 0`, `total_amount >= 0`).
* **Validacion 5:** Duración de viaje estrictamente positiva (`> 0`) eliminando viajes instantáneos erróneos que corrompen medias aritméticas.
* **Validacion 6:** Control de atípicos máximos paramétricos (`passenger_count BETWEEN 0 AND 9`).
* **Validacion 7:** Exclusión categórica del ruido por GPS en distancias imposibles o saltos espacio-temporales (`avg_speed_mph <= 150 mph`).

### Auditoría Estadística Activa (Monitoreo en `04_validaciones_y_exploracion.ipynb`)
* Se emplean DataFrames matriciales que interrogan **absolutamente todos los registros** integrados (+800 millones) en Snowflake descartando típicas muestras estáticas limitadas a 100 mil filas de Pandas.
* Comprueba analíticamente la efectividad integral de las 8 validaciones de la OBT analizando ceros perdidos y verificando conteos transaccionales absolutos.


---

## 8. Hallazgos y Conclusiones de Negocio (05_data_analysis.ipynb)

El análisis intensivo de los más de 800 millones de registros a través del particionamiento distribuido con PySpark evidenció múltiples tendencias estructurales del ecosistema de transporte neoyorquino:

* **Densidad Geográfica:** Existe un clásico monopolio de los viajes *Yellow* sobre el Centro Comercial y núcleo financiero (Manhattan Sur, Aeropuerto JFK). En contraste directo, los Taxis *Green* capitalizan exitosamente todo el sector norte y los *Boroughs* periféricos, probando una repartición logística del territorio que no fomenta superposición desleal.
* **Correlación de Precios y Pagos:** El efectivo (*Cash*) presenta estadísticamente un porcentaje de propina declarada marginal (tendiendo al 0% formal). Quienes usan tarjeta de crédito inyectan propinas considerables (tip_pct) que robustecen sustancialmente el ingreso del conductor mensual.
* **Estacionalidad y Eventualidad (YoY):** Se documentó formalmente el impacto avasallador de la pandemia COVID-19 con un vacío crítico en la serie temporal 2020. Posterior a esto, el volumen mensual de viajes presenta una franca pero lenta recuperación; sin embargo, el costo fijo del ticket (*Total Amount*) ha experimentado inflación año tras año para amortizar costos de operación.
* **Paradigma del Tráfico y Reloj:** Se determinaron colapsos absolutos de la fluidez del tránsito (la *Avg Speed Mph* decae radicalmente por debajo de un peatón) entre las 17:00 y las 20:00 (Rush-hour nocturno urbano). Los picos de volumen dominan netamente el espectro post-oficina. Las tarifas de Congestión en estas zonas sobrecargan fuertemente al pasajero pero apenas rentabilizan el viaje base del chofer.
