# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

From v9.3.1 onwards, version numbers track the Elastic Stack version the skill is tested
against. Prior releases (1.0.0, 1.1.0) used independent semantic versioning.

## [Unreleased]

## [9.3.1-4] - 2026-04-10

### Added

- Workflow `/test` endpoint documentation (dry-run, identical to `/run` but marked as test)
- KQL string expression syntax for `if` conditions: field equality, ranges, wildcards, and logical operators
- `foreach` context variables: `foreach.index` (zero-based), `foreach.total`, `foreach.items`
- Bracket notation for dotted field names in foreach: `{{ foreach.item['service.name'] }}`
- Known issue: workflows cannot self-trigger (with external HTTP service workaround)
- Known issue: stale reads from `elasticsearch.search` step (with `elasticsearch.request` workaround and explicit refresh step)
- "release-ready" trigger phrase in AGENTS.md for automated end-to-end release workflow

### Changed

- Updated `foreach` example to demonstrate `foreach.index` and `foreach.total` usage
- Updated Template Variables table with `foreach.index` and `foreach.total` entries
- Improved release workflow in AGENTS.md: automated version determination, `zip -j` for flat root files, automated release notes extraction from CHANGELOG

## [9.3.1-3] - 2026-03-13

### Fixed

- `if` condition syntax: corrected `{{ }}` (template) to `${{ }}` (expression) — template syntax renders to the string `"true"/"false"` which the engine does not recognise as a boolean, causing conditions to always evaluate to false
- `ai.agent` output path: corrected from `steps.NAME.output.message` to `steps.NAME.output` (plain string, not nested)
- Updated Step Output Paths table to reflect correct `ai.agent` output root

### Added

- "Known Issues" section in Workflows reference replacing "API Naming Gotcha", documenting:
  - `lastExecution` always returns `null` bug with workaround via executions endpoint
  - Cases `_find` tags parameter uses OR logic (undocumented) with composite-tag workaround
  - No public API for `.workflows` system connector on alert rules, with `actions` array workaround

### Changed

- Reformatted markdown tables in Workflows reference for consistent column alignment

## [9.3.1-2] - 2026-03-09

### Added

- Workflows REST API reference: full CRUD, execution, stats, validation endpoints with `x-elastic-internal-origin: kibana` header requirement
- Workflows `rrule` scheduled trigger syntax (cron-like with timezone support)
- Step Output Paths table documenting output root and common paths per step type
- Workflow Output section (`output:` top-level key) for controlling data returned to callers
- Agent Builder workflow tool type (`type: workflow`) for invoking workflows from agents
- `platform.core.get_workflow_execution_status` built-in tool in Agent Builder
- Workflow tools best practices in Agent Builder reference

### Changed

- Corrected `kibana.createCaseDefaultSpace` to `kibana.createCase` with required fields (`owner`, `connector`, `settings`)
- Corrected `ai.agent` fields from `agentId`/`input` to `agent_id`/`message`
- Documented `elasticsearch.search` sort limitation (string-only; object form causes silent `valid: false`)
- Updated alert enrichment recipe to use `elasticsearch.request` for descending sort
- Added agent output routing pattern for `ai.agent` (free-text output, `contains` operator)
- Updated SKILL.md Workflows section to reference REST API availability

## [9.3.1-1] - 2026-03-03

### Added

- Workflows (Kibana Automation) reference: YAML structure, triggers, step types, data/templating, error handling, and ready-to-use recipes (`references/WORKFLOWS.MD`)
- Workflows section in SKILL.md with overview and link to reference
- "Workflows (Kibana automation)" added to skill description

## [9.3.1] - 2026-03-03

### Changed

