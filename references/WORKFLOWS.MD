# Workflows (Kibana Automation)

Workflows is a Kibana feature (Preview since Stack 9.3, Preview on Serverless) for building automated sequences that turn data insights into actions. Workflows are defined in YAML and can be authored in Kibana's built-in editor or managed programmatically via the REST API.

**Location:** Kibana > Analytics > Workflows

## Prerequisites

- Kibana privilege: `All` for Analytics > Workflows
- Feature flag enabled: `workflows:ui:enabled`
- For AI steps: an LLM connector configured in Kibana (OpenAI, Gemini, etc.)
- For external steps (Slack, Jira): the relevant Kibana connector configured

## REST API

While Workflows is in **Technical Preview**, all API requests require an additional header:

```
x-elastic-internal-origin: kibana
```

Without this header, every endpoint returns `400 "exists but is not available with the current configuration"`.

### Required Headers (All Requests)

```bash
curl -s "${KIBANA_URL%/}/api/workflows/<endpoint>" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  -H "x-elastic-internal-origin: kibana"
```

### Workflow CRUD

| Method | Endpoint | Body | Purpose |
| --- | --- | --- | --- |
| `POST` | `/api/workflows` | `{yaml: "..."}` | Create workflow |
| `PUT` | `/api/workflows/{id}` | `{yaml: "..."}` or `{name: "..."}` | Update workflow |
| `GET` | `/api/workflows/{id}` | — | Get workflow by ID |
| `DELETE` | `/api/workflows/{id}` | — | Delete workflow |
| `DELETE` | `/api/workflows` | `{ids: [...]}` | Bulk delete |
| `POST` | `/api/workflows/search` | `{limit, page, query, enabled}` | Search/list workflows |
| `POST` | `/api/workflows/{id}/clone` | — | Clone workflow |

### Execution

| Method | Endpoint | Body | Purpose |
| --- | --- | --- | --- |
| `POST` | `/api/workflows/{id}/run` | `{inputs: {...}}` | Run workflow (returns `{workflowExecutionId}`) |
| `GET` | `/api/workflowExecutions/{execId}` | — | Execution details (status, steps, duration) |
| `GET` | `/api/workflowExecutions/{execId}/logs` | — | Execution logs |
| `POST` | `/api/workflowExecutions/{execId}/cancel` | — | Cancel execution |

### Statistics & Validation

| Method | Endpoint | Purpose |
| --- | --- | --- |
| `GET` | `/api/workflows/stats` | Workflow counts + execution history |
| `GET` | `/api/workflows/workflow-json-schema?loose=false` | YAML validation schema |
| `GET` | `/api/workflows/connectors` | Available connector types |

### Create Example

```bash
curl -s -X POST "${KIBANA_URL%/}/api/workflows" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  -H "x-elastic-internal-origin: kibana" \
  -d '{"yaml": "name: My Workflow\nenabled: true\ntriggers:\n  - type: manual\nsteps:\n  - name: hello\n    type: console\n    with:\n      message: hello world\n"}'
```

Response includes `id`, `name`, `enabled`, `valid`, and `definition`.

### Run Example

```bash
curl -s -X POST "${KIBANA_URL%/}/api/workflows/${WF_ID}/run" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -H "kbn-xsrf: true" \
  -H "x-elastic-internal-origin: kibana" \
  -d '{"inputs": {"query": "error"}}'
```

### API Naming Gotcha

The `name` field in the workflow's top-level API representation comes from parsing the YAML. If the YAML fails validation (`valid: false`), the name cannot be extracted and the workflow appears as "Untitled workflow". To set the name explicitly, send a follow-up `PUT` with only `{"name": "My Workflow"}` (no `yaml` key) — the API ignores `name` when `yaml` is present in the same request body.

## YAML Structure

Every workflow has five top-level sections:

