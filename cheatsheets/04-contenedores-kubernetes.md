# 04 · Contenedores y orquestación: Docker, Kubernetes, Helm, HPA/KEDA

> Naturaleza: **empaquetado y ejecución**. Convierten código + modelo en una unidad portable y la operan a escala.
> [← Índice](00-indice-roadmap.md) · Anterior: [03](03-procesamiento-datos.md) · Siguiente: [05 CI/CD](05-ci-cd.md)

---

## Docker

**Para qué sirve:** empaquetar aplicación + dependencias + runtime en una imagen inmutable que corre igual en cualquier lado.

**Escenario de uso:** imagen de la API de inferencia, imagen de entrenamiento con CUDA, devcontainers (`dev_container_images`), Lambdas basadas en imagen.

**Comandos típicos**
```bash
docker build -t api-modelo:1.0 .
docker build --platform linux/amd64 -t api-modelo:1.0 .     # desde Mac ARM
docker run --rm -p 8080:8080 -e MODEL_URI=... api-modelo:1.0
docker run --gpus all nvidia/cuda:12.4.1-runtime-ubuntu22.04 nvidia-smi
docker ps -a; docker logs -f <id>; docker exec -it <id> sh
docker images; docker system prune -af                     # limpieza
docker tag api-modelo:1.0 <registry>/api-modelo:1.0 && docker push <registry>/api-modelo:1.0
docker compose up -d --build; docker compose logs -f
docker scout cves api-modelo:1.0  # o: trivy image api-modelo:1.0
```
```dockerfile
# Multi-stage, imagen mínima, usuario no root
FROM python:3.12-slim AS build
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=build /install /usr/local
COPY src/ ./src
RUN useradd -m appuser
USER appuser
EXPOSE 8080
HEALTHCHECK CMD python -c "import urllib.request;urllib.request.urlopen('http://localhost:8080/health')"
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**Variantes:** Docker / Podman (sin daemon) / Buildah / Kaniko (builds en CI sin privilegios); imágenes base: slim, distroless, alpine, NVIDIA CUDA; registries: ECR, OCIR, ACR, Artifact Registry, Docker Hub.

**Requisitos:** Docker Desktop o engine (WSL2 en Windows), NVIDIA Container Toolkit para GPU; capas y caché, `.dockerignore`, redes y volúmenes, seguridad de imágenes.

**Conexión:** CI construye y publica ([05](05-ci-cd.md)), Trivy escanea ([12](12-seguridad-gobierno.md)), Kubernetes/ECS/OKE ejecutan, BentoML genera imágenes ([10](10-serving.md)), SageMaker/OCI Data Science aceptan BYOC ([14](14-cloud-aws-azure-gcp.md)/[15](15-cloud-oci.md)).

**Aprende más:** docs.docker.com/get-started · *Docker Deep Dive* (N. Poulton) · Play with Docker.

---

## Kubernetes (K8s)

**Para qué sirve:** orquestar contenedores: despliegue declarativo, autorecuperación, escalado, service discovery, rollouts.

**Escenario de uso:** APIs de inferencia con alta disponibilidad, jobs de entrenamiento con GPU, plataforma de ML (Kubeflow, KServe, Argo).

**Comandos típicos**
```bash
kubectl config get-contexts; kubectl config use-context prod
kubectl get pods -n ml -o wide; kubectl get all -n ml
kubectl describe pod <pod> -n ml                  # eventos: ImagePullBackOff, OOMKilled...
kubectl logs -f <pod> -c <container> --previous   # logs del contenedor que crasheó
kubectl exec -it <pod> -- sh
kubectl apply -f k8s/ ; kubectl delete -f k8s/
kubectl rollout status deploy/api-modelo; kubectl rollout undo deploy/api-modelo
kubectl scale deploy/api-modelo --replicas=5
kubectl port-forward svc/api-modelo 8080:80
kubectl top pods -n ml                             # requiere metrics-server
kubectl create secret generic db --from-literal=pass=xxx
kubectl get events -n ml --sort-by=.lastTimestamp
```
```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: api-modelo, namespace: ml }
spec:
  replicas: 2
  selector: { matchLabels: { app: api-modelo } }
  strategy: { rollingUpdate: { maxSurge: 1, maxUnavailable: 0 } }
  template:
    metadata: { labels: { app: api-modelo } }
    spec:
      containers:
        - name: api
          image: <registry>/api-modelo:1.0
          ports: [{ containerPort: 8080 }]
          resources:
            requests: { cpu: "500m", memory: "1Gi" }
            limits:   { memory: "2Gi" }          # GPU: nvidia.com/gpu: 1
          readinessProbe: { httpGet: { path: /health, port: 8080 } }
          livenessProbe:  { httpGet: { path: /health, port: 8080 }, initialDelaySeconds: 20 }
          envFrom: [{ secretRef: { name: db } }]
