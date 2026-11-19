# 11 · Monitoreo y observabilidad: Prometheus, Grafana, ELK/OpenSearch, OpenTelemetry, Evidently, WhyLabs

> Naturaleza: **visibilidad del sistema y del modelo en producción**. Dos capas: *observabilidad de sistema* (latencia, errores, recursos) y *monitoreo de modelo* (drift, calidad de predicción).
> [← Índice](00-indice-roadmap.md) · Anterior: [10](10-serving.md) · Siguiente: [12 Seguridad](12-seguridad-gobierno.md)

| Señal | Herramienta | Pregunta que responde |
|---|---|---|
| Métricas | Prometheus → Grafana | ¿Cuánta latencia/errores/CPU hay ahora y en el tiempo? |
| Logs | ELK / OpenSearch (o CloudWatch Logs, OCI Logging) | ¿Qué pasó exactamente en esta petición/ejecución? |
| Trazas | OpenTelemetry → Jaeger/Tempo/X-Ray/OCI APM | ¿Dónde se va el tiempo entre servicios? |
| Drift / calidad del modelo | Evidently, WhyLabs, SageMaker Model Monitor | ¿Cambiaron los datos o se degradó el modelo? |

**Golden signals (SRE):** latencia, tráfico, errores, saturación. **Para ML, además:** distribución de features, distribución de predicciones, performance real (cuando llega el label), frescura de datos.

---

## Prometheus

**Para qué sirve:** base de datos de series de tiempo que *scrapea* métricas vía HTTP (`/metrics`) y evalúa reglas de alerta.

**Escenario de uso:** latencia p95 de la API de inferencia, tasa de errores 5xx, uso de GPU, lag de Kafka, duración de DAGs.

**Comandos típicos**
```python
from prometheus_client import Counter, Histogram, start_http_server
PRED = Counter("predicciones_total", "Predicciones", ["model_version", "clase"])
LAT  = Histogram("inferencia_segundos", "Latencia", buckets=[.01, .05, .1, .25, .5, 1])
with LAT.time(): y = model.predict(x)
PRED.labels("7", str(y)).inc()
# FastAPI: from prometheus_fastapi_instrumentator import Instrumentator; Instrumentator().instrument(app).expose(app)
```
```promql
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])        # tasa de error
histogram_quantile(0.95, sum by (le) (rate(inferencia_segundos_bucket[5m])))      # p95
sum by (model_version, clase) (rate(predicciones_total[1h]))                      # mezcla de predicciones
```
```yaml
# Regla de alerta
groups:
  - name: ml
    rules:
      - alert: LatenciaAltaInferencia
        expr: histogram_quantile(0.95, sum by (le) (rate(inferencia_segundos_bucket[5m]))) > 0.5
        for: 10m
        labels: { severity: page }
        annotations: { summary: "p95 > 500ms en API churn" }
```
```bash
helm install kps prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
promtool check rules rules.yml
```

**Variantes:** kube-prometheus-stack (Prometheus Operator + Grafana + Alertmanager), Amazon Managed Prometheus, Thanos/Mimir (largo plazo), DCGM exporter (GPU).

**Requisitos:** tipos de métrica (counter, gauge, histogram), labels con baja cardinalidad, PromQL, Alertmanager.

**Aprende más:** prometheus.io/docs · *Prometheus: Up & Running* · promlabs.com/promql-cheat-sheet.

---

## Grafana

**Para qué sirve:** dashboards y alertas sobre múltiples fuentes (Prometheus, Loki, OpenSearch, CloudWatch, OCI Monitoring, SQL).

**Comandos típicos:** importar dashboards por ID (grafana.com/dashboards), dashboards como código (JSON en Git, provisioning, Terraform provider `grafana`), variables de plantilla (`$model_version`), alerting unificado con contact points (Teams/Slack/email).

**Variantes:** Grafana OSS, Grafana Cloud, Amazon Managed Grafana; stack LGTM (Loki logs, Grafana, Tempo trazas, Mimir métricas). Plugin oficial de OCI Metrics/Logs.

**Aprende más:** grafana.com/docs · grafana.com/tutorials.

---

## ELK / OpenSearch

**Para qué sirve:** ingestar, indexar y buscar logs (y eventos) a escala; dashboards con Kibana/OpenSearch Dashboards.

**Escenario de uso:** buscar todos los errores de un `request_id`, analizar logs del agente revisor de ejecuciones, auditoría.

**Comandos típicos**
```python
# Log estructurado JSON (clave para buscar y correlacionar)
import structlog
log = structlog.get_logger()
log.info("prediccion", request_id=rid, model_version="7", latency_ms=42, score=0.83)
```
```text
# KQL (Kibana/OpenSearch Dashboards)
level:ERROR and service:"api-churn" and model_version:7
```
```bash
curl -s "$OS/_cat/indices?v"
curl -s "$OS/logs-*/_search" -H 'Content-Type: application/json' -d '{"query":{"match":{"level":"ERROR"}}}'
```

