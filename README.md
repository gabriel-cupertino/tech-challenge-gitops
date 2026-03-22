# tech-challenge-gitops

Repositório de manifestos Kubernetes do projeto **ToggleMaster** (POSTECH/FIAP – Fase 3).
Monitorado pelo ArgoCD para sincronização automática no cluster EKS.

---

## Estrutura

```
tech-challenge-gitops/
├── argocd/                    # ArgoCD Application CRDs (um por serviço)
│   ├── auth-service.yaml
│   ├── flag-service.yaml
│   ├── targeting-service.yaml
│   ├── evaluation-service.yaml
│   └── analytics-service.yaml
├── auth-service/              # configmap.yaml, job.yaml, deployment.yaml, service.yaml, ingress.yaml
├── flag-service/              # configmap.yaml, job.yaml, deployment.yaml, service.yaml, ingress.yaml
├── targeting-service/         # configmap.yaml, job.yaml, deployment.yaml, service.yaml, ingress.yaml
├── evaluation-service/        # deployment.yaml, service.yaml, ingress.yaml
└── analytics-service/         # deployment.yaml, service.yaml
```

## Serviços

| Serviço | Namespace | Porta | Banco |
|---|---|---|---|
| `auth-service` | auth-service | 8001 | PostgreSQL (auth_db) |
| `flag-service` | flag-service | 8002 | PostgreSQL (flag_db) |
| `targeting-service` | targeting-service | 8003 | PostgreSQL (targeting_db) |
| `evaluation-service` | evaluation-service | 8004 | Redis + SQS |
| `analytics-service` | analytics-service | 8005 | DynamoDB + SQS |

## Fluxo GitOps

```
Push no tech-challenge-apps
  └─► Pipeline CI/CD
        └─► Build + scan + push da imagem para ECR
        └─► Atualiza deployment.yaml neste repo (nova tag de imagem)
              └─► ArgoCD detecta a mudança
                    └─► Sync automático no cluster EKS
```

## DB Init (PreSync Hook)

Para os serviços com PostgreSQL (`auth`, `flag`, `targeting`), o schema é inicializado via **ArgoCD PreSync Hook**:

1. ArgoCD executa o `Job` (`job.yaml`) antes de aplicar o `Deployment`
2. O Job roda o SQL do `ConfigMap` contra o RDS
3. Após sucesso, o Job é removido automaticamente (`HookSucceeded`)
4. O `Deployment` sobe com o banco já preparado

O `configmap.yaml` de cada serviço é atualizado automaticamente pela pipeline CI sempre que o `db/init.sql` muda no `tech-challenge-apps`.

## Registrar as Applications no ArgoCD

```bash
kubectl apply -f argocd/auth-service.yaml
kubectl apply -f argocd/flag-service.yaml
kubectl apply -f argocd/targeting-service.yaml
kubectl apply -f argocd/evaluation-service.yaml
kubectl apply -f argocd/analytics-service.yaml
```

O ArgoCD monitora o branch `main` e sincroniza automaticamente com `selfHeal: true` e `prune: true`.

## Repositórios relacionados

| Repositório | Conteúdo |
|---|---|
| `tech-challenge-apps` | Código-fonte dos 5 microsserviços + pipelines CI/CD |
| `tech-challenge-gitops` | Este repositório — manifestos Kubernetes |
| `tech-challenge-iac` | Infraestrutura Terraform (EKS, RDS, Redis, SQS, ECR) |
