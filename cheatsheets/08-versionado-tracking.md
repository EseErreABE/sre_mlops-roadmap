# 08 · Versionado y tracking: MLflow, DVC, Weights & Biases, Model Registry

> Naturaleza: **trazabilidad y reproducibilidad**. Responden: ¿qué datos, código, parámetros y métricas produjeron este modelo, y cuál está en producción?
> [← Índice](00-indice-roadmap.md) · Anterior: [07](07-orquestacion-pipelines.md) · Siguiente: [09 Feature store](09-feature-store-calidad.md)

| Qué se versiona | Herramienta |
|---|---|
| Código | Git ([05](05-ci-cd.md)) |
| Datos y artefactos grandes | DVC, versionado de buckets (S3/Object Storage), Delta/Iceberg |
| Experimentos (params, métricas, artefactos) | MLflow Tracking, W&B |
| Modelos y su estado (staging/prod) | MLflow Model Registry, SageMaker Model Registry, OCI Data Science Model Catalog, W&B Registry |

---

## MLflow

**Para qué sirve:** plataforma open source para tracking de experimentos, empaquetado estándar de modelos (`pyfunc`), registry y serving; también trazas de LLM/GenAI.

**Escenario de uso:** comparar corridas de entrenamiento, registrar el modelo campeón, cargar "el modelo de producción" por alias desde la API de inferencia.

**Comandos típicos**
```bash
mlflow server --backend-store-uri postgresql://... --default-artifact-root s3://mlflow-artifacts --host 0.0.0.0
mlflow ui                                            # local
export MLFLOW_TRACKING_URI=http://mlflow:5000
mlflow models serve -m "models:/churn@champion" -p 5001 --env-manager local
mlflow models build-docker -m "models:/churn@champion" -n churn-api
```
```python
import mlflow
mlflow.set_experiment("churn")
mlflow.sklearn.autolog()                       # params, métricas y modelo automáticos
with mlflow.start_run(run_name="lgbm-v3") as run:
    mlflow.log_params({"num_leaves": 63}); mlflow.log_metric("auc_val", 0.81)
    mlflow.log_artifact("reports/shap.png")
    mlflow.sklearn.log_model(pipe, name="model", registered_model_name="churn",
                             input_example=X_te[:5])       # firma de entrada/salida

from mlflow import MlflowClient
c = MlflowClient()
c.set_registered_model_alias("churn", "champion", version=7)    # aliases reemplazan stages
model = mlflow.pyfunc.load_model("models:/churn@champion")
mlflow.search_runs(experiment_names=["churn"], order_by=["metrics.auc_val DESC"]).head()

# Scoring batch distribuido
udf = mlflow.pyfunc.spark_udf(spark, "models:/churn@champion")
```

**Variantes:** MLflow OSS self-hosted, Databricks MLflow, SageMaker managed MLflow, flavors (sklearn, xgboost, pytorch, transformers, langchain), MLflow Tracing/Evaluate para LLM.

**Requisitos:** servidor con DB (Postgres/MySQL) + almacenamiento de artefactos (S3/Object Storage); conceptos de experimento/run/artifact/model signature, aliases y tags.

**Conexión:** frameworks ML ([02](02-frameworks-ml.md)), Spark UDF ([03](03-procesamiento-datos.md)), quality gate en CI ([05](05-ci-cd.md)), orquestadores registran corridas ([07](07-orquestacion-pipelines.md)), serving carga por alias o genera imagen ([10](10-serving.md)), evaluación de agentes LLM ([13](13-optimizacion-llm-agentes.md)).

