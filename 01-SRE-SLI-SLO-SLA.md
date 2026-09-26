# SRE — SLI, SLO e SLA do `donation-service`

> Serviço em escopo: **`donation-service`** — Hot Path / Caminho Crítico da
> SolidaryTech (processamento de doações). É o único serviço com garantias
> formais de nível de serviço definidas neste documento; os demais
> (`ngo-service`, `volunteer-service`, `notification-service`) são
> monitorados pela mesma stack, mas sem SLA formal.

## 1. Golden Metrics aplicadas

As quatro Golden Metrics, mapeadas para o `donation-service`:

| Golden Metric | Aplicação no `donation-service` | Fonte |
|---|---|---|
| **Latência** | Tempo de resposta das requisições ao serviço (rota de negócio: `POST`/`GET /donations`) | `solidarytech_http_request_duration_seconds` (histograma) |
| **Tráfego** | Requisições por segundo | `solidarytech_http_requests_total` (contador) |
| **Erros** | Taxa de respostas HTTP 5xx | `solidarytech_http_requests_total{http_status_code=~"5.."}` |
| **Saturação** | Uso de CPU dos pods vs. `requests` (o painel atual não mede memória) | `container_cpu_usage_seconds_total` / `kube_pod_container_resource_requests` (painel "Rightsizing" do dashboard Overview) |

> **Escopo real das métricas (`otel.go`, `main.go`):** o middleware
> `withMetrics` envolve o roteador inteiro, então **todas as rotas** do
> serviço são medidas, inclusive `/health`, que as readiness probes (a cada
> 10 s) e liveness probes (a cada 15 s) chamam. As queries abaixo filtram só
> `service_name` e **não** `http_route`, portanto essas chamadas de health
> check entram nos SLIs.

Este documento formaliza **dois SLIs**, conforme exigido pelo desafio
(Erros e Latência — as duas Golden Metrics mais diretamente ligadas à
experiência do doador).

## 2. SLI 1 — Taxa de Sucesso (Erros)

**Definição:** proporção de requisições HTTP ao `donation-service` que
não resultam em erro de servidor (status `5xx`), em relação ao total de
requisições, numa janela móvel de 30 dias.

**Fórmula (PromQL, a mesma usada no dashboard SRE do Grafana):**

```promql
100 * (1 - (
  (sum(increase(solidarytech_http_requests_total{service_name="donation-service", http_status_code=~"5.."}[30d])) or vector(0))
  /
  sum(increase(solidarytech_http_requests_total{service_name="donation-service"}[30d]))
))
```

