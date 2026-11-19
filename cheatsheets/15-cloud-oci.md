# 15 · Cloud: OCI (Oracle Cloud Infrastructure)

> Naturaleza: **plataforma de ejecución gestionada** (segundo proveedor en la empresa junto con AWS). Aquí están los servicios OCI útiles para MLOps y cómo se conectan con el resto del stack.
> [← Índice](00-indice-roadmap.md) · Anterior: [14 AWS/Azure/GCP](14-cloud-aws-azure-gcp.md)

## Conceptos propios de OCI (diferencias con AWS)
| Concepto | Qué es | Equivalente aproximado AWS |
|---|---|---|
| Tenancy | Cuenta raíz de la organización | Organization / cuenta |
| **Compartment** | Contenedor lógico de recursos para aislar y aplicar políticas (anidable) | Cuenta / OU + tags |
| OCID | Identificador único de cada recurso (`ocid1.<tipo>.oc1...`) | ARN |
| **Policies** | Reglas en lenguaje natural: `Allow group X to manage Y in compartment Z` | IAM policies (JSON) |
| Dynamic groups / Resource principals | Identidad para recursos (Functions, Jobs, Instances) sin llaves | IAM roles para servicios |
| Identity Domains | Gestión de usuarios/SSO/federación | IAM Identity Center |
| VCN | Red virtual (subnets, gateways, security lists/NSG) | VPC |
| Region / AD / FD | Región, Availability Domain, Fault Domain | Region / AZ |

## Requisitos
- **Técnicos:** usuario en el tenancy con grupo y políticas en el compartment del proyecto; OCI CLI (`pip install oci-cli`); llave API (`~/.oci/config`) o *session token*; SDK Python `oci`; para ML, la librería `oracle-ads`.
- **Conocimiento:** compartments y políticas, VCN, OCIDs, resource principals, diferencias con AWS, Terraform provider `oracle/oci`.

```bash
oci setup config                                    # genera ~/.oci/config y llave API
oci session authenticate --region sa-santiago-1     # sesión SSO temporal (usar --auth security_token)
oci iam compartment list --all --compartment-id-in-subtree true
oci iam region list
export C=ocid1.compartment.oc1..xxxx
```

---

## Almacenamiento y datos

| Servicio | Para qué | Comandos típicos |
|---|---|---|
| **Object Storage** | Data lake, artefactos, modelos, estado Terraform; API compatible con S3 | `oci os object put -bn modelos --file model.pkl` · `oci os object list -bn modelos --prefix churn/` · `oci os object bulk-download -bn b --download-dir ./d` · `oci os preauth-request create -bn b -o f --access-type ObjectRead --time-expires 2026-12-01` |
| **Autonomous Database (ADW / ATP)** | Warehouse / transaccional autogestionado; SQL Oracle, **AI Vector Search (23ai)**, Select AI (NL→SQL) | `oci db autonomous-database list -c $C` · Python: `oracledb.connect(user, password, dsn, config_dir=wallet)` |
| **Data Flow** | Spark serverless (PySpark/Scala/SQL) | `oci data-flow application create ...` · `oci data-flow run create --application-id $APP -c $C --display-name run1` |
| **Data Integration** | ETL/ELT visual y orquestación de tareas de datos | Tareas y pipelines desde consola/SDK |
| **Data Catalog** | Catálogo de metadatos y linaje | Harvest de Object Storage/ADB |
| **GoldenGate** | Replicación/CDC en tiempo real | Fuente para Streaming |

Object Storage vía API S3 (útil para DVC, MLflow, boto3, Spark): endpoint `https://<namespace>.compat.objectstorage.<region>.oraclecloud.com` con *Customer Secret Keys*.
```bash
oci os ns get                                        # namespace del tenancy
```

## Cómputo y contenedores