```yaml
# Metadata
name: My Workflow
description: What this workflow does
enabled: true
tags: ["ops", "production"]

# Constants — fixed values reused throughout
consts:
  indexName: "logs-app"
  threshold: 100
  endpoints:
    primary: "https://api.example.com"
    fallback: "https://backup.example.com"

# Inputs — parameters passed at trigger time
inputs:
  - name: environment
    type: string
    required: true
    default: "staging"
    description: "Target environment"
  - name: dryRun
    type: boolean
    default: true

# Triggers — how/when the workflow starts
triggers:
  - type: manual

# Steps — the workflow logic, executed in order
steps:
  - name: search_errors
    type: elasticsearch.search
    with:
      index: "{{ consts.indexName }}"
      query:
        bool:
          filter:
            - range:
                "@timestamp":
                  gte: "now-1h"
```

All sections except `name` and `steps` are optional, though a workflow without triggers can only be run manually from the editor.

## Triggers

Every workflow needs at least one trigger. Multiple triggers of different types are allowed.

### Manual

Run on demand from the Kibana UI or via the REST API (`POST /api/workflows/{id}/run`).

```yaml
triggers:
  - type: manual
```

### Scheduled

Run on a recurring interval or cron expression.

```yaml
# Interval — every 5 minutes
triggers:
  - type: scheduled
    with:
      every: 5m

# Other interval examples: 30s, 1h, 1d
```

**rrule** — for cron-like schedules with timezone support:

```yaml
# Every weekday at 09:00 London time
triggers:
  - type: scheduled
    with:
      rrule:
        freq: DAILY
        interval: 1
        tzid: Europe/London
        byhour: [9]
        byminute: [0]
        byday: [MO, TU, WE, TH, FR]
```

### Alert

Run when a Kibana alerting or detection rule fires. The full alert payload is available via `{{ event }}`.

```yaml
triggers:
  - type: alert
```

### Trigger Context

All triggers provide execution context accessible in steps:

| Variable | Description |
| --- | --- |
| `execution.id` | Unique ID for this run |
| `execution.startedAt` | ISO timestamp when the run began |
| `event` | Full trigger payload (alert body, schedule info, or manual params) |

## Step Types

Steps are the individual units of logic. Each step has `name`, `type`, and usually `with` (parameters).

### Elasticsearch

**`elasticsearch.search`** — run a search query:

```yaml
- name: find_errors
  type: elasticsearch.search
  with:
    index: "logs-*"
    query:
      bool:
        must:
          - match:
              level: "error"
        filter:
          - range:
              "@timestamp":
                gte: "now-24h"
    size: 50
    sort:
      - "@timestamp"
```

> **Sort limitation:** `elasticsearch.search` only accepts simple string sort values (e.g. `- "@timestamp"`). The object form (`- "@timestamp": {order: desc}`) silently causes `valid: false`. Ascending is the default; for descending or complex sort, use `elasticsearch.request` instead:

```yaml
- name: find_latest_errors
  type: elasticsearch.request
  with:
    method: POST
    path: "/logs-*/_search"
    body:
      query:
        bool:
          must:
            - match:
                level: "error"
          filter:
            - range:
                "@timestamp":
                  gte: "now-24h"
      size: 50
      sort:
        - "@timestamp":
            order: desc
```

**`elasticsearch.update`** — update a document by ID:

```yaml
- name: mark_processed
  type: elasticsearch.update
  with:
    index: "cases"
    id: "{{ steps.find_case.output.hits.hits.0._id }}"
    doc:
      status: "processed"
      processed_at: "{{ execution.startedAt }}"
```

**`elasticsearch.indices.create`** — create an index with mappings:

```yaml
- name: create_index
  type: elasticsearch.indices.create
  with:
    index: "reports-{{ inputs.environment }}"
    body:
      settings:
        number_of_shards: 1
      mappings:
        properties:
          title:
            type: text
          created_at:
            type: date
```

**`elasticsearch.request`** — arbitrary Elasticsearch API call (for anything not covered by the typed steps):

```yaml
- name: bulk_index
  type: elasticsearch.request
  with:
    method: POST
    path: "/my-index/_bulk?refresh=wait_for"
    headers:
      Content-Type: application/x-ndjson
    body: |
      {"index":{}}
      {"name":"Doc 1","status":"active"}
      {"index":{}}
      {"name":"Doc 2","status":"active"}

- name: delete_old_index
  type: elasticsearch.request
  with:
    method: DELETE
    path: "/logs-2025.01"
```

### Kibana

**`kibana.createCase`** — create a Kibana case:

