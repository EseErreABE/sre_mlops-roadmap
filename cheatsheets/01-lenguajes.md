# 01 · Lenguajes: Python, SQL, Bash (+ Go/Java)

> Naturaleza: **fundamento**. Todo el resto del stack se opera, automatiza o extiende con estos lenguajes.
> [← Índice](00-indice-roadmap.md) · Siguiente: [02 Frameworks ML](02-frameworks-ml.md)

---

## Python

**Para qué sirve:** lenguaje principal de ML, datos, APIs de serving, DAGs de orquestación, automatización y agentes LLM.

**Escenario de uso:** entrenar y servir modelos, escribir pipelines (Airflow/Prefect), Lambdas/OCI Functions, librerías internas (`pe_analytics_libraries`), scripts de soporte.

**Comandos típicos**
```bash
# Entornos y dependencias
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt
pip freeze > requirements.txt
uv venv && uv pip install -r requirements.txt           # alternativa rápida
poetry init / poetry add fastapi / poetry install        # gestión con lockfile

# Calidad
ruff check . --fix        # lint + autofix
ruff format .             # formateo (reemplaza black)
mypy src/                 # tipado estático
pytest -q --cov=src       # tests + cobertura
pre-commit install && pre-commit run --all-files

# Empaquetado
python -m build           # genera wheel/sdist (pyproject.toml)
pip install -e .          # instalación editable
```
```python
# Patrones que se usan a diario
from pathlib import Path
import logging, os
from dataclasses import dataclass
from typing import Any
import asyncio

logging.basicConfig(level=os.getenv("LOG_LEVEL", "INFO"))
log = logging.getLogger(__name__)

@dataclass(frozen=True)
class Settings:
    bucket: str = os.environ["BUCKET"]

async def fetch_all(urls: list[str]) -> list[Any]:
    return await asyncio.gather(*(fetch(u) for u in urls))
```

**Variantes:** CPython 3.10–3.12 (estándar), PyPy (rendimiento puro Python), gestores pip / uv / Poetry / conda (conda útil con dependencias CUDA nativas).

**Requisitos**
- Técnicos: Python ≥3.10, gestor de entornos, IDE con linter.
- Conocimiento: tipado, POO y dataclasses, manejo de excepciones, `asyncio` (APIs con FastAPI), concurrencia (threads vs procesos vs async, ver repo `concurrency_samples`), empaquetado `pyproject.toml`, testing con pytest.

**Conexión con el stack:** base de [Frameworks ML](02-frameworks-ml.md), [Datos](03-procesamiento-datos.md) (Pandas/PySpark), [Serving](10-serving.md) (FastAPI/BentoML), [Orquestación](07-orquestacion-pipelines.md) (DAGs), [Tracking](08-versionado-tracking.md) (SDK MLflow), [LLM/Agentes](13-optimizacion-llm-agentes.md), SDKs cloud (`boto3`, `oci`).

**Aprende más:** docs.python.org (tutorial oficial) · *Fluent Python* (L. Ramalho) · Real Python · docs de `uv`, `ruff`, `pytest`.

---

## SQL

**Para qué sirve:** consultar, transformar y validar datos estructurados en data warehouses y bases transaccionales.

**Escenario de uso:** extracción de datasets de entrenamiento, features agregadas, validaciones de calidad, consultas en Athena/Redshift/Autonomous DB, modelos dbt, text-to-SQL con LLM.

**Comandos típicos**
```sql
-- Agregación + ventana (features típicas)
SELECT cliente_id,
       COUNT(*)                                   AS n_llamadas_30d,
       AVG(duracion_seg)                          AS dur_prom,
       ROW_NUMBER() OVER (PARTITION BY cliente_id ORDER BY fecha DESC) AS rn
FROM   llamadas
WHERE  fecha >= CURRENT_DATE - INTERVAL '30' DAY
GROUP  BY cliente_id, fecha, duracion_seg;

-- CTE para legibilidad
WITH base AS (SELECT * FROM ventas WHERE anio = 2026)
SELECT region, SUM(monto) FROM base GROUP BY region;

-- Evitar fuga temporal: join "point-in-time"
SELECT e.*, f.score
FROM eventos e
LEFT JOIN features f
  ON f.id = e.id AND f.fecha_feature <= e.fecha_evento;

EXPLAIN ANALYZE SELECT ...;   -- plan de ejecución
```

