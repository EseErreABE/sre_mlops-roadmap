# 09 · Feature store y calidad de datos: Feast, Great Expectations, Deepchecks, Pandera

> Naturaleza: **confiabilidad de los datos que entran al modelo**. Garantizan features consistentes entre entrenamiento e inferencia y datos válidos antes de usarlos.
> [← Índice](00-indice-roadmap.md) · Anterior: [08](08-versionado-tracking.md) · Siguiente: [10 Serving](10-serving.md)

| Herramienta | Rol | Momento |
|---|---|---|
| Feast | Feature store (offline + online) | Entrenamiento (históricos point-in-time) e inferencia (baja latencia) |
| Great Expectations | Validación de datos declarativa con reportes | Pipelines ETL/batch, quality gates |
| Pandera | Validación de esquemas de DataFrames en código | Funciones Python, tests, entrada de APIs batch |
| Deepchecks | Suites de validación de datos *y modelos* | Antes de entrenar, tras entrenar, en monitoreo |

---

## Feast

**Para qué sirve:** definir features una vez y servirlas de forma consistente: *offline store* (warehouse/Parquet) para entrenamiento con joins point-in-time y *online store* (Redis/DynamoDB) para inferencia en milisegundos.

**Escenario de uso:** la API de scoring necesita "nº llamadas últimos 30 días" del cliente en tiempo real, calculado igual que en el entrenamiento (evitar *training-serving skew*).

**Comandos típicos**
```bash
feast init repo_features && cd repo_features/feature_repo
feast apply                                            # registra definiciones
feast materialize-incremental $(date -u +%Y-%m-%dT%H:%M:%S)   # offline -> online
feast feature-views list
feast serve                                            # servidor HTTP de features
```
```python
from feast import Entity, FeatureView, Field, FileSource, FeatureStore
from feast.types import Float32, Int64
from datetime import timedelta

cliente = Entity(name="cliente", join_keys=["cliente_id"])
src = FileSource(path="s3://features/llamadas.parquet", timestamp_field="event_ts")
fv = FeatureView(name="llamadas_30d", entities=[cliente], ttl=timedelta(days=2),
                 schema=[Field(name="n_llamadas", dtype=Int64), Field(name="dur_prom", dtype=Float32)],
                 source=src)

store = FeatureStore(repo_path=".")
train_df = store.get_historical_features(entity_df=eventos_df,
            features=["llamadas_30d:n_llamadas", "llamadas_30d:dur_prom"]).to_df()
online = store.get_online_features(features=["llamadas_30d:n_llamadas"],
            entity_rows=[{"cliente_id": 123}]).to_dict()
```

**Variantes:** Feast (OSS), SageMaker Feature Store, Vertex AI Feature Store, Databricks Feature Store, Tecton, Hopsworks. En OCI: patrón Feast con Autonomous DB/Object Storage + Redis (OCI Cache) u OCI Data Science Feature Store.

**Requisitos:** offline store (S3/Parquet, Redshift, Snowflake, BigQuery) + online store (Redis, DynamoDB); entender entidades, TTL, point-in-time join, materialización.

**Conexión:** Spark/dbt generan las tablas fuente ([03](03-procesamiento-datos.md)), Airflow materializa ([07](07-orquestacion-pipelines.md)), API de serving consulta online ([10](10-serving.md)), Terraform crea Redis/DynamoDB ([06](06-iac.md)).

**Aprende más:** docs.feast.dev · featurestore.org · artículo "What is a feature store" (Tecton blog).

---

## Great Expectations (GX)

**Para qué sirve:** definir *expectations* sobre datos (no nulos, rangos, unicidad, distribución) y validarlas generando reportes HTML (Data Docs).

**Escenario de uso:** quality gate en el DAG: si la carga diaria trae 40% de nulos en `monto`, se detiene antes de entrenar o scorear.

