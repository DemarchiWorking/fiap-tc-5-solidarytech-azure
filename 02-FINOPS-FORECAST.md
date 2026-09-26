# FinOps — Forecast de Custos e Otimizações (SolidaryTech / Fase 5)

> Os preços unitários foram consultados na **Azure Retail Prices API**
> (`https://prices.azure.com/api/retail/prices`) em 2026-09, regiões
> `eastus`/`eastus2` (as regiões definidas em
> `fiap-tc-5-terraform/variables.tf`). Exceção: o Load Balancer, estimado
> por valor de lista (ver nota na tabela). As quantidades vêm do código
> (Terraform e values dos addons); volumes de uso (operações, RUs) são
> estimativas.

## 1. Estratégia de Tagging (evidência)

Todos os recursos do Terraform que aceitam o argumento `tags` recebem as
tags obrigatórias exigidas pelo desafio, definidas uma única vez em
[`main.tf`](fiap-tc-5-terraform/main.tf) e aplicadas via `merge()` em
cada recurso:

```hcl
locals {
  mandatory_tags = {
    Project     = "SolidaryTech"
    Environment = "Production"
    CostCenter  = "NGO-Core"
  }

  tags = merge(local.mandatory_tags, {
    ManagedBy = "terraform"
    Fase      = "5"
  })
}
```

Cada recurso soma um `Component` específico (ex.: `Component =
"donation-service"` + `Criticality = "hot-path"` no Postgres de doações,
`Component = "nosql"` no Cosmos DB), permitindo quebrar o custo por
componente **dentro** do mesmo `CostCenter` no Azure Cost Management. O
`README.md` do Terraform cita a consulta via `az rest` contra
`Microsoft.CostManagement/query`, a mesma técnica usada no fim da Fase 4.

Recursos com tags no código (9): Resource Group, VNet, ACR, AKS, 2x
PostgreSQL Flexible Server, namespace do Service Bus, conta do Cosmos DB e
Storage Account do Velero ([`velero.tf`](fiap-tc-5-terraform/velero.tf)).
Sub-recursos (subnet, bancos, regras de firewall, fila, tabelas, container)
e role assignments não recebem tags no código. O node pool do AKS não
declara `tags`, e os Load Balancers e discos criados pelo próprio AKS não
são marcados pelo Terraform.

## 2. Forecast mensal — breakdown por recurso

| Recurso | Preço unitário (Retail Prices API) | Quantidade | Custo/mês |
|---|---|---|---|
| AKS — nós `Standard_B2s` (control plane no tier Free = US$0) | US$0,0416/hora | 3 nós (piso do autoscaling, `aks_min_count`) × 730h | **US$91,10** |
| ACR Basic | US$0,1666/dia | 1 registry × 30 dias | **US$5,00** |
| PostgreSQL Flexible Server — compute `Burstable B1MS` | US$0,017/hora | 2 servidores (`ngo`, `donation`) × 730h | **US$24,82** |
| PostgreSQL Flexible Server — storage | US$0,115/GB/mês | 2 × 32GB = 64GB | **US$7,36** |
| Azure Service Bus — Basic (operações) | US$0,05 / milhão de operações | ~1M ops/mês (estimativa de tráfego de demo) | **US$0,50** |
| **Cosmos DB Table API — Serverless** (ver §3) | US$0,25 / milhão de RUs | ~0,5M RUs/mês (estimativa) | **US$0,13** |
| PVC (discos gerenciados) — Prometheus + Loki + Grafana | preço de disco Standard SSD LRS | ~40GB agregados (os values declaram 32 Gi: Prometheus 20 + Loki 10 + Grafana 2) | **US$5,16** |
| Load Balancer Standard | tarifa de LB Standard + regra | 1x ingress-nginx (público) + 1x outbound LB do AKS (padrão em cluster Standard SKU) | **US$39,50** (estimativa — API de preços de LB não retornou meter exato p/ a região; usado valor de lista) |
| **Total estimado** | | | **≈ US$173,57/mês** |

> **Itens do código fora da tabela:** o Service do ArgoCD também é do tipo
> `LoadBalancer` (`argocd.tf`); os PVCs do SonarQube (5 Gi + 5 Gi do
> PostgreSQL embutido) e do Alertmanager (1 Gi) não estão somados; os
> serviços SaaS (Datadog, PagerDuty) não fazem parte deste forecast.

> O Velero (`velero.tf`) usa um Storage Account Standard LRS para os
> backups. Estimativa deste documento: com o volume de dados do projeto, o
> custo fica abaixo de US$1/mês, dentro da margem de arredondamento da
> tabela.

## 3. Otimização #1 — **aplicada no código**: Cosmos DB Serverless em vez de provisionado

