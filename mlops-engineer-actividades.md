# Machine Learning Operations (MLOps) Engineer — Actividades

## Actividades principales (por relevancia / criticidad)

1. **Despliegue y serving de modelos en producción** — empaquetado, APIs/batch/streaming, estrategias canary/blue-green y rollback.
2. **Monitoreo y observabilidad de modelos** — latencia, disponibilidad, data drift, concept drift, degradación de performance, alertas.
3. **Pipelines de CI/CD para ML** — automatización de build, pruebas, validación y promoción de modelos entre ambientes.
4. **Orquestación de pipelines de entrenamiento y reentrenamiento** — Airflow, Kubeflow, Vertex/SageMaker/Azure ML; reentrenamiento automático por drift o calendario.
5. **Gestión de infraestructura (IaC)** — contenedores, Kubernetes, GPUs/cómputo, autoscaling, Terraform.
6. **Seguridad, gobierno y cumplimiento** — control de accesos, gestión de secretos, trazabilidad, auditoría, privacidad de datos.
7. **Versionado y registro de artefactos** — model registry, versionado de datos, código y experimentos (MLflow, DVC, W&B).
8. **Feature stores y calidad de datos** — validación de datos, consistencia entre entrenamiento e inferencia.
9. **Gestión de incidentes y soporte** — troubleshooting, análisis de causa raíz, post-mortems, SLAs/SLOs.
10. **Optimización de costos y performance** — right-sizing, cuantización, batching, uso eficiente de GPU.
11. **Colaboración con Data Science, Data Engineering y DevOps** — estándares, plantillas y buenas prácticas de productivización.
12. **Documentación y mejora continua** — runbooks, arquitecturas de referencia, estandarización de procesos.

## Desarrollo de plataformas (actividades adicionales del rol)

### Auto Code Reviewer
- Revisión automática de pull requests (calidad, estilo, seguridad, buenas prácticas ML).
- Detección de vulnerabilidades, secretos expuestos y malas prácticas en código y pipelines.
- Comentarios y sugerencias de corrección directamente en el repositorio, integrado al CI.

### Agente revisor de ejecuciones de modelos
- Análisis automático de logs y ejecuciones de entrenamiento, evaluación e inferencia.
- Listado estructurado de errores detectados (fallos de datos, memoria, dependencias, métricas anómalas, timeouts).
- Propuesta de posibles soluciones por error, priorizadas por probabilidad e impacto.
- Reporte/alerta a los equipos responsables y registro histórico para identificar patrones recurrentes.

## Stack tecnológico

### Conocimiento técnico general
| Área | Tecnologías / conceptos |
|---|---|
| Lenguajes | Python (principal), SQL, Bash; deseable Go/Java para servicios |
| ML y frameworks | scikit-learn, XGBoost/LightGBM, PyTorch, TensorFlow; fundamentos de entrenamiento, evaluación y métricas |
| Serving | FastAPI, BentoML, Seldon, KServe, Triton, TorchServe; batch, streaming y online |
| Contenedores y orquestación | Docker, Kubernetes, Helm, autoscaling (HPA/KEDA) |
| CI/CD | Git, GitHub Actions / GitLab CI / Jenkins, Azure DevOps; pruebas automatizadas, quality gates |
| IaC | Terraform, Ansible, Pulumi |
| Orquestación de pipelines | Airflow, Kubeflow Pipelines, Argo Workflows, Prefect/Dagster |
| Versionado y tracking | MLflow, DVC, Weights & Biases, model registry |
| Feature store y calidad de datos | Feast, Great Expectations, Deepchecks, Pandera |
| Procesamiento de datos | Spark, Pandas/Polars, Kafka, dbt |
| Monitoreo y observabilidad | Prometheus, Grafana, ELK/OpenSearch, OpenTelemetry, Evidently, WhyLabs (drift) |
| Seguridad y gobierno | Vault, gestión de secretos, RBAC, escaneo de vulnerabilidades (Trivy, Snyk), SAST, auditoría y trazabilidad |
| Optimización | Cuantización, ONNX, TensorRT, batching, uso eficiente de GPU |
| LLM / agentes (plataformas) | APIs de LLM, LangChain/LangGraph, MCP, RAG, evaluación de agentes (Auto Code Reviewer, agente revisor) |
| Prácticas | Metodologías ágiles, SRE (SLAs/SLOs), post-mortems, documentación técnica |

### Stack en la nube
| Servicio | AWS | Azure | GCP | OCI |
|---|---|---|---|---|
| Plataforma ML | SageMaker | Azure Machine Learning | Vertex AI | OCI Data Science (Model Deployment, Jobs, Pipelines) |
| Cómputo / contenedores | EC2, ECS, EKS, Lambda | AKS, Container Apps, Functions | GKE, Cloud Run | Compute (VM/BM/GPU), OKE, Container Instances, OCI Functions |
| Almacenamiento y datos | S3, Glue, Athena, Redshift | Blob/ADLS, Synapse, Data Factory, Databricks | GCS, BigQuery, Dataflow | Object Storage, Autonomous Database (ADW/ATP), Data Flow (Spark), Data Integration, Data Catalog |
| Streaming | Kinesis, MSK | Event Hubs | Pub/Sub | OCI Streaming (compatible con Kafka) |
| CI/CD | CodePipeline, CodeBuild, ECR | Azure DevOps, ACR | Cloud Build, Artifact Registry | OCI DevOps, Container Registry (OCIR), Artifact Registry |
| IaC | CloudFormation, Terraform | Bicep, Terraform | Deployment Manager, Terraform | Resource Manager (Terraform gestionado), Terraform |
| Monitoreo | CloudWatch, SageMaker Model Monitor | Azure Monitor, App Insights | Cloud Monitoring, Vertex Model Monitoring | OCI Monitoring, Logging, Logging Analytics, APM |
| Seguridad | IAM, KMS, Secrets Manager | Entra ID, Key Vault | IAM, Secret Manager, KMS | IAM (compartments, policies), Identity Domains, Vault (keys y secrets), Cloud Guard |
| IA generativa | Bedrock | Azure OpenAI / AI Foundry | Vertex AI (Model Garden) | OCI Generative AI (Cohere, Llama), Generative AI Agents, AI Services (Speech, Language, Document Understanding) |

> Nota: se listan las tres nubes principales más OCI (Oracle Cloud Infrastructure), usada en la empresa junto con AWS. Ver cheatsheets detallados en [cheatsheets/](cheatsheets/00-indice-roadmap.md).
