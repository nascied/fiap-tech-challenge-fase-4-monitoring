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
- **Grafana** — visualização, com dois dashboards customizados e datasources provisionados
- **OpenTelemetry Collector** — peça central de roteamento de telemetria (métricas → Prometheus, logs → Loki, traces → Datadog)

Este repositório também deixa **pronta a rota de traces para o APM** (exporter Datadog configurado no Collector, Requisito 1/2), mas não inclui:

- Instrumentação do código-fonte dos microsserviços (Requisito 3 — repositório `fiap-tech-challenge-fase-3-services`)
- Visualização de Distributed Tracing e Service Map no painel do Datadog (Requisito 3)
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
| `otel-collector` | `open-telemetry/opentelemetry-collector` | Receiver OTLP + exporters para Prometheus/Loki/Datadog |
| `monitoring-dashboards` | manifesto próprio | ConfigMaps dos dashboards customizados do Grafana |
| `grafana-ingress` | manifesto próprio | Exposição do Grafana via `ingress-nginx` |

## 📁 Estrutura do Projeto

```text
├── argocd/
│   ├── root.yaml               ← Application raiz, único arquivo aplicado manualmente
│   └── application/             ← Applications reais (chart público + values deste repo)
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
│   └── otel-collector.yaml      ← inclui exporter Datadog (traces) via DD_API_KEY
├── dashboards/
│   ├── toggle-master-overview.json   ← saúde dos 5 microsserviços
│   ├── toggle-master-infra.json      ← saúde/capacidade do cluster (nodes)
│   └── kustomization.yaml
├── manifests/
│   └── grafana-ingress.yaml
└── README.md
```

## ✅ Status Atual

| Componente | Situação | Observação |
|---|---|---|
| `kube-prometheus-stack` (Prometheus, Grafana, Alertmanager, node-exporter, kube-state-metrics) | 🟢 Operacional | Validado direto no cluster (`fiap-tc-f4-eks`) |
| `loki` + `promtail` | 🟢 Operacional | Logs dos 5 microsserviços chegando normalmente |
| Dashboard "ToggleMaster - Infraestrutura do Cluster" | 🟢 Operacional | Depende só de `node-exporter`/`kube-state-metrics`, já ativos |
| Dashboard "ToggleMaster - Visão Geral" — CPU/Memória/Pods/Logs por serviço | 🟢 Operacional | |
| Dashboard "ToggleMaster - Visão Geral" — painéis de Requisições (métrica `http_server_duration_milliseconds_count`) | 🟡 Aguardando dados | Pipeline OTel Collector → Prometheus testado e correto; painel fica "No data" até a instrumentação de código (Requisito 3, Sandro) começar a emitir métricas |
| `otel-collector` — pipeline de traces → Datadog | 🟡 Configurado, aguardando dados | Exporter e roteamento prontos; sem o Secret `datadog-secret` o pod **não sobe** (ver pré-requisito abaixo); sem instrumentação de código, não há traces a enviar |

> Este quadro reflete o estado no momento da última verificação (via `kubectl` direto no cluster). Atualize antes de gerar o relatório final.

## Requisitos

- Cluster Kubernetes (EKS) e ArgoCD já em execução (Fase 3).
- `kubectl` configurado apontando para o cluster.
- Os 5 microsserviços do ToggleMaster operacionais.
- Secret `datadog-secret` criado no namespace `monitoring` (ver seção abaixo) — obrigatório antes de sincronizar o `otel-collector`, senão o pod falha ao subir por variável de ambiente sem Secret correspondente.

Configure o `kubectl` antes da execução:

```bash
aws eks update-kubeconfig --name <nome-do-cluster> --region us-east-1
```

### Pré-requisito: Secret do Datadog

O `otel-collector` referencia `DD_API_KEY` via `secretKeyRef` (`values/otel-collector.yaml`). Esse Secret **não é versionado neste repositório** (credencial sensível) e precisa ser criado manualmente antes do ArgoCD sincronizar a Application `otel-collector`:

```bash
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic datadog-secret -n monitoring --from-literal=apiKey='<SUA_DD_API_KEY>'
```

Gere a chave em `https://app.datadoghq.com/organization-settings/api-keys` (ajuste o domínio se sua conta não for na região US1).

## Values por Chart

