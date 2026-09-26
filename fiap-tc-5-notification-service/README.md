# notification-service

Serviço **novo** (não fazia parte do código-fonte original do desafio) —
consome os eventos de doação publicados pelo `donation-service` no Azure
Service Bus e grava o registro de cada notificação processada no Cosmos DB
Table API. Fecha o trace distribuído assíncrono
`donation-service -> Service Bus -> notification-service`, mesmo padrão
`evaluation-service -> analytics-service` da Fase 3/4.

- Linguagem: Python 3.11 / Flask + worker em thread
- Mensageria: Azure Service Bus (fila `donation-events`)
- Banco: Azure Cosmos DB Table API (tabela `DonationNotifications`)
- Porta: 8084

## Como funciona

O Flask só expõe `/health` — o processamento de verdade acontece numa
thread em background (`start_worker`, iniciada na importação do módulo,
mesmo padrão do `analytics-service` da Fase 3/4), que fica consumindo a
fila do Service Bus continuamente.

Para cada mensagem: extrai o `traceparent`/`tracestate` propagado pelo
`donation-service` das propriedades da mensagem
(`message.application_properties`), abre um span filho
(`notification.process_event`, `SpanKind.CONSUMER`) linkado ao mesmo
trace, e grava o registro no Cosmos DB (`PartitionKey = ngo_id`,
`RowKey` = UUID gerado).

## Variáveis de ambiente

| Variável | Obrigatória | Descrição |
|---|---|---|
| `PORT` | não (default 8084) | Porta HTTP |
| `AZURE_SERVICEBUS_CONNECTION_STRING` | sim | Connection string do Service Bus |
| `AZURE_SERVICEBUS_QUEUE_NAME` | não (default `donation-events`) | Fila a consumir |
| `AZURE_COSMOSDB_CONNECTION_STRING` | sim | Connection string do Cosmos DB |
| `AZURE_COSMOSDB_TABLE_NAME` | não (default `DonationNotifications`) | Tabela de destino |
| `OTEL_*` | não | Configuração padrão do OpenTelemetry SDK (ver `fiap-tc-5-gitops/apps/notification`) |

## Observabilidade

- `/health` é auto-instrumentado via `opentelemetry-instrument`.
- O worker consumidor usa o tracer/meter **manuais** do OTel (não é
  coberto pela auto-instrumentação, que só cobre o ciclo de
  request/response HTTP do Flask).
- Métricas: `solidarytech_http_requests_total` /
  `..._duration_seconds` (HTTP) e `solidarytech_messages_processed_total`
  (mensagens consumidas, com atributo `status=success|error`).

## CI/CD

`.github/workflows/build-push.yaml`: Lint (flake8) → Test → SonarQube
(SAST) → Trivy (SCA, ignora CVEs sem fix disponível) → Build/Push no ACR →
atualiza o GitOps (`fiap-tc-5-gitops`).

`.github/workflows/self-heal.yml`: disparado pelo Datadog via
`repository_dispatch` (se `notification-service` estiver em
`monitored_services` no Terraform).

### Secrets necessários no GitHub

| Secret | Uso |
|---|---|
| `ACR_USERNAME` / `ACR_PASSWORD` | push da imagem |
| `GITOPS_TOKEN` | commit no repo `fiap-tc-5-gitops` |
| `SONAR_TOKEN` / `SONAR_HOST_URL` | SonarQube |
| `AZURE_CREDENTIALS` / `AKS_RESOURCE_GROUP` / `AKS_CLUSTER_NAME` | self-healing |