```

**Objetos clave:** Pod, Deployment, StatefulSet, Job/CronJob, Service, Ingress/Gateway, ConfigMap, Secret, PVC, Namespace, ServiceAccount/RBAC, NetworkPolicy.

**Variantes:** EKS (AWS), **OKE (OCI)**, AKS, GKE; local: kind, minikube, k3d, Docker Desktop.

**Requisitos:** cluster + `kubectl` + kubeconfig; networking (Service, DNS, Ingress), requests/limits, probes, RBAC, troubleshooting de pods.

**Conexión:** Helm/Kustomize empaquetan manifiestos; Argo CD despliega vía GitOps ([05](05-ci-cd.md)); KServe/Seldon/Triton corren encima ([10](10-serving.md)); Kubeflow/Argo Workflows ([07](07-orquestacion-pipelines.md)); Prometheus/Grafana ([11](11-monitoreo-observabilidad.md)); Vault/External Secrets ([12](12-seguridad-gobierno.md)); Terraform crea el cluster ([06](06-iac.md)).

**Aprende más:** kubernetes.io/docs/tutorials · *Kubernetes Up & Running* · killercoda.com (labs gratis) · certificaciones CKAD/CKA.

---

## Helm

**Para qué sirve:** gestor de paquetes de Kubernetes: plantillas parametrizables (charts) con versionado y rollback.

**Escenario de uso:** desplegar la misma API en dev/qa/prod con distintos `values`, instalar componentes (Prometheus, KServe, Airflow, MLflow).

**Comandos típicos**
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update
helm search repo airflow
helm create api-modelo                                  # scaffold
helm template api-modelo ./chart -f values-prod.yaml    # render sin aplicar
helm lint ./chart
helm upgrade --install api-modelo ./chart -n ml -f values-prod.yaml --set image.tag=1.1 --atomic
helm list -n ml; helm history api-modelo -n ml
helm rollback api-modelo 3 -n ml
helm uninstall api-modelo -n ml
```

**Variantes:** Kustomize (overlays sin plantillas, nativo en `kubectl -k`), Helmfile (múltiples releases), charts OCI en registries.

**Requisitos:** Kubernetes; Go templates, estructura de chart (`Chart.yaml`, `values.yaml`, `templates/`).

**Conexión:** Argo CD/Flux sincronizan charts ([05](05-ci-cd.md)); Terraform `helm_release` ([06](06-iac.md)).

**Aprende más:** helm.sh/docs · artifacthub.io (catálogo de charts).

---

## Autoscaling: HPA y KEDA

**Para qué sirve:** ajustar réplicas automáticamente según carga. **HPA** usa CPU/memoria/métricas custom; **KEDA** escala por eventos (lag de Kafka, colas SQS, cron, Prometheus) e incluso a cero.

**Escenario de uso:** API de inferencia con picos de tráfico (HPA), consumidores de cola o batch esporádico (KEDA, scale-to-zero).

**Comandos típicos**
```bash
kubectl autoscale deploy api-modelo --cpu-percent=70 --min=2 --max=10
kubectl get hpa -w
helm install keda kedacore/keda -n keda --create-namespace
kubectl get scaledobjects -A
```
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata: { name: scoring-consumer }
spec:
  scaleTargetRef: { name: scoring-consumer }
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
    - type: kafka
      metadata: { bootstrapServers: broker:9092, consumerGroup: scoring, topic: eventos, lagThreshold: "100" }
```

**Variantes:** VPA (ajusta requests), Cluster Autoscaler / Karpenter (nodos), autoscaling nativo de KServe (Knative).

**Requisitos:** metrics-server, requests definidos; métricas, cold start, límites de cuota.

**Conexión:** Kafka/SQS/OCI Queue como fuentes ([03](03-procesamiento-datos.md)), Prometheus como métrica ([11](11-monitoreo-observabilidad.md)), costos ([13](13-optimizacion-llm-agentes.md)).

**Aprende más:** kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale · keda.sh/docs.

---

## Hilando soluciones
**API de inferencia productiva:** Dockerfile multi-stage → CI construye, escanea con Trivy y publica en ECR/OCIR → Helm chart con `values-prod.yaml` → Argo CD sincroniza en EKS/OKE → HPA escala por CPU/latencia → Prometheus scrapea `/metrics` → rollback con `helm rollback` si falla el SLO.
