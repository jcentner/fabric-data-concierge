## Primary project context

I'd like to create a video and blog series that shows off some of the most useful and interesting features of Azure AI Agents, with the target audiences being potential enterprise developers considering it for their companies, cloud solution architects looking to upskill, and anyone interested in developing agents. 

The goal is to keep each session relatively brief (<30-45 minutes), incrementally adding functionality onto an Agent to meet a set of functional requirements. Each video will essentially follow the blog post as a script. We'll start with all the blog posts and produce the videos afterward. The first will cover setting up an environment for Agents. 

The end product will be an Agent that is capable of something relevant to an enterprise building with Azure AI Foundry/Agent Service. It should eventually have enterprise features like observability (perhaps through App Insights integrations), production-ready security, infrastructure-as-code for deployment of resources for scalability, evaluations, and so on. There will be integrations to services like Fabric, Purview, and perhaps Copilot Studio. Connection mechanisms like MCP (model context protocol), OpenAPI 3.0, and Azure Functions will be covered.



## Fabric Data Concierge (self-service analytics + governed access)

### What it does

An agent that lets analysts and PMs: discover governed datasets, ask natural-language questions against Fabric semantic models, generate quick visuals, and request data access—with lineage/PII checks and audit trails.

### Why enterprises care

Bridges the gap between business users and governed data. Demonstrates Fabric + Purview + enterprise auth/observability in a single, credible scenario.

### Core functional requirements

- RAG over catalog: Search and summarize datasets, tables, and KPIs with lineage/context.
- NL Q&A on Fabric: Translate questions to DAX/SQL over a chosen model, return results and a simple chart.
- Access workflow: Create a data-access request if the user lacks permissions; track status.
- Governance checks: Before answering, query Purview for sensitivity labels; if sensitive, mask/deny.
- Audit & observability: Log every query, data source, policy check, and decision.
- Channels: Web/chat UI; optional Copilot Studio hand-off for M365 users.

### Integrations to showcase
- Fabric (semantic models / SQL endpoints / OneLake)
- Purview (classifications, lineage, labels)
- Azure AI Search (indexing data docs/FAQs)
- Azure Functions or Logic Apps (access request workflow)
- OpenAPI 3.0 tools (Service desk/ITSM API; Fabric SQL endpoint wrapper)
- MCP tools (filesystem/git notes, lightweight CSV export)
- App Insights / OpenTelemetry


### 8-episode build path (each 30–45 min)

- IaC foundation: Bicep deploys hub/project, private networking, Key Vault, App Insights.
- Hello, Agent: Single tool (health check). Add Entra auth/RBAC and structured logging.
- Grounding: Index Fabric metadata into AI Search; implement dataset discovery tool.
- NL→SQL/DAX: Tool that runs parameterized queries against a Fabric warehouse/model; return tabular JSON + file attachment.
- Purview policy checks: Pre-answer guard that inspects labels; apply masking/deny + user-friendly rationale.
- Access requests: OpenAPI tool to create/track tickets in ITSM (or Azure DevOps) + status polling.
- Observability & evals: Add trace spans, custom metrics (answerable? latency, denials). Run offline eval set (accuracy/policy adherence).
- Channels & hardening: Surface via Copilot Studio; add rate limits, retries, cost caps, RBAC tests, e2e load test.