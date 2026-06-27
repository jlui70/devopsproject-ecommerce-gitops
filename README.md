# devopsproject-ecommerce GitOps

Repositório GitOps que gerencia todos os manifestos Kubernetes da aplicação `devopsproject-ecommerce`.

Utiliza **Helm charts** para empacotar cada microsserviço e **Kustomize** para aplicar overlays transversais (namespace `dpe`, prefixo `dpe-`, sufixo `-prod`, labels comuns e imagens centralizadas). O ArgoCD reconcilia este repositório continuamente contra o cluster Kubernetes (ADR-0002).

---

## Estrutura

```
devopsproject-ecommerce-gitops/
├── argocd/
│   ├── application.yml          # Application ArgoCD (ns argocd → ns dpe)
│   └── argocd-cm-patch.yml      # Habilita --enable-helm no ArgoCD
└── production/
    ├── kustomization.yml        # namePrefix dpe-, nameSuffix -prod, images:
    ├── configurations/
    │   └── namereference.yml    # Resolve scaleTargetRef do HPA para o Rollout renomeado
    ├── infrastructure/
    │   ├── namespace.yml
    │   ├── config-map.yml       # Variáveis de ambiente compartilhadas (preencher placeholders TF)
    │   ├── service-account.yml
    │   ├── ingress.yml          # ALB interno HTTPS:443, preview antes de active
    │   ├── analysis-template.yml
    │   ├── network-policies/
    │   │   ├── ingress.yml
    │   │   └── egress.yml
    │   └── secrets/
    │       ├── ecr-image-pull-credentials.yml
    │       ├── kestrel-certificate.yml
    │       └── mongo-certificate.yml
    └── application/             # 1 Helm chart por microsserviço
        ├── main/                # Rollout Blue/Green + HPA
        ├── order/               # Rollout Blue/Green + HPA
        ├── identity-server/     # Rollout Blue/Green + HPA
        ├── health-checker/      # Deployment + Service
        ├── invoice-generator/   # Deployment + KEDA ScaledObject
        └── notificator/         # Deployment + KEDA ScaledObject
```

---

## Pré-requisitos

Antes de ativar o sync do ArgoCD, os passos abaixo devem estar concluídos:

### 1. Stacks Terraform aplicadas

As seguintes stacks devem estar em execução:

- **ADR-0001** (networking) — VPC, subnets, Route53
- **ADR-0002** (server) — cluster Kubernetes, ECR, ACM, ALB Controller, external-dns, ArgoCD instalados via Ansible
- **ADR-0003** (serverless) — Aurora, RDS Proxy, DocumentDB, SQS, SNS, Secrets Manager

### 2. Preencher placeholders no repositório

Após o apply das stacks, substituir os placeholders nos manifests e commitar:

#### `production/infrastructure/ingress.yml`

```bash
ACM_ARN=$(terraform -chdir=devopsproject-ecommerce-iac/terraform/server output -raw acm_certificate_arn)
# Substituir <TERRAFORM_OUTPUT:acm_certificate_arn> pelo valor de $ACM_ARN
```

#### `production/infrastructure/config-map.yml`

```bash
RDS_PROXY=$(terraform -chdir=devopsproject-ecommerce-iac/terraform/serverless output -raw rds_proxy_endpoint)
DOCDB=$(terraform -chdir=devopsproject-ecommerce-iac/terraform/serverless output -raw documentdb_cluster_endpoint)
# Substituir os placeholders <TERRAFORM_OUTPUT:...> e <SECRETS_MANAGER:...> pelos valores reais
# Senhas: recuperar do AWS Secrets Manager — nunca colar em plaintext no repositório
```

### 3. Criar Secrets no cluster

```bash
# ECR image pull credentials (renovar periodicamente)
kubectl create secret docker-registry ecr-image-pull-credentials \
    --docker-username=AWS \
    --docker-password=$(aws ecr get-login-password --region us-east-1) \
    --docker-server=794038226274.dkr.ecr.us-east-1.amazonaws.com \
    --dry-run=client -o yaml \
    > production/infrastructure/secrets/ecr-image-pull-credentials.yml

# Certificado Kestrel (PFX) e CA do DocumentDB (PEM)
# Gerar/obter os certificados e criar os secrets conforme processo documentado
```

### 4. Imagens publicadas no ECR

O CI/CD (ADR-0005) deve ter executado ao menos uma build e publicado as imagens antes do primeiro sync. O ArgoCD não consegue criar pods sem imagens disponíveis no registry.

---

## Ativar o ArgoCD

### 1. Aplicar o patch do ConfigMap

```bash
kubectl apply -f argocd/argocd-cm-patch.yml
```

### 2. Registrar o repositório GitOps no ArgoCD

```bash
argocd repo add https://github.com/jlui70/devopsproject-ecommerce-gitops \
    --username <github-user> \
    --password <github-token>
```

### 3. Criar a Application

```bash
kubectl apply -f argocd/application.yml
```

O ArgoCD passará a reconciliar automaticamente (`automated: prune: true, selfHeal: true`) o path `production` deste repositório contra o namespace `dpe` do cluster.

---

## Acesso local ao cluster (via SSM Port Forward)

```bash
# Obter o kubeconfig do nó control-plane via SSM
aws ssm start-session --target <CONTROL_PLANE_INSTANCE_ID>
sudo cat /etc/kubernetes/admin.conf
# Copiar o conteúdo para ~/.kube/config e substituir o DNS do NLB por 127.0.0.1

# Abrir túnel SSM para o kube-apiserver
aws ssm start-session \
    --target <CONTROL_PLANE_INSTANCE_ID> \
    --document-name AWS-StartPortForwardingSession \
    --parameters 'portNumber=6443,localPortNumber=6443'

export KUBECONFIG=~/.kube/config
kubectl get nodes
```

---

## Rollback

```bash
# Reverter para o commit anterior (ArgoCD reconcilia automaticamente)
git revert HEAD
git push origin main
```

Durante uma promoção Blue/Green em andamento, abortar o Rollout mantém a versão active:

```bash
kubectl argo rollouts abort <rollout-name> -n dpe
```