Não é uma recomendação: está **implementada** em
[`cosmosdb.tf`](fiap-tc-5-terraform/cosmosdb.tf):

```hcl
capabilities { name = "EnableTable" }
capabilities { name = "EnableServerless" }
```

e as duas tabelas (`Volunteers`, `DonationNotifications`) não declaram
`throughput`. O repositório já nasce em Serverless (commit inicial
`82421a8`); a comparação com o modo provisionado (400 RU/s por tabela) está
registrada no comentário do próprio `cosmosdb.tf`.

**Por quê:** o modo provisionado cobra um valor **fixo**, independente do
uso — 400 RU/s por tabela × 2 tabelas = 800 RU/s, a
US$0,008/hora por 100 RU/s ⇒ **US$46,72/mês fixos**, mesmo em uma madrugada
sem nenhuma requisição. No volume de tráfego esperado deste projeto
(dezenas de milhares de operações/mês, não milhões — voluntários e
notificações de doação, não um app de escala global), o modo
**Serverless** cobra US$0,25 por milhão de RUs efetivamente consumidas.
Na estimativa, o custo passa de **US$46,72/mês fixos** (provisionado) para
**menos de US$0,20/mês** (serverless, no volume estimado), uma diferença
de **~99,6%** nesse componente.

**Trade-off aceito conscientemente:** Serverless tem teto de 5.000 RU/s e
50GB por container (folgado para o volume do projeto) e não suporta
geo-replicação multi-região — não é uma limitação real aqui, já que o
projeto usa uma única `geo_location`.

## 4. Otimização #2 — recomendada (não aplicada nesta fase)

**Azure Reserved Instances / Savings Plan de 1 ano para os nós do AKS.**
Os 3 nós `Standard_B2s` sob demanda somam ~US$91,10/mês; com um
compromisso de 1 ano (Reserved Instance ou Savings Plan de compute), a
faixa de desconto estimada é de **30-40%** sobre o preço sob demanda
(estimativa; não há cotação de reserva nesta pasta) — uma economia
projetada de **~US$27 a US$36/mês** (~US$330-440/ano) só nesse item, sem
qualquer mudança de arquitetura.

**Por que não foi aplicada agora:** reservas de 1 ano são um compromisso
financeiro de longo prazo, adequado para uma carga de produção estável —
não para o ciclo de vida de um ambiente de hackathon/demo que será
destruído ao final da avaliação. A reserva só deve ser avaliada quando a
carga de produção real se confirmar estável por alguns ciclos de billing.

## 5. Rightsizing (Kubernetes)

Valores declarados em [`fiap-tc-5-gitops/apps/*/base/deployment.yaml`](fiap-tc-5-gitops/apps)
(requests → limits):

- **`donation-service`** (Hot Path): 100m/64Mi → 500m/256Mi — os maiores
  limits entre os serviços e o único com HPA (`overlays/prod/hpa.yaml`,
  2–10 réplicas, CPU 70%). O rightsizing aqui prioriza *requests*
  realistas para o HPA escalar com base em uso real.
- **`ngo-service`** e **`notification-service`**: 50m/64Mi → 200m/128Mi
  (mesmo perfil); **`volunteer-service`**: 50m/96Mi → 250m/192Mi. Perfis
  mais enxutos, sem HPA — evita pagar por capacidade ociosa nos nós do AKS.
  O manifesto do `ngo-service` registra que são "valores iniciais
  conservadores", a revisar com o painel de rightsizing.

A evidência operacional fica no painel **"Rightsizing"**, adicionado ao
dashboard `SolidaryTech - Overview` (Grafana,
[`addons/grafana-dashboards/solidarytech-overview-dashboard.yaml`](fiap-tc-5-gitops/addons/grafana-dashboards/solidarytech-overview-dashboard.yaml)),
que compara `container_cpu_usage_seconds_total` (uso real) contra
`kube_pod_container_resource_requests` (o que foi reservado), **só para
CPU** (o painel não cobre memória) — se o uso
real ficar consistentemente abaixo do request, é o sinal para reduzir o
request no próximo ciclo (menos "vacância" reservada = mais pods por nó =
menos nós necessários no autoscaling).

## 6. Resumo executivo

| | Alternativa provisionada (estimada) | Configuração no código (serverless) |
|---|---|---|
| Cosmos DB | US$46,72/mês fixos | **~US$0,13/mês** (estimativa pelo volume esperado de RUs) |
| Forecast total mensal | ~US$220/mês | **~US$173,57/mês** |
| Diferença estimada entre as duas configurações | — | **~US$46,59/mês (~21% do forecast com provisionado)** |
| Economia adicional **recomendada** (não aplicada) | — | ~US$27-36/mês via Reserved Instances (AKS) |
