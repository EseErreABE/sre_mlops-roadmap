# 06 · Infraestructura como Código: Terraform, Ansible, Pulumi

> Naturaleza: **provisión y configuración declarativa de infraestructura**. Crea y mantiene de forma reproducible los recursos donde corre todo lo demás.
> [← Índice](00-indice-roadmap.md) · Anterior: [05](05-ci-cd.md) · Siguiente: [07 Orquestación](07-orquestacion-pipelines.md)

| Herramienta | Enfoque | Uso principal |
|---|---|---|
| Terraform / OpenTofu | Declarativo (HCL), con estado | Provisionar recursos cloud (redes, clusters, buckets, IAM) |
| Pulumi | Declarativo con lenguajes reales (Python/TS/Go) | Igual que Terraform, cuando se quiere lógica en Python |
| Ansible | Procedural/idempotente, sin agente (SSH) | Configurar servidores/VMs, instalar software |
| Nativos | CloudFormation/CDK (AWS), **OCI Resource Manager** (Terraform gestionado), Bicep (Azure) | Cuando se queda en un solo proveedor |

---

## Terraform

**Para qué sirve:** definir infraestructura en HCL, planificar cambios y aplicarlos de forma reproducible en cualquier proveedor.

**Escenario de uso:** crear buckets S3/Object Storage, clusters EKS/OKE, roles IAM/políticas OCI, Lambdas/Functions, endpoints SageMaker, ambientes dev/qa/prod idénticos (repo `infra_terraform`).

**Comandos típicos**
```bash
terraform init                      # descarga providers, configura backend
terraform fmt -recursive && terraform validate
terraform plan -out=tfplan -var-file=env/prod.tfvars
terraform apply tfplan
terraform destroy -target=aws_s3_bucket.tmp
terraform state list; terraform state show aws_s3_bucket.data
terraform import aws_s3_bucket.data mi-bucket       # traer recurso existente
terraform output -json
terraform workspace new qa                           # alternativa: carpetas por ambiente
tflint; checkov -d .  # o: trivy config .           # lint y seguridad de IaC
```
```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
    oci = { source = "oracle/oci",    version = "~> 6.0" }
  }
  backend "s3" { bucket = "tf-state-ml", key = "mlops/prod.tfstate", region = "us-east-1", dynamodb_table = "tf-lock" }
}

module "api_lambda" {
  source        = "./modules/lambda"
  name          = "scoring-${var.env}"
  image_uri     = var.image_uri
  memory_mb     = 1024
}

resource "oci_objectstorage_bucket" "modelos" {
  compartment_id = var.compartment_ocid
  namespace      = data.oci_objectstorage_namespace.ns.namespace
  name           = "modelos-${var.env}"
  versioning     = "Enabled"
}
```

**Variantes:** Terraform (HashiCorp, licencia BSL), OpenTofu (fork open source), Terragrunt (DRY multi-ambiente), Terraform Cloud/HCP, Atlantis (plan/apply desde PRs), OCI Resource Manager (stacks Terraform gestionados).

**Requisitos**
- Técnicos: CLI Terraform, credenciales cloud (perfil AWS, `~/.oci/config`), backend remoto con locking.
- Conocimiento: providers, recursos, data sources, módulos, variables/outputs, estado y drift, ciclo de vida (`create_before_destroy`, `prevent_destroy`), redes cloud (VPC/VCN, subnets, gateways).

**Conexión con el stack:** crea clusters K8s y `helm_release` ([04](04-contenedores-kubernetes.md)), corre en CI con `plan` en PR y `apply` en merge ([05](05-ci-cd.md)), provisiona MWAA/Airflow, SageMaker, OCI Data Science ([14](14-cloud-aws-azure-gcp.md)/[15](15-cloud-oci.md)), IAM/Vault/KMS ([12](12-seguridad-gobierno.md)), escaneo con Checkov/Trivy.

**Aprende más:** developer.hashicorp.com/terraform/tutorials · registry.terraform.io (docs de providers aws/oci) · *Terraform: Up & Running* (Y. Brikman) · docs OCI Resource Manager.

---

## Pulumi

**Para qué sirve:** IaC usando Python/TypeScript/Go, con bucles, funciones y tests nativos del lenguaje.

**Escenario de uso:** equipos Python que quieren abstracciones reutilizables (componentes) y testing unitario de infraestructura.

**Comandos típicos**
```bash
pulumi new aws-python
pulumi stack init prod; pulumi config set aws:region us-east-1
pulumi config set --secret dbPassword xxx
pulumi preview; pulumi up; pulumi destroy
pulumi stack output
```
```python
import pulumi, pulumi_aws as aws
bucket = aws.s3.BucketV2("modelos", force_destroy=False)
pulumi.export("bucket", bucket.id)
```

**Variantes:** Pulumi Cloud / backend self-managed (S3), providers AWS/OCI/Azure/GCP/K8s; CDK (AWS) y CDKTF como alternativas similares.

**Requisitos:** Python, conceptos de IaC iguales a Terraform.

**Conexión:** igual que Terraform; facilita tests con `pytest`.

**Aprende más:** pulumi.com/docs/get-started.

---

## Ansible

**Para qué sirve:** configurar y mantener servidores de forma idempotente vía SSH, sin agentes.

**Escenario de uso:** preparar VMs con GPU (drivers NVIDIA, Docker), servidores on-premise, parches, despliegues en máquinas que no son K8s.

**Comandos típicos**
```bash
ansible all -i inventory.ini -m ping
ansible-playbook -i inventory.ini site.yml --check --diff     # dry-run
ansible-playbook -i inventory.ini site.yml --limit gpu_nodes --tags docker
ansible-vault encrypt group_vars/prod/secrets.yml
ansible-galaxy install -r requirements.yml
```
```yaml
- hosts: gpu_nodes
  become: true
  tasks:
    - name: Instalar Docker
      ansible.builtin.apt: { name: docker.io, state: present, update_cache: true }
    - name: Servicio Docker activo
      ansible.builtin.service: { name: docker, state: started, enabled: true }
```

**Variantes:** AWX / Ansible Automation Platform (UI), inventarios dinámicos (aws_ec2, oracle.oci); alternativas: Chef, Puppet, cloud-init.

**Requisitos:** SSH, Python en el host destino; YAML, roles, handlers, idempotencia.

**Conexión:** Terraform crea la VM → Ansible la configura; ejecutable desde CI ([05](05-ci-cd.md)); secretos con Ansible Vault/HashiCorp Vault ([12](12-seguridad-gobierno.md)).

**Aprende más:** docs.ansible.com · *Ansible for DevOps* (J. Geerling).

---

## Hilando soluciones
**Ambiente ML reproducible:** módulo Terraform `ml-platform` (VPC/VCN, cluster EKS/OKE, buckets, registry, roles) → `plan` comentado en el PR por Atlantis/GitHub Actions → `apply` en merge → `helm_release` instala KServe, Prometheus y MLflow → Ansible configura nodos GPU on-prem → Checkov bloquea buckets públicos.
