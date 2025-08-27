
# Fabric Data Concierge — Plan

**Scope**: Detailed syllabus (with acceptance criteria) and target architecture for the Fabric Data Concierge.  
**Note**: No Bicep templates in this document; this is the “what” and “how it fits together.”

---

## Overview

**Goal**: Build an enterprise-ready agent that helps users discover governed datasets, ask natural-language questions over Microsoft Fabric models/warehouses, enforce data governance via Purview, and handle access requests—with full observability and auditability.

**Core capabilities**  
1. **Governed discovery**: Find/summarize datasets, tables, KPIs, and lineage context.  
2. **NL → SQL/DAX**: Answer questions over a selected Fabric warehouse/lakehouse/semantic model; return tables and quick charts.  
3. **Governance checks**: Interrogate Purview classifications/sensitivity labels before answering; mask or deny if required.  
4. **Access workflow**: If the user lacks permissions, open a request and track status.  
5. **Observability & audit**: Trace every step (tools, policy decisions, errors) with structured telemetry.

---

## Target Architecture

### High-level components
- **Agent Runtime (Azure AI Foundry / Agents Service)**
  - System prompt, tools registry, safety settings, server-side runs/threads.
  - Uses **Managed Identity** to call internal tools/services.
- **Tools (server-side)**
  1. **Catalog Search Tool**: Queries **Azure AI Search** index of Fabric metadata (datasets, tables, measures, lineage, descriptions).
  2. **Query Runner Tool**: Executes parameterized SQL (warehouse/lakehouse endpoint) or DAX against a Power BI semantic model.
  3. **Purview Policy Tool**: Looks up resource sensitivity labels/classifications and lineage; returns policy verdicts + masking instructions.
  4. **Access Request Tool**: Creates and checks tickets in ITSM (ServiceNow/Azure DevOps) via OpenAPI.
  5. **File/Chart Tool** (optional): Creates CSV/PNG artifacts for answers; stores in secure blob with short-lived SAS for retrieval.
- **Data & Services**
  - **Microsoft Fabric**: Warehouse/Lakehouse SQL endpoints; Power BI semantic models (for DAX).
  - **Microsoft Purview**: Classifications, sensitivity labels, lineage graph.
  - **Azure AI Search**: Indexed catalog (titles, columns, measures, owners, glossary terms, lineage text).
  - **ITSM**: ServiceNow/Azure DevOps REST endpoint.
- **Security/Networking**
  - **Azure Entra ID/RBAC** for users and service principals; **Managed Identity** for server-side calls.
  - **Private Endpoints/VNet** for Fabric SQL, Search, Storage, Purview (where applicable).
  - **Key Vault** for secrets (e.g., ITSM PAT or OAuth client).
  - **Row-level/column-level security** honored by Query Runner Tool (no elevation).
- **Observability**
  - **Application Insights** + **OpenTelemetry**: traces, metrics (latency, cost, tool error rate), logs (policy denials, masked fields).
  - **Audit store** (App Insights `customEvents` + optional Storage table) with: user, data object, labels, decision, tool call hashes, ticket IDs.
- **Channels**
  - **Web chat** (internal app) + optional **Copilot Studio** connector for M365 users.

### Identity & authorization model
- **End-user** authenticates via Entra ID → web app → passes a user assertion to the Agent (for personalization and policy context).
- **Agent** uses **Managed Identity** to call:
  - AI Search (read), Purview (read), Fabric SQL endpoints (delegated or MI-scoped read, never bypassing RLS), ITSM (app credential).
- **Least privilege**: Query Runner Tool only has read to selected Fabric endpoints; ITSM tool only allowed to specific project/queue.

### Core data flows
**A) Dataset discovery**  
1. User asks: “What sales data is available for Europe?”  
2. Agent → Catalog Search Tool (AI Search) with facets (domain=Sales, region=EU).  
3. Tool returns top entities with snippets + owners + lineage.  
4. Agent composes a ranked, cited summary and suggests a target model.

**B) NL → SQL/DAX answer**  
1. User selects a dataset/model.  
2. Agent calls Purview Policy Tool for the specific objects/columns implied by the query intent.  
3. If **allow/mask** → Agent generates parameterized SQL/DAX; calls Query Runner Tool.  
4. Tool returns shaped rows; Agent returns table + inline chart + citations; stores CSV/PNG via File/Chart Tool.

**C) Access request**  
1. Policy Tool says **deny** OR Query Runner indicates permission error.  
2. Agent explains the reason and calls Access Request Tool with resource identifiers + justification template.  
3. Ticket ID returned; Agent posts status + “notify me” option; logs decision.

**D) Observability**  
- Every step emits spans (agent run, tool call), attributes (model, dataset, label, verdict), metrics (latency, tokens), and custom events (policy_denial, access_request_created).

