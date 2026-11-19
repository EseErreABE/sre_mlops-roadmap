# SRE / MLOps — Documentación

Material de referencia para el rol de **MLOps Engineer / SRE**: actividades del rol y cheatsheets del stack tecnológico.

## Contenido

| Recurso | Descripción |
|---|---|
| [mlops-engineer-actividades.md](mlops-engineer-actividades.md) | Actividades principales del rol (por criticidad), desarrollo de plataformas (Auto Code Reviewer, agente revisor de ejecuciones de modelos) y stack tecnológico. |
| [cheatsheets/](cheatsheets/00-indice-roadmap.md) | 15 cheatsheets del stack, con índice y roadmap de aprendizaje. |

## Cheatsheets

Empieza por el [índice y roadmap](cheatsheets/00-indice-roadmap.md), que incluye el mapa de cómo se conectan las piezas.

| # | Tema | Contenido |
|---|---|---|
| 01 | [Lenguajes](cheatsheets/01-lenguajes.md) | Python, SQL, Bash, Go/Java |
| 02 | [Frameworks de ML](cheatsheets/02-frameworks-ml.md) | scikit-learn, XGBoost/LightGBM, PyTorch, TensorFlow |
| 03 | [Procesamiento de datos](cheatsheets/03-procesamiento-datos.md) | Pandas/Polars, Spark, Kafka, dbt |
| 04 | [Contenedores y Kubernetes](cheatsheets/04-contenedores-kubernetes.md) | Docker, Kubernetes, Helm, HPA/KEDA |
| 05 | [CI/CD](cheatsheets/05-ci-cd.md) | Git, GitHub Actions, GitLab CI, Jenkins, Azure DevOps, GitOps |
| 06 | [Infraestructura como código](cheatsheets/06-iac.md) | Terraform, Pulumi, Ansible |
| 07 | [Orquestación de pipelines](cheatsheets/07-orquestacion-pipelines.md) | Airflow, Kubeflow, Argo, Prefect, Dagster |
| 08 | [Versionado y tracking](cheatsheets/08-versionado-tracking.md) | MLflow, DVC, W&B, model registry |
| 09 | [Feature store y calidad](cheatsheets/09-feature-store-calidad.md) | Feast, Great Expectations, Pandera, Deepchecks |
| 10 | [Serving](cheatsheets/10-serving.md) | FastAPI, BentoML, KServe, Seldon, Triton, TorchServe |
| 11 | [Monitoreo y observabilidad](cheatsheets/11-monitoreo-observabilidad.md) | Prometheus, Grafana, ELK, OpenTelemetry, Evidently, SRE |
| 12 | [Seguridad y gobierno](cheatsheets/12-seguridad-gobierno.md) | Vault, IAM/RBAC, Trivy, Snyk, SAST, IA responsable |
| 13 | [Optimización y LLM/agentes](cheatsheets/13-optimizacion-llm-agentes.md) | ONNX, cuantización, TensorRT, RAG, LangGraph, MCP |
| 14 | [Cloud AWS + Azure/GCP](cheatsheets/14-cloud-aws-azure-gcp.md) | SageMaker, Lambda, EKS, S3, Bedrock y equivalencias |
| 15 | [Cloud OCI](cheatsheets/15-cloud-oci.md) | Data Science, OKE, Functions, Generative AI |

## Estructura de cada cheatsheet

Para qué sirve · Escenario de uso · Comandos típicos · Variantes · Requisitos técnicos y de conocimiento · Conexión con el stack · Aprende más · Hilando soluciones.

## Estructura de carpetas

```
sre_mlops/
├── README.md
├── mlops-engineer-actividades.md
└── cheatsheets/
    ├── 00-indice-roadmap.md
    └── 01 … 15-*.md
```
