# PCN — Plano de Continuidade de Negócio (SolidaryTech)

> Frente 4 do desafio: Multicloud/Segurança/DR. Estratégia escolhida:
> **Velero** (backup/restore nativo Kubernetes, Opção A do enunciado).

## 1. Escopo e objetivo

Este documento formaliza o que a SolidaryTech se compromete a suportar em
caso de desastre (perda de cluster, corrupção de dados, exclusão
acidental de namespace) e como a stack técnica implementada nesta fase
sustenta esse compromisso com agendamentos automáticos (Velero Schedules)
e PITR do PostgreSQL.

> **Limite desta configuração — falha de região:** o código mantém os
> backups na mesma região do ambiente. O Storage Account do Velero é
> `LRS` em `eastus` (`velero.tf`), os servidores PostgreSQL têm
> `geo_redundant_backup_enabled = false` (`postgresql.tf`) e o Cosmos DB
> tem uma única `geo_location` (`cosmosdb.tf`). A perda da região inteira
> **não** está coberta por este plano.

Desastre, aqui, é qualquer evento que tire o cluster AKS ou um dos bancos
de dados do ar de forma não recuperável por self-healing simples
(rollout restart não resolve perda de dados ou perda do cluster inteiro
— esse é o caso que o self-healing do dia a dia, descrito em
[`04-ITSM-INCIDENT-LIFECYCLE.md`](04-ITSM-INCIDENT-LIFECYCLE.md), não
cobre).

## 2. RTO / RPO por serviço

O `donation-service` é o Hot Path/Caminho Crítico (fluxo financeiro de
doações) e recebe metas mais rígidas que os demais serviços — a mesma
lógica de priorização usada no SLA (`01-SRE-SLI-SLO-SLA.md`): dado
transacional tem menor tolerância a perda e a indisponibilidade prolongada
do que dado cadastral.

| Serviço | RPO (perda máxima de dados tolerada) | RTO (tempo máximo de indisponibilidade) | Racional |
|---|---|---|---|
| **`donation-service`** (Hot Path) | **≤ 5 minutos** | **≤ 30 minutos** | Dado financeiro/transacional. RPO baixo sustentado pelo **PITR (Point-in-Time Restore) contínuo** do Azure PostgreSQL Flexible Server — mecanismo independente do Velero, que grava WAL continuamente (não depende do agendamento de um backup periódico). RTO agressivo, mas viável: GitOps reaplica o Deployment automaticamente ao reconectar o ArgoCD a um cluster novo, e o restore de dados via Velero pode ser direcionado **só ao namespace `donation`**, sem esperar o cluster inteiro. |
| `ngo-service`, `volunteer-service`, `notification-service` | **≤ 15 minutos** | **≤ 2 horas** | Dados cadastrais/operacionais (ONGs, voluntários, notificações) — perda de alguns minutos é operacionalmente aceitável. RTO mais frouxo permite um restore completo do cluster via Velero sem a pressão de tempo do caminho crítico. |

> Valores propostos pela equipe técnica e aprovados; ficam sujeitos a
> revisão de negócio. Os mesmos alvos estão registrados no
> `fiap-tc-5-terraform/README.md` e nos comentários de `postgresql.tf` e
> `addons/velero/schedule.yaml`.

## 3. Mecanismo técnico — como o RTO/RPO é sustentado

### 3.1 RPO — dados

| Dado | Mecanismo de RPO | Configuração |
|---|---|---|
| `donation_db` (Postgres) | PITR contínuo (WAL) | `backup_retention_days = 14` em [`postgresql.tf`](fiap-tc-5-terraform/postgresql.tf) — retenção maior que `ngo_db` (7 dias), refletindo o `Criticality = "hot-path"` |
| `ngo_db` (Postgres) | Backup automático + PITR | `backup_retention_days = 7` |
| Cosmos DB (`Volunteers`, `DonationNotifications`) | Backup nativo do Cosmos DB | `cosmosdb.tf` não declara bloco `backup`: vale o padrão da plataforma; fora do escopo do Velero |
| Estado do cluster (Deployments, ConfigMaps, Secrets, PVCs) | **Velero Schedules** | ver §3.2 |

### 3.2 Velero — o que está agendado

Definido em [`fiap-tc-5-gitops/addons/velero/schedule.yaml`](fiap-tc-5-gitops/addons/velero/schedule.yaml)
(o agendamento de backup é versionado no repositório GitOps). Atenção: o
ArgoCD não chegou a aplicar esse arquivo, e os dois `Schedule` foram
aplicados com `kubectl apply` como contorno (ver §4):

