# Serverless Expense Approval — Azure Durable Functions vs Logic Apps

A dual implementation of the same expense approval workflow built on Azure, comparing two fundamentally different approaches to serverless orchestration. Both versions handle the same business logic: expense validation, manager approval with timeout escalation, and employee notification.

![Azure](https://img.shields.io/badge/Azure-Durable_Functions-0078D4?logo=microsoftazure&logoColor=white)
![Azure](https://img.shields.io/badge/Azure-Logic_Apps-0078D4?logo=microsoftazure&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)
![Azure Service Bus](https://img.shields.io/badge/Azure-Service_Bus-0078D4?logo=microsoftazure&logoColor=white)

---

## What this project is about

Serverless orchestration sounds like a solved problem — until you need human-in-the-loop approval, timeouts, and multi-step branching. This project builds the same workflow twice to expose where each tool is the right fit and where it breaks down.

**The workflow:**
1. An expense request is submitted
2. It is validated and routed for manager approval
3. If the manager does not respond within the timeout, the request is automatically escalated
4. The employee is notified of the outcome by email

---

## Two Implementations

### Version A — Azure Durable Functions (`/version-a-durable-functions`)

Implemented in Python using the v2 programming model. Six functions: an HTTP trigger, an orchestrator, three activity functions, and a manager response endpoint.

The core design is the **Human Interaction pattern**: the orchestrator races a durable timer against an external event using `context.task_any([approval_task, timer_task])`. If the manager responds in time, the timer is cancelled and the decision is applied. If the timer fires first, the status is set to `escalated`. Employee notifications are delivered via Azure Communication Services Email.

**Key technical challenges resolved:**
- The correct PyPI package is `azure-functions-durable`, not `azure-durable-functions` — the wrong name causes a silent import failure
- The orchestrator has strict replay rules: no `datetime.now()`, no logging, no direct I/O inside the orchestrator body
- `approval_task.result` returns a JSON string, not a parsed dict — requires `json.loads()` before reading fields
- The `extensionBundle` block in `host.json` is required for durable bindings to register

### Version B — Azure Logic Apps + Service Bus (`/version-b-logic-apps`)

Implemented visually using a Logic App (Consumption tier) triggered by an Azure Service Bus queue. Validation is delegated to a lightweight Azure Function. Outcomes are published to a Service Bus topic with `approved`, `rejected`, and `escalated` subscriptions. Manager approval uses the built-in **Send Approval Email (V2)** connector.

The escalation branch is wired through "configure run after" — if the approval Scope is skipped (timeout), a downstream action publishes the escalated outcome. This differs from Durable Functions, where the timer is a first-class entity the orchestrator explicitly races against.

**Key technical challenges resolved:**
- Logic Apps rejects Azure Functions with custom routes — the Python v2 model requires `route=""` (empty string), which is undocumented
- The Service Bus trigger delivers message content as base64 — a Compose decode step (`base64ToString(...)`) is required upstream of the validation function
- Logic Apps' visual Condition designer treats all values as strings: comparing a JSON boolean to `"True"` (capital T) caused `ActionFailed` — the fix is `@equals(body('Parse_JSON')?['valid'], true)` with a lowercase boolean
- The validation function must always return a `data` field regardless of outcome, otherwise all downstream expressions resolve to null

---

## Comparison

| Dimension | Durable Functions | Logic Apps |
|-----------|------------------|-----------|
| **Development** | Python, full IDE support, explicit code | Visual designer, faster initial setup |
| **Local testing** | Full offline testing with Azurite + `func start` | Not possible — requires deployed Azure resources |
| **Escalation logic** | Timer race in code (`task_any`) | "Configure run after: Scope is skipped" |
| **Error handling** | Standard Python `try/except`, full control | Per-action "run after" settings |
| **Observability** | Application Insights + orchestrator status API | Visual run history with inputs/outputs per action |
| **Cost at scale** | More cost-efficient at high volume (consumption model) | Per-action pricing becomes expensive above ~10k/day |

### When to use each

**Logic Apps** is the stronger choice when:
- Non-developers need to monitor and modify workflows
- The built-in connector ecosystem covers most integration needs (email, approval, queues)
- Speed of delivery matters more than deep testability

**Durable Functions** is the better choice when:
- The workflow is complex enough that a visual designer becomes a liability
- Local testability and CI/CD integration are requirements
- High volume makes per-action pricing prohibitive
- Custom approval flows (multi-step, delegation, custom UI) are needed

---

## Tech Stack

`Python` `Azure Durable Functions` `Azure Logic Apps` `Azure Service Bus` `Azure Communication Services` `Azurite` `Bicep`

---

## Infrastructure

`main.bicep` at the root provisions the full Azure environment for both versions: Function App, Logic App, Service Bus namespace, Communication Services, and supporting resources.

---

## Running Locally (Version A)

```bash
cd version-a-durable-functions
pip install -r requirements.txt
func start
```

Requires Azurite running locally for storage emulation. Use `test-durable.http` to trigger all scenarios including escalation (set `APPROVAL_TIMEOUT_SECONDS=30` in `local.settings.json`).

Version B cannot be tested locally — it requires a deployed Logic App and Service Bus instance.
