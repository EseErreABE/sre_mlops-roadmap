# 12 · Seguridad y gobierno: Vault, secretos, RBAC, Trivy, Snyk, SAST, auditoría

> Naturaleza: **transversal / protección y cumplimiento**. Aplica a cada capa: código, imágenes, infraestructura, datos, modelos y accesos.
> [← Índice](00-indice-roadmap.md) · Anterior: [11](11-monitoreo-observabilidad.md) · Siguiente: [13 Optimización y LLM](13-optimizacion-llm-agentes.md)

| Capa | Riesgo típico | Control | Herramienta |
|---|---|---|---|
| Código | Vulnerabilidades, secretos hardcodeados | SAST, secret scanning, code review | Semgrep, Bandit, CodeQL, SonarQube, gitleaks |
| Dependencias | Librerías con CVEs | SCA | Snyk, pip-audit, Dependabot, Trivy fs |
| Imágenes | CVEs del SO/base | Escaneo de imágenes, firma | Trivy, Grype, cosign |
| IaC | Buckets públicos, puertos abiertos | Escaneo de IaC, policy-as-code | Checkov, Trivy config, tfsec, OPA/Conftest |
| Secretos en runtime | Fuga de credenciales | Gestor de secretos, rotación | Vault, AWS Secrets Manager, **OCI Vault** |
| Accesos | Privilegios excesivos | IAM mínimo privilegio, RBAC | IAM, OCI policies/compartments, K8s RBAC |
| Datos | Exposición de PII | Cifrado, enmascaramiento, anonimización | KMS, OCI Vault keys, Data Safe, Macie |
| Modelos/LLM | Prompt injection, fuga de datos, sesgo | Guardrails, evaluación, model cards | Bedrock Guardrails, NeMo Guardrails, OWASP LLM Top 10 |

---

## HashiCorp Vault / gestores de secretos

**Para qué sirve:** almacenar, controlar y auditar acceso a secretos (passwords, API keys, certificados) y generar credenciales dinámicas de corta duración.

**Escenario de uso:** la API de inferencia necesita la password de la DB o la API key del LLM sin que estén en el código ni en el repo.

**Comandos típicos**
```bash
# Vault
vault login -method=oidc
vault kv put secret/ml/api-churn db_pass=xxx llm_key=yyy
vault kv get -field=db_pass secret/ml/api-churn
vault policy write api-churn policy.hcl
vault auth enable kubernetes          # pods se autentican con su ServiceAccount

# AWS Secrets Manager
aws secretsmanager get-secret-value --secret-id ml/api-churn --query SecretString --output text

# OCI Vault
oci vault secret list --compartment-id $C
oci secrets secret-bundle get --secret-id $SECRET_OCID --query 'data."secret-bundle-content".content' --raw-output | base64 -d
```
```python
import boto3, json
secret = json.loads(boto3.client("secretsmanager").get_secret_value(SecretId="ml/api-churn")["SecretString"])
```

**Variantes:** Vault OSS/Enterprise/HCP, AWS Secrets Manager / SSM Parameter Store, **OCI Vault (secrets + master keys)**, Azure Key Vault, GCP Secret Manager; en K8s: External Secrets Operator, Secrets Store CSI Driver, Sealed Secrets/SOPS para GitOps.

**Requisitos:** IAM del proveedor, conceptos de lease/rotación, cifrado envelope (KMS), principio de mínimo privilegio.

**Conexión:** K8s ([04](04-contenedores-kubernetes.md)), CI con OIDC en vez de llaves estáticas ([05](05-ci-cd.md)), Terraform (`vault`/`oci_vault_secret`) ([06](06-iac.md)), Lambdas/Functions ([14](14-cloud-aws-azure-gcp.md)/[15](15-cloud-oci.md)).

**Aprende más:** developer.hashicorp.com/vault/tutorials · docs AWS Secrets Manager · docs.oracle.com → OCI Vault.

---

## IAM y RBAC

**Para qué sirve:** definir *quién* (usuario, rol, servicio) puede hacer *qué* sobre *qué recurso*.

