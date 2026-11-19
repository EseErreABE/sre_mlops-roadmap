# 07 · Orquestación de pipelines: Airflow, Kubeflow Pipelines, Argo Workflows, Prefect, Dagster

> Naturaleza: **coordinación de flujos de trabajo**. Definen *qué* corre, *en qué orden*, *cuándo* y *qué pasa si falla* (ETL, entrenamiento, reentrenamiento, scoring batch).
> [← Índice](00-indice-roadmap.md) · Anterior: [06](06-iac.md) · Siguiente: [08 Versionado](08-versionado-tracking.md)

| Herramienta | Modelo | Corre sobre | Ideal para |
|---|---|---|---|
| Airflow | DAGs en Python, scheduler | VM, K8s, MWAA (AWS), Composer (GCP) | ETL + ML batch, orquestar servicios cloud |
| Kubeflow Pipelines | DAG de contenedores (SDK `kfp`) | Kubernetes | Pipelines ML con GPU, Vertex Pipelines |
| Argo Workflows | CRD YAML, cada paso un pod | Kubernetes | Workflows cloud-native, CI pesado, fan-out masivo |
| Prefect | Flows Python dinámicos | Cualquier lugar (workers) | Flujos Python con poca ceremonia |
| Dagster | Assets de datos declarativos | Cualquier lugar | Linaje de datos/modelos, dbt + ML |

Equivalentes gestionados: SageMaker Pipelines, Step Functions (AWS), **OCI Data Science Pipelines, OCI Data Integration** (OCI), Vertex AI Pipelines, Azure ML Pipelines.

---

## Apache Airflow

**Para qué sirve:** programar y monitorear workflows definidos como DAGs en Python.

**Escenario de uso:** ETL diario → features → scoring batch → carga de resultados; reentrenamiento mensual; disparar Glue/EMR/SageMaker/Lambda (repo `airflow_demo_etl`).

**Comandos típicos**
```bash
airflow standalone                                   # local rápido
airflow dags list; airflow dags list-import-errors
airflow dags test etl_scoring 2026-11-19             # ejecutar un DAG completo sin scheduler
airflow tasks test etl_scoring extraer 2026-11-19
airflow dags trigger etl_scoring --conf '{"mes":"2026-10"}'
airflow dags backfill etl_scoring -s 2026-10-01 -e 2026-10-31
airflow connections add aws_default --conn-type aws
airflow variables set umbral_auc 0.75
```
```python
from airflow.decorators import dag, task
from airflow.providers.amazon.aws.operators.sagemaker import SageMakerTrainingOperator
from pendulum import datetime

@dag(schedule="0 6 * * *", start_date=datetime(2026, 1, 1), catchup=False,
     default_args={"retries": 2, "retry_delay": 300}, tags=["ml"])
def etl_scoring():
    @task
    def extraer(ds=None) -> str: return f"s3://raw/{ds}/"
    @task
    def validar(path: str) -> str:
        # Great Expectations / Pandera; lanzar excepción si falla
        return path
    @task.branch
    def hay_drift(path: str): return "reentrenar" if detectar(path) else "scoring"
    @task
    def scoring(path: str): ...
    p = validar(extraer())
    hay_drift(p) >> [scoring(p)]
etl_scoring()
```

**Variantes:** Airflow 2.x/3.x, MWAA (AWS), Cloud Composer (GCP), Astronomer; executors: Local, Celery, Kubernetes (`KubernetesPodOperator` para aislar dependencias).

**Requisitos:** Python, base de datos de metadata, scheduler/webserver/workers; idempotencia de tareas, `ds`/data intervals, XCom (sólo metadatos, no datos), sensores, reintentos, SLAs.

**Conexión:** operators para Spark/Glue/EMR/OCI Data Flow ([03](03-procesamiento-datos.md)), dbt vía Cosmos, SageMaker/OCI Data Science ([14](14-cloud-aws-azure-gcp.md)/[15](15-cloud-oci.md)), MLflow ([08](08-versionado-tracking.md)), Great Expectations ([09](09-feature-store-calidad.md)), alertas a Teams/Slack y métricas StatsD → Prometheus ([11](11-monitoreo-observabilidad.md)).

**Aprende más:** airflow.apache.org/docs · Astronomer Academy (gratis) · *Data Pipelines with Apache Airflow* (Manning).

