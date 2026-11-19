# 05 · CI/CD: Git, GitHub Actions, GitLab CI, Jenkins, Azure DevOps

> Naturaleza: **automatización del ciclo de entrega**. Lleva código, datos y modelos de un commit a producción con quality gates.
> [← Índice](00-indice-roadmap.md) · Anterior: [04](04-contenedores-kubernetes.md) · Siguiente: [06 IaC](06-iac.md)

---

## Git

**Para qué sirve:** control de versiones distribuido; fuente de verdad del código, configuración, IaC y (vía DVC) punteros a datos/modelos.

**Escenario de uso:** todo. Branching, code review con PR, trazabilidad commit → modelo desplegado.

**Comandos típicos**
```bash
git switch -c feature/drift-monitor
git add -p                                  # stage selectivo
git commit -m "feat: monitor de drift con evidently"
git fetch && git rebase origin/main
git push -u origin feature/drift-monitor
git log --oneline --graph --decorate -20
git diff main...HEAD
git stash push -m wip; git stash pop
git restore --staged archivo; git restore archivo
git revert <sha>                            # deshacer en ramas compartidas
git bisect start; git bisect bad; git bisect good <sha>   # encontrar el commit que rompió
git tag -a v1.2.0 -m "modelo churn v1.2" && git push --tags
```

**Variantes / flujos:** trunk-based (recomendado para CI/CD), GitHub Flow, GitFlow; Conventional Commits; hosting: GitHub, GitLab, Bitbucket, Azure Repos, OCI DevOps Code Repositories.

**Requisitos:** Git instalado + SSH/token; ramas, merge vs rebase, resolución de conflictos, `.gitignore` (nunca datos ni secretos).

**Conexión:** dispara pipelines CI, base de GitOps (Argo CD), DVC versiona datos encima ([08](08-versionado-tracking.md)), pre-commit + gitleaks ([12](12-seguridad-gobierno.md)), Auto Code Reviewer sobre PRs ([13](13-optimizacion-llm-agentes.md)).

**Aprende más:** git-scm.com/book (Pro Git, gratis) · learngitbranching.js.org · ohshitgit.com.

---

## Pipelines CI/CD (concepto común)

Etapas típicas de un pipeline de ML:
1. **Lint & tests** (ruff, mypy, pytest) — 2. **Validación de datos** (Great Expectations/Pandera) — 3. **Seguridad** (SAST, gitleaks, Trivy) — 4. **Build** imagen — 5. **Entrenamiento/evaluación** (opcional, o disparar orquestador) — 6. **Quality gate de modelo** (métrica ≥ umbral y ≥ modelo actual) — 7. **Registro** en model registry — 8. **Deploy** a staging → pruebas de humo → aprobación → producción (canary/blue-green) — 9. **Rollback** automático si fallan SLOs.

