# Elasticsearch Skill

Markdown-based skill that teaches AI coding assistants (Cursor, Claude Code etc.) and desktop apps (Claude Desktop) how to interact with Elasticsearch and Kibana via REST/curl — no SDK or MCP required.

## What's in it?

This skill provides AI assistants with expert knowledge of Elasticsearch operations, enabling them to help you query data, manage indices, build aggregations, troubleshoot clusters, and
work with Kibana—all through curl-based REST API commands.

| File                             | What it covers                                                                                |
| -------------------------------- | --------------------------------------------------------------------------------------------- |
| `SKILL.md`                       | Auth, search, CRUD, bulk ops, index management, cluster health, ILM, ES\|QL, ingest pipelines |
| `references/QUERY-DSL.MD`        | Bool, match, term, range, nested, geo, wildcards, runtime fields, `search_after`              |
| `references/AGGREGATIONS.MD`     | Metric, bucket, and pipeline aggregations — plus SRE patterns (error rates, top-N)            |
| `references/AGGREGATIONS-API.MD` | Aggregation API request/response structure and options                                        |
| `references/SEARCH-API.MD`       | Search API parameters, sorting, pagination, highlighting                                      |
| `references/DOCUMENT-API.MD`     | Single-doc and bulk CRUD operations                                                           |
| `references/INDEX-API.MD`        | Index creation, settings, aliases, reindex                                                    |
| `references/MAPPING-API.MD`      | Field types, dynamic mappings, mapping updates                                                |
| `references/CLUSTER-API.MD`      | Cluster health, node stats, shard allocation                                                  |
| `references/KIBANA-API.MD`       | Dashboards, data views, saved objects                                                         |
| `references/AGENT-BUILDER.MD`    | Kibana Agent Builder integration                                                              |
| `references/OTEL-DATA.MD`        | OpenTelemetry data patterns in Elasticsearch                                                  |

The main `SKILL.md` covers day-to-day operations. Reference files load on demand when the LLM needs deeper detail.

## What you can do

- **Search** — full-text, bool, term, range, nested, geo, and wildcard queries via Query DSL
- **Document CRUD** — index, get, update, delete, and bulk operations
- **Aggregations** — terms, date histograms, metrics, percentiles, pipeline aggs, SRE patterns
- **Index management** — create, delete, mappings, aliases, templates, reindex
- **Cluster ops** — health checks, node stats, shard allocation, troubleshooting
- **Data streams and ILM** — lifecycle policies, rollover, hot/warm/delete tiers
- **ES|QL** — pipe-based query language with joins, inline stats, change-point detection, and more
- **Ingest pipelines** — grok, date parsing, enrichment processors
- **Kibana API** — dashboards, data views, saved objects
- **Agent Builder** — Kibana AI agents that query data via natural language
- **OpenTelemetry** — query patterns for OTel logs, traces, and metrics

## Configuration

Set your cluster credentials as environment variables before starting your assistant:

```bash
export ES_URL="https://your-cluster.es.cloud.elastic.co:443"
export ES_API_KEY="your-base64-api-key"
```

Optionally, for Kibana API access:

```bash
export KIBANA_URL="https://your-cluster.kb.cloud.elastic.co:443"
```

No server to run, no dependencies to install.

## Example usage

Once configured, just ask your assistant in natural language:

- "Show me error logs from the last hour"
- "Which services have the highest error rate today?"
- "Create an index for user events with timestamp, action, and user_id fields"
- "Build a date histogram of orders per day for the last 30 days"
- "Why are there unassigned shards in my cluster?"
- "Set up an ILM policy that rolls over at 50 GB and deletes after 90 days"

The assistant reads the skill, writes the `curl` commands, runs them, and explains the results.

## Installation

### Claude Desktop (no git required)

1. Download the zip from the [latest release](https://github.com/face0b1101/elasticsearch-skill/releases/latest)
2. In Claude, go to **Settings > Capabilities** and ensure **Code execution** is enabled
3. Under **Skills**, click **Upload skill** and select the zip

Claude will automatically use the skill when your prompt involves Elasticsearch. See [Using Skills in Claude](https://support.claude.com/en/articles/12512180-using-skills-in-claude) for more detail.

### Cursor

Clone into the Cursor skills directory:

```bash
git clone https://github.com/face0b1101/elasticsearch-skill.git \
  ~/.cursor/skills/elasticsearch
```

Cursor loads skills from `~/.cursor/skills/` automatically.

### Claude Code — personal skill (all projects)

```bash
git clone https://github.com/face0b1101/elasticsearch-skill.git /tmp/elasticsearch-skill
mkdir -p ~/.claude/skills/elasticsearch
cp /tmp/elasticsearch-skill/SKILL.md ~/.claude/skills/elasticsearch/
cp -r /tmp/elasticsearch-skill/references ~/.claude/skills/elasticsearch/
```

### Claude Code — project skill (one repo)

From your project root:

```bash
git clone https://github.com/face0b1101/elasticsearch-skill.git /tmp/elasticsearch-skill
mkdir -p .claude/skills/elasticsearch
cp /tmp/elasticsearch-skill/SKILL.md .claude/skills/elasticsearch/
cp -r /tmp/elasticsearch-skill/references .claude/skills/elasticsearch/
```

Commit the `.claude/skills/elasticsearch/` directory so your team gets it too.

### Verify (Claude Code)

Restart Claude Code, then run:

```text
/skills
```

You should see `elasticsearch` in the list.

## Why a skill and not MCP?

Elasticsearch is a REST API with a single auth header. It doesn't need a protocol translation layer — it needs a good explanation.

**A skill is just markdown** that tells the LLM how authentication works, gives it `curl` patterns for every operation, and provides query language references to load on demand. The LLM reads this, then writes and executes the actual API calls.

**An MCP server** would mean a separate process to run and maintain, tool schemas consuming context tokens on every turn, and version compatibility issues between the server and your cluster.

Skills achieve progressive disclosure by design: the main `SKILL.md` loads when triggered, and deeper reference files load only when the LLM actually needs them. No tool schema overhead, no server process, zero dependencies.

MCP still makes sense for stateful connections, binary protocols, complex auth flows (OAuth), and non-HTTP tools. Elasticsearch doesn't need any of that.

## Licence

[CC BY 4.0](LICENCE)
