# Azure Microservices + APIM

AI-powered email security inspection pipeline and chat assistant running on AKS with Azure API Management gateway.

## Architecture

```
                     Azure API Management (APIM)
                              |
        +----------+----------+----------+----------+
        |          |          |          |          |
      /chat     /publish   /inspect   /view      /echo
        |          |          |          |
   [chat svc] [publisher] [inspector] [viewer]
        |          |          |          |
   Azure OpenAI  Service    Azure     Table
   + Key Vault    Bus       OpenAI    Storage
                   |          |
              [receiver] ----+--- Table Storage
```

### Email Security Pipeline

1. **Publisher** receives messages via `POST /message` and sends them to Azure Service Bus
2. **Receiver** (daemon) consumes from Service Bus, calls Inspector, stores results
3. **Inspector** classifies emails as `normal` / `spam` / `fraud` using Azure OpenAI GPT-3.5
4. **Viewer** exposes classified results from Azure Table Storage via `GET /emails`

### Chat Assistant

- **Chat** service forwards messages to Azure OpenAI GPT-3.5-turbo
- **ui-chat** React SPA with syntax highlighting and Mermaid diagram rendering

## Services

| Service | Tech | Endpoints | Azure Dependencies |
|---------|------|-----------|-------------------|
| chat | FastAPI | `GET /ping`, `POST /message` | Key Vault, OpenAI |
| publisher | FastAPI | `GET /ping`, `POST /message` | Service Bus |
| inspector | FastAPI | `GET /ping`, `POST /inspect` | Key Vault, OpenAI |
| receiver | Python daemon | - | Service Bus, Table Storage |
| viewer | FastAPI | `GET /ping`, `GET /emails` | Table Storage |
| ui-chat | React 19 | SPA | Static Web App |

## Project Structure

```
chat/                    # AI chat service
inspector/               # Email classifier service
publisher/               # Service Bus message publisher
receiver/                # Async message consumer daemon
viewer/                  # Email results viewer
ui-chat/                 # React frontend
releases/
  {service}/
    HELM/                # Helm chart (Deployment, Service, ConfigMap, SA)
    APIMConfig/          # APIM API definitions, policies, OpenAPI specs
    pipeline-*.yaml      # Azure DevOps release pipelines
```

## CI/CD

**Build** (per service): Docker build + push to ACR, triggered on changes to service directory.

**Release** (per service):
1. Pull image reference from build artifact
2. Helm upgrade on AKS
3. Wait for internal LoadBalancer IP
4. Push APIM configuration via APIOps

All pipelines run on a self-hosted Azure DevOps agent.

## Key Design Decisions

- **Workload Identity** - each service has its own User Managed Identity federated to a K8s service account (zero secrets in cluster)
- **Internal LoadBalancers** - services are not internet-accessible; all traffic flows through APIM
- **APIOps** - APIM configuration (APIs, policies, diagnostics) is version-controlled and deployed via `azure/apiops`
- **Namespace isolation** - each service runs in its own K8s namespace

## APIM Endpoints

| Path | Service | Versioned |
|------|---------|-----------|
| `/chat` | chat | v1 |
| `/publish` | publisher | v1 |
| `/inspect` | inspector | v1 |
| `/view` | viewer | v1 |
| `/echo` | echo | no |
| `/mock` | mock-response | no |
