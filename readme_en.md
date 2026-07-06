[한국어](#top) | **English**

# Azure SRE Agent — Specification & Cost Structure Overview

> Basis: Microsoft Learn official documentation (as of 2026-07). Rates/models/regions are subject to change, so
> re-verify in the [Azure pricing calculator](https://azure.microsoft.com/pricing/details/sre-agent/).

---

## 1. Summary

- **There is no officially documented upper limit on the "maximum number of resources (N AKS clusters, N PostgreSQL instances)" a single agent can handle.**
- An agent's coverage is determined **not by a count limit but by RBAC scope** (resource group level or subscription level).
- A single agent can **manage many resources within its scope simultaneously**, and Microsoft recommends consolidating workloads into a single agent (to reduce always-on cost).
- An official documentation example cites a single agent recognizing **"251 resources across 3 resource groups (5 Container Apps, 2 AKS clusters, etc.)"** — this is only an example, not a limit.

---

## 2. Scope a Single Agent Can Cover (Specification)

### 2.1 Coverage Model — "Scope," Not "Count"

The agent's management targets are determined by the **RBAC access scope** granted at creation time.

| Access grant method | Description |
|---|---|
| **Resource group level** | Access to all resources within the selected resource group(s) (multiple selectable) |
| **Subscription-level** | When the managed identity is granted the **Reader** role over the entire subscription, it recognizes all resources in the subscription |

- That is, hard-coded per-resource-type count limits such as **a maximum number of AKS clusters / PostgreSQL instances** do not exist in the official documentation.
- The practical limit is determined by the size of the granted scope, the token/AAU cost consumed during investigation (active flow), and the context window.
- **"Can a single agent handle multiple workloads?"** → Official FAQ answer: **"Yes. A single agent can monitor multiple resources within its scope, and consolidating workloads into one reduces always-on cost compared to running multiple separate agents."**

> Note: For AKS diagnostics or PostgreSQL diagnostics, there is no documented product-level quantitative cap on the number of clusters/instances itself.
> For large environments, the realistic consideration is to separate/design agents based on **investigation cost (AAU), region/scope configuration, and response performance** rather than a count limit.

### 2.2 Managed Azure Services

| Category | Supported services |
|---|---|
| **Compute** | Virtual Machines, App Service, Container Apps, **AKS**, Azure Functions, etc. |
| **Storage** | Blob storage, File shares, Managed disks, Storage accounts |
| **Networking** | Virtual networks, Load balancers, Application gateways, NSG |
| **Databases** | Azure SQL Database, Cosmos DB, **PostgreSQL**, MySQL, Redis |
| **Monitoring/Management** | Azure Monitor, Log Analytics, Application Insights, Resource Manager |

- Virtually all **Azure CLI operations** can be automated via runbooks / subagents / agent hooks.

### 2.3 Extension Primitives (Agent Capability Building Blocks)

| Primitive | Description |
|---|---|
| **Skills** | Individual capabilities such as marketplace runbooks and Azure CLI scripts |
| **Subagents** | 5 built-in by default (architecture, logs/metrics, source code, root cause analysis, scanning) + custom configurable |
| **Python tools** | Custom logic / data transformation / API integration |
| **MCP servers** | 40+ connectors (Datadog, Prometheus, Grafana, New Relic, Splunk, Dynatrace, AWS CloudWatch, GCP, etc.) or custom |
| **Agent hooks** | Lifecycle-event-based automation (command hook / prompt hook) |
| **Permission gate** | A pre-approval and policy-validation layer before every tool call |

### 2.4 Integrations

- **Monitoring/Observability**: Azure Monitor, Application Insights, Log Analytics, Grafana
- **Incident management**: Azure Monitor Alerts, PagerDuty, ServiceNow
- **Source/CI-CD**: GitHub, Azure DevOps
- **Data sources**: Azure Data Explorer (Kusto), MCP servers
- **Communication**: Slack, Microsoft Teams

### 2.5 Execution Isolation & Security (Key Limits)

- **Dedicated sandbox group per agent** (micro VM based on Azure Dedicated Compute) — separates the reasoning loop from tool execution.
- **Full per-customer isolation**: compute (dedicated ADC), DB (per-customer Cosmos DB), Blob (per-agent storage account), network (per-agent proxy), credentials (per-agent managed identity).
- **Code execution environment**: 700+ Python packages pre-installed, **arbitrary package installation is not allowed**, and the egress proxy allows access only to known domains.
- **No automatic execution — human approval required**: Review mode requires approval for write operations; Autonomous mode executes directly via the managed identity.

### 2.6 Creation Requirements / Constraints

| Item | Details |
|---|---|
| **Subscription** | `Microsoft.App` resource provider registration required |
| **Permissions** | **Owner** or **User Access Administrator** role on the subscription required (to assign RBAC to the managed identity) |
| **Regions** | Must be able to create resources in Sweden Central, East US 2, or Australia East |
| **Network** | Firewall must allow `*.azuresre.ai` |
| **Auto-created resources** | Application Insights, a Log Analytics workspace, and a Managed Identity are created together when the agent is created |

> **Note on language support**
> The official documentation (overview) states that the "chat interface supports English only,"
> but **in practice, Korean input/response has been observed to work.** However, since it is not an officially supported language, quality/consistency may not be guaranteed.

---

## 3. Cost Structure

Billing unit: **AAU (Azure Agent Unit)** — the standard processing measurement unit common to all prebuilt Azure agents. Monthly billing consists of the sum of the two flows below.

### 3.1 Always-on flow (fixed cost)

| Item | Rate |
|---|---|
| Always-on flow | **4 AAU per agent / hour** |

- Charged as long as the agent exists, regardless of whether work is actually performed (from creation → **deletion**).
- **Always-on cost continues even after Stop** → a **Delete** is required to fully stop it.

### 3.2 Active flow (variable cost) — based on token consumption during processing

Four token types are each metered and then summed: **Input / Output / Cache read / Cache write**

**Per-model AAU rate (per 1M tokens)**

| Model | Input | Output | Cache read | Cache write |
|---|---|---|---|---|
| Claude Opus 4.6 | 100 | 500 | 10 | 125 |
| GPT 5.3 Codex | 35 | 280 | 3.5 | 0 |
| GPT 5.2 | 35 | 280 | 3.5 | 0 |

- **Only processing time is billed** — time spent waiting for a user response is not billed.
- **The consumption counter resets at the start of each month.**
- The model provider is specified in the agent settings (the selected model determines the rate).

### 3.3 Consumption Examples by Task Type

| Scenario | Input | Output | Claude Opus 4.6 | GPT 5.3 Codex | Example |
|---|---|---|---|---|---|
| Quick question | ~20K | ~2K | ~3.8 AAU | ~1.3 AAU | "Show me recent alerts" |
| Incident investigation | ~200K | ~15K | ~35.3 AAU | ~11.7 AAU | Azure Monitor automatic incident |
| Full remediation | ~500K | ~40K | ~86.5 AAU | ~30.1 AAU | "Diagnose and fix a failed deployment" |

**Calculation example (Claude Opus 4.6, Quick question)**

| Token type | Tokens | Rate per 1M | AAU |
|---|---|---|---|
| Input | 20K | 100 | 2.0 |
| Output | 2K | 500 | 1.0 |
| Cache read | 15K | 10 | 0.15 |
| Cache write | 5K | 125 | 0.625 |
| **Total** | | | **3.775 AAU** |

### 3.4 Cost Management / Monitoring

- **Active flow spending limit**: Configurable in the range of **500 ~ 1,000,000 AAU** per month. When the limit is reached, chat/actions for that month stop (always-on continues). Increases apply immediately; decreases apply the following month.
- **Monitoring**: Settings > Agent consumption (per-thread/per-day consumption), Azure Cost Management.

**Billing impact by action**

| Action | Active flow | Always-on | How to resume next month |
|---|---|---|---|
| Budget limit reached | Stopped | Continues to be billed | Auto-resets at the start of the month |
| Stop agent | Stopped | Continues to be billed | Start from Settings > Basics |
| Delete agent | Stopped | Stopped | Create a new agent |

### 3.5 Cost Optimization Tips

- Reduce token waste by adding context (skills/knowledge/documents).
- Filter incidents by severity/service/keyword using a Response plan.
- Batch work with scheduled tasks (daily/weekly instead of constant polling).
- Validate prompts in chat/Playground before automating.
- Stop idle agents; Delete unused agents.

### 3.6 Free Tier

- **No free tier.** Billing starts **from the moment the agent is created.**
- Prices may vary by region.

---

## 3-A. Estimated Monthly Cost Simulation (Based on Regular Diagnostics + Report Delivery)

> **Base rate (official pricing page): 1 AAU = $0.10 (USD)**
> - Always-on (fixed): 4 AAU/hour × $0.10 = **$0.40/hour/agent**
> - Active flow (variable): consumed AAU × $0.10
> - The amounts below are per **single agent**, in USD. They may vary by exchange rate/region/discount agreement.

### (1) Fixed cost (Always-on) — incurred every month regardless of usage

| Period | Calculation | Amount |
|---|---|---|
| Per hour | 4 AAU × $0.10 | $0.40 |
| Per day (24h) | $0.40 × 24 | $9.60 |
| **Per month (~730 hours)** | $0.40 × 730 | **~$292** |

> Just keeping the agent on (even without running diagnostics) incurs a base of **~$290+ per month**. This cost continues even after Stop and **only stops after Delete**.

### (2) Unit price per task (Active flow)

| Task type | Claude Opus 4.6 | GPT 5.3 Codex |
|---|---|---|
| Quick question (~3.8 / ~1.3 AAU) | ~$0.38 | ~$0.13 |
| **One diagnostic report (Incident-investigation-level, ~35.3 / ~11.7 AAU)** | **~$3.53** | **~$1.17** |
| Diagnose + fix (Full remediation, ~86.5 / ~30.1 AAU) | ~$8.65 | ~$3.01 |

> One "diagnose then receive report" is roughly assumed to be at the **Incident investigation level** (varies by complexity).

### (3) Estimated monthly total by usage intensity (fixed $292 + variable)

**Scenario A — Light (weekly regular report)**
- 4 regular reports + 20 ad-hoc queries + 2 incident investigations / month

| Model | Variable (Active) | Fixed (Always-on) | **Monthly total** |
|---|---|---|---|
| Claude Opus 4.6 | ~$28.8 | $292 | **~$321** |
| GPT 5.3 Codex | ~$9.6 | $292 | **~$302** |

**Scenario B — Medium (daily regular report)**
- 30 regular reports + 100 ad-hoc queries + 10 incident investigations / month

| Model | Variable (Active) | Fixed (Always-on) | **Monthly total** |
|---|---|---|---|
| Claude Opus 4.6 | ~$179 | $292 | **~$471** |
| GPT 5.3 Codex | ~$60 | $292 | **~$352** |

**Scenario C — Heavy (aggressive automation + remediation)**
- 60 reports + 300 ad-hoc queries + 30 incident investigations + 10 diagnose+fix / month

| Model | Variable (Active) | Fixed (Always-on) | **Monthly total** |
|---|---|---|---|
| Claude Opus 4.6 | ~$518 | $292 | **~$810** |
| GPT 5.3 Codex | ~$174 | $292 | **~$466** |

### (4) Interpretation & Caveats

- **In light usage, the fixed cost ($292) is the majority of the total** — since the base fee is large even without frequent diagnostics, consolidating multiple workloads into **a single agent** is advantageous (number of agents = multiplier of the fixed cost).
- **As usage grows, the Active flow (variable cost) becomes dominant** — here the model choice has a large impact (GPT is about 3× cheaper than Opus).
- The AAU figures above are **average example values** from the official documentation; actual token consumption can vary greatly depending on resource size/log volume/investigation depth.
- The **monthly spending cap (500~1,000,000 AAU)** can be set to prevent variable-cost runaway (the fixed cost continues separately).
- Ingestion costs for the auto-created Application Insights / Log Analytics may be incurred **separately**.

> **Quick summary**: Keeping one agent always on costs **at least ~$290/month (fixed)**; on top of that, depending on usage,
> it is estimated at **~$300s** for light, **~$350~470** for medium, and **~$470~810** for heavy usage.

---

## 4. Conclusion — The Answer to "How Many Maximum?"

- Per Microsoft's official documentation, **there is no defined per-resource-type (AKS, PostgreSQL, etc.) count cap.**
- A single agent can **manage many resources simultaneously by scoping it to an entire subscription or multiple resource groups.**
- Practical limits manifest through the following factors:
  1. **Cost (AAU)** — the more diagnostic targets/frequency, the higher the active flow AAU.
  2. **Context window** — a limit on the amount of data that can be handled in a single investigation.
  3. **Region/scope configuration** — an agent is created in a specific region and its scope is limited by RBAC.
- Therefore, in large environments, the key is to design **how to split/consolidate agents based on cost, performance, and management boundaries** rather than "how many are supported."

---

## References (Microsoft Learn)

- Overview: https://learn.microsoft.com/en-us/azure/sre-agent/overview
- Pricing and billing: https://learn.microsoft.com/en-us/azure/sre-agent/pricing-billing
- Security overview: https://learn.microsoft.com/en-us/azure/sre-agent/security-overview
- Create agent: https://learn.microsoft.com/en-us/azure/sre-agent/create-agent
- Pricing calculator: https://azure.microsoft.com/pricing/details/sre-agent/