| Servicio | Para qué | Comandos típicos |
|---|---|---|
| Compute (VM, Bare Metal, **GPU** A10/A100/H100) | Entrenamiento, servidores de inferencia propios | `oci compute instance list -c $C` · `oci compute instance action --action STOP --instance-id $I` |
| **OKE** (Container Engine for Kubernetes) | Kubernetes gestionado (nodos gestionados o virtual nodes serverless) | `oci ce cluster create-kubeconfig --cluster-id $K --file ~/.kube/config --region sa-santiago-1 --token-version 2.0.0` |
| Container Instances | Contenedores serverless sin cluster | `oci container-instances container-instance create ...` |
| **OCI Functions** | Serverless basado en Fn Project (imágenes en OCIR) | `fn init --runtime python f` · `fn -v deploy --app ml-app` · `fn invoke ml-app f` · `oci fn function invoke --function-id $F --file out.json --body '{}'` |
| **OCIR** (Container Registry) | Registry de imágenes | `docker login <region-key>.ocir.io -u '<namespace>/<usuario>' ` (password = auth token) · `docker push <region-key>.ocir.io/<ns>/api:1.0` |
| API Gateway | Exponer Functions/HTTP backends con auth y rate limit | Deployments con rutas y políticas JWT |
| Queue / Notifications / Events | Mensajería, alertas (email, Slack, webhook), eventos de recursos | `oci ons message publish --topic-id $T --body "falló DAG"` |

## OCI Data Science (plataforma ML)

**Para qué sirve:** notebooks gestionados, Jobs (entrenamiento/batch), Pipelines, Model Catalog (registry), Model Deployments (endpoints HTTP), integración con conda environments y MLflow/ADS.

**Comandos típicos**
```python
import ads
ads.set_auth("resource_principal")          # dentro de notebook/job; local: ads.set_auth("api_key")

# Job de entrenamiento
from ads.jobs import Job, DataScienceJob, PythonRuntime
job = (Job(name="train-churn")
       .with_infrastructure(DataScienceJob().with_shape_name("VM.Standard.E4.Flex").with_shape_config_details(ocpus=4, memory_in_gbs=64)
                            .with_log_group_id(LOG_GROUP).with_log_id(LOG))
       .with_runtime(PythonRuntime().with_source("train.py").with_service_conda("generalml_p311_cpu_x86_64_v1")))
job.create(); run = job.run(); run.watch()

# Registrar y desplegar modelo
from ads.model.framework.sklearn_model import SklearnModel
m = SklearnModel(estimator=pipe, artifact_dir="./art")
m.prepare(inference_conda_env="generalml_p311_cpu_x86_64_v1", X_sample=X_te[:5], force_overwrite=True)
m.verify(X_te[:5].tolist())                 # prueba local del score.py
m.save(display_name="churn-v7")             # Model Catalog
m.deploy(display_name="churn-api", deployment_instance_shape="VM.Standard.E4.Flex")
m.predict(X_te[:5].tolist())
```
```bash
ads opctl run -f job.yaml                   # Jobs desde YAML
oci data-science model list -c $C --project-id $P
oci data-science model-deployment list -c $C
odsc conda list; odsc conda install -s generalml_p311_cpu_x86_64_v1
```
**BYOC:** Jobs y Model Deployments aceptan imágenes propias desde OCIR (p. ej. FastAPI o Triton).
**AI Quick Actions:** desplegar y afinar LLMs open source (vLLM/TGI) desde Data Science.

## IA generativa y servicios de IA
| Servicio | Para qué |
|---|---|
| **Generative AI** | Modelos de chat y embeddings gestionados (Cohere, Meta Llama y otros), fine-tuning en *dedicated AI clusters* |
| **Generative AI Agents** | Agentes con RAG gestionado sobre Object Storage, OpenSearch u Oracle AI Vector Search, y tools |
| AI Speech | Speech-to-text (alternativa a Whisper/Transcribe para `speech_to_text*`) |
| AI Language | NER, sentimiento, clasificación, PII |
| AI Document Understanding | OCR, extracción de tablas/formularios (boletas, declaraciones juradas) |
| AI Vision | Clasificación y detección en imágenes |

```bash
oci generative-ai model list -c $C --region us-chicago-1    # GenAI está disponible en regiones específicas
oci ai-speech transcription-job create ...
```
```python
# Integración LangChain
from langchain_community.chat_models import ChatOCIGenAI
llm = ChatOCIGenAI(model_id=MODEL_ID, service_endpoint=ENDPOINT, compartment_id=C, auth_type="API_KEY")
```

## Streaming
**OCI Streaming:** compatible con API de Kafka (usar clientes Kafka con SASL/SSL y auth token) → reutiliza todo lo de [03](03-procesamiento-datos.md).
```bash
oci streaming admin stream create --name eventos --partitions 3 -c $C
```
**Connector Hub:** mueve datos entre Logging, Streaming, Object Storage, Functions y Monitoring sin código.