**Aprende más:** mlflow.org/docs/latest · tutoriales "MLflow Model Registry" · *Practical MLOps* (O'Reilly).

---

## DVC (Data Version Control)

**Para qué sirve:** versionar datasets y modelos grandes junto a Git (Git guarda punteros `.dvc`, los datos viven en S3/Object Storage), y definir pipelines reproducibles.

**Escenario de uso:** reproducir exactamente el dataset de entrenamiento de un modelo de hace 3 meses; pipelines `dvc repro` que sólo re-ejecutan etapas cambiadas.

**Comandos típicos**
```bash
dvc init
dvc remote add -d storage s3://dvc-store/proyecto     # OCI: endpoint S3-compatible de Object Storage
dvc add data/raw/llamadas.parquet && git add data/raw/llamadas.parquet.dvc .gitignore
dvc push; dvc pull
git checkout v1.2 && dvc checkout                      # volver a los datos de esa versión
dvc stage add -n train -d src/train.py -d data/feat -o models/model.pkl -M metrics.json python src/train.py
dvc repro; dvc dag
dvc exp run -S train.lr=0.01; dvc exp show
dvc metrics diff main
```

**Variantes:** DVC + CML (reportes en PR), lakeFS (Git sobre data lakes), versionado nativo de buckets, Delta Lake time travel.

**Requisitos:** Git, almacenamiento remoto; entender hashing de contenido y separación código/datos.

**Conexión:** Git y CI ([05](05-ci-cd.md)), MLflow (DVC para datos, MLflow para experimentos), S3/Object Storage ([14](14-cloud-aws-azure-gcp.md)/[15](15-cloud-oci.md)).

**Aprende más:** dvc.org/doc/start · learn.iterative.ai.

---

## Weights & Biases (W&B)

**Para qué sirve:** SaaS de tracking de experimentos con dashboards ricos, sweeps de hiperparámetros, artefactos, registry y evaluación de LLM (Weave).

**Escenario de uso:** deep learning con muchas corridas y curvas de entrenamiento, tuning colaborativo, comparar prompts/agents.

**Comandos típicos**
```bash
wandb login
wandb sweep sweep.yaml && wandb agent <entity/project/sweep_id>
```
```python
import wandb
run = wandb.init(project="stt-finetune", config={"lr": 3e-4})
wandb.log({"loss": loss, "wer": wer})
art = wandb.Artifact("modelo", type="model"); art.add_file("model.pt"); run.log_artifact(art)
run.finish()
```

**Variantes:** W&B SaaS / dedicado / self-managed; alternativas: Neptune, Comet, ClearML, TensorBoard.

**Requisitos:** cuenta/licencia, revisar política de datos corporativa (SaaS externo).

**Conexión:** PyTorch/Transformers callbacks ([02](02-frameworks-ml.md)), evaluación de LLM ([13](13-optimizacion-llm-agentes.md)).

**Aprende más:** docs.wandb.ai · cursos gratis en wandb.ai/courses.

---

## Model Registry (concepto)

Catálogo central de modelos con: versión, linaje (run, datos, commit), firma, métricas, estado/alias (`champion`, `challenger`), aprobaciones y quién lo promovió.

```bash
# AWS SageMaker Model Registry
aws sagemaker list-model-packages --model-package-group-name churn
aws sagemaker update-model-package --model-package-arn <arn> --model-approval-status Approved
# OCI Data Science Model Catalog
oci data-science model list --compartment-id $C --project-id $P
```
```python
# OCI con ADS
from ads.model.generic_model import GenericModel
m = GenericModel(estimator=pipe, artifact_dir="art"); m.prepare(inference_conda_env="generalml_p311_cpu_x86_64_v1")
m.save(display_name="churn-v7"); m.deploy(display_name="churn-api")
```

**Requisitos de conocimiento:** gobierno del ciclo de vida, criterios de promoción, rollback a versión anterior, auditoría.

## Hilando soluciones
**Reproducibilidad end-to-end:** commit Git `abc123` + `dvc.lock` (hash de datos) + run MLflow (params/métricas, tag `git_sha`) → modelo `churn v7` alias `champion` → imagen `api:abc123` en ECR/OCIR → deployment en K8s con label `model_version=7` → los logs de predicción incluyen `model_version` → cualquier predicción es trazable a sus datos y código.