CI/CD/**CT** (continuous training): el pipeline de reentrenamiento puede dispararse por calendario o drift, no sólo por commit.

---

## GitHub Actions

**Para qué sirve:** CI/CD integrado en GitHub, workflows YAML disparados por eventos.

**Comandos típicos**
```yaml
# .github/workflows/ci.yml
name: ci
on:
  pull_request:
  push: { branches: [main] }
permissions: { contents: read, id-token: write }   # OIDC hacia la nube, sin llaves estáticas
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12", cache: pip }
      - run: pip install -r requirements.txt
      - run: ruff check . && pytest -q --cov=src
  build:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with: { role-to-assume: arn:aws:iam::123:role/gha, aws-region: us-east-1 }
      - uses: aws-actions/amazon-ecr-login@v2
      - run: docker build -t $REG/api:${{ github.sha }} . && docker push $REG/api:${{ github.sha }}
      - uses: aquasecurity/trivy-action@0.28.0
        with: { image-ref: "$REG/api:${{ github.sha }}", exit-code: "1", severity: CRITICAL }
```
```bash
gh run list; gh run watch; gh run rerun <id> --failed
gh workflow run retrain.yml -f dataset=2026-11
act -j test           # ejecutar workflows localmente
```

**Requisitos:** repo en GitHub, secrets/OIDC configurados; YAML, matrices, caché, environments con aprobaciones, reusable workflows.

---

## GitLab CI

```yaml
# .gitlab-ci.yml
stages: [test, build, deploy]
test:
  image: python:3.12
  script: [pip install -r requirements.txt, pytest -q]
build:
  stage: build
  image: docker:27
  services: [docker:27-dind]
  script: [docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA ., docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA]
  rules: [{ if: '$CI_COMMIT_BRANCH == "main"' }]
deploy_prod:
  stage: deploy
  when: manual
  environment: production
  script: [helm upgrade --install api ./chart --set image.tag=$CI_COMMIT_SHA]
```
`gitlab-runner exec docker test` · runners self-hosted · `include:` para plantillas compartidas.

---

## Jenkins

```groovy
// Jenkinsfile (declarativo)
pipeline {
  agent { docker { image 'python:3.12' } }
  stages {
    stage('Test')  { steps { sh 'pip install -r requirements.txt && pytest -q' } }
    stage('Build') { when { branch 'main' } steps { sh 'docker build -t api:$GIT_COMMIT .' } }
    stage('Deploy'){ input { message 'Desplegar a prod?' } steps { sh './deploy.sh' } }
  }
  post { failure { echo 'notificar a Teams' } }
}
```
Útil on-premise y con plugins legacy; requiere mantener el servidor y sus plugins.

---

## Azure DevOps Pipelines

```yaml
trigger: [main]
pool: { vmImage: ubuntu-latest }
stages:
  - stage: Test
    jobs: [{ job: t, steps: [{ script: pip install -r requirements.txt && pytest -q }] }]
  - stage: Deploy
    dependsOn: Test
    jobs:
      - deployment: prod
        environment: production        # aprobaciones en el environment
        strategy: { runOnce: { deploy: { steps: [{ script: ./deploy.sh }] } } }
```

---

## Equivalentes cloud y GitOps
- **AWS:** CodePipeline + CodeBuild (`buildspec.yml`) + ECR. **OCI:** OCI DevOps (build pipelines `build_spec.yaml`, deployment pipelines a OKE/Functions/Instances) + OCIR.
- **GitOps (Argo CD / Flux):** el cluster se sincroniza con el estado declarado en Git. `argocd app sync api-modelo`, `argocd app rollback api-modelo <rev>`.

**Variantes de despliegue:** rolling, blue-green, canary (Argo Rollouts, Flagger, KServe canary), shadow (tráfico espejado sin responder).

**Requisitos de conocimiento (todas):** YAML, gestión de secretos en CI, OIDC federado, artefactos y caché, entornos y aprobaciones, estrategias de release y rollback.

**Conexión con el stack:** Docker ([04](04-contenedores-kubernetes.md)), Terraform plan/apply en PR ([06](06-iac.md)), dispara Airflow/Kubeflow ([07](07-orquestacion-pipelines.md)), consulta MLflow registry para quality gate ([08](08-versionado-tracking.md)), Trivy/Snyk/SAST ([12](12-seguridad-gobierno.md)), Auto Code Reviewer con LLM ([13](13-optimizacion-llm-agentes.md)), notificaciones a Teams (`sendnotificationsteams`).

**Aprende más:** docs.github.com/actions · docs.gitlab.com/ee/ci · jenkins.io/doc · learn.microsoft.com/azure/devops · argo-cd.readthedocs.io · *Continuous Delivery* (Humble & Farley) · Google "MLOps: Continuous delivery and automation pipelines in ML" (cloud.google.com/architecture).

---

## Hilando soluciones
**Promoción de modelo con quality gate:** PR → GitHub Actions (tests + Trivy + Auto Code Reviewer) → merge → pipeline dispara entrenamiento en SageMaker/OCI Data Science → script compara métrica vs modelo "Production" en MLflow → si mejora, transiciona alias `champion` → Argo CD despliega canary 10% → Prometheus valida SLO → 100% o rollback automático.
