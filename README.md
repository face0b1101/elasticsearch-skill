# Elasticsearch Skill

[![Release](https://img.shields.io/github/v/release/face0b1101/elasticsearch-skill)](https://github.com/face0b1101/elasticsearch-skill/releases)
[![License](https://img.shields.io/github/license/face0b1101/elasticsearch-skill)](LICENSE)

> Comprehensive Elasticsearch skill that teaches AI coding assistants and Claude Desktop how to interact with Elasticsearch clusters via REST APIs.

This skill provides AI assistants with expert knowledge of Elasticsearch operations, enabling them to help you query data, manage indices, build aggregations, troubleshoot clusters, and work with Kibana—all through curl-based REST API commands.

## Features

- **Search & Query DSL**: Full support for Elasticsearch Query DSL including bool, match, term, range, wildcard, regexp, and nested queries
- **Document Operations**: Complete CRUD operations with bulk indexing, updates, and deletes
- **Aggregations**: Terms, date histogram, cardinality, percentiles, and nested aggregations with sub-aggregations
- **Index Management**: Index creation, mappings, aliases, templates, reindexing, and data stream operations
- **Cluster Health**: Monitoring, diagnostics, shard allocation, task management, and troubleshooting
- **Data Streams & ILM**: Index lifecycle management for time-series data with hot/warm/cold tier policies
- **Ingest Pipelines**: Data transformation, enrichment, and preprocessing with grok, date, and custom processors
- **ES|QL Support**: Pipe-based query syntax for Elasticsearch 8.11+ with natural language-like queries
- **Kibana API**: Dashboards, data views, saved objects, alerting rules, and visualization management
- **OpenTelemetry Patterns**: Query patterns for logs, traces, metrics, and correlation across OTEL data
- **Agent Builder**: Kibana AI-powered agents that can query Elasticsearch data using natural language
- **Serverless Compatible**: Works with both traditional and serverless Elasticsearch deployments

## Installation

### For Claude Desktop

1. Download the latest release:

   ```bash
   curl -LO https://github.com/face0b1101/elasticsearch-skill/releases/download/v1.0.0/elasticsearch-skill.zip
   ```

2. Configure in Claude Desktop's MCP settings (`~/Library/Application Support/Claude/claude_desktop_config.json`):

   ```json
   {
     "mcpServers": {
       "elasticsearch": {
         "command": "mcp-server-skill",
         "args": ["path/to/elasticsearch-skill.zip"]
       }
     }
   }
   ```

3. Restart Claude Desktop

### For Cursor

1. Download the latest release:

   ```bash
   curl -LO https://github.com/face0b1101/elasticsearch-skill/releases/download/v1.0.0/elasticsearch-skill.zip
   ```

2. Place the zip file in your skills directory:

   ```bash
   mkdir -p ~/.cursor/skills
   unzip elasticsearch-skill.zip -d ~/.cursor/skills/elasticsearch
   ```

3. The skill will be automatically available in Cursor

### For Claude Code (VSCode Extension)

1. Download and extract to your agent skills directory:

   ```bash
   mkdir -p ~/.claude/skills
   curl -L https://github.com/face0b1101/elasticsearch-skill/releases/download/v1.0.0/elasticsearch-skill.zip -o /tmp/elasticsearch-skill.zip
   unzip /tmp/elasticsearch-skill.zip -d ~/.claude/skills/elasticsearch
   ```

2. The skill will be automatically detected by Claude Code

## Quick Start

### Prerequisites

You'll need:

- An Elasticsearch cluster URL (Cloud, self-managed, or serverless)
- An API key with appropriate permissions

### Example Usage

Ask your AI assistant:

```txt
"Query my Elasticsearch cluster at https://my-cluster.es.cloud.elastic.co
for error logs in the last hour"
```

The assistant will use the skill to:

1. Set up authentication with your cluster
2. Construct the appropriate Query DSL
3. Execute the search via curl
4. Parse and explain the results

### Basic Commands

The skill teaches AI assistants to execute commands like:

```bash
# Check cluster health
curl -s "${ES_URL%/}/_cluster/health" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" | jq .

# Search for documents
curl -s "${ES_URL%/}/my-index/_search" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "query": { "match": { "message": "error" } },
    "size": 10
  }' | jq .

# Build aggregations
curl -s "${ES_URL%/}/my-index/_search?size=0" \
  -H "Authorization: ApiKey $(printenv ES_API_KEY)" \
  -H "Content-Type: application/json" \
  -d '{
    "aggs": {
      "by_level": { "terms": { "field": "level" } }
    }
  }' | jq '.aggregations'
```

For comprehensive documentation, see [SKILL.md](SKILL.md).

## Documentation

### Core Documentation

- **[SKILL.md](SKILL.md)**: Complete skill documentation with examples and best practices
- **[CHANGELOG.md](CHANGELOG.md)**: Version history and release notes

### API References

Located in the [`references/`](references/) directory:

- **[QUERY-DSL.MD](references/QUERY-DSL.MD)**: Complete Query DSL reference (match, bool, term, range, wildcard, regexp, nested, exists)
- **[SEARCH-API.MD](references/SEARCH-API.MD)**: Search API parameters, sorting, pagination, and highlighting
- **[AGGREGATIONS.MD](references/AGGREGATIONS.MD)**: Aggregation types and patterns with real-world examples
- **[AGGREGATIONS-API.MD](references/AGGREGATIONS-API.MD)**: Detailed aggregation API reference
- **[DOCUMENT-API.MD](references/DOCUMENT-API.MD)**: Document CRUD operations and bulk API
- **[INDEX-API.MD](references/INDEX-API.MD)**: Index management, aliases, templates, and settings
- **[MAPPING-API.MD](references/MAPPING-API.MD)**: Mapping configuration and field type reference
- **[CLUSTER-API.MD](references/CLUSTER-API.MD)**: Cluster health, stats, and troubleshooting APIs
- **[KIBANA-API.MD](references/KIBANA-API.MD)**: Kibana REST API for dashboards, data views, and saved objects
- **[OTEL-DATA.MD](references/OTEL-DATA.MD)**: OpenTelemetry data patterns (logs, traces, metrics, correlation)
- **[AGENT-BUILDER.MD](references/AGENT-BUILDER.MD)**: Kibana Agent Builder for AI-powered data queries

## Project Structure

```sh
elasticsearch-skill/
├── SKILL.md                 # Main skill documentation
├── README.md                # This file
├── CHANGELOG.md             # Version history
├── AGENTS.md                # Agent/AI assistant rules
└── references/              # Detailed API references
    ├── QUERY-DSL.MD
    ├── SEARCH-API.MD
    ├── AGGREGATIONS.MD
    ├── AGGREGATIONS-API.MD
    ├── DOCUMENT-API.MD
    ├── INDEX-API.MD
    ├── MAPPING-API.MD
    ├── CLUSTER-API.MD
    ├── KIBANA-API.MD
    ├── OTEL-DATA.MD
    └── AGENT-BUILDER.MD
```

## Development

### Building the Release Zip

To create a distribution zip file:

```bash
cd elasticsearch-skill
zip -r elasticsearch-skill.zip . -x "*.git*" -x "*/.DS_Store" -x ".gitignore" -x "*.zip"
```

### Contributing

Contributions are welcome! Please feel free to submit issues, feature requests, or pull requests.

When contributing:

1. Follow the existing documentation structure
2. Include practical examples
3. Test commands against a real Elasticsearch cluster
4. Update the CHANGELOG.md with your changes

### Release Process

1. Update `CHANGELOG.md` with changes in the `[Unreleased]` section
2. Create a new version section in the changelog
3. Commit changes: `git commit -m "chore: prepare vX.Y.Z release"`
4. Create and push tag: `git tag -a vX.Y.Z -m "Release vX.Y.Z" && git push origin vX.Y.Z`
5. Build the zip file
6. Create GitHub release with the zip as an attachment

See [Keep a Changelog](https://keepachangelog.com/en/1.0.0/) for changelog format guidelines.

## Related Resources

- [Elasticsearch Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Kibana Documentation](https://www.elastic.co/guide/en/kibana/current/index.html)
- [Elasticsearch REST API Reference](https://www.elastic.co/guide/en/elasticsearch/reference/current/rest-apis.html)
- [Query DSL](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)
- [Aggregations](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations.html)

## License

This project is open source. See the repository for license details.

## Releases

- [Latest Release](https://github.com/face0b1101/elasticsearch-skill/releases/latest)
- [All Releases](https://github.com/face0b1101/elasticsearch-skill/releases)
- [Changelog](CHANGELOG.md)

______________________________________________________________________

**Note**: This skill requires an Elasticsearch cluster (version 7.x or 8.x) and valid API credentials. It works with Elastic Cloud, self-managed deployments, and serverless Elasticsearch.
