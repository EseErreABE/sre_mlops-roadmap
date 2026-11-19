# 13 · Optimización de modelos y LLM/Agentes

> Naturaleza: dos bloques avanzados.
> **A. Optimización:** hacer que el modelo sea más rápido y barato (cuantización, ONNX, TensorRT, batching, GPU).
> **B. LLM y agentes:** construir plataformas con modelos de lenguaje (APIs de LLM, LangChain/LangGraph, MCP, RAG, evaluación), como el Auto Code Reviewer y el agente revisor de ejecuciones.
> [← Índice](00-indice-roadmap.md) · Anterior: [12](12-seguridad-gobierno.md) · Siguiente: [14 Cloud](14-cloud-aws-azure-gcp.md)

---

# A. Optimización

## ONNX / ONNX Runtime

**Para qué sirve:** formato estándar de modelos e *engine* de inferencia optimizado y portable (CPU, GPU, edge), independiente del framework de entrenamiento.

**Escenario de uso:** acelerar un modelo sklearn/PyTorch en CPU 2–5x, servirlo en Triton o en una Lambda/Function sin instalar el framework completo.

**Comandos típicos**
```python
# sklearn -> ONNX
from skl2onnx import to_onnx
onx = to_onnx(pipe, X_tr[:1].astype("float32")); open("model.onnx", "wb").write(onx.SerializeToString())
# PyTorch -> ONNX
torch.onnx.export(model, dummy, "model.onnx", dynamo=True)
# Inferencia
import onnxruntime as ort
sess = ort.InferenceSession("model.onnx", providers=["CUDAExecutionProvider", "CPUExecutionProvider"])
y = sess.run(None, {sess.get_inputs()[0].name: x})
```
```bash
python -m onnxruntime.quantization.preprocess --input model.onnx --output model-pre.onnx
```
**Aprende más:** onnxruntime.ai/docs · github.com/onnx/tutorials.

## Cuantización

**Para qué sirve:** reducir precisión de pesos/activaciones (FP32 → FP16/BF16/INT8/INT4) para bajar memoria y latencia con pérdida mínima de calidad.

| Técnica | Cuándo |
|---|---|
| Dinámica INT8 (post-training) | Rápido, modelos NLP en CPU |
| Estática INT8 (con calibración) | Mejor latencia, requiere dataset de calibración |
| QAT (quantization-aware training) | Cuando la PTQ pierde demasiada calidad |
| LLM: GPTQ, AWQ, GGUF (llama.cpp), bitsandbytes 4/8-bit, FP8 | Servir LLMs grandes en menos GPU |

```python
from onnxruntime.quantization import quantize_dynamic, QuantType
quantize_dynamic("model.onnx", "model.int8.onnx", weight_type=QuantType.QInt8)
# PyTorch: torch.ao.quantization ; HF: BitsAndBytesConfig(load_in_4bit=True)
```
**Siempre:** medir calidad antes/después en el mismo set de evaluación.

## TensorRT / TensorRT-LLM

**Para qué sirve:** compilador de NVIDIA que fusiona capas y optimiza kernels para GPU específicas; TensorRT-LLM para LLMs (in-flight batching, KV cache paginado).
```bash
trtexec --onnx=model.onnx --saveEngine=model.plan --fp16
```
Se sirve con Triton ([10](10-serving.md)). **Requisitos:** GPU NVIDIA, versión de CUDA/TensorRT alineada con el driver.

## Batching y uso eficiente de GPU
- **Dynamic batching** (Triton, BentoML, vLLM continuous batching): agrupa requests para usar la GPU al máximo.
- Perfilar: `nvidia-smi dmon`, `nvtop`, PyTorch Profiler, Nsight Systems, DCGM exporter → Prometheus.
- Mixed precision (`torch.autocast`), `torch.compile`, data loaders con `num_workers` y `pin_memory`.
- Compartir GPU: MIG, time-slicing, múltiples modelos por GPU (Triton instance groups).
- **Costos:** right-sizing de instancias, spot/preemptible para entrenamiento, scale-to-zero (KEDA/KServe), batch en vez de online cuando el negocio lo permite, apagar notebooks ociosos, tags de costo por proyecto.

