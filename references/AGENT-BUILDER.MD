# Agent Builder (Kibana AI)

Agent Builder is a Kibana feature (GA since 9.3) for creating custom AI agents that can query and interact with your Elasticsearch data. Agents can use tools (ES|QL queries, index searches, platform tools) to answer questions and perform tasks.

## Prerequisites

- LLM connector configured in Kibana (OpenAI, Azure OpenAI, Amazon Bedrock, etc.)
- Kibana API access with appropriate permissions
- API key with necessary privileges

## Authentication

Agent Builder uses Kibana APIs (not Elasticsearch APIs). Set up authentication:

```bash
# Set these for your session
KIBANA_URL="https://your-cluster.kb.cloud.elastic.co:443"
ES_API_KEY="your-base64-api-key"

# All requests to Kibana follow this pattern:
curl -s "${KIBANA_URL%/}/api/agent_builder/<endpoint>" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json"

# For mutations (POST, PUT, DELETE), add kbn-xsrf header:
curl -s -X POST "${KIBANA_URL%/}/api/agent_builder/<endpoint>" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{ ... }'
```

**Important:** Use `KIBANA_URL` (not `ES_URL`) and always include `kbn-xsrf: true` header for POST/PUT/DELETE operations.

## Spaces Support

Kibana Spaces allow you to organize dashboards, saved objects, and agents. To work with non-default spaces, add `/s/<space_name>` before `/api/agent_builder`:

```bash
# Default space (no prefix needed)
curl -s "${KIBANA_URL%/}/api/agent_builder/tools" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .

# Non-default space (e.g., "my-space")
curl -s "${KIBANA_URL%/}/s/my-space/api/agent_builder/tools" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .
```

## Tools API

Tools are reusable components that agents can use to query data or perform actions.

### List All Tools

```bash
curl -s "${KIBANA_URL%/}/api/agent_builder/tools" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .
```

### Create a Tool (ES|QL Type)

```bash
curl -s -X POST "${KIBANA_URL%/}/api/agent_builder/tools" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "analyze-financial-trades",
    "type": "esql",
    "description": "Analyze financial trades with time filtering and aggregation",
    "tags": ["analytics", "finance", "trading"],
    "configuration": {
      "query": "FROM financial_trades | WHERE execution_timestamp >= ?startTime | STATS trade_count=COUNT(*), avg_price=AVG(execution_price) BY symbol | SORT trade_count DESC | LIMIT ?limit",
      "params": {
        "startTime": {
          "type": "date",
          "description": "Start time for the analysis in ISO format (e.g., 2026-01-01T00:00:00Z)"
        },
        "limit": {
          "type": "integer",
          "description": "Maximum number of results to return"
        }
      }
    }
  }' | jq .
```

### Create a Tool (Index Search Type)

```bash
curl -s -X POST "${KIBANA_URL%/}/api/agent_builder/tools" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "search-logs-tool",
    "type": "index_search",
    "description": "Search application logs for errors and warnings",
    "tags": ["logs", "troubleshooting"],
    "configuration": {
      "indices": ["logs-*"],
      "description": "Searches application logs with full-text search capabilities"
    }
  }' | jq .
```

### Create a Tool (Workflow Type)

Register a Kibana Workflow as a tool so agents can trigger it during conversations:

```bash
curl -s -X POST "${KIBANA_URL%/}/api/agent_builder/tools" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "enrich-flight-data",
    "type": "workflow",
    "description": "Gather enrichment data for a flight investigation. Call this when the user asks about a specific aircraft by ICAO hex address.",
    "tags": ["enrichment", "flights"],
    "configuration": {
      "workflow_id": "workflow-abc123-your-workflow-id"
    }
  }' | jq .
```

The `description` tells the agent *when* to call the tool — be specific about trigger conditions. A vague description leads to the agent calling the tool at wrong times or not at all.

The workflow must exist and be `valid: true` before registering it as a tool. Workflow inputs become the tool's parameters automatically.

### Get Tool by ID

```bash
curl -s "${KIBANA_URL%/}/api/agent_builder/tools/analyze-financial-trades" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .
```

### Update a Tool