---

## Kubeflow Pipelines (KFP)

**Para qué sirve:** pipelines de ML donde cada paso es un contenedor en Kubernetes, con tracking de artefactos y caché.

**Escenario de uso:** entrenamiento con GPU, tuning (Katib), pipelines portables a Vertex AI Pipelines.

**Comandos típicos**
```python
from kfp import dsl, compiler

@dsl.component(base_image="python:3.12", packages_to_install=["scikit-learn"])
def entrenar(data: dsl.Input[dsl.Dataset], model: dsl.Output[dsl.Model], c: float = 1.0): ...

@dsl.pipeline(name="churn")
def pipeline(c: float = 1.0):
    t = entrenar(data=prep_op().outputs["out"], c=c)
    t.set_accelerator_type("nvidia.com/gpu").set_accelerator_limit(1)

compiler.Compiler().compile(pipeline, "churn.yaml")
```
```bash
kfp run create --experiment-name churn --package-file churn.yaml
```

**Variantes:** Kubeflow completo (Notebooks, Katib, Training Operator, KServe), Vertex AI Pipelines (KFP gestionado).

**Requisitos:** cluster K8s con Kubeflow; contenedores, artefactos tipados, K8s básico.

**Conexión:** KServe para servir ([10](10-serving.md)), MLflow/registry ([08](08-versionado-tracking.md)), CI compila y sube pipelines ([05](05-ci-cd.md)).

**Aprende más:** kubeflow.org/docs/components/pipelines.

---

## Argo Workflows

**Para qué sirve:** motor de workflows nativo de Kubernetes (CRDs), cada paso un pod; DAGs y fan-out masivos.

**Comandos típicos**
```bash
argo submit -n argo wf.yaml --watch -p dataset=2026-11
argo list -n argo; argo logs @latest; argo retry <wf>
argo cron create cron-wf.yaml
```
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata: { generateName: scoring- }
spec:
  entrypoint: main
  templates:
    - name: main
      dag:
        tasks:
          - { name: prep,  template: py, arguments: { parameters: [{ name: cmd, value: prep.py }] } }
          - { name: score, template: py, dependencies: [prep], arguments: { parameters: [{ name: cmd, value: score.py }] } }
    - name: py
      inputs: { parameters: [{ name: cmd }] }
      container: { image: "<registry>/ml:1.0", command: [python, "{{inputs.parameters.cmd}}"] }
```

**Variantes:** Argo Events (disparadores), Argo CD (GitOps), Argo Rollouts (canary).
**Requisitos:** K8s, YAML, artefactos en S3/Object Storage.
**Aprende más:** argo-workflows.readthedocs.io.

---

## Prefect / Dagster

**Prefect** — flows Python con decoradores, reintentos y caché, despliegues en workers.
```python
from prefect import flow, task
@task(retries=3, cache_key_fn=None)
def extraer(): ...
@flow(log_prints=True)
def pipeline(): entrenar(extraer())
```
```bash
prefect server start; prefect deploy; prefect deployment run 'pipeline/prod'
```

**Dagster** — orientado a *software-defined assets* (tablas, features, modelos) con linaje y materializaciones.
```python
from dagster import asset, Definitions
@asset
def features(raw_llamadas): ...
@asset
def modelo_churn(features): ...
defs = Definitions(assets=[features, modelo_churn])
```
```bash
dagster dev; dagster asset materialize --select modelo_churn
```

**Aprende más:** docs.prefect.io · docs.dagster.io · Dagster University (gratis).

---

## Requisitos de conocimiento comunes
DAGs y dependencias, idempotencia y re-ejecución, backfills, parametrización por fecha, paso de artefactos por almacenamiento (no en memoria), reintentos/timeouts, observabilidad del pipeline, separación orquestación vs cómputo (el orquestador *dispara*, el cómputo ocurre en Spark/K8s/servicio gestionado).

## Hilando soluciones
**Reentrenamiento por drift:** Airflow (diario) → Spark/Glue calcula features → Great Expectations valida → Evidently compara con referencia → si drift > umbral: `SageMakerTrainingOperator`/OCI Data Science Job entrena → evaluación vs champion en MLflow → si gana, registra alias `challenger` y abre aprobación → CI/CD despliega → notificación a Teams.