**Variantes / dialectos:** ANSI, PostgreSQL, Presto/Trino (Athena), Redshift, Spark SQL, Oracle SQL / PL-SQL (Autonomous DB en OCI), BigQuery SQL, T-SQL.

**Requisitos**
- Técnicos: cliente SQL (DBeaver, psql, SQL Developer), credenciales del warehouse.
- Conocimiento: joins, CTEs, funciones de ventana, particionamiento, índices, planes de ejecución, modelado dimensional (estrella), point-in-time correctness.

**Conexión con el stack:** dbt ([03](03-procesamiento-datos.md)), Feast offline store ([09](09-feature-store-calidad.md)), Athena/Redshift ([14](14-cloud-aws-azure-gcp.md)), Autonomous DB ([15](15-cloud-oci.md)), text-to-SQL con LLM ([13](13-optimizacion-llm-agentes.md)).

**Aprende más:** Mode SQL Tutorial · *SQL Performance Explained* (use-the-index-luke.com) · docs Trino/Athena · Oracle Live SQL.

---

## Bash

**Para qué sirve:** automatizar tareas del sistema, *glue* de CI/CD, entrypoints de contenedores, operación de servidores.

**Escenario de uso:** scripts en pipelines de CI, `entrypoint.sh` de imágenes Docker, troubleshooting en pods/VMs, procesamiento rápido de logs.

**Comandos típicos**
```bash
#!/usr/bin/env bash
set -euo pipefail                      # fallar ante errores y variables no definidas
trap 'echo "error en línea $LINENO"' ERR

for f in data/*.csv; do echo "$f"; done
grep -rn "ERROR" logs/ | sort | uniq -c | sort -nr | head
tail -f app.log | grep --line-buffered "Timeout"
find . -name "*.pkl" -mtime +30 -delete
jq '.items[] | {name: .metadata.name}' pods.json
curl -sS -X POST localhost:8080/predict -H 'Content-Type: application/json' -d @payload.json
xargs -P 4 -I{} python procesa.py {} < lista.txt     # paralelismo simple
df -h; free -m; top; ps aux | grep python; ss -lntp   # diagnóstico
```

**Variantes:** bash, zsh, sh (POSIX, en imágenes alpine), PowerShell (Windows), Git Bash.

**Requisitos**
- Técnicos: terminal Linux/WSL o Git Bash.
- Conocimiento: pipes y redirecciones, variables de entorno, exit codes, permisos, `jq`, `curl`, `grep/sed/awk`.

**Conexión con el stack:** [Docker](04-contenedores-kubernetes.md) (entrypoints), [CI/CD](05-ci-cd.md) (steps), [IaC](06-iac.md) (user-data, Ansible), CLIs cloud (`aws`, `oci`).

**Aprende más:** *The Linux Command Line* (W. Shotts, gratis) · shellcheck.net · explainshell.com.

---

## Go / Java (deseable)

**Para qué sirve:** servicios de alto rendimiento o baja latencia, operadores de Kubernetes (Go), integración con sistemas empresariales (Java).

**Escenario de uso:** gateways de inferencia, sidecars, operadores/controladores de K8s, consumidores Kafka de alto throughput.

**Comandos típicos**
```bash
go mod init svc && go build ./... && go test ./...
mvn clean package / gradle build
```

**Requisitos de conocimiento:** concurrencia (goroutines/channels; threads JVM), tipado estático, gRPC/protobuf.

**Conexión:** Kubernetes ([04](04-contenedores-kubernetes.md)), Kafka ([03](03-procesamiento-datos.md)), Triton/gRPC ([10](10-serving.md)).

**Aprende más:** go.dev/tour · *Effective Java* (J. Bloch).

---

## Hilando soluciones
**Script de reentrenamiento ligero:** Bash (`entrypoint.sh` con `set -euo pipefail`) → Python lee datos vía SQL (Athena / Autonomous DB) → entrena con scikit-learn → registra en MLflow → sale con exit code que el orquestador (Airflow) interpreta.