```bash
curl -s -X PUT "${KIBANA_URL%/}/api/agent_builder/tools/analyze-financial-trades" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "description": "Updated: Analyze financial trades with enhanced filtering",
    "tags": ["analytics", "finance", "trading", "updated"],
    "configuration": {
      "query": "FROM financial_trades | WHERE execution_timestamp >= ?startTime AND execution_timestamp <= ?endTime | STATS trade_count=COUNT(*), avg_price=AVG(execution_price), total_volume=SUM(volume) BY symbol | SORT trade_count DESC | LIMIT ?limit",
      "params": {
        "startTime": {
          "type": "date",
          "description": "Start time for the analysis in ISO format"
        },
        "endTime": {
          "type": "date",
          "description": "End time for the analysis in ISO format"
        },
        "limit": {
          "type": "integer",
          "description": "Maximum number of results to return"
        }
      }
    }
  }' | jq .
```

### Delete a Tool

```bash
curl -s -X DELETE "${KIBANA_URL%/}/api/agent_builder/tools/analyze-financial-trades" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" | jq .
```

### Execute a Tool

```bash
curl -s -X POST "${KIBANA_URL%/}/api/agent_builder/tools/_execute" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "tool_id": "platform.core.search",
    "tool_params": {
      "query": "find john doe email address from the employee index"
    }
  }' | jq .
```

## Agents API

Agents are AI assistants configured with specific instructions and tools.

### List All Agents

```bash
curl -s "${KIBANA_URL%/}/api/agent_builder/agents" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .
```

### Create an Agent

```bash
curl -s -X POST "${KIBANA_URL%/}/api/agent_builder/agents" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "data-analyst-agent",
    "name": "Data Analyst Helper",
    "description": "I can help you analyze data in your indices starting with \"analytics-\" prefix",
    "labels": ["analytics", "data-analysis"],
    "avatar_color": "#BFDBFF",
    "avatar_symbol": "DA",
    "configuration": {
      "instructions": "You are a data analyst agent that helps users query and analyze data from indices with the \"analytics-\" prefix. Use ES|QL queries for efficient analysis. Always explain your findings clearly.",
      "tools": [
        {
          "tool_ids": [
            "platform.core.search",
            "platform.core.list_indices",
            "platform.core.get_index_mapping",
            "platform.core.execute_esql"
          ]
        }
      ]
    }
  }' | jq .
```

### Get Agent by ID

```bash
curl -s "${KIBANA_URL%/}/api/agent_builder/agents/data-analyst-agent" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .
```

### Update an Agent

```bash
curl -s -X PUT "${KIBANA_URL%/}/api/agent_builder/agents/data-analyst-agent" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Advanced Data Analyst",
    "description": "Enhanced data analyst with additional capabilities",
    "labels": ["analytics", "data-analysis", "advanced"],
    "avatar_color": "#BFDBFF",
    "avatar_symbol": "AD",
    "configuration": {
      "instructions": "You are an advanced data analyst agent specializing in complex queries and analysis. Use ES|QL for efficient operations and provide detailed insights with visualizations when appropriate.",
      "tools": [
        {
          "tool_ids": [
            "platform.core.search",
            "platform.core.list_indices",
            "platform.core.get_index_mapping",
            "platform.core.execute_esql",
            "platform.core.generate_esql"
          ]
        }
      ]
    }
  }' | jq .
```

### Delete an Agent

```bash
curl -s -X DELETE "${KIBANA_URL%/}/api/agent_builder/agents/data-analyst-agent" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" | jq .
```

## Chat & Conversations API

Interact with agents through conversations.

### Send a Chat Message (Synchronous)

```bash
curl -s -X POST "${KIBANA_URL%/}/api/agent_builder/converse" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "input": "What is Elasticsearch and how does it work?",
    "agent_id": "elastic-ai-agent"
  }' | jq .
```

### Send a Chat Message with Conversation Context

```bash
curl -s -X POST "${KIBANA_URL%/}/api/agent_builder/converse" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "input": "Can you show me the top 10 errors from the last hour?",
    "agent_id": "data-analyst-agent",
    "conversation_id": "existing-conversation-id-123"
  }' | jq .
```

### Stream Chat Responses (Async)

For real-time streaming responses, use the async endpoint. This returns Server-Sent Events (SSE):

