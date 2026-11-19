# 14 · Cloud: AWS (foco) + equivalencias Azure y GCP

> Naturaleza: **plataforma de ejecución gestionada**. Proveen cómputo, datos, ML gestionado, seguridad y monitoreo como servicio. OCI está en [15](15-cloud-oci.md).
> [← Índice](00-indice-roadmap.md) · Anterior: [13](13-optimizacion-llm-agentes.md) · Siguiente: [15 OCI](15-cloud-oci.md)

## Requisitos comunes
- **Técnicos:** cuenta con acceso SSO, CLI instalada y perfil configurado, permisos IAM por ambiente.
- **Conocimiento:** redes (VPC, subnets públicas/privadas, NAT, security groups, endpoints privados), IAM, modelo de responsabilidad compartida, costos y tagging, regiones/AZ, límites de servicio.

---

## AWS: servicios clave para MLOps

### Configuración e identidad
```bash
aws configure sso; aws sso login --profile empresa-dev
export AWS_PROFILE=empresa-dev
aws sts get-caller-identity
```

### Almacenamiento y datos
| Servicio | Para qué | Comandos típicos |
|---|---|---|
| S3 | Data lake, artefactos de modelos, estado Terraform | `aws s3 ls s3://b/ --recursive --human-readable` · `aws s3 sync ./out s3://b/out/` · `aws s3 presign s3://b/f --expires-in 3600` |
| Glue | Catálogo de datos + ETL Spark serverless + Data Quality | `aws glue start-job-run --job-name etl` · `aws glue get-tables --database-name raw` |
| Athena | SQL serverless sobre S3 (Trino) | `aws athena start-query-execution --query-string "SELECT ..." --work-group primary` · Python: `awswrangler.athena.read_sql_query` |
| Redshift | Data warehouse | `aws redshift-data execute-statement --sql "..."` |
| EMR | Spark/Hadoop gestionado | `aws emr add-steps --cluster-id j-XXX --steps Type=Spark,...` |

### Cómputo y contenedores
| Servicio | Para qué | Comandos típicos |
|---|---|---|
| Lambda | Funciones serverless (APIs ligeras, triggers S3, ETL pequeño) | `aws lambda invoke --function-name f --payload fileb://in.json out.json` · `aws logs tail /aws/lambda/f --follow` |
| ECS / Fargate | Contenedores sin gestionar K8s | `aws ecs update-service --cluster c --service s --force-new-deployment` |
| EKS | Kubernetes gestionado | `aws eks update-kubeconfig --name ml-cluster` · `eksctl create nodegroup ...` |
| EC2 | VMs (incl. GPU g5/g6/p4/p5) | `aws ec2 describe-instances --filters Name=tag:proyecto,Values=ml` |
| ECR | Registry de imágenes | `aws ecr get-login-password \| docker login --username AWS --password-stdin <acct>.dkr.ecr.<reg>.amazonaws.com` |
| Step Functions | Orquestación serverless de estados | `aws stepfunctions start-execution --state-machine-arn ...` |
| MWAA | Airflow gestionado | DAGs en S3, `aws mwaa create-cli-token` |

### SageMaker (plataforma ML)
```python
from sagemaker.sklearn import SKLearn
est = SKLearn(entry_point="train.py", role=ROLE, instance_type="ml.m5.xlarge", framework_version="1.2-1")
est.fit({"train": "s3://b/train/"})
pred = est.deploy(initial_instance_count=1, instance_type="ml.m5.large")   # endpoint real-time
```
```bash
aws sagemaker list-endpoints
aws sagemaker describe-training-job --training-job-name <job>
aws sagemaker-runtime invoke-endpoint --endpoint-name churn --body fileb://in.json --content-type application/json out.json
```
Componentes: Training Jobs, Processing, Pipelines, Model Registry, Endpoints (real-time, serverless, async), Batch Transform, Model Monitor, Feature Store, Clarify (sesgo/explicabilidad), managed MLflow, Studio.