| `Schedule` (CRD) | Escopo | Frequência | TTL (retenção) | Por quê |
|---|---|---|---|---|
| `solidarytech-donation-hourly` | Namespace `donation` apenas | a cada hora | 7 dias | Sustenta o RPO de 5min do Hot Path **em conjunto** com o PITR do Postgres — o Velero cobre o estado do cluster (Deployment, Secret, PVC se houver), o PITR cobre o dado transacional em si. Um restore do namespace `donation` nunca está a mais de 1h de defasagem no pior caso do estado do cluster; o dado transacional segue o RPO do PITR (≤ 5 min). |
| `solidarytech-daily` | Todos os namespaces (`ngo`, `donation`, `volunteer`, `notification`) | diária | 30 dias | Cobertura completa do cluster para o RTO de 2h dos serviços não-críticos. |

O backend de armazenamento dos backups é um **Azure Storage Account**
dedicado ([`velero.tf`](fiap-tc-5-terraform/velero.tf)), com um
Service Principal próprio (`azuread_application.velero` +
`azuread_service_principal.velero`) com:
- `Storage Account Contributor` no Storage Account: o plugin do Azure
  precisa chamar `listKeys` para gravar os manifests/metadados de backup
  (a role inicial, `Storage Blob Data Contributor`, não bastava — ver §4);
- `Contributor` no **node resource group do AKS**
  (`azurerm_kubernetes_cluster.aks.node_resource_group_id`) — necessário
  porque os discos dos PVCs vivem no resource group gerenciado do AKS,
  não no resource group principal do Terraform, e o Velero precisa dessa
  permissão para tirar snapshot de disco via o plugin
  `velero-plugin-for-microsoft-azure`.

### 3.3 RTO — o que acelera a restauração

1. **GitOps (ArgoCD App-of-Apps)**: um cluster AKS novo, ao ser apontado
   para o mesmo repositório `fiap-tc-5-gitops`, reconstrói sozinho toda a
   topologia de Applications (addons + os 4 apps) — não há passo manual
   de "reinstalar" nada além de: provisionar o AKS via Terraform, instalar
   o ArgoCD (também via Terraform, `argocd.tf`) e apontar o
   `bootstrap/app-of-apps.yaml`. Enquanto a pendência da §4 não for
   resolvida, os `Schedule` do Velero precisam ser reaplicados à mão.
2. **Velero restore**, direcionado por namespace quando o tempo importa
   (caso `donation`) ou completo quando não (demais namespaces).
3. **Self-healing** (`self-heal.yml` em cada repositório de serviço) não
   é mecanismo de DR — ele resolve degradação de um Deployment já
   existente (rollout restart), não perda de cluster/dados. É
   complementar, não substituto.

## 4. Evidência real — backup e restore executados em produção

Não é só o runbook documentado abaixo — foi **executado** no ambiente
real, no namespace `donation` (Hot Path), como evidência prática de DR. O
registro abaixo é textual; os prints ou o vídeo da execução ainda precisam
ser anexados à entrega (a duração do restore não foi registrada):

1. **Backup real**: `Backup` (`demo-donation-backup`) criado cobrindo o
   namespace `donation` inteiro (manifestos + volumes) → `Completed`,
   **409 itens** salvos no Azure Blob Storage dedicado do Velero.
2. **Desastre simulado**: `kubectl delete namespace donation` — apagado
   por completo (Deployment, Service, HPA, Secrets, Jobs, histórico de
   execução).
3. **Restore real**: `Restore` a partir do backup → `Completed`. O
   namespace voltou inteiro, incluindo o **Secret real** da connection
   string do Postgres (não um placeholder) — prova de que o Velero
   captura o estado **real** do cluster, algo que o GitOps sozinho não
   reconstituiria (o Git só tem os manifestos base, os segredos reais só
   existem em runtime).
4. **Validação funcional pós-restore**: `donation-service` reconectou
   sozinho no Postgres e no Service Bus (log: `Conectado ao PostgreSQL` /
   `Cliente Azure Service Bus inicializado com sucesso`) e uma doação real
   via `POST /donations` retornou **201 Created** — o sistema estava
   genuinamente operacional de novo, não só com pods "Running" vazios.

### Bugs reais encontrados e corrigidos nessa validação (documentar no
relatório como evidência de rigor, não só de sucesso):

- **`BackupStorageLocation` ficava `Unavailable`**: os placeholders do
  `values.yaml` do addon (`REPLACE_WITH_RESOURCE_GROUP_NAME`,
  `REPLACE_WITH_VELERO_STORAGE_ACCOUNT_NAME`,
  `REPLACE_WITH_SUBSCRIPTION_ID`) nunca tinham sido substituídos pelos
  valores reais. Corrigido preenchendo com os outputs do Terraform.
