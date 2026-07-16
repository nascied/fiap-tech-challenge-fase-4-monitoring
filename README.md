# 🚀 Tech Challenge Fase 4 - Observabilidade ToggleMaster

[![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white)](https://prometheus.io/)
[![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white)](https://grafana.com/)
[![Loki](https://img.shields.io/badge/Loki-F5A623?style=for-the-badge&logo=grafana&logoColor=white)](https://grafana.com/oss/loki/)
[![OpenTelemetry](https://img.shields.io/badge/OpenTelemetry-425CC7?style=for-the-badge&logo=opentelemetry&logoColor=white)](https://opentelemetry.io/)
[![ArgoCD](https://img.shields.io/badge/ArgoCD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)](https://argo-cd.readthedocs.io/)

> ⚠️ **PROJETO DIDÁTICO** - Este projeto foi desenvolvido como parte do Tech Challenge Fase 4 da Pós-Tech FIAP em Arquitetura Cloud e DevOps.

## 📋 Sobre o Projeto

Este projeto representa o escopo de **infraestrutura de observabilidade** da Fase 4 do Tech Challenge, implementando a stack Opensource de monitoramento (Prometheus, Loki, Grafana) e o OpenTelemetry Collector sobre o cluster Kubernetes já provisionado nas Fases 1, 2 e 3, com entrega 100% via GitOps (ArgoCD).

## Escopo

Este repositório contém os Helm charts e as configurações de observabilidade do ToggleMaster, entregues como `Application` do ArgoCD (padrão app-of-apps). Cobre os **Requisitos 1 e 2** do Tech Challenge Fase 4:

- **Prometheus** — armazenamento e consulta de métricas de infraestrutura (via `kube-prometheus-stack`)
- **Loki + Promtail** — centralização e indexação de logs de todos os containers do cluster
- **Grafana** — visualização, com dashboard customizado e datasources provisionados
- **OpenTelemetry Collector** — peça central de roteamento de telemetria (métricas → Prometheus, logs → Loki, traces → APM)

Não fazem parte do escopo deste repositório:

- Instrumentação do código-fonte dos microsserviços (Requisito 3)
- Integração com a ferramenta de APM (Datadog/New Relic) e Distributed Tracing (Requisito 3)
- Alertas, PagerDuty/OpsGenie, ChatOps e Self-Healing (Requisito 4)
- Cluster EKS, Terraform e os 5 microsserviços em si (Fases 1, 2 e 3 — pré-requisito, não escopo desta fase)

## 🛠️ Tecnologias Utilizadas

| Tecnologia | Uso |
|---|---|
| **Prometheus** | Armazenamento e consulta de métricas |
| **Grafana** | Visualização de métricas e logs |
| **Loki** | Centralização e indexação de logs |
| **Promtail** | Agente de coleta de logs dos containers |
| **OpenTelemetry Collector** | Recebimento, processamento e roteamento de telemetria |
| **ArgoCD** | Entrega contínua via GitOps |
| **Helm** | Empacotamento e parametrização dos charts |

## Componentes Provisionados

| Application (ArgoCD) | Chart | Responsabilidade |
|---|---|---|
| `kube-prometheus-stack` | `prometheus-community/kube-prometheus-stack` | Prometheus, Grafana, Alertmanager, node-exporter, kube-state-metrics |
| `loki` | `grafana/loki` | Armazenamento e indexação de logs (modo `SingleBinary`, storage em filesystem) |
| `promtail` | `grafana/promtail` | Coleta de logs de todos os pods do cluster |
| `otel-collector` | `open-telemetry/opentelemetry-collector` | Receiver OTLP + exporters para Prometheus/Loki/APM |
| `monitoring-dashboards` | manifesto próprio | ConfigMap do dashboard customizado do Grafana |
| `grafana-ingress` | manifesto próprio | Exposição do Grafana via `ingress-nginx` |

## 📁 Estrutura do Projeto

```text
├── argocd/
│   ├── root.yaml               ← Application raiz, único arquivo aplicado manualmente
│   └── children/                ← Applications reais (chart público + values deste repo)
│       ├── application-kube-prometheus-stack.yaml
│       ├── application-loki.yaml
│       ├── application-promtail.yaml
│       ├── application-otel-collector.yaml
│       ├── application-dashboards.yaml
│       ├── application-grafana-ingress.yaml
│       └── kustomization.yaml
├── values/
│   ├── kube-prometheus-stack.yaml
│   ├── loki.yaml
│   ├── promtail.yaml
│   └── otel-collector.yaml
├── dashboards/
│   ├── toggle-master-overview.json
│   └── kustomization.yaml
├── manifests/
│   └── grafana-ingress.yaml
└── README.md
```

## Requisitos

- Cluster Kubernetes (EKS) e ArgoCD já em execução (Fase 3).
- `kubectl` configurado apontando para o cluster.
- Os 5 microsserviços do ToggleMaster operacionais.

Configure o `kubectl` antes da execução:

```bash
aws eks update-kubeconfig --name <nome-do-cluster> --region us-east-1
```

## Values por Chart

Cada `Application` usa fonte múltipla (multi-source do ArgoCD): o chart público do respectivo projeto + o `values.yaml` correspondente deste repositório.

| Arquivo | Chart afetado | Principais ajustes |
|---|---|---|
| `values/kube-prometheus-stack.yaml` | `kube-prometheus-stack` | retenção do Prometheus, descoberta de `ServiceMonitor` em qualquer namespace, sidecar de dashboards, datasource do Loki |
| `values/loki.yaml` | `loki` | modo `SingleBinary`, storage em filesystem, sem autenticação multi-tenant |
| `values/promtail.yaml` | `promtail` | endpoint de push apontando para o Service do Loki |
| `values/otel-collector.yaml` | `opentelemetry-collector` | receiver OTLP, exporters de métricas/logs, pipeline de traces pronto para o exporter do APM |

## Modelo de Uso manual

Acesse o diretório do repositório:

```bash
cd fiap-tech-challenge-fase-4-monitoring
```

Aplique a Application raiz (único comando necessário):

```bash
kubectl apply -f argocd/root.yaml
```

Acompanhe a sincronização:

```bash
kubectl get applications -n argocd
```

## Sincronização via ArgoCD (GitOps)

A `Application` raiz (`monitoring-root`) sincroniza a pasta `argocd/children/`, que cria as 6 `Application` reais. `kube-prometheus-stack` e `loki` sobem primeiro (`sync-wave: "-1"`); `promtail`, `otel-collector`, `monitoring-dashboards` e `grafana-ingress` sobem na sequência (`sync-wave: "0"`), já que dependem do Prometheus/Loki estarem disponíveis.

| Evento | Ação | Objetivo |
|---|---|---|
| Push no branch `main` | ArgoCD detecta divergência (polling automático) | Atualizar o cluster sem intervenção manual |
| Alteração manual no cluster | `selfHeal` reverte para o estado do Git | Garantir que o Git continue sendo a única fonte da verdade |

## Acesso

- **Grafana**: `http://<domínio>/grafana` (usuário `admin`, senha em `values/kube-prometheus-stack.yaml`)
- **Dashboard customizado**: "ToggleMaster - Visão Geral" — uso de recursos do cluster, taxa de requisição por microsserviço (via métricas do `ingress-nginx`) e painel de logs em tempo real (Loki)

## Arquivos Ignorados

```text
*.tgz
.helm/
```

## Boas Práticas

- Revisar o `helm template` de cada chart localmente antes de commitar alterações de `values`.
- Nunca commitar credenciais reais (ex: senha do Grafana) em texto plano — usar Secret gerenciado externamente quando sair do escopo didático.
- Manter o pipeline de *traces* do OTel Collector documentado como placeholder até a instrumentação de código (Requisito 3) estar pronta.
- Usar `ServerSideApply=true` em Applications que instalam CRDs grandes (evita o limite de 256KB de annotation do `kubectl apply` padrão).

## 👨‍💻 Autor

**Edson Leandro da Silva Nascimento**
- Pós-Tech FIAP - Arquitetura Cloud e DevOps
- Tech Challenge Fase 4 — Requisitos 1 e 2 (Infraestrutura de Observabilidade)

---

## 📄 Licença

Este projeto é apenas para fins educacionais como parte do programa de pós-graduação em Arquitetura Cloud e DevOps da instituição FIAP.