## Seguridad y gobierno
- **IAM:** compartments por proyecto/ambiente; políticas mínimas; dynamic groups para que Jobs/Functions accedan a Object Storage/Vault con *resource principal*.
- **Vault:** master keys (KMS) y secrets. `oci vault secret create-base64 ...`
- **Cloud Guard** (postura de seguridad), **Security Zones** (impiden configuraciones inseguras), **Bastion** (acceso a subnets privadas), **Audit** (registro de llamadas API), **Data Safe** (seguridad y enmascaramiento en bases Oracle), escaneo de vulnerabilidades en OCIR.
Ver [12](12-seguridad-gobierno.md).

## Monitoreo
| Servicio | Para qué |
|---|---|
| Monitoring | Métricas de servicios + custom, consultas MQL, alarmas → Notifications |
| Logging | Logs de servicios, custom logs (agente), logs de Functions/Jobs/Model Deployment |
| Logging Analytics | Búsqueda/análisis avanzado y detección de patrones en logs |
| APM | Trazas distribuidas (compatible OpenTelemetry), monitoreo sintético |

```bash
oci monitoring metric-data summarize-metrics-data -c $C --namespace oci_datascience_modeldeploy \
  --query-text 'PredictLatency[5m].percentile(0.95)'
oci logging-search search-logs --search-query "search \"$C\" | where data.level = 'ERROR'" \
  --time-start 2026-11-18T00:00:00Z --time-end 2026-11-19T00:00:00Z
```
Grafana tiene plugin oficial para OCI Metrics y Logging. Ver [11](11-monitoreo-observabilidad.md).

## IaC y CI/CD
- **Terraform provider `oracle/oci`** y **Resource Manager** (stacks Terraform gestionados, con estado y drift detection). `oci resource-manager job create-apply-job --stack-id $S --execution-plan-strategy AUTO_APPROVED`
- **OCI DevOps:** repositorios de código, *build pipelines* (`build_spec.yaml`), artifacts, *deployment pipelines* a OKE, Functions o instancias, con canary y aprobaciones.
- También funciona con GitHub Actions/GitLab usando la CLI o acciones oficiales de OCI.

```yaml
# build_spec.yaml (OCI DevOps)
version: 0.1
component: build
steps:
  - type: Command
    name: test
    command: pip install -r requirements.txt && pytest -q
outputArtifacts:
  - name: api_image
    type: DOCKER_IMAGE
    location: api:latest
```

## Multi-cloud AWS ↔ OCI
- **Interconexión:** OCI FastConnect / VPN con AWS para tráfico privado entre nubes.
- **Datos:** Object Storage API S3-compatible permite usar `boto3`, `s3fs`, DVC y MLflow apuntando a OCI.
- **Identidad:** federar ambos con el mismo IdP corporativo.
- **Patrón típico:** datos/warehouse en Autonomous DB (OCI), procesamiento o GenAI en el proveedor que tenga el servicio requerido; homologar con Terraform (dos providers en el mismo repo) y observabilidad centralizada en Grafana/OpenTelemetry.
- Repos de referencia: `Analytics_AWS_OCI_Management`, `oci_chatbot_back`, `oci_chatbot_front`.

**Aprende más:** docs.oracle.com/iaas · **Oracle University (cursos y certificaciones gratis):** OCI Foundations → OCI AI Foundations → OCI Generative AI Professional → OCI Data Science Professional → OCI Architect Associate · github.com/oracle-samples/oci-data-science-ai-samples · accelerated-data-science.readthedocs.io (ADS) · docs.oracle.com/iaas/Content/generative-ai · Oracle LiveLabs (labs guiados gratis).

## Hilando soluciones (OCI)
**Modelo + GenAI en OCI de punta a punta:** datos en Autonomous DB → Data Flow (Spark) calcula features a Object Storage → Data Science Job entrena (resource principal, logs en OCI Logging) → Model Catalog registra → Model Deployment o imagen FastAPI en OKE (desde OCIR, desplegado por OCI DevOps) → API Gateway con JWT → Generative AI Agents explica el score al asesor usando RAG sobre políticas → Monitoring + alarmas → Notifications a Teams → todo definido en Terraform vía Resource Manager.