- **Versioning:** Switched from independent semver to Elastic Stack version alignment. Version numbers now track the Stack version the skill is tested against.
- Replace removed `moving_avg` aggregation with `moving_fn` in AGGREGATIONS-API.MD and AGGREGATIONS.MD (removed in ES 9.0)
- Remove Kibana saved object CRUD endpoints from KIBANA-API.MD (removed in Kibana 9.0); only `_import`/`_export` remain
- Rewrite ES|QL section with 9.x features: LOOKUP JOIN (GA 9.1), INLINE STATS (GA 9.3), CHANGE_POINT (GA 9.2)
- Rename "Data Streams & ILM" section to "Data Streams & Lifecycle" and add data stream lifecycle as the newer alternative to ILM
- Add `_cluster/reroute` response change note in CLUSTER-API.MD (no longer returns cluster state in 9.0)
- Update README ES|QL description to reference 9.x

### Added

- `Tested against: Elastic Stack 9.3.1` version indicator in SKILL.md
- Agent Builder GA status note (GA since 9.3) in AGENT-BUILDER.MD
- Data stream lifecycle examples (`_data_stream/<name>/_lifecycle`) in SKILL.md
- ES|QL examples for LOOKUP JOIN, INLINE STATS, and CHANGE_POINT commands
- "Reviewing Against a New Stack Version" process documentation in AGENTS.md
- Stack-aligned versioning scheme documentation in AGENTS.md

## [1.1.0] - 2026-02-23

### Added

- LICENCE file (CC BY 4.0)
- "What you can do" capabilities overview in README
- Claude Desktop installation instructions (zip upload via Settings > Capabilities)
- Example usage section with natural-language prompt examples
- Link to Claude skills documentation

### Changed

- Rewrite README to be concise and consistent with skill conventions
- Update licence from MIT to CC BY 4.0

## [1.0.0] - 2026-02-09

### Added

- Initial release of Elasticsearch skill for AI coding assistants and Claude Desktop
- Comprehensive REST API documentation for Elasticsearch interactions via curl
- Authentication guide with API key format and environment variable setup
- Search capabilities using Query DSL (match, bool, term, range queries)
- Document CRUD operations (create, read, update, delete, bulk operations)
- Index management operations (create, delete, mappings, aliases, templates)
- Aggregations support (terms, date histogram, metrics, nested aggregations)
- Cluster health and troubleshooting commands
- Data streams and ILM (Index Lifecycle Management) policies
- ES|QL (Elasticsearch Query Language) support for Elasticsearch 8.11+
- Ingest pipeline configuration and testing
- Serverless Elasticsearch compatibility notes and limitations
- Reference documentation:
  - `QUERY-DSL.MD`: Complete Query DSL reference with examples
  - `AGGREGATIONS.MD`: Aggregation types and patterns
  - `AGGREGATIONS-API.MD`: Aggregation API reference
  - `SEARCH-API.MD`: Search API parameters and options
  - `DOCUMENT-API.MD`: Document operations reference
  - `INDEX-API.MD`: Index management reference
  - `MAPPING-API.MD`: Mapping configuration and field types
  - `CLUSTER-API.MD`: Cluster health and troubleshooting APIs
  - `KIBANA-API.MD`: Kibana REST API for dashboards, data views, and saved objects
  - `OTEL-DATA.MD`: OpenTelemetry data patterns (logs, traces, metrics)
  - `AGENT-BUILDER.MD`: Kibana Agent Builder for AI-powered data queries
- Best practices and tips for working with Elasticsearch
- Support for both traditional and serverless Elasticsearch deployments

[unreleased]: https://github.com/face0b1101/elasticsearch-skill/compare/v9.3.1-4...HEAD
[9.3.1-4]: https://github.com/face0b1101/elasticsearch-skill/compare/v9.3.1-3...v9.3.1-4
[9.3.1-3]: https://github.com/face0b1101/elasticsearch-skill/compare/v9.3.1-2...v9.3.1-3
[9.3.1-2]: https://github.com/face0b1101/elasticsearch-skill/compare/v9.3.1-1...v9.3.1-2
[9.3.1-1]: https://github.com/face0b1101/elasticsearch-skill/compare/v9.3.1...v9.3.1-1
[9.3.1]: https://github.com/face0b1101/elasticsearch-skill/compare/v1.1.0...v9.3.1
[1.1.0]: https://github.com/face0b1101/elasticsearch-skill/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/face0b1101/elasticsearch-skill/releases/tag/v1.0.0