**Comandos típicos**
```python
import great_expectations as gx
ctx = gx.get_context()
ds = ctx.data_sources.add_pandas("pandas")
batch = ds.add_dataframe_asset("llamadas").add_batch_definition_whole_dataframe("b").get_batch(
            batch_parameters={"dataframe": df})
suite = ctx.suites.add(gx.ExpectationSuite(name="llamadas"))
suite.add_expectation(gx.expectations.ExpectColumnValuesToNotBeNull(column="cliente_id"))
suite.add_expectation(gx.expectations.ExpectColumnValuesToBeBetween(column="duracion", min_value=0, max_value=36000))
suite.add_expectation(gx.expectations.ExpectTableRowCountToBeBetween(min_value=10_000))
result = batch.validate(suite)
assert result.success, "Datos inválidos"
```

**Variantes:** GX Core (OSS), GX Cloud; integración Airflow (`GreatExpectationsOperator`), Spark/SQL datasources. Alternativas: Soda, dbt tests, AWS Glue Data Quality.

**Requisitos:** Python; conocer el dominio para definir expectativas útiles; versión de API (cambió fuerte en 1.x).

**Conexión:** Airflow/Dagster ([07](07-orquestacion-pipelines.md)), CI ([05](05-ci-cd.md)), reportes publicados en S3/Object Storage, alertas ([11](11-monitoreo-observabilidad.md)).

**Aprende más:** docs.greatexpectations.io.

---

## Pandera

**Para qué sirve:** validar esquemas de DataFrames (tipos, rangos, checks custom) directamente en el código Python, con sintaxis tipo Pydantic.

**Escenario de uso:** validar input/output de funciones de feature engineering, tests unitarios de datos, payloads batch.

**Comandos típicos**
```python
import pandera.pandas as pa
from pandera.typing import Series, DataFrame

class Llamadas(pa.DataFrameModel):
    cliente_id: Series[int] = pa.Field(ge=1)
    duracion: Series[float] = pa.Field(ge=0, le=36000)
    canal: Series[str] = pa.Field(isin=["voz", "chat", "app"])
    class Config: strict = True; coerce = True

@pa.check_types
def features(df: DataFrame[Llamadas]) -> DataFrame: ...

Llamadas.validate(df, lazy=True)    # reporta todos los errores juntos
```

**Variantes:** soporte Pandas, Polars, PySpark; Pydantic para validar payloads JSON de APIs.

**Requisitos:** Python/Pandas, tipado.

**Conexión:** pytest ([05](05-ci-cd.md)), FastAPI + Pydantic en serving ([10](10-serving.md)), Pandas/Polars ([03](03-procesamiento-datos.md)).

**Aprende más:** pandera.readthedocs.io.

---

## Deepchecks

**Para qué sirve:** suites listas de chequeos sobre datos y modelos: integridad, train-test drift, leakage, performance por segmento.

**Escenario de uso:** antes de promover un modelo, correr la suite y adjuntar el reporte al PR o al registro del modelo.

**Comandos típicos**
```python
from deepchecks.tabular import Dataset
from deepchecks.tabular.suites import data_integrity, train_test_validation, model_evaluation
tr = Dataset(train_df, label="churn", cat_features=["canal"])
te = Dataset(test_df,  label="churn", cat_features=["canal"])
r = train_test_validation().run(tr, te); r.save_as_html("validacion.html")
model_evaluation().run(tr, te, model).passed()
```

**Variantes:** módulos tabular, NLP, visión; alternativas: Evidently (ver [11](11-monitoreo-observabilidad.md)), whylogs.

**Requisitos:** Python, conceptos de drift, leakage, evaluación por segmentos.

**Conexión:** CI quality gate ([05](05-ci-cd.md)), MLflow `log_artifact` del reporte ([08](08-versionado-tracking.md)).

**Aprende más:** docs.deepchecks.com.

---

## Hilando soluciones
**Consistencia train/serve:** dbt/Spark construye features → Great Expectations valida la tabla → Feast registra la FeatureView → entrenamiento usa `get_historical_features` (sin leakage) → Deepchecks valida train vs test → Airflow materializa a Redis → FastAPI llama `get_online_features` y valida con Pydantic → mismo cálculo en ambos lados.