---

## 8-Episode Syllabus (30–45 min each)

### Episode 1 — Secure foundation & “Hello, Concierge”
**Goal**: Stand up the minimal, secure runtime with one trivial tool; wire up logging.  
**Scope**
- Create the Agent (system prompt, enterprise tone, refusal/masking policy stubs).
- Add **HealthCheck Tool** (returns OK + environment metadata).
- Configure **Managed Identity**, Key Vault references (no secrets inline).
- Enable **Application Insights** (basic traces, run IDs).
**Acceptance Criteria**
- AC1: Agent responds and invokes HealthCheck Tool on prompt “status?”
- AC2: App Insights shows traces for: thread → run → tool call, with correlation IDs.
- AC3: No secrets in code; MI is the only identity used for platform calls.

### Episode 2 — Catalog discovery (AI Search)
**Goal**: Users can find governed data assets.  
**Scope**
- Build an AI Search index from Fabric metadata export (titles, descriptions, owners, columns, lineage text).
- Implement **Catalog Search Tool** (query + filters: domain, sensitivity, owner).
- Response schema: list of entities with IDs, type, summary, owners, lineage snippet, sensitivity (if known).
**Acceptance Criteria**
- AC1: Query “customer churn datasets in marketing” returns relevant assets with owner & lineage snippet.
- AC2: Latency P50 < 1.5s tool time for top-5 results.
- AC3: Trace includes query text, filter facets, and selected entity IDs.

### Episode 3 — Purview policy check (allow/mask/deny)
**Goal**: Enforce governance before any data retrieval.  
**Scope**
- Implement **Purview Policy Tool**: given resource IDs/columns, return sensitivity labels, PII types, lineage parents, and a **verdict**: {allow, mask:[col…], deny, reason}.
- Add agent pre-flight step: every Q&A run calls policy first.
- Add **masking schema** (e.g., redact, bucketize, null).
**Acceptance Criteria**
- AC1: For a resource with “Confidential/PII”, the tool returns **mask** with specified columns.
- AC2: For a resource outside allowed domain, the tool returns **deny** with a human-readable reason.
- AC3: Agent explains denials/masking clearly; traces include the verdict and impacted columns.

### Episode 4 — NL → SQL (Warehouse/Lakehouse)
**Goal**: Natural language answers over a chosen dataset with SQL.  
**Scope**
- Implement **Query Runner Tool (SQL)**: parameterized statements only; returns rows + schema + execution time; honors RLS/CLS.
- Prompting pattern for **SQL generation + verification** (schema-aware, safe casts, guards).
- Basic chart generation via File/Chart Tool (e.g., small PNG for top-N bar).
**Acceptance Criteria**
- AC1: “Top 5 products by revenue last quarter” returns a correct table (spot-checked) and a chart.
- AC2: Masked columns are redacted according to Episode 3 rules.
- AC3: SQL text and execution stats logged to telemetry (redacted where needed).

### Episode 5 — NL → DAX (Semantic model / Power BI)
**Goal**: Support questions over a Power BI model (measures/KPIs).  
**Scope**
- Extend Query Runner with a **DAX endpoint** option; safe, parameterized template for filter values.
- Add selection logic: if user references KPIs/measures, prefer DAX; otherwise SQL.
**Acceptance Criteria**
- AC1: “What was Gross Margin % by month YTD?” returns a measure series matching the model.
- AC2: Agent cites the model and measure names used.
- AC3: P50 answer end-to-end < 12s on a warm path.

### Episode 6 — Access requests (ITSM integration)
**Goal**: Graceful path when users lack permissions.  
**Scope**
- **Access Request Tool** via OpenAPI (ServiceNow/ADO): create ticket with resource ID, owner, justification; **Status Tool** to poll.
- Agent patterns: propose request, confirm user intent, then create and return ticket link + ETA.
**Acceptance Criteria**
- AC1: When a permission error occurs, user is offered to create a request; on “yes,” a ticket is created and ID returned.
- AC2: “Check my access request” surfaces latest status.
- AC3: Audit log contains {user, resource, reason, ticketId}.

### Episode 7 — Evaluations & reliability guardrails
**Goal**: Quantify quality/safety; add reliability features.  
**Scope**
- Offline **eval set**: ~50 prompts covering discovery, SQL, DAX, masking/denial cases.
- Metrics: answerability, grounding correctness (schema used), numeric accuracy (tolerances), policy adherence, latency, tool failure rate.
- Guardrails: retries with backoff for transient SQL; cost/latency budget per run; circuit-break on repeated failures.
**Acceptance Criteria**
- AC1: Eval harness runs and produces a report with pass/fail per category.
- AC2: Zero policy violations on the eval set.
- AC3: P95 end-to-end latency under a configured threshold (e.g., 18s) for green-path queries.

