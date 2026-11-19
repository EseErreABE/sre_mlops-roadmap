# 10 · Serving de modelos: FastAPI, BentoML, Seldon, KServe, Triton, TorchServe

> Naturaleza: **exposición del modelo para consumo** (online, batch o streaming). Es donde el modelo genera valor y donde se mide latencia y disponibilidad.
> [← Índice](00-indice-roadmap.md) · Anterior: [09](09-feature-store-calidad.md) · Siguiente: [11 Monitoreo](11-monitoreo-observabilidad.md)

## Patrones de inferencia
| Patrón | Latencia | Ejemplo | Implementación típica |
|---|---|---|---|
| Online (request/response) | ms | Score de riesgo al registrar una venta | FastAPI/BentoML/KServe detrás de API Gateway |
| Batch | min–horas | Score mensual de churn a toda la base | Spark UDF, SageMaker Batch Transform, OCI Data Science Job |
| Streaming | segundos | Detección de anomalías en eventos | Consumer Kafka/OCI Streaming + modelo |
| Serverless | ms–s (cold start) | APIs de baja frecuencia | AWS Lambda, OCI Functions |

## Cuál elegir
| Herramienta | Nivel | Cuándo |
|---|---|---|
| FastAPI | Framework web genérico | Control total, lógica de negocio, APIs LLM/agentes; equipo Python |
| BentoML | Framework de serving ML | Empaquetar modelo + API + imagen rápidamente, batching adaptativo |
| KServe | Plataforma en K8s (CRD) | Muchos modelos en K8s, autoscaling a cero, canary nativo |
| Seldon Core | Plataforma en K8s | Grafos de inferencia (pre/post, A/B, ensembles), explicadores |
| Triton | Servidor de inferencia de alto rendimiento | GPU, múltiples frameworks, dynamic batching, LLM con TensorRT-LLM |
| TorchServe | Servidor para PyTorch | Modelos PyTorch con handlers custom (en mantenimiento limitado; preferir Triton/KServe para nuevos) |

---

## FastAPI

**Para qué sirve:** framework web Python asíncrono con validación Pydantic y OpenAPI automático.

**Escenario de uso:** API de inferencia, middleware de IA (`ai_api_middleware`), backends de chatbots y RAG, templates (`fastapi_template_test`).

**Comandos típicos**
```python
from contextlib import asynccontextmanager
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field
import mlflow

class Entrada(BaseModel):
    cliente_id: int = Field(ge=1)
    duracion: float = Field(ge=0)

class Salida(BaseModel):
    score: float
    model_version: str

state = {}
@asynccontextmanager
async def lifespan(app: FastAPI):
    state["model"] = mlflow.pyfunc.load_model("models:/churn@champion")   # cargar 1 vez
    yield

app = FastAPI(lifespan=lifespan)

@app.get("/health")
def health(): return {"status": "ok"}

@app.post("/predict", response_model=Salida)
def predict(x: Entrada):
    try:
        s = float(state["model"].predict([[x.duracion]])[0])
    except Exception as e:
        raise HTTPException(500, str(e))
    return Salida(score=s, model_version="7")
```
```bash
uvicorn app.main:app --reload --port 8080                 # dev
fastapi run app/main.py --workers 4                       # prod simple
gunicorn app.main:app -k uvicorn.workers.UvicornWorker -w 4 -b 0.0.0.0:8080
curl localhost:8080/docs                                  # Swagger UI
```
En Lambda: `Mangum(app)` como handler.

**Requisitos:** Python async, Pydantic, HTTP/REST, concurrencia (CPU-bound → workers/procesos; I/O-bound → async).

**Conexión:** Docker/K8s ([04](04-contenedores-kubernetes.md)), MLflow ([08](08-versionado-tracking.md)), Feast online ([09](09-feature-store-calidad.md)), `prometheus-fastapi-instrumentator` y OpenTelemetry ([11](11-monitoreo-observabilidad.md)), OAuth2/JWT ([12](12-seguridad-gobierno.md)), LangChain/LangGraph ([13](13-optimizacion-llm-agentes.md)).

**Aprende más:** fastapi.tiangolo.com/tutorial · testdriven.io.

---

## BentoML

**Para qué sirve:** definir un servicio de modelo en Python y convertirlo en "Bento" (paquete) e imagen Docker, con batching adaptativo y runners.

