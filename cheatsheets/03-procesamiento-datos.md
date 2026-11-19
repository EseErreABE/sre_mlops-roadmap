# 03 · Procesamiento de datos: Pandas/Polars, Spark, Kafka, dbt

> Naturaleza: **datos en movimiento y transformación** (batch y streaming). Alimentan entrenamiento, features e inferencia.
> [← Índice](00-indice-roadmap.md) · Anterior: [02](02-frameworks-ml.md) · Siguiente: [04 Contenedores](04-contenedores-kubernetes.md)

| Herramienta | Tipo | Volumen típico | Cuándo elegirla |
|---|---|---|---|
| Pandas | DataFrame en memoria | < pocos GB | Exploración, prototipos, ETL pequeño |
| Polars | DataFrame columnar, lazy | hasta decenas de GB en 1 máquina | ETL rápido en un nodo |
| Spark | Cómputo distribuido | GB–PB | Datos que no caben en un nodo, joins masivos |
| Kafka | Log de eventos distribuido | Streams continuos | Eventos en tiempo real, desacoplar productores/consumidores |
| dbt | Transformación SQL en el warehouse | Lo que soporte el warehouse | Modelado analítico versionado y testeado |

---

## Pandas / Polars

**Para qué sirve:** manipular datos tabulares en Python.

**Escenario de uso:** EDA, limpieza, feature engineering pequeño, preparación de payloads, reportería.

**Comandos típicos**
```python
import pandas as pd
df = pd.read_parquet("s3://bucket/data.parquet")          # requiere s3fs
df.info(); df.describe(); df.isna().mean()
df = (df.query("monto > 0")
        .assign(mes=lambda d: d.fecha.dt.to_period("M"))
        .groupby(["cliente_id", "mes"], as_index=False)
        .agg(total=("monto", "sum"), n=("monto", "size")))
df.merge(clientes, on="cliente_id", how="left", validate="m:1")
df.to_parquet("out.parquet", index=False)
```
```python
import polars as pl
out = (pl.scan_parquet("data/*.parquet")                  # lazy: optimiza el plan
         .filter(pl.col("monto") > 0)
         .group_by("cliente_id")
         .agg(pl.col("monto").sum().alias("total"))
         .collect())
```

**Variantes:** Pandas 2 con backend PyArrow, Polars, DuckDB (SQL embebido sobre Parquet), Modin/Dask (Pandas distribuido).

**Requisitos:** Python; formatos (CSV vs Parquet), tipos de datos, vectorización (evitar `apply` fila a fila), uso de memoria.

**Conexión:** Pandera/Great Expectations ([09](09-feature-store-calidad.md)), sklearn ([02](02-frameworks-ml.md)), Lambdas/Functions con datos pequeños ([14](14-cloud-aws-azure-gcp.md)/[15](15-cloud-oci.md)).

**Aprende más:** pandas.pydata.org/docs/user_guide · docs.pola.rs · *Python for Data Analysis* (W. McKinney, gratis online) · duckdb.org/docs.

---

## Apache Spark (PySpark)

**Para qué sirve:** procesamiento distribuido batch y streaming; SQL, ML (MLlib) y DataFrames a gran escala.

**Escenario de uso:** ETL de CDRs/llamadas, agregaciones históricas, feature pipelines masivos, scoring batch de millones de registros.

**Comandos típicos**
```python
from pyspark.sql import SparkSession, functions as F, Window
spark = SparkSession.builder.appName("features").getOrCreate()
df = spark.read.parquet("s3a://bucket/llamadas/")
w = Window.partitionBy("cliente_id").orderBy(F.col("fecha").desc())
feat = (df.filter(F.col("fecha") >= F.date_sub(F.current_date(), 30))
          .withColumn("rn", F.row_number().over(w))
          .groupBy("cliente_id").agg(F.count("*").alias("n"), F.avg("duracion").alias("dur")))
feat.write.mode("overwrite").partitionBy("anio_mes").parquet("s3a://bucket/features/")
feat.explain()                     # plan físico
df.repartition(200, "cliente_id"); df.cache(); F.broadcast(dim_pequena)
```
```bash
spark-submit --master yarn --deploy-mode cluster --num-executors 10 \
  --executor-memory 8g --conf spark.sql.shuffle.partitions=400 job.py
```

**Variantes:** Spark en EMR / Glue (AWS), OCI Data Flow (serverless), Databricks, Dataproc (GCP), Spark Structured Streaming, Delta Lake / Iceberg / Hudi (formatos de tabla).

**Requisitos:** cluster o servicio gestionado, Java runtime; particionamiento, shuffles, skew, lazy evaluation, Spark UI para diagnóstico.

