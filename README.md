# fabric-data-concierge
Building an enterprise-grade Fabric data Agent episodically.

## Pitch

Fabric Data Concierge is a governed, self-service analytics agent for Microsoft Fabric. It lets employees discover approved datasets, ask questions in natural language, and get accurate, auditable answers while enforcing Purview sensitivity labels and  existing RLS/CLS policies by design. When access is missing, it automatically opens and tracks requests in our ITSM, keeping the workflow compliant and traceable. It runs on Azure AI Agents with managed identities, private networking, and full telemetry via Application Insights, so it’s production-ready and measurable from day one. The result is faster decisions without sacrificing governance, fewer ad-hoc BI/engineering tickets, and clear accountability over how data is used.

## Quick definitions

- **RLS (Row-Level Security):** Rules that filter which rows a user can read based on identity/role.
- **CLS (Column-Level Security):** Rules that hide or mask specific columns (e.g., PII) for certain users.
- **ITSM (using ADO):** IT Service Management workflows; in our case we’ll use Azure DevOps (ADO) to open/track access-request tickets.
- **DAX (Data Analysis Expressions):** The formula language in Power BI/Fabric semantic models for measures, KPIs, and time-intelligence.

## Tech overviews

### Azure AI Agents (Azure AI Foundry)
Managed runtime for building/hosting LLM agents with tools and secure identities. It handles threads/runs, tool calling, safety, and observability. You register tools (Search, Purview, OpenAPI, Azure Functions, MCP) and the service orchestrates reliable, auditable calls under a managed identity.

### Microsoft Fabric
End-to-end analytics SaaS on OneLake: lakehouse, warehouse (SQL), pipelines, notebooks, and Power BI semantic models. You can query via SQL endpoints or DAX and keep security models (RLS/CLS) in one place.

### Microsoft Purview
Data governance suite: catalog your data, track lineage, classify/sensitivity-label assets, and enforce policies. Think “data catalog + governance + compliance signals” your apps can query before touching data.

### Azure DevOps (ADO)
Work management, repos, and pipelines. We’ll use Boards/Work Items as our ITSM system for access requests—created/read via REST/OpenAPI so requests for access and approvals are tracked and auditable.

### Application Insights
Telemetry platform (part of Azure Monitor) for logs, metrics, and distributed traces. We’ll emit spans for agent runs and tool calls, plus custom events for policy decisions, access requests, cost, and latency.