**Variantes:** Elasticsearch + Logstash/Beats + Kibana; OpenSearch (fork AWS, Amazon OpenSearch Service); Fluent Bit/Fluentd como shippers; Loki (más barato, indexa sólo labels); nativos: CloudWatch Logs Insights, **OCI Logging + Logging Analytics**.

**Requisitos:** logs estructurados, políticas de retención (ILM), mapping de índices, costos de almacenamiento.

**Aprende más:** opensearch.org/docs · elastic.co/guide.

---

## OpenTelemetry (OTel)

**Para qué sirve:** estándar abierto y SDKs para generar trazas, métricas y logs de forma neutral al proveedor, con un *Collector* que los enruta.

**Escenario de uso:** seguir una petición: API Gateway → FastAPI → Feast → modelo → LLM; medir cuánto tarda cada paso (incluyendo llamadas a LLM y tools en agentes).

**Comandos típicos**
```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install
OTEL_SERVICE_NAME=api-churn OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317 \
  opentelemetry-instrument uvicorn app.main:app
```
```python
from opentelemetry import trace
tracer = trace.get_tracer(__name__)
with tracer.start_as_current_span("feature_lookup") as span:
    span.set_attribute("cliente_id", cid)
```

**Variantes:** backends Jaeger, Tempo, AWS X-Ray (ADOT), **OCI APM** (acepta OTLP), Datadog, Langfuse/Phoenix (trazas de LLM sobre OTel).

**Requisitos:** conceptos de span, trace context, sampling, Collector (receivers/processors/exporters).

**Aprende más:** opentelemetry.io/docs.

---

## Evidently

**Para qué sirve:** reportes y tests de data drift, target drift, calidad de datos y performance de modelos (clasificación, regresión, LLM/texto).

**Escenario de uso:** job diario compara predicciones/features de producción vs dataset de referencia; si el drift supera umbral, alerta o dispara reentrenamiento.

**Comandos típicos**
```python
from evidently import Report
from evidently.presets import DataDriftPreset, DataSummaryPreset
report = Report([DataDriftPreset(), DataSummaryPreset()])
snap = report.run(reference_data=ref_df, current_data=prod_df)
snap.save_html("drift.html")
res = snap.dict()      # extraer métricas para Prometheus / decisión en Airflow
```
**Métodos de drift comunes:** PSI, KS test, Jensen-Shannon, Wasserstein, chi-cuadrado (categóricas).

**Variantes:** Evidently OSS (reportes, UI self-hosted), Evidently Cloud; alternativas: WhyLabs/whylogs, NannyML (estima performance sin labels), SageMaker Model Monitor, Vertex Model Monitoring, Arize.

**Requisitos:** dataset de referencia bien definido, logging de inputs/outputs en producción, estadística de distribuciones.

**Aprende más:** docs.evidentlyai.com · curso gratuito "ML Observability" de Evidently.

---

## WhyLabs / whylogs

**Para qué sirve:** *profiling* ligero de datos (whylogs, OSS) que genera estadísticas sin guardar datos crudos (privacidad), con plataforma SaaS de monitoreo.
```python
import whylogs as why
profile = why.log(df).profile(); profile.view().to_pandas()
```
**Nota:** verificar estado actual del servicio SaaS antes de adoptarlo; whylogs sigue siendo útil como librería de perfiles.

---

## Equivalentes cloud
AWS: CloudWatch (métricas, logs, alarmas, Logs Insights), X-Ray, SageMaker Model Monitor. **OCI:** Monitoring (MQL, alarmas → Notifications), Logging, Logging Analytics, APM, Events. Ver [14](14-cloud-aws-azure-gcp.md) y [15](15-cloud-oci.md).

## SLOs y gestión de incidentes
- SLI: % de requests < 300ms; SLO: 99% mensual; error budget = 1%.
- Alertar por **síntomas** (SLO quemándose), no por causas; runbook enlazado en cada alerta.
- Post-mortem sin culpables: línea de tiempo, causa raíz, acciones con dueño.
- Lectura: *Site Reliability Engineering* y *The SRE Workbook* (sre.google, gratis).

## Hilando soluciones
**Observabilidad completa de una API de ML:** FastAPI instrumentada con OTel (trazas → Tempo/OCI APM) + `/metrics` (Prometheus) + logs JSON con `request_id` y `model_version` (Fluent Bit → OpenSearch/OCI Logging) → predicciones también a Kafka → job Airflow diario con Evidently → métrica `drift_score` publicada en Prometheus → Grafana unifica todo → Alertmanager notifica a Teams → el *agente revisor de ejecuciones* ([13](13-optimizacion-llm-agentes.md)) lee logs y propone causas.
