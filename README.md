# gitops-bu-acme-demo

Repositório GitOps da **Business Unit ACME** — contém ferramentas e aplicações específicas da BU,
gerenciadas via ArgoCD pelo hub `gerencia-ho` (homologação) e `gerencia-pr` (produção).

Faz parte da estratégia de [3 repositórios GitOps](https://github.com/SEU_USUARIO/gitops-global-demo/blob/main/docs/ADR-001-three-repo-gitops-strategy.md).

> ⚠️ **Importante:** Substitua `SEU_USUARIO` nos arquivos YAML pelo seu usuário/organização GitHub.

---

## Como Funciona

As ferramentas desta BU são **auto-descobertas** pelo `gitops-global-demo` através de um
**ApplicationSet Matrix Generator**:

```
gitops-bu-acme-demo/ho/tools/<pasta>/   ← Git Generator detecta cada subpasta
         +
bu-acme-placement-ho (OCM)              ← seleciona apenas o cluster bu-acme-ho
         ↓
ApplicationSet gera Application         → deploya no cluster bu-acme-ho via ArgoCD
```

O ArgoCD do hub `gerencia-ho` sincroniza o conteúdo para o cluster `bu-acme-ho`. Para cada pasta
criada em `ho/tools/`, uma nova ArgoCD Application é gerada automaticamente —
**sem alteração no `gitops-global-demo`**.

---

## Estrutura do Repositório

```text
gitops-bu-acme-demo/
├── ho/                             # Ambiente Homologação
│   └── tools/
│       ├── namespace-config/       # ✅ Namespace, ResourceQuota, LimitRange, Ingress
│       │   ├── namespace.yaml
│       │   ├── resource-quota.yaml
│       │   ├── limit-range.yaml
│       │   ├── ingress.yaml        # 🌐 Ingress: sample-app-bu-acme-ho.local
│       │   └── kustomization.yaml
│       └── sample-app/             # 🎯 Aplicação de demonstração
│           ├── configmap.yaml      # HTML da página + variáveis de ambiente
│           ├── deployment.yaml     # Nginx com a página customizada
│           ├── service.yaml        # ClusterIP (exposto via Ingress)
│           └── kustomization.yaml
│
└── pr/                             # Ambiente Produção (mesma estrutura)
    └── tools/
        ├── namespace-config/
        └── sample-app/
```

---

## Acesso Local (DNS .local)

Os serviços são expostos via **HAProxy Ingress Controller** instalado no cluster `bu-acme-ho`:

| URL | Serviço |
|---|---|
| http://sample-app-bu-acme-ho.local | Aplicação de demonstração (HO) |
| http://headlamp-bu-acme-ho.local | Dashboard Kubernetes (HO) |
| http://sample-app-bu-acme-pr.local | Aplicação de demonstração (PR) |
| http://headlamp-bu-acme-pr.local | Dashboard Kubernetes (PR) |

> Os nomes DNS são resolvidos via `/etc/hosts` do host. Após reboot do Docker,
> rode `./scripts/fix-ips.sh` no repositório `gitops-ocm-foundation-demo`.

---

## Como Adicionar uma Nova Ferramenta

1. Crie uma pasta em `ho/tools/<nome-da-tool>/`
2. Adicione os manifests Kubernetes + `kustomization.yaml`
3. (Opcional) Adicione uma entrada de `Ingress` em `namespace-config/ingress.yaml`
4. Faça commit e push para `main`
5. O ArgoCD no hub `gerencia-ho` detecta e deploya automaticamente no cluster `bu-acme-ho`
6. Repita em `pr/tools/` quando pronto para promover para produção

---

## Fluxo de Promoção HO → PR

```
ho/tools/sample-app/ ──[testado e aprovado]──► pr/tools/sample-app/
                                                    │
                                                    └── ArgoCD deploya em bu-acme-pr
```

---

## Clusters Alvo

| Ambiente | Cluster | Hub | Contexto |
|---|---|---|---|
| Homologação | `bu-acme-ho` | `gerencia-ho` | `kind-bu-acme-ho` |
| Produção | `bu-acme-pr` | `gerencia-pr` | `kind-bu-acme-pr` |

---

## Verificação de Status

```bash
# Aplicações geradas pelo ApplicationSet
kubectl --context kind-gerencia-ho get applications -n argocd -l domain=bu-acme

# Recursos no cluster bu-acme-ho
kubectl --context kind-bu-acme-ho get all -n bu-acme-workloads
kubectl --context kind-bu-acme-ho get ingress -A

# Status de sincronização
kubectl --context kind-gerencia-ho get applications -n argocd
```