**Comandos típicos**
```python
import bentoml
@bentoml.service(resources={"cpu": "2"}, traffic={"timeout": 10})
class Churn:
    def __init__(self): self.model = bentoml.sklearn.load_model("churn:latest")
    @bentoml.api(batchable=True)
    def predict(self, x: list[list[float]]) -> list[float]:
        return self.model.predict_proba(x)[:, 1].tolist()
```
```bash
bentoml serve service:Churn
bentoml build && bentoml containerize churn:latest
bentoml models list
```
**Variantes:** BentoCloud (gestionado), Yatai (K8s). **Aprende más:** docs.bentoml.com.

---

## KServe

**Para qué sirve:** CRD `InferenceService` en Kubernetes para servir modelos estándar (sklearn, xgboost, pytorch, onnx, huggingface) o contenedores custom, con autoscaling (incl. a cero), canary y protocolo de inferencia v2.

**Comandos típicos**
```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata: { name: churn, namespace: ml }
spec:
  predictor:
    canaryTrafficPercent: 10
    minReplicas: 1
    model:
      modelFormat: { name: sklearn }
      storageUri: s3://modelos/churn/v7
```
```bash
kubectl apply -f isvc.yaml; kubectl get isvc -n ml
curl -H "Host: churn.ml.example.com" http://$INGRESS/v1/models/churn:predict -d @in.json
```
**Requisitos:** K8s + KServe (con Knative o modo raw), acceso a storage. **Aprende más:** kserve.github.io/website.

---

## Seldon Core

**Para qué sirve:** serving en K8s con grafos de inferencia (transformers, routers A/B, combiners) y explicadores (Alibi).
```bash
kubectl apply -f seldon-model.yaml; kubectl get models -n seldon
```
**Notas:** Seldon Core v2 usa licencia BSL; revisar condiciones. **Aprende más:** docs.seldon.ai.

---

## NVIDIA Triton Inference Server

**Para qué sirve:** servidor de inferencia de alto rendimiento multi-framework (ONNX, TensorRT, PyTorch, TF, Python, FIL para árboles) con dynamic batching, ejecución concurrente y métricas Prometheus.

**Comandos típicos**
```text
model_repository/
└── churn/
    ├── config.pbtxt
    └── 1/model.onnx
```
```protobuf
name: "churn"
backend: "onnxruntime"
max_batch_size: 64
dynamic_batching { max_queue_delay_microseconds: 2000 }
instance_group [{ kind: KIND_GPU, count: 1 }]
```
```bash
docker run --gpus all -p8000:8000 -p8001:8001 -p8002:8002 -v $PWD/model_repository:/models \
  nvcr.io/nvidia/tritonserver:24.10-py3 tritonserver --model-repository=/models
curl localhost:8000/v2/health/ready
perf_analyzer -m churn --concurrency-range 1:16          # benchmark
```
**Requisitos:** GPU NVIDIA (o CPU), conocer formatos de modelo y tuning de batching. **Aprende más:** docs.nvidia.com/deeplearning/triton-inference-server · Triton tutorials (GitHub).

---

## TorchServe

```bash
torch-model-archiver --model-name stt --version 1.0 --serialized-file model.pt --handler handler.py
torchserve --start --model-store model_store --models stt=stt.mar
curl -X POST localhost:8080/predictions/stt -T audio.wav
```
**Nota:** el proyecto está en mantenimiento limitado; para nuevos desarrollos considerar Triton, KServe o vLLM (LLM).

---

## Serving de LLM (referencia rápida)
vLLM (`vllm serve <modelo>` con API compatible OpenAI), TGI (Hugging Face), Triton + TensorRT-LLM, Ollama (local). Gestionados: Bedrock, OCI Generative AI, Azure OpenAI, Vertex AI. Ver [13](13-optimizacion-llm-agentes.md).

## Estrategias de despliegue
Rolling · Blue-green · Canary (KServe `canaryTrafficPercent`, Argo Rollouts) · Shadow (espejo de tráfico) · A/B por segmento. Siempre: health/readiness, versión del modelo en la respuesta y logs, rollback documentado.

## Hilando soluciones
**Serving online gestionado vs propio:**
- *Gestionado:* MLflow → `sagemaker` endpoint / OCI Data Science Model Deployment → API Gateway → CloudWatch/OCI Monitoring.
- *Propio en K8s:* modelo ONNX → Triton (GPU) o KServe → Istio/Ingress → HPA/KEDA → Prometheus/Grafana → Evidently recibe logs de predicción vía Kafka.