> O `or vector(0)` no numerador é necessário: enquanto não houver nenhum
> erro 5xx real no período, a série com o label `http_status_code=~"5.."`
> simplesmente não existe no Prometheus (diferente de existir com valor
> 0), e uma divisão vetorial contra uma série ausente retorna vazio ("No
> data" no Grafana) em vez de 0% — bug real encontrado na validação
> end-to-end, corrigido nas queries do dashboard e aqui.

> **O Monitor do Datadog não usa esta PromQL** (`fiap-tc-5-terraform/datadog.tf`):
> ele calcula a taxa de erro a partir das métricas de trace
> `trace.http.server.request.errors` / `trace.http.server.request.hits` e
> dispara quando ela passa de **5% em 5 minutos**. É um alerta de
> degradação aguda, e não o cálculo do SLO de 30 dias.

**Fonte de dados:** métrica customizada `solidarytech_http_requests_total`,
emitida pelo código do `donation-service` (OTel SDK manual em Go),
roteada pelo OTel Collector via `prometheusremotewrite` para o Prometheus.

## 3. SLI 2 — Latência (p95)

**Definição:** percentil 95 do tempo de resposta das requisições ao
`donation-service`, medido numa janela móvel de 5 minutos.

**Fórmula (PromQL):**

```promql
histogram_quantile(0.95,
  sum by (le) (rate(solidarytech_http_request_duration_seconds_bucket{service_name="donation-service"}[5m]))
)
```

**Fonte de dados:** métrica customizada
`solidarytech_http_request_duration_seconds` (histograma, buckets de 5ms a
10s), emitida no mesmo ponto de instrumentação do SLI 1.

## 4. SLOs (Service Level Objectives — metas internas de engenharia)

| SLI | SLO | Janela | Error Budget |
|---|---|---|---|
| Taxa de sucesso | **≥ 99.9%** | 30 dias móveis | 0.1% de falhas permitidas ≈ **43,2 minutos/mês** de indisponibilidade equivalente |
| Latência p95 | **≤ 500ms** | 5 minutos móveis (avaliação contínua) | — (é uma meta pontual, não acumulativa) |

### Error Budget — cálculo e consumo

```
Error Budget total (30d)     = (1 - 0.999) × total_de_requisições_30d
Error Budget consumido (%)   = (taxa_de_erro_real / 0.001) × 100
Error Budget restante (%)    = 100 - Error Budget consumido (%)
```

O painel **"Error Budget Restante (30d)"** do dashboard
`SolidaryTech - SRE (SLO & Error Budget)` (Grafana) calcula isso em tempo
real. Quando o budget chega a 0%, qualquer erro adicional já viola o SLO
mensal — é o sinal para a equipe desacelerar mudanças/deploys até o
budget se recompor.

## 5. SLA (Service Level Agreement — compromisso externo com as ONGs parceiras)

O SLA é deliberadamente **mais frouxo que o SLO interno** — a prática
padrão de SRE é manter uma margem entre a meta de engenharia (SLO) e o
compromisso contratual (SLA), para que violações do SLO sejam um alerta
interno *antes* de se tornarem uma violação de contrato visível ao
cliente.

| Item | Compromisso |
|---|---|
| Disponibilidade mensal do fluxo de doações | **≥ 99.5%** (≈ 3h39min de indisponibilidade tolerada/mês) |
| Tempo de resposta a incidentes críticos (Sev1: doações fora do ar) | Reconhecimento em **até 15 minutos** (PagerDuty), atualização de status a cada 30min |
| Tempo de resolução de incidentes críticos | **até 2 horas** (ou mitigação via self-healing/rollback, o que ocorrer primeiro) |
| Janela de manutenção programada | Não conta para o cálculo de disponibilidade, comunicada com 48h de antecedência |
| Compensação em caso de violação | Créditos operacionais / revisão de prioridade de roadmap com a ONG afetada (a formalizar comercialmente) |

## 6. MTTR — como a stack reduz o tempo de resposta a incidentes

O cenário descrito no desafio (Fase 4) ilustra o "antes": **6 horas** até a
equipe ser avisada por usuários, mais **4 horas** de troubleshooting manual
em logs espalhados — **10 horas de MTTR** para um incidente simples.

Com a stack implementada, medimos o fluxo completo **de ponta a ponta, ao
vivo, sem intervenção humana**, na validação da Fase 4 (mesmo padrão
replicado aqui):

| Etapa | Tempo medido (real, Fase 4) |
|---|---|
| Erro real acontecendo → Datadog Monitor entra em `Alert` | detecção em até 1 min (ciclo de avaliação do Monitor) |
| Monitor `Alert` → Incidente aberto no PagerDuty | **2 segundos** |
| Monitor `Alert` → GitHub Actions self-healing disparado (`repository_dispatch`) | **4 segundos** |
| Self-healing (`kubectl rollout restart` + rollout completo) | **~35-46 segundos** |

**MTTR total, da detecção à mitigação automática: menos de 2 minutos** —
uma redução de **~99,7%** em relação ao cenário manual de 10 horas, sem
nenhuma intervenção humana na cadeia detecção→notificação→mitigação. O
tempo humano só entra depois, no *post-mortem* (ver
`04-ITSM-INCIDENT-LIFECYCLE.md`), já com o serviço mitigado.

> Estes tempos são da **validação da Fase 4**. A mesma cadeia está
> declarada no código da Fase 5 (`datadog.tf`, `pagerduty.tf`,
> `self-heal.yml`), mas a pasta de entrega não contém medição feita no
> ambiente SolidaryTech: repetir o teste e anexar a evidência.

Esse ganho vem de três fatores multiplicativos:
1. **Detecção ativa** (Datadog Monitors de taxa de erro 5xx e de latência
   p95 avaliados continuamente) em vez de reativa (usuário reportando).
2. **Notificação automática multi-canal** (PagerDuty + Discord) em vez de
   dependência de alguém "ver" o problema.
3. **Mitigação automática** (self-healing via GitHub Actions: `kubectl
   rollout restart` disparado pelo Monitor de taxa de erro 5xx) em vez de
   esperar um engenheiro investigar, decidir e executar manualmente.
