---
name: definite-onprem
description: >-
  Query data, build data apps, and run automations on a self-hosted (on-prem)
  Definite deployment via its MCP server. Use whenever the user asks to query
  their Definite data, run SQL or a semantic query, search the ontology, build or
  edit a data app, set up or inspect an integration or sync, run an automation, or
  drive the `definite` CLI on their deployment. Triggers include "definite",
  "our data", "query my data", "data app", "run sql", "semantic query", "semantic
  model", "ontology", "metric", "integration", "sync", "automation", "pipeline",
  "on-prem", "self-hosted definite".
---

# Definite (on-prem)

The MCP server for a self-hosted Definite deployment is connected. For any data,
analytics, data app, or pipeline request, use these tools directly: do not ask
the user to "use the connector," just use it. A caller only sees what their
Definite user is permitted to see.

## Find the right metric before writing SQL

Definite has a layered model. Search the top layers first so you use certified
metrics and the correct columns, instead of guessing against raw tables.

- **`search_ontology(query)`**: start here for business language (customer,
  revenue, order, returns, SKU, buyer, funnel). Each concept links down to
  measures, dimensions, models, tables/columns, scripts, or docs.
- **`describe_ontology_object(name)`**: full definition of one concept with its
  aliases, guidance, and typed links.
- **`search_semantic(query)`**: find the certified metric/dimension and learn
  which raw columns to avoid.
- **`list_semantic_models()`** / **`describe_semantic_model(name)`**: list models
  with their grain, then get the full definition (dimensions, measures,
  relationships, AI context).

## Querying data

- **`run_semantic_query(model, measures, dimensions, filters, limit)`**: prefer
  this whenever the thing you want is already in the semantic layer. Name
  dimensions and measures from a model and get grouped rows back, plus the
  generated `sql`. `filters` target dimensions; operators include `equals`,
  `not_equals`, `in`, `not_in`, `gt`, `gte`, `lt`, `lte`, `contains`, `is_null`,
  `is_not_null`.
- **`run_sql_query(sql)`**: row-level detail, ad-hoc joins, schema discovery.
  - Discover schema with `information_schema`, not `SHOW TABLES`.
  - Always include a `LIMIT` (100 to 2000).
  - For JSON columns use arrow syntax: `->` returns JSON, `->>` returns text;
    cast for math, e.g. `(d ->> 'price')::DOUBLE`.

## Building a data app (end to end)

The build (tsc + eslint + esbuild) runs inside the deployment, so you author
source files and the server compiles them. No node or npm on your side.

1. **`scaffold_data_app()`**: returns the starter template's editable files
   (`app.json` manifest + `src/` React components) plus authoring notes.
2. Edit the manifest's `resources.<key>.source.sql` and the components, then
   **`save_data_app(slug, files, mode="create")`**.
   - `mode="create"` (default) is create-only. An existing slug returns
     `{ok: false, stage: "slug_conflict"}`: pick a new slug, or ask the user
     before switching to `mode="update"`.
   - `mode="update"` merges `files` over the app's stored source. Send only the
     changed files; remove files with `delete_paths`.
   - A failed build returns `{ok: false, stage: "build", error: <compiler
     output>}`: fix the source and call again.
   - On success you get the app record, its `/apps/<slug>` URL, and a per-resource
     SQL validation report.
3. **`query_data_app_resource(slug, resource_key, limit)`**: spot-check the rows
   feeding each chart (default 50, max 500) without opening the app.
4. **`validate_data_app(slug)`**: dry-run every SQL resource (`SELECT ... LIMIT 0`)
   to catch unknown tables/columns and type mismatches any time.
- **`list_data_apps()`** / **`get_data_app(slug, include_source=True)`**: browse,
  or pull an existing app's source to edit it.

Manifest contract: `version: 2`, each `resources.<key>.source = {type: "sql",
sql: "..."}` against lakehouse tables. Do not include build config, `runtime/`,
`types/`, or `node_modules`: the server overlays those at build time.

**Date gotcha:** comparing a DATE column to a `::TEXT`-cast expression is a DuckDB
binder error that zeros out every KPI. Build date predicates from the picker's
`YYYY-MM-DD` `from`/`to` strings client-side instead.

## Integrations & automations

- **`list_integrations()`** / **`get_integration(id_or_name)`**: list or inspect
  integrations. Secrets are never returned.
- **`list_automations()`** / **`run_automation(pipeline_id)`**: list pipelines, or
  trigger a run and get back the run record.

## The `definite` CLI over MCP

Everything not covered by a dedicated tool (automation/script/agent CRUD,
semantic and ontology writes, transform projects, file loads, permission grants)
is available through the CLI.

- **`run_cli(args)`**: runs an allowlisted `definite` CLI command, authenticated
  as you, output as JSON. Call **`run_cli([])`** to get `definite --help`; every
  subcommand accepts `--help`.
  - Allowlisted groups: `run`, `semantic`, `ontology`, `transform`, `version`.
    Deploy/ops commands (`init`, `upgrade`, `admin`, ...) and interactive
    `run fi` are blocked.
  - Bounded: 120s timeout, output truncated past 64 KiB stdout / 16 KiB stderr,
    concurrent calls capped.
- **`write_workspace_file(path, content)`** / **`read_workspace_file(path)`** /
  **`list_workspace_files()`**: a private scratch dir that is the CLI's working
  directory. Use it for file-taking commands (`semantic save -f`,
  `run automation create <file>`, `transform apply <dir>`) and to read outputs
  back (`semantic pull`, `run app scaffold`). The workspace is ephemeral: a pod
  restart clears it.

## Permissions & errors

- Reads respect your user's grants. Authoring a data app (`save_data_app`)
  requires the editor role; editing an existing app requires edit access on it.
- A `401` means the connection token expired (sessions last 14 days): reconnect
  the connector or re-add the MCP with a fresh token.

## Tool index

Discovery: `search_ontology`, `describe_ontology_object`, `search_semantic`,
`list_semantic_models`, `describe_semantic_model`
Querying: `run_semantic_query`, `run_sql_query`
Data apps: `scaffold_data_app`, `save_data_app`, `validate_data_app`,
`query_data_app_resource`, `list_data_apps`, `get_data_app`
Integrations & automations: `list_integrations`, `get_integration`,
`list_automations`, `run_automation`
CLI: `run_cli`, `write_workspace_file`, `read_workspace_file`,
`list_workspace_files`