### IA generativa
Bedrock (modelos fundacionales vía API Converse, Knowledge Bases para RAG, Agents, Guardrails, Model Evaluation), Textract (documentos), Transcribe (speech-to-text), Comprehend. Ver [13](13-optimizacion-llm-agentes.md).

### Streaming
Kinesis Data Streams / Firehose, MSK (Kafka gestionado), SQS/SNS/EventBridge (mensajería y eventos).

### Seguridad y monitoreo
IAM (roles, Pod Identity/IRSA para EKS), KMS, Secrets Manager, CloudTrail, GuardDuty, Security Hub, Inspector; CloudWatch (métricas, logs, alarmas, Logs Insights), X-Ray. Ver [11](11-monitoreo-observabilidad.md) y [12](12-seguridad-gobierno.md).
```bash
aws logs start-query --log-group-name /aws/lambda/f --start-time $(date -d '-1 hour' +%s) --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | limit 50'
aws cloudwatch put-metric-alarm --alarm-name lat-alta --metric-name Latency ...
```

### IaC y CI/CD
CloudFormation, CDK (Python/TS), Terraform ([06](06-iac.md)); CodePipeline, CodeBuild, CodeDeploy ([05](05-ci-cd.md)). SAM para serverless: `sam build && sam deploy --guided` (repos `lambda-template`, `lambda-tutoria`).

**Aprende más:** docs.aws.amazon.com · AWS Skill Builder (gratis) · workshops.aws (labs de SageMaker, Bedrock, EKS) · **certificaciones:** Cloud Practitioner → Solutions Architect Associate → Machine Learning Engineer Associate → (opcional) DevOps Professional / ML Specialty · *Well-Architected Framework, Machine Learning Lens*.

---

## Equivalencias Azure y GCP (resumen)

| Necesidad | AWS | Azure | GCP |
|---|---|---|---|
| Plataforma ML | SageMaker | Azure Machine Learning | Vertex AI |
| Serverless funciones | Lambda | Functions | Cloud Run functions |
| Contenedores serverless | Fargate | Container Apps | Cloud Run |
| Kubernetes | EKS | AKS | GKE |
| Objeto | S3 | Blob / ADLS Gen2 | GCS |
| Warehouse / SQL serverless | Redshift / Athena | Synapse / Fabric | BigQuery |
| Spark | EMR / Glue | Databricks / Synapse Spark | Dataproc / Dataflow (Beam) |
| Streaming | Kinesis / MSK | Event Hubs | Pub/Sub |
| Orquestación | MWAA / Step Functions | Data Factory / AML Pipelines | Composer / Vertex Pipelines |
| Registry | ECR | ACR | Artifact Registry |
| Secretos / llaves | Secrets Manager / KMS | Key Vault | Secret Manager / KMS |
| Monitoreo | CloudWatch | Azure Monitor / App Insights | Cloud Monitoring / Logging |
| GenAI | Bedrock | Azure OpenAI / AI Foundry | Vertex AI / Model Garden |
| IaC nativo | CloudFormation / CDK | Bicep / ARM | Infrastructure Manager |

```bash
# Azure
az login; az account set -s <sub>; az ml job create -f job.yml; az aks get-credentials -g rg -n aks
# GCP
gcloud auth login; gcloud config set project <p>; gcloud ai custom-jobs create ...; gcloud container clusters get-credentials gke
```

**Aprende más:** learn.microsoft.com (ruta AI-102 / DP-100) · cloud.google.com/learn (Professional ML Engineer) · cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning.

## Hilando soluciones (AWS)
**Pipeline serverless de documentos (tipo `analizador_boletas_back`):** S3 (upload) → EventBridge → Lambda (imagen en ECR) → Textract/Bedrock extrae datos → validación Pydantic → DynamoDB/RDS → API Gateway + Lambda (FastAPI + Mangum) para consulta → CloudWatch alarmas → Terraform/SAM define todo → GitHub Actions con OIDC despliega.