**Aprende más:** NVIDIA Deep Learning Performance Guide · Hugging Face "Optimum" docs · *Efficient Deep Learning* (efficientdlbook.com).

---

# B. LLM y agentes

## APIs de LLM

**Para qué sirve:** consumir modelos fundacionales (generación, clasificación, extracción, resumen, embeddings) vía API.

**Escenario de uso:** chatbots internos (`empresa_gpt`, `oci_chatbot_back`), análisis de boletas/documentos, tipificación de llamadas, text-to-SQL, generación de contenido.

**Comandos típicos**
```python
# AWS Bedrock (API Converse, unificada para varios proveedores)
import boto3
br = boto3.client("bedrock-runtime")
r = br.converse(modelId=MODEL_ID,                       # ID del modelo habilitado en la cuenta
                messages=[{"role": "user", "content": [{"text": "Resume este log: ..."}]}],
                inferenceConfig={"maxTokens": 1024, "temperature": 0})
print(r["output"]["message"]["content"][0]["text"])

# OCI Generative AI
import oci
cfg = oci.config.from_file()
gen = oci.generative_ai_inference.GenerativeAiInferenceClient(cfg, service_endpoint=ENDPOINT)
# chat_detail = oci.generative_ai_inference.models.ChatDetails(compartment_id=C, serving_mode=..., chat_request=...)
```
```bash
aws bedrock list-foundation-models --query 'modelSummaries[].modelId'
oci generative-ai model list --compartment-id $C
```

**Conceptos clave:** tokens y ventana de contexto, temperature, system prompt, *structured outputs* (JSON schema), *tool use / function calling*, streaming, prompt caching, rate limits, costo por token, latencia (time-to-first-token).

**Variantes:** Bedrock (Anthropic Claude, Llama, Mistral, Amazon Nova…), **OCI Generative AI (Cohere, Llama, otros)**, Azure OpenAI / AI Foundry, Vertex AI, APIs directas de proveedores, modelos self-hosted (vLLM, Ollama). Gateways: LiteLLM, middleware propio (`ai_api_middleware`).

## RAG (Retrieval-Augmented Generation)

**Para qué sirve:** responder con información propia (documentos, políticas, tickets) recuperando fragmentos relevantes e inyectándolos en el prompt.

**Pipeline:** ingesta → parsing (Docling, Textract, OCI Document Understanding) → chunking → embeddings → vector store → recuperación (híbrida: vectorial + BM25) → re-ranking → generación con citas → evaluación.

```python
# Esquema mínimo
chunks = split(docs, size=800, overlap=100)
vecs = embed(chunks)                         # Titan/Cohere embed (Bedrock / OCI GenAI)
store.upsert(ids, vecs, metadata)            # OpenSearch k-NN, pgvector, Oracle AI Vector Search, FAISS
ctx = store.search(embed(pregunta), k=5)
respuesta = llm(f"Responde sólo con este contexto:\n{ctx}\n\nPregunta: {pregunta}")
```
**Variantes:** Bedrock Knowledge Bases, OCI Generative AI Agents (RAG gestionado), **Oracle Database 23ai AI Vector Search**, OpenSearch, pgvector, Chroma/FAISS (local). Repos de referencia: `rag_aws`, `rag_serverless`, `embedding_gen_ai`.

## LangChain / LangGraph

**Para qué sirve:** LangChain ofrece integraciones (LLMs, vector stores, loaders, tools); **LangGraph** modela agentes como grafos de estado con ciclos, memoria, checkpoints y *human-in-the-loop*.

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class S(TypedDict):
    log: str
    errores: list
    propuesta: str

def clasificar(s: S): return {"errores": llm_extraer_errores(s["log"])}
def proponer(s: S):  return {"propuesta": llm_proponer(s["errores"])}

