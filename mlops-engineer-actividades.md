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

## Niveles de carrera (escalones de conocimiento)

Hay tres niveles y son **acumulativos**: cada nivel incluye todo lo del anterior y agrega más herramientas, más profundidad y más alcance. En los tres niveles se trabajan las mismas soluciones (ver [conocimiento transversal por solución](#conocimiento-transversal-por-solución)), pero cada nivel con más responsabilidad.

| Nivel | Rol | Alcance | Autonomía | Actividades principales a cargo |
|---|---|---|---|---|
| **4** | **MLOpsE Jr** | Tareas y servicios puntuales sobre plantillas, pipelines e infraestructura existentes | Ejecuta con guía y revisión de un nivel superior | 1, 3, 7 y 12 (sobre plantillas); 2 y 9 en primera línea (dashboards, logs, escalamiento) |
| **5** | **MLOpsE Specialist** | Soluciones end-to-end de un producto o dominio, de los datos al monitoreo | Autónomo en su solución; revisa el trabajo de los Jr | 1 a 12 completas para su dominio; aporta a las plataformas (Auto Code Reviewer, agente revisor) |
| **6** | **MLOpsE Specialist Senior** | Plataforma ML multi-modelo, multi-equipo y multi-cloud; estándares de la empresa | Define arquitectura y estándares; lidera incidentes mayores y mentorea | Todas, con foco en 4, 5, 6, 10 y 11 a nivel plataforma; diseña y lidera las plataformas del rol |

### Herramientas por nivel (acumulativo)

Cómo leer la tabla: el **Jr** domina su columna; el **Specialist** domina la del Jr **más** la suya; el **Senior** domina las tres.

| Área | Nivel 4 · MLOpsE Jr | + Nivel 5 · MLOpsE Specialist | + Nivel 6 · MLOpsE Specialist Senior |
|---|---|---|---|
| Lenguajes | Python (pytest, venv, `pyproject.toml`), SQL (joins, agregaciones), Bash básico | SQL avanzado (ventanas, CTEs, tuning), Bash scripting robusto, typing y async en Python | Go o Java para servicios, profiling y performance de Python |
| ML y frameworks | scikit-learn, métricas básicas, entender el modelo que entrega Data Science | XGBoost/LightGBM, PyTorch o TensorFlow, SHAP, prevención de data leakage | Entrenamiento distribuido (DDP, Ray), fine-tuning, criterio de selección de modelo por caso de negocio |
| Serving | FastAPI, inferencia batch | BentoML, streaming, estrategias canary y blue-green con rollback | KServe, Seldon, Triton, TorchServe, multi-model serving, diseño de SLAs de inferencia |
| Contenedores y orquestación | Docker, Docker Compose, `kubectl` básico (get, logs, describe) | Kubernetes (Deployments, Services, ConfigMaps, requests/limits), Helm, HPA | KEDA, operators/CRDs, GPU scheduling, network policies, multi-cluster |
| CI/CD | Git (branching, PR, rebase), GitHub Actions / GitLab CI sobre plantillas | Diseño de pipelines con quality gates, Jenkins / Azure DevOps, OIDC, registries | GitOps (Argo CD / Flux), plantillas reutilizables para la empresa, promoción multiambiente |
| IaC | Leer y aplicar Terraform existente (`plan`/`apply`, variables) | Escribir módulos Terraform, state remoto, Ansible | Pulumi, arquitectura de módulos para toda la organización, policy as code (OPA/Sentinel), drift de infraestructura |
| Orquestación de pipelines | Ejecutar y monitorear DAGs de Airflow existentes | Diseñar DAGs de Airflow, Prefect/Dagster, reentrenamiento por calendario | Kubeflow Pipelines, Argo Workflows, reentrenamiento automático por drift, orquestación multi-equipo |
| Versionado y tracking | MLflow tracking y registry (uso) | DVC, Weights & Biases, linaje modelo–datos–código | Estrategia de model registry corporativo, gobierno de promoción, reproducibilidad auditable |
| Feature store y calidad de datos | Pandera, validaciones simples | Great Expectations, Deepchecks, Feast | Diseño de feature store online/offline, contratos de datos, control de training-serving skew |
| Procesamiento de datos | Pandas, SQL | Polars, Spark, Kafka (producir/consumir), dbt | Arquitectura de streaming (Kafka/MSK/OCI Streaming), tuning de Spark, lakehouse |
| Monitoreo y observabilidad | Leer dashboards de Grafana, logs en CloudWatch / OCI Logging | Prometheus, alertas, ELK/OpenSearch, Evidently, SLIs/SLOs | OpenTelemetry (trazas distribuidas), WhyLabs, error budgets, observabilidad de LLM |
| Seguridad y gobierno | Buenas prácticas de secretos (nada hardcodeado), Trivy | Vault / Secrets Manager / OCI Vault, IAM de mínimo privilegio, Snyk, SAST | Threat modeling, gobierno de modelos, auditoría, IA responsable, OWASP Top 10 for LLM |
| Optimización | Medir latencia y costo | Batching, right-sizing, ONNX | Cuantización, TensorRT, uso eficiente de GPU, FinOps |
| LLM / agentes | Consumir APIs de LLM (Bedrock, OCI Generative AI), prompting | RAG, LangChain, salida estructurada, evaluación básica | LangGraph, MCP, evaluación de agentes en CI, guardrails; diseño del Auto Code Reviewer y del agente revisor |
| Cloud | AWS base (S3, IAM, EC2, Lambda, CloudWatch) y fundamentos de OCI | AWS + OCI: SageMaker / OCI Data Science, EKS / OKE, ECR / OCIR, Functions, Secrets Manager / OCI Vault | Multi-cloud (AWS + OCI y equivalencias Azure/GCP), landing zones, arquitecturas de referencia, costos |
| Prácticas | Ágil, seguir runbooks, participar en post-mortems | Escribir runbooks, liderar incidentes de su servicio, code review | Liderar incidentes mayores, definir estándares y ADRs, mentoría |

### Conocimiento transversal por solución

Las mismas soluciones atraviesan los tres niveles; cambia qué parte resuelve cada uno.

| Solución | Nivel 4 · MLOpsE Jr | Nivel 5 · MLOpsE Specialist | Nivel 6 · MLOpsE Specialist Senior |
|---|---|---|---|
| Modelo batch (churn, propensión) | Ejecuta y monitorea el DAG, valida datos con Pandera, revisa métricas en MLflow | Diseña el pipeline completo (datos → validación → entrenamiento → registro) en Airflow sobre AWS/OCI | Estandariza el patrón batch para todos los equipos, con gobierno de promoción y costos |
| API de inferencia en tiempo real | Dockeriza el modelo con la plantilla FastAPI y lo despliega por CI | Diseña la API, el CI con quality gates, canary con rollback y SLOs | Plataforma multi-modelo (KServe/Triton) con autoscaling por GPU y SLAs corporativos |
| Reentrenamiento automático | Revisa reportes de drift y escala alertas | Implementa detección de drift con Evidently y reentrenamiento con evaluación champion/challenger | Define la política de reentrenamiento y monitoreo de toda la plataforma |
| Plataforma ML en Kubernetes | Despliega con Helm charts existentes y diagnostica pods | Opera el cluster EKS/OKE con Terraform, HPA y Helm | Diseña la plataforma (GitOps, KEDA, Kubeflow/Argo, multi-cluster) |
| RAG / chatbot corporativo | Integra la API de LLM y prueba prompts | Construye el RAG (embeddings, vector store, evaluación) y lo despliega | Define arquitectura GenAI, guardrails, evaluación continua y costo por request |
| Auto Code Reviewer | Usa el reviewer y reporta falsos positivos | Mantiene reglas, prompts e integración en CI | Diseña la plataforma, su evaluación y su gobierno |
| Agente revisor de ejecuciones | Usa el reporte de errores para soporte | Agrega tools y fuentes de logs, mide su calidad | Diseña la arquitectura del agente (LangGraph + MCP) y su evaluación con dataset dorado |

El detalle de aprendizaje por nivel está en el [roadmap de cheatsheets](cheatsheets/00-indice-roadmap.md#niveles-de-carrera-y-roadmap).