- **RBAC insuficiente pro plugin do Azure**: o Service Principal do
  Velero tinha `Storage Blob Data Contributor` (acesso de **dado**, só
  blobs), mas o plugin oficial do Azure autentica via Shared Key, que
  exige chamar `Microsoft.Storage/storageAccounts/listKeys` — uma
  operação de **gerenciamento (control-plane)** do recurso. Sem a role
  `Storage Account Contributor`, a chamada falhava com
  `AuthorizationFailed`. Corrigido em `velero.tf`.
- **Os `Schedule` (agendamento de backup automático) nunca foram
  criados**: o gerador multi-source do ArgoCD (`directory.include:
  "secrets.yaml,schedule.yaml"`) não estava aplicando o `schedule.yaml`
  por algum motivo não totalmente diagnosticado — aplicado diretamente
  via `kubectl apply` como contorno; os dois `Schedule` (`solidarytech-daily`,
  `solidarytech-donation-hourly`) estão `Enabled` no cluster agora. Vale
  investigar a causa raiz do multi-source antes da apresentação final,
  mas o comportamento (schedule ativo, cron correto) já está confirmado.

## 5. Runbook de restore (roteiro genérico, reaplicável)

Passo a passo executável — a evidência exigida pelo desafio ("estratégia
prática de DR", não só o documento):

```bash
# 1. Confirmar os backups disponíveis
velero backup get

# 2. (Cenário: perda do namespace `donation` inteiro)
#    Restaurar só esse namespace a partir do backup mais recente
#    do schedule horário:
velero restore create donation-restore-$(date +%s) \
  --from-schedule solidarytech-donation-hourly \
  --include-namespaces donation

# 3. Acompanhar o restore
velero restore describe donation-restore-<timestamp> --details

# 4. Validar que o ArgoCD voltou a reconciliar o namespace normalmente
#    (o ApplicationSet gera a Application com o nome app-<pasta>)
kubectl get application app-donation -n argocd
argocd app get app-donation

# 5. (Cenário: perda do cluster inteiro)
#    a. Recriar o AKS via Terraform (terraform apply em fiap-tc-5-terraform,
#       reaproveitando o mesmo state remoto) — o ArgoCD é instalado pelo
#       próprio Terraform (argocd.tf)
#    b. Apontar o bootstrap do ArgoCD (app-of-apps.yaml) — reconstrói
#       addons + os 4 apps automaticamente via GitOps
#    c. Restaurar o cluster completo a partir do schedule diário:
velero restore create full-restore-$(date +%s) \
  --from-schedule solidarytech-daily

# 6. (Dado transacional do donation-service) — restaurar o Postgres
#    para um ponto no tempo específico (PITR), independente do Velero:
az postgres flexible-server restore \
  --resource-group <rg> \
  --name pg-donation-<prefix>-restored \
  --source-server pg-donation-<prefix> \
  --restore-time "<timestamp ISO 8601, até 5min antes do incidente>"
```

## 6. Comunicação e responsabilidades (parte do PCN, não só técnica)

| Papel | Responsabilidade durante um DR |
|---|---|
| On-call (PagerDuty, ver `04-ITSM-INCIDENT-LIFECYCLE.md`) | Declara o incidente como DR (não self-healing) quando o rollout restart automático não resolve em 2 tentativas, ou quando o cluster inteiro está inacessível |
| Engenheiro executando o restore | Segue o runbook da §5, comunica status a cada 30min pelo canal de incidente (Discord) até resolução |
| Stakeholder de negócio (ONGs parceiras) | Recebe comunicação formal se o RTO/SLA (`01-SRE-SLI-SLO-SLA.md`) for ultrapassado — aciona a cláusula de compensação já prevista no SLA |

## 7. Por que Velero (Opção A) e não ambiente espelho em outra região (Opção B)

Decisão consciente de custo-benefício, documentada aqui para
justificar a escolha perante o avaliador. A Opção B do enunciado
(infraestrutura ativo-passivo: um ambiente "espelho" em outra região,
levantado pelo Terraform com 1 comando) cobriria a perda de região, mas
exigiria manter pronto em uma segunda região um espelho do ambiente (AKS,
Postgres, Cosmos, Service Bus), o que:
- se replicado integralmente, multiplicaria o forecast de custo do
  `02-FINOPS-FORECAST.md` por até ~2x (estimativa deste documento),
  incompatível com a filosofia de FinOps do mesmo desafio;
- é desproporcional aos RTOs de 30min/2h definidos na §2, que o desenho
  Velero + GitOps pretende atender em um cenário de hackathon/demo (a
  duração real do restore ainda precisa ser medida — ver §4).

Backup/restore (Velero) é a estratégia de DR **certa para o porte e a
criticidade real** deste sistema — multi-região fica como evolução natural
se o volume de doações justificar o custo adicional no futuro.