g = StateGraph(S)
g.add_node("clasificar", clasificar); g.add_node("proponer", proponer)
g.add_edge(START, "clasificar"); g.add_edge("clasificar", "proponer"); g.add_edge("proponer", END)
app = g.compile()
app.invoke({"log": open("train.log").read()})
```
**Variantes:** LlamaIndex (RAG), OpenAI Agents SDK, smolagents, Claude Agent SDK, CrewAI, Strands Agents, Bedrock Agents, OCI Generative AI Agents (repos `poc-openai-agents`, `smolagents_poc`, `desafio_agente_llama`).

## MCP (Model Context Protocol)

**Para qué sirve:** protocolo abierto para exponer *tools*, *resources* y *prompts* a agentes LLM de forma estándar (un servidor MCP = un conector reutilizable).

**Escenario de uso:** dar al agente revisor acceso controlado a logs de CloudWatch/OCI Logging, al registry de MLflow o a Jira/Confluence.
```python
from mcp.server.fastmcp import FastMCP
mcp = FastMCP("mlops-tools")
@mcp.tool()
def ultimo_run(experimento: str) -> dict:
    """Devuelve métricas del último run de MLflow."""
    return buscar_run(experimento)
if __name__ == "__main__": mcp.run()
```
**Seguridad:** mínimo privilegio por tool, validación de inputs, confirmación humana para acciones destructivas, cuidado con prompt injection desde datos leídos.

## Evaluación de LLM y agentes

**Qué medir:** exactitud/faithfulness (RAG), relevancia del contexto, tasa de alucinación, éxito de tarea del agente, uso correcto de tools, latencia, costo, seguridad (jailbreak, PII).

**Cómo:** datasets dorados versionados, LLM-as-judge con rúbricas, regresión en CI ante cambios de prompt/modelo, trazas (Langfuse, LangSmith, Phoenix, MLflow Tracing), A/B en producción, feedback de usuarios.

**Herramientas:** Ragas, DeepEval, promptfoo, MLflow Evaluate, W&B Weave, Bedrock Model Evaluation.

```bash
promptfoo eval -c promptfooconfig.yaml      # regresión de prompts en CI
```

## Plataformas del rol

**Auto Code Reviewer**
- Disparo: webhook de PR → GitHub Actions/GitLab CI.
- Entrada: diff + reglas del repo (estilo, seguridad, buenas prácticas ML) + resultados de Semgrep/Trivy/ruff como contexto.
- LLM con salida estructurada (JSON: archivo, línea, severidad, sugerencia) → comentarios inline vía API de Git.
- Guardrails: no exponer secretos al LLM, límites de tokens por PR, métricas de aceptación de comentarios.

**Agente revisor de ejecuciones de modelos**
- Fuentes vía tools/MCP: logs (CloudWatch/OCI Logging/OpenSearch), métricas (Prometheus), runs (MLflow), estado de DAGs (Airflow).
- Grafo LangGraph: recolectar → clasificar error (datos, memoria/OOM, dependencias, métricas anómalas, timeout) → buscar casos similares (RAG sobre post-mortems) → proponer soluciones priorizadas → notificar (Teams) → guardar histórico para patrones.

**Requisitos (bloque B):** Python async, APIs REST, prompt engineering, embeddings y búsqueda vectorial, diseño de tools, evaluación, costos por token, seguridad LLM (OWASP LLM Top 10).

**Aprende más:** docs de Bedrock y OCI Generative AI · python.langchain.com y langchain-ai.github.io/langgraph · modelcontextprotocol.io · Anthropic "Building effective agents" (anthropic.com/research) · DeepLearning.AI short courses (RAG, agentes, LangGraph, evaluación) · *AI Engineering* (Chip Huyen) · *Designing Machine Learning Systems* (Chip Huyen).

---

## Hilando soluciones
**RAG productivo en AWS u OCI:** documentos en S3/Object Storage → evento dispara Lambda/OCI Function → Docling/Textract parsea → embeddings (Bedrock/OCI GenAI) → OpenSearch k-NN / Oracle AI Vector Search → API FastAPI con LangGraph (retrieve → rerank → generate con citas) → trazas OTel/Langfuse → evaluación Ragas en CI con dataset dorado → guardrails + secretos en Vault → costos por token en Grafana.
