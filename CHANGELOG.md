# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

From v9.3.1 onwards, version numbers track the Elastic Stack version the skill is tested
against. Prior releases (1.0.0, 1.1.0) used independent semantic versioning.

## [Unreleased]

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

[unreleased]: https://github.com/face0b1101/elasticsearch-skill/compare/v9.3.1...HEAD
[9.3.1]: https://github.com/face0b1101/elasticsearch-skill/compare/v1.1.0...v9.3.1
[1.1.0]: https://github.com/face0b1101/elasticsearch-skill/compare/v1.0.0...v1.1.0
[1.0.0]: https://github.com/face0b1101/elasticsearch-skill/releases/tag/v1.0.0