Cada `Application` usa fonte múltipla (multi-source do ArgoCD): o chart público do respectivo projeto + o `values.yaml` correspondente deste repositório.

| Arquivo | Chart afetado | Principais ajustes |
|---|---|---|
| `values/kube-prometheus-stack.yaml` | `kube-prometheus-stack` | retenção do Prometheus, descoberta de `ServiceMonitor` em qualquer namespace, sidecar de dashboards, datasource do Loki |
| `values/loki.yaml` | `loki` | modo `SingleBinary`, storage em filesystem, sem autenticação multi-tenant |
| `values/promtail.yaml` | `promtail` | endpoint de push apontando para o Service do Loki |
| `values/otel-collector.yaml` | `opentelemetry-collector` | receiver OTLP; exporters de métricas (Prometheus) e logs (Loki); traces roteados para o Datadog via `DD_API_KEY` (Secret externo) |

## Como Executar

1. Configure o `kubectl` apontando para o cluster (ver comando acima).
2. Crie o Secret `datadog-secret` (ver pré-requisito acima) — sem ele, a Application `otel-collector` fica `Degraded`.
3. Acesse o diretório do repositório:

   ```bash
   cd fiap-tech-challenge-fase-4-monitoring
   ```

4. Aplique a Application raiz (único comando necessário — todo o resto sobe via GitOps):

   ```bash
   kubectl apply -f argocd/root.yaml
   ```

5. Acompanhe a sincronização até todas ficarem `Synced`/`Healthy`:

   ```bash
   kubectl get applications -n argocd -w
   ```

6. Valide os componentes:

   ```bash
   kubectl get pods -n monitoring
   ```

7. Acesse o Grafana (ver seção "Acesso" abaixo) e confira os dois dashboards.

## Sincronização via ArgoCD (GitOps)

A `Application` raiz (`monitoring-root`) sincroniza a pasta `argocd/application/`, que cria as 6 `Application` reais. `kube-prometheus-stack` e `loki` sobem primeiro (`sync-wave: "-1"`); `promtail`, `otel-collector`, `monitoring-dashboards` e `grafana-ingress` sobem na sequência (`sync-wave: "0"`), já que dependem do Prometheus/Loki estarem disponíveis.

| Evento | Ação | Objetivo |
|---|---|---|
| Push no branch `main` | ArgoCD detecta divergência (polling automático) | Atualizar o cluster sem intervenção manual |
| Alteração manual no cluster | `selfHeal` reverte para o estado do Git | Garantir que o Git continue sendo a única fonte da verdade |

## Acesso

- **Grafana**: `http://<domínio>/grafana` (usuário `admin`, senha em `values/kube-prometheus-stack.yaml`)
- **Dashboard "ToggleMaster - Visão Geral"**: recursos de CPU/memória por pod, taxa de requisições por microsserviço (via métrica `http_server_duration_milliseconds_count`, OTel Collector) e painel de logs em tempo real (Loki), com uma seção detalhada por microsserviço
- **Dashboard "ToggleMaster - Infraestrutura do Cluster"**: nodes ready, pods pendentes/com restart, CPU/memória/disco/rede por node, capacidade (requests vs. alocável)

## Arquivos Ignorados

```text
*.tgz
.helm/
```

## Boas Práticas

- Revisar o `helm template` de cada chart localmente antes de commitar alterações de `values`.
- Nunca commitar credenciais reais (ex: senha do Grafana, `DD_API_KEY`) em texto plano — usar Secret gerenciado externamente (`datadog-secret`), fora deste repositório.
- Verificar se os Deployments dos microsserviços têm as env vars `OTEL_EXPORTER_OTLP_ENDPOINT` etc. antes de esperar dados nos painéis de requisições — sem isso, o Collector não recebe nada mesmo com o roteamento correto.
- Usar `ServerSideApply=true` em Applications que instalam CRDs grandes (evita o limite de 256KB de annotation do `kubectl apply` padrão).

## 👨‍💻 Autor

**Edson Leandro da Silva Nascimento**
- Pós-Tech FIAP - Arquitetura Cloud e DevOps
- Tech Challenge Fase 4 — Requisitos (Infraestrutura de Observabilidade)

---

## 📄 Licença

Este projeto é apenas para fins educacionais como parte do programa de pós-graduação em Arquitetura Cloud e DevOps da instituição FIAP.
