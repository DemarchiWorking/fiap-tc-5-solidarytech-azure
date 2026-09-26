# volunteer-service

Match entre voluntários e campanhas das ONGs parceiras da SolidaryTech.

- Linguagem: Python 3.11 / Flask
- Banco: NoSQL — Azure Cosmos DB (Table API) em produção
  (`CLOUD_PROVIDER=azure`) ou AWS DynamoDB em dev local (default)
- Porta: 8083

## Endpoints

| Método | Rota | Descrição |
|---|---|---|
| GET | `/health` | Health check |
| POST | `/volunteers` | Registra um voluntário (`name`, `email`, `ngo_id`) |
| GET | `/volunteers/<ngo_id>` | Lista voluntários de uma ONG |

## Variáveis de ambiente

| Variável | Obrigatória | Descrição |
|---|---|---|
| `PORT` | não (default 8083) | Porta HTTP |
| `CLOUD_PROVIDER` | não (default `aws`) | `azure` usa Cosmos DB Table API; qualquer outro valor usa DynamoDB |
| `AZURE_COSMOSDB_CONNECTION_STRING` / `AZURE_COSMOSDB_TABLE_NAME` | se `CLOUD_PROVIDER=azure` | Credenciais do Cosmos DB |
| `AWS_REGION` / `AWS_DYNAMODB_TABLE` | se `CLOUD_PROVIDER != azure` | Credenciais do DynamoDB (dev local) |
| `OTEL_*` | não | Configuração padrão do OpenTelemetry SDK (ver `fiap-tc-5-gitops/apps/volunteer`) |

No Cosmos DB Table API, `PartitionKey = ngo_id` e `RowKey = volunteer_id` —
permite consultar voluntários de uma ONG via filtro de partição, mais
eficiente que o `scan` completo usado na versão DynamoDB original.

## Observabilidade

Auto-instrumentado via `opentelemetry-instrument` (ver `Dockerfile`) +
métricas customizadas `solidarytech_http_requests_total` e
`solidarytech_http_request_duration_seconds`.

## Nota de correção

O `requirements.txt` original não fixava a versão do `Werkzeug` — com
`Flask==2.2.2` e um `Werkzeug` recente instalado, a aplicação nem sobe
(`ImportError: cannot import name 'url_quote'`). Fixado em
`Werkzeug==2.2.3`.

## CI/CD

`.github/workflows/build-push.yaml`: Lint (flake8) → Test → SonarQube
(SAST) → Trivy (SCA, ignora CVEs sem fix disponível) → Build/Push no ACR →
atualiza o GitOps (`fiap-tc-5-gitops`).

`.github/workflows/self-heal.yml`: disparado pelo Datadog via
`repository_dispatch` (se `volunteer-service` estiver em
`monitored_services` no Terraform).

### Secrets necessários no GitHub

| Secret | Uso |
|---|---|
| `ACR_USERNAME` / `ACR_PASSWORD` | push da imagem |
| `GITOPS_TOKEN` | commit no repo `fiap-tc-5-gitops` |
| `SONAR_TOKEN` / `SONAR_HOST_URL` | SonarQube |
| `AZURE_CREDENTIALS` / `AKS_RESOURCE_GROUP` / `AKS_CLUSTER_NAME` | self-healing |