```bash
curl -s -X POST "${KIBANA_URL%/}/api/agent_builder/converse/async" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" \
  -H "Content-Type: application/json" \
  -d '{
    "input": "Analyze the error patterns in my logs",
    "agent_id": "data-analyst-agent",
    "conversation_id": "conversation-id-456"
  }'
```

### List Conversations

```bash
curl -s "${KIBANA_URL%/}/api/agent_builder/conversations" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .
```

### Get Conversation by ID

```bash
curl -s "${KIBANA_URL%/}/api/agent_builder/conversations/conversation-id-123" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .
```

### Delete Conversation by ID

```bash
curl -s -X DELETE "${KIBANA_URL%/}/api/agent_builder/conversations/conversation-id-123" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "kbn-xsrf: true" | jq .
```

## Built-in Platform Tools

Agent Builder provides several built-in tools that agents can use:

- `platform.core.search` - Natural language search across indices
- `platform.core.list_indices` - List available indices with patterns
- `platform.core.get_index_mapping` - Retrieve index field mappings
- `platform.core.get_document_by_id` - Fetch specific documents
- `platform.core.execute_esql` - Execute ES|QL queries
- `platform.core.generate_esql` - Generate ES|QL from natural language
- `platform.core.index_explorer` - Explore index patterns and metadata
- `platform.core.get_workflow_execution_status` - Poll status of an async workflow execution (use with workflow tools)

Reference these by `tool_ids` when creating agents.

## Tips & Best Practices

**Tool Design:**

- Keep ES|QL queries focused and parameterized for reusability
- Use descriptive tool IDs and clear parameter descriptions
- Tag tools appropriately for easier discovery
- Test tools with `_execute` endpoint before adding to agents

**Agent Configuration:**

- Write clear, specific instructions for agent behavior
- Include relevant context about data structure and naming conventions
- Assign appropriate tools based on the agent's purpose
- Use meaningful avatar colors and symbols for visual identification

**Workflow Tools:**

- When an `ai.agent` step invokes an agent from a workflow, the agent's full response is free text at `steps.NAME.output.message` — there is no structured JSON output. Use text markers and the `contains` operator in workflow `if` conditions for routing.
- Long-running workflows execute asynchronously. Include `platform.core.get_workflow_execution_status` in the agent's tools so it can poll for completion and report progress.
- Workflow inputs become the tool's parameters. The agent sees the workflow's `output` section (or all step outputs if no `output` is defined).

**Error Handling:**

- Check for `401 Unauthorized` - verify API key and permissions
- `400 Bad Request` - validate JSON structure and required fields
- `404 Not Found` - confirm tool/agent IDs and space names
- `403 Forbidden` - ensure LLM connector is configured and accessible

**Performance:**

- Use ES|QL tools for large-scale aggregations (faster than index_search)
- Limit result sets with LIMIT clauses in queries
- Cache frequently-used tool executions when possible

**Security:**

- Scope API keys to necessary indices and operations only
- Review agent instructions to prevent unintended data access
- Use spaces to isolate agents and tools by team/project
- Audit conversation logs for sensitive information

## Integration with ES|QL

ES|QL tools are particularly powerful for agents. Example patterns:

```bash
# Time-series analysis
"query": "FROM logs-* | WHERE @timestamp >= ?start AND @timestamp <= ?end | STATS count=COUNT(*) BY level | SORT count DESC"

# Aggregation with filtering
"query": "FROM metrics-* | WHERE host.name IN (?hosts) | STATS avg_cpu=AVG(system.cpu.percent) BY host.name"

# Pattern detection
"query": "FROM logs-* | WHERE message LIKE ?pattern | STATS occurrences=COUNT(*) BY service.name | SORT occurrences DESC | LIMIT ?limit"
```

## Reference

For complete API specifications, request/response schemas, and advanced parameters:

- [Agent Builder Kibana API Documentation](https://www.elastic.co/docs/explore-analyze/ai-features/agent-builder/kibana-api)
- [Kibana API Reference](https://www.elastic.co/docs/api/doc/kibana)

For ES|QL syntax and capabilities, see [references/QUERY-DSL.MD](QUERY-DSL.MD) and ES|QL documentation.