**Comandos típicos**
```bash
# AWS
aws sts get-caller-identity
aws iam simulate-principal-policy --policy-source-arn <role> --action-names s3:GetObject --resource-arns arn:aws:s3:::modelos/*
# Kubernetes
kubectl auth can-i create deployments -n ml --as system:serviceaccount:ml:api
kubectl create rolebinding api-read --role=reader --serviceaccount=ml:api -n ml
```
```text
# OCI policy (lenguaje de políticas por compartment)
Allow group MLOps to manage data-science-family in compartment analytics
Allow dynamic-group ds-jobs to read secret-bundles in compartment analytics
Allow any-user to read objects in compartment analytics where request.principal.type = 'datasciencejob'
```

**Conceptos clave:** roles vs usuarios, *instance/resource principals* (OCI) e *IAM roles for service accounts / Pod Identity* (EKS), federación SSO, separación de ambientes (cuentas AWS / compartments OCI), break-glass.

**Aprende más:** docs AWS IAM best practices · docs OCI IAM policies · kubernetes.io/docs/reference/access-authn-authz/rbac.

---

## Escaneo de vulnerabilidades: Trivy, Snyk y SAST

**Comandos típicos**
```bash
trivy image --severity HIGH,CRITICAL --exit-code 1 <registry>/api:1.0
trivy fs --scanners vuln,secret,misconfig .
trivy config infra/                                     # IaC
trivy image --format cyclonedx -o sbom.json <img>       # SBOM
snyk test --file=requirements.txt; snyk container test <img>; snyk iac test infra/
pip-audit -r requirements.txt
bandit -r src/ -ll
semgrep --config p/python --config p/secrets src/
gitleaks detect --source . --redact
checkov -d infra/
cosign sign <img>; cosign verify <img>                  # firma de imágenes
```
```yaml
# .pre-commit-config.yaml
repos:
  - { repo: https://github.com/gitleaks/gitleaks, rev: v8.21.2, hooks: [{ id: gitleaks }] }
  - { repo: https://github.com/astral-sh/ruff-pre-commit, rev: v0.7.4, hooks: [{ id: ruff }] }
  - { repo: https://github.com/PyCQA/bandit, rev: 1.7.10, hooks: [{ id: bandit, args: ["-ll"] }] }
```

**Variantes:** CodeQL/GitHub Advanced Security, SonarQube, Grype, ECR scanning (Inspector), OCIR vulnerability scanning, Cloud Guard (OCI), Security Hub (AWS).

**Requisitos:** entender CVSS, falsos positivos, políticas de excepción con fecha de expiración, SBOM y supply chain (SLSA).

**Conexión:** quality gates en CI ([05](05-ci-cd.md)), escaneo de imágenes antes de push ([04](04-contenedores-kubernetes.md)), Auto Code Reviewer con LLM agrega contexto ([13](13-optimizacion-llm-agentes.md)).

**Aprende más:** aquasecurity.github.io/trivy · docs.snyk.io · semgrep.dev/docs · owasp.org (Top 10, ASVS, **OWASP Top 10 for LLM Applications**).

---

## Gobierno, auditoría y trazabilidad

**Qué cubrir:**
- **Auditoría de accesos:** AWS CloudTrail, OCI Audit, Vault audit device, K8s audit logs.
- **Trazabilidad de modelos:** linaje datos→código→modelo→despliegue ([08](08-versionado-tracking.md)); quién aprobó la promoción.
- **Documentación:** model cards (propósito, datos, métricas, limitaciones, sesgos), datasheets de datasets.
- **Privacidad:** clasificación de datos, minimización, anonimización (repo `etl_65_aws_anon`), retención, Ley N.º 29733 de protección de datos personales (Perú).
- **IA responsable:** evaluación de sesgo por segmentos, explicabilidad (SHAP), revisión humana en decisiones sensibles, guardrails en LLM.

```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteBucket
oci audit event list --compartment-id $C --start-time 2026-11-01T00:00:00Z --end-time 2026-11-19T00:00:00Z
```

**Aprende más:** NIST AI RMF · ISO/IEC 42001 · Google Model Cards paper · docs OCI Data Safe.

## Hilando soluciones
**Pipeline seguro de punta a punta:** pre-commit (gitleaks, bandit) → CI con SAST (Semgrep), SCA (pip-audit/Snyk), Trivy de imagen e IaC, SBOM y firma cosign → push a ECR/OCIR con scanning → K8s con External Secrets desde Vault/OCI Vault, RBAC y NetworkPolicy → workload identity (sin llaves) → CloudTrail/OCI Audit → model card adjunta en el registry.