```yaml
- name: open_case
  type: kibana.createCase
  with:
    owner: securitySolution
    connector:
      id: "none"
      name: "none"
      type: ".none"
      fields: null
    settings:
      syncAlerts: true
    title: "Alert: {{ event.rule.name }}"
    description: "Auto-created from workflow. Severity: {{ event.severity }}"
    tags: ["auto-created", "{{ event.rule.category }}"]
```

Required fields: `owner` (`"cases"` | `"observability"` | `"securitySolution"`), `connector`, `settings`, `title`, `description`, `tags`. Use `connector.id: "none"` to create a case without an external connector.

**`kibana.updateCase`** — update an existing case (status, tags, assignees, etc.). Takes the same fields as the [Update case API](https://www.elastic.co/docs/api/doc/kibana/v9/operation/operation-updatecase).

**`kibana.request`** — arbitrary Kibana API call (`kbn-xsrf` and `Content-Type` added automatically):

```yaml
- name: get_dashboard
  type: kibana.request
  with:
    method: GET
    path: "/api/dashboards/dashboard/my-dashboard-id"
```

### External Systems

**`http`** — direct HTTP request to any URL:

```yaml
- name: call_webhook
  type: http
  with:
    url: "https://hooks.example.com/incident"
    method: POST
    headers:
      Authorization: "Bearer {{ consts.webhookToken }}"
    body:
      alert_id: "{{ event.id }}"
      severity: "{{ event.severity }}"
```

**`slack`** — send a Slack message via a configured Kibana connector:

```yaml
- name: notify_team
  type: slack
  connector-id: "slack-ops-channel"
  with:
    message: |
      *Alert:* {{ event.rule.name }}
      *Severity:* {{ event.severity }}
      *Hits:* {{ steps.search_errors.output.hits.total.value }}
```

**`jira`** — create or update a Jira issue via connector:

```yaml
- name: create_ticket
  type: jira
  connector-id: "jira-ops"
  with:
    subAction: pushToService
    subActionParams:
      incident:
        summary: "Alert: {{ event.rule.name }}"
        description: "Auto-created by workflow"
        labels: ["auto-created"]
```

`connector-id` accepts either the connector's UUID or its display name.

### Flow Control

**`if`** — conditional branching:

```yaml
- name: check_severity
  type: if
  condition: "{{ event.severity == 'critical' }}"
  steps:
    - name: page_oncall
      type: slack
      connector-id: "slack-urgent"
      with:
        message: "CRITICAL: {{ event.rule.name }}"
  else:
    - name: log_info
      type: console
      with:
        message: "Non-critical alert: {{ event.rule.name }}"
```

**`foreach`** — loop over an array. The current item is `{{ foreach.item }}`:

```yaml
- name: process_hits
  type: foreach
  foreach: "${{ steps.find_errors.output.hits.hits }}"
  steps:
    - name: log_hit
      type: console
      with:
        message: "Processing: {{ foreach.item._source.message }}"
    - name: update_status
      type: elasticsearch.update
      with:
        index: "logs-app"
        id: "{{ foreach.item._id }}"
        doc:
          reviewed: true
```

**`wait`** — pause execution:

```yaml
- name: pause
  type: wait
  with:
    duration: "30s"
```

**`console`** — log a message (useful for debugging and audit trails):

```yaml
- name: log_result
  type: console
  with:
    message: "Found {{ steps.search_errors.output.hits.total.value }} errors"
```

### AI

**`ai.prompt`** — send a prompt to an LLM via a configured AI connector:

```yaml
- name: classify_alert
  type: ai.prompt
  with:
    prompt: |
      Classify the following alert and suggest a remediation action.
      Alert name: {{ event.rule.name }}
      Alert details: {{ event | json:2 }}
    connectorId: "openai-gpt4"
    temperature: 0.3
    outputSchema:
      type: object
      properties:
        category:
          type: string
          enum: ["infrastructure", "security", "application", "data"]
        severity_assessment:
          type: string
        recommended_action:
          type: string
```

`connectorId` is optional — omit it to use the default AI connector. `outputSchema` enforces structured JSON output.

**`ai.agent`** — invoke an Agent Builder agent:

```yaml
- name: ask_agent
  type: ai.agent
  with:
    agent_id: "data-analyst-agent"
    message: "Summarise the top error patterns in the last hour from logs-*"
```

The agent's response is available as **free text** at `steps.NAME.output.message`. There is no `outputSchema` support for `ai.agent` (unlike `ai.prompt`). For downstream routing based on agent output, instruct the agent to include a parseable marker in its response and use the `contains` operator:

```yaml
- name: route_verdict
  type: if
  condition: "{{ steps.ask_agent.output.message contains '[CRITICAL]' }}"
  steps:
    - name: escalate
      type: slack
      connector-id: "slack-urgent"
      with:
        message: "{{ steps.ask_agent.output.message }}"
```

## Step Output Paths

Different step types expose their output at different paths. Getting this wrong is the most common cause of "workflow succeeded but returned null".

| Step Type | Output Root | Common Paths |
| --- | --- | --- |
| `elasticsearch.search` | `steps.NAME.output` | `.output.hits.hits`, `.output.hits.total.value` |
| `elasticsearch.request` | `steps.NAME.output` | `.output.hits.hits`, `.output.aggregations` |
| `http` | `steps.NAME.output.data` | `.output.data.results`, `.output.data.field` |
| `kibana.request` | `steps.NAME.output` | `.output.cases`, `.output.total` |
| `kibana.createCase` | `steps.NAME.output` | `.output.id`, `.output.version` |
| `ai.prompt` + `outputSchema` | `steps.NAME.output` | `.output.<field>` (structured per schema) |
| `ai.agent` | `steps.NAME.output.message` | Free text only (string) |

> Both `elasticsearch.search` and `elasticsearch.request` expose the ES response body directly at `.output`. The output structure is identical — the difference is that `elasticsearch.search` has stricter parameter validation (e.g. sort syntax) while `elasticsearch.request` passes the body through verbatim.

## Data and Templating

### Accessing Data

| Syntax | Source | Example |
| --- | --- | --- |
| `steps.<name>.output` | Output of a previous step | `{{ steps.find_errors.output.hits.total.value }}` |
| `steps.<name>.error` | Error from a failed step | `{{ steps.api_call.error.message }}` |
| `consts.<name>` | Workflow constants | `{{ consts.indexName }}` |
| `inputs.<name>` | Trigger-time parameters | `{{ inputs.environment }}` |
| `event` | Trigger payload | `{{ event.severity }}` |
| `execution.id` | Current run ID | `{{ execution.id }}` |
| `execution.startedAt` | Run start time | `{{ execution.startedAt }}` |
| `foreach.item` | Current item in a loop | `{{ foreach.item._source.host }}` |
| `workflow.name` | Workflow name | `{{ workflow.name }}` |

### Template Expressions

- **`{{ }}`** — renders the value as a **string**. Use for messages, index names, IDs.
- **`${{ }}`** — preserves the original type (**arrays, objects, numbers**). Use when passing structured data between steps.

```yaml
# String interpolation
message: "Found {{ steps.search.output.hits.total.value }} results"

# Preserving an array for foreach
foreach: "${{ steps.search.output.hits.hits }}"

# Preserving an object as a query body
query: "${{ consts.savedQuery }}"
```

### Liquid Filters

Templates support [Liquid](https://shopify.github.io/liquid/) filters:

```yaml
message: "{{ event.rule.name | upcase }}"
details: "{{ event | json:2 }}"
```

### Workflow Output

Workflows can define a top-level `output` section to control what data they return. This is especially important for **workflow-as-tool** patterns where an Agent Builder agent invokes the workflow and receives the output:

```yaml
name: Enrich Data
enabled: true
triggers:
  - type: manual
inputs:
  - name: query
    type: string
    required: true
steps:
  - name: search
    type: elasticsearch.search
    with:
      index: "my-index"
      query:
        match:
          content: "{{ inputs.query }}"
      size: 5

output:
  total: "{{ steps.search.output.hits.total.value }}"
  results: "${{ steps.search.output.hits.hits }}"
```

Without an `output` section, the agent receives all step outputs. With one, you control exactly what the agent sees.

## Error Handling

Steps can define `on-failure` behaviour. Processing order: retry, then fallback, then continue.

### Retry

```yaml
- name: call_api
  type: http
  with:
    url: "https://api.example.com/action"
    method: POST
  on-failure:
    retry:
      max-attempts: 3
      delay: "5s"
```

### Fallback

Run alternative steps when retries are exhausted:

```yaml
- name: primary_notify
  type: slack
  connector-id: "slack-primary"
  with:
    message: "Alert: {{ event.rule.name }}"
  on-failure:
    retry:
      max-attempts: 2
    fallback:
      - name: backup_notify
        type: http
        with:
          url: "{{ consts.endpoints.fallback }}/notify"
          method: POST
          body:
            message: "Slack failed. Alert: {{ event.rule.name }}"
```

### Continue

Allow the workflow to proceed even if the step fails:

```yaml
- name: optional_enrichment
  type: http
  with:
    url: "https://enrichment.example.com/lookup"
    method: GET
  on-failure:
    continue: true
```

### Workflow-Level Error Handling

Set defaults for all steps under `settings`. Individual step `on-failure` overrides:

```yaml
settings:
  on-failure:
    retry:
      max-attempts: 2
    continue: true

steps:
  - name: step_one
    type: elasticsearch.search
    with:
      index: "logs-*"
      query: { match_all: {} }
  - name: critical_step
    type: http
    with:
      url: "https://api.example.com"
      method: POST
    on-failure:
      retry:
        max-attempts: 5
        delay: "10s"
```

Flow-control steps (`if`, `foreach`) cannot use workflow-level `on-failure`; define it on the inner steps instead.

## Common Patterns

### Alert Enrichment and Slack Notification

Triggered by a Kibana alert rule. Searches for related errors, then posts a summary to Slack.

```yaml
name: Alert Enrichment and Notify
description: Enrich alerts with recent error context and notify the ops channel
enabled: true
tags: ["alerting", "ops"]

consts:
  lookbackWindow: "now-15m"

triggers:
  - type: alert

steps:
  - name: search_context
    type: elasticsearch.request
    with:
      method: POST
      path: "/logs-*/_search"
      body:
        query:
          bool:
            must:
              - match:
                  service.name: "{{ event.service.name }}"
            filter:
              - range:
                  "@timestamp":
                    gte: "{{ consts.lookbackWindow }}"
              - term:
                  level: "error"
        size: 5
        sort:
          - "@timestamp":
              order: desc

  - name: notify_ops
    type: slack
    connector-id: "slack-ops"
    with:
      message: |
        *Alert:* {{ event.rule.name }}
        *Service:* {{ event.service.name }}
        *Severity:* {{ event.severity }}
        *Recent errors (last 15m):* {{ steps.search_context.output.hits.total.value }}
        *Latest:* {{ steps.search_context.output.hits.hits.0._source.message }}
```

### Scheduled Index Cleanup

Runs daily to delete indices older than a retention window.

```yaml
name: Nightly Index Cleanup
description: Delete staging indices older than 30 days
enabled: true
tags: ["maintenance", "scheduled"]

consts:
  retentionDays: 30

triggers:
  - type: scheduled
    with:
      every: 1d

steps:
  - name: find_old_indices
    type: elasticsearch.request
    with:
      method: GET
      path: "/_cat/indices/staging-*?format=json&h=index,creation.date.string"

  - name: log_candidates
    type: console
    with:
      message: "Found {{ steps.find_old_indices.output | size }} staging indices to evaluate"

  - name: cleanup_old
    type: foreach
    foreach: "${{ steps.find_old_indices.output }}"
    steps:
      - name: delete_index
        type: elasticsearch.request
        with:
          method: DELETE
          path: "/{{ foreach.item.index }}"
        on-failure:
          continue: true
```

### AI-Powered Alert Triage

Uses an LLM to classify alerts and route them based on the classification.

```yaml
name: AI Alert Triage
description: Classify alerts with AI and route to the appropriate team
enabled: true
tags: ["ai", "triage", "alerting"]

triggers:
  - type: alert

steps:
  - name: classify
    type: ai.prompt
    with:
      prompt: |
        You are a site-reliability classification engine.
        Given the following alert, return a JSON classification.

        Rule: {{ event.rule.name }}
        Message: {{ event.message }}
        Tags: {{ event.tags | join:", " }}
      outputSchema:
        type: object
        properties:
          team:
            type: string
            enum: ["platform", "security", "app-dev", "data-eng"]
          priority:
            type: string
            enum: ["p1", "p2", "p3"]
          summary:
            type: string

  - name: route_alert
    type: if
    condition: "{{ steps.classify.output.priority == 'p1' }}"
    steps:
      - name: page_oncall
        type: slack
        connector-id: "slack-oncall"
        with:
          message: |
            *P1 — {{ steps.classify.output.team }}*
            {{ steps.classify.output.summary }}
            Rule: {{ event.rule.name }}
    else:
      - name: log_ticket
        type: jira
        connector-id: "jira-ops"
        with:
          subAction: pushToService
          subActionParams:
            incident:
              summary: "[{{ steps.classify.output.priority | upcase }}] {{ event.rule.name }}"
              description: "{{ steps.classify.output.summary }}"
              labels: ["{{ steps.classify.output.team }}", "auto-triage"]
```

### Foreach with Conditional Processing

Loops over search hits and applies different actions based on each document's content.

```yaml
name: Process and Route Logs
description: Examine recent critical logs and take per-document actions
enabled: true
tags: ["ops", "processing"]

triggers:
  - type: manual

steps:
  - name: find_critical
    type: elasticsearch.search
    with:
      index: "logs-*"
      query:
        bool:
          filter:
            - term:
                level: "critical"
            - range:
                "@timestamp":
                  gte: "now-1h"
      size: 100

  - name: process_each
    type: foreach
    foreach: "${{ steps.find_critical.output.hits.hits }}"
    steps:
      - name: check_service
        type: if
        condition: "{{ foreach.item._source.service.name == 'payments' }}"
        steps:
          - name: urgent_notify
            type: slack
            connector-id: "slack-payments"
            with:
              message: "CRITICAL in payments: {{ foreach.item._source.message }}"
        else:
          - name: standard_log
            type: console
            with:
              message: "Critical in {{ foreach.item._source.service.name }}: {{ foreach.item._source.message }}"
```

## Tips

- **Start with `manual` trigger** while developing, then switch to `scheduled` or `alert` once the workflow is tested.
- **Use `console` steps** liberally during development — they appear in the Executions log and help trace data flow.
- **Use `${{ }}` (dollar-brace)** when passing arrays or objects between steps. Plain `{{ }}` stringifies the value, which breaks `foreach` and structured bodies.
- **Keep step names unique and descriptive** — they serve as the keys in `steps.<name>.output`.
- **Constants vs inputs:** put values that never change per run in `consts` (index names, thresholds, endpoints). Put values that vary per invocation in `inputs` (environment, dry-run flag).
- **Test individual steps** using the "Run step" button in the Kibana editor before running the full workflow.
- **Error handling defaults:** set `settings.on-failure` at the workflow level for sensible defaults, then override on critical steps that need stricter retry policies.
- **Connector names or IDs:** `connector-id` accepts either the connector UUID or its display name from Kibana's Connectors page.
- **Silent validation failures:** workflows with invalid YAML deploy as `valid: false`, `enabled: false`, `name: "Untitled workflow"` with no error message. Always verify `valid: true` via `GET /api/workflows/{id}` after creation. Common causes: object-form sort in `elasticsearch.search`, use of `kibana.createCaseDefaultSpace` (use `kibana.createCase` instead).
- **`elasticsearch.search` vs `elasticsearch.request`:** prefer `elasticsearch.search` for simple queries (cleaner syntax, less nesting). Switch to `elasticsearch.request` when you need descending sort, complex sort expressions, or any ES API parameter that `elasticsearch.search` doesn't expose.

## Reference

- [Workflows overview](https://www.elastic.co/docs/explore-analyze/workflows)
- [Get started with workflows](https://www.elastic.co/docs/explore-analyze/workflows/get-started)
- [Triggers](https://www.elastic.co/docs/explore-analyze/workflows/triggers)
- [Steps](https://www.elastic.co/docs/explore-analyze/workflows/steps)
- [Data and error handling](https://www.elastic.co/docs/explore-analyze/workflows/data)
- [Author workflows](https://www.elastic.co/docs/explore-analyze/workflows/author-workflows)