**Conexión:** Airflow (`SparkSubmitOperator`, Glue/EMR/Data Flow operators) ([07](07-orquestacion-pipelines.md)), Feast offline ([09](09-feature-store-calidad.md)), MLflow (`mlflow.pyfunc.spark_udf` para scoring batch) ([08](08-versionado-tracking.md)), Kafka como fuente de streaming.

**Aprende más:** spark.apache.org/docs/latest · *Spark: The Definitive Guide* · *Learning Spark 2ª ed.* (gratis de Databricks) · docs OCI Data Flow.

---

## Apache Kafka

**Para qué sirve:** plataforma de streaming de eventos: publicar/suscribir, persistir y reprocesar flujos.

**Escenario de uso:** inferencia en tiempo real (evento → modelo → acción), captura de predicciones para monitoreo, CDC de bases de datos, desacoplar microservicios.

**Comandos típicos**
```bash
kafka-topics.sh --bootstrap-server $BROKER --create --topic predicciones --partitions 6 --replication-factor 3
kafka-topics.sh --bootstrap-server $BROKER --list
kafka-console-producer.sh --bootstrap-server $BROKER --topic eventos
kafka-console-consumer.sh --bootstrap-server $BROKER --topic eventos --from-beginning --group dbg
kafka-consumer-groups.sh --bootstrap-server $BROKER --describe --group scoring   # ver LAG
```
```python
from confluent_kafka import Consumer, Producer
c = Consumer({"bootstrap.servers": BROKER, "group.id": "scoring", "auto.offset.reset": "earliest",
              "enable.auto.commit": False})
c.subscribe(["eventos"])
while True:
    msg = c.poll(1.0)
    if msg is None or msg.error(): continue
    pred = model.predict(parse(msg.value()))
    producer.produce("predicciones", value=serialize(pred)); c.commit(msg)
```

**Variantes:** Apache Kafka, Confluent, AWS MSK, **OCI Streaming (API compatible con Kafka)**, Redpanda; alternativas: Kinesis, Pub/Sub, Event Hubs. Ecosistema: Kafka Connect, Schema Registry (Avro/Protobuf), Kafka Streams, Flink.

**Requisitos:** cluster o servicio gestionado; topics, particiones, offsets, consumer groups, garantías de entrega (at-least-once / exactly-once), esquemas.

**Conexión:** Spark Structured Streaming / Flink (procesamiento), servicios de [Serving](10-serving.md) como consumidores, KEDA escala por lag ([04](04-contenedores-kubernetes.md)), métricas en Prometheus ([11](11-monitoreo-observabilidad.md)).

**Aprende más:** kafka.apache.org/documentation · developer.confluent.io (cursos gratuitos) · *Kafka: The Definitive Guide*.

---

## dbt (data build tool)

**Para qué sirve:** transformar datos *dentro* del warehouse con SQL versionado, modular, testeado y documentado.

**Escenario de uso:** capas staging → intermedio → marts; tablas de features en SQL; linaje y documentación de datos.

**Comandos típicos**
```bash
dbt init proyecto
dbt debug                       # valida conexión (profiles.yml)
dbt run --select staging+       # ejecuta modelos y dependientes
dbt test                        # tests de datos (unique, not_null, relationships)
dbt build                       # run + test + seeds + snapshots
dbt docs generate && dbt docs serve
```
```sql
-- models/marts/fct_llamadas.sql
{{ config(materialized='incremental', unique_key='llamada_id') }}
select * from {{ ref('stg_llamadas') }}
{% if is_incremental() %} where fecha > (select max(fecha) from {{ this }}) {% endif %}
```
```yaml
# models/marts/schema.yml
models:
  - name: fct_llamadas
    columns:
      - name: llamada_id
        tests: [unique, not_null]
```

**Variantes:** dbt Core (CLI, open source) / dbt Cloud; adaptadores: Redshift, Athena, Postgres, Snowflake, BigQuery, Databricks, Spark, Oracle.

**Requisitos:** warehouse soportado; SQL sólido, Jinja básico, modelado dimensional, Git.

**Conexión:** Airflow/Dagster (Cosmos, `dagster-dbt`) ([07](07-orquestacion-pipelines.md)), Feast offline store ([09](09-feature-store-calidad.md)), CI con `dbt build --select state:modified+` ([05](05-ci-cd.md)).

**Aprende más:** docs.getdbt.com · learn.getdbt.com (cursos gratis) · *Analytics Engineering with SQL and dbt*.

---

## Hilando soluciones
**Scoring en tiempo real:** sistema origen → Kafka/OCI Streaming (`eventos`) → servicio FastAPI/consumer en K8s (escala con KEDA por lag) → modelo de MLflow → topic `predicciones` → Spark diario agrega predicciones + reales → Evidently calcula drift.