### Episode 8 — Channels, hardening & UX polish
**Goal**: Ready for pilot usage.  
**Scope**
- Minimal **web UI**: login, chat, dataset picker, results table, inline chart, “Request access,” “Export CSV.”
- Optional **Copilot Studio** connection for M365 channel.
- Rate limits, content policy refinements, cost dashboards, RBAC smoke tests.
**Acceptance Criteria**
- AC1: Authenticated user can discover data, run a query, see chart/table, export CSV.
- AC2: Rate limiting in place (per-user qps) and recorded in telemetry.
- AC3: Playbook document exists for on-call (where logs live, common errors, rollback steps).

---

## Tool Interfaces (concise schemas)

**Catalog Search Tool (AI Search)**  
- **Input**:  
  ```json
  { "query": "string", "filters": { "domain": ["..."], "sensitivity": ["..."], "owner": ["..."] }, "top": 10 }
  ```
- **Output**:  
  ```json
  { "items": [{ "id": "...", "type": "dataset|table|kpi", "name": "...", "description": "...", "owners": ["..."], "lineage": "...", "sensitivity": "..." }], "queryTimeMs": 123 }
  ```

**Purview Policy Tool**  
- **Input**:  
  ```json
  { "resources": [{ "id": "...", "type": "table|column|model" }], "columns": ["..."] }
  ```
- **Output**:  
  ```json
  { "verdict": "allow|mask|deny", "maskedColumns": ["..."], "reason": "...", "labels": [{ "name": "...", "source": "Purview" }], "lineage": ["..."] }
  ```

**Query Runner Tool**  
- **Input (SQL mode)**:  
  ```json
  { "mode": "sql", "endpointId": "...", "database": "...", "statementTemplate": "SELECT ... WHERE date BETWEEN @start AND @end", "parameters": { "start": "2024-01-01", "end": "2024-03-31" } }
  ```
- **Input (DAX mode)**:  
  ```json
  { "mode": "dax", "datasetId": "...", "expressionTemplate": "CALCULATE([Gross Margin %], DATESYTD('Date'[Date]))", "parameters": {} }
  ```
- **Output**:  
  ```json
  { "rows": [{ "col": "val" }], "schema": [{ "name": "col", "type": "string" }], "stats": { "durationMs": 345, "rowCount": 5 }, "artifact": { "csvUrl": "https://...", "chartPngUrl": "https://..." } }
  ```

**Access Request Tool**  
- **Input**:  
  ```json
  { "resourceId": "...", "resourceType": "table|dataset|model", "reason": "...", "userContext": { "upn": "user@contoso.com" }, "urgency": "low|normal|high" }
  ```
- **Output**:  
  ```json
  { "ticketId": "INC12345", "url": "https://...", "status": "new|in_progress|resolved" }
  ```

**Status Tool**  
- **Input**:  
  ```json
  { "ticketId": "INC12345" }
  ```
- **Output**:  
  ```json
  { "status": "in_progress", "lastUpdate": "2024-07-18T12:34:56Z", "assignee": "jane.doe" }
  ```

**File/Chart Tool**  
- **Input**:  
  ```json
  { "table": [{ "x": "A", "y": 10 }], "chartSpec": { "x": "x", "y": "y", "type": "bar" } }
  ```
- **Output**:  
  ```json
  { "csvUrl": "https://...", "chartPngUrl": "https://..." }
  ```

---

## Telemetry & Audit (minimum fields)

- **Tracing**: `runId`, `threadId`, `tool.name`, `tool.latencyMs`, `tokenUsage`, `model`, `cacheHit`  
- **Policy decisions**: `resourceId`, `labels[]`, `verdict`, `maskedColumns[]`, `reason`  
- **Queries**: `mode`, `endpointId/datasetId`, `execMs`, `rowCount` (no PII values)  
- **Tickets**: `ticketId`, `status`, `resourceId`, `requester`

---

## Quality & Safety Bars (series-level Definition of Done)

- No query runs without a **prior policy verdict**.  
- Masking/denials are **explained** to the user in plain language.  
- All tool invocations and decisions are **traceable** in App Insights.  
- **Least privilege** confirmed via RBAC tests; no hard-coded secrets.  
- Eval set shows **zero** policy violations and acceptable accuracy/latency targets.

---

## Risks & Mitigations

- **Schema drift** → Regenerate AI Search index nightly; tool fetches live schema before SQL generation.  
- **Ambiguous NL** → Enforce disambiguation turns (e.g., pick a dataset/model) before generating queries.  
- **Permission gaps** → MI must not bypass RLS—verify with negative tests.  
- **Cost/latency** → Enable caching for frequent catalog answers; cap result sizes; paginate tables.
