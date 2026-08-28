---
name: definite-onprem
description: >-
  Query data, build data apps, run automations, and update the semantic layer
  and ontology on a self-hosted (on-prem) Definite deployment via its MCP
  server. Use whenever the user asks to query their Definite data, run SQL or a
  semantic query, resolve or search the ontology, define or edit a metric or
  semantic model, build or edit a data app, set up or inspect an integration or
  sync, run an automation, or drive the `definite` CLI on their deployment.
  Triggers include "definite", "our data", "query my data", "data app", "run
  sql", "semantic query", "semantic model", "ontology", "metric", "integration",
  "sync", "automation", "pipeline", "on-prem", "self-hosted definite".
---

# Definite (on-prem)

The MCP server for a self-hosted Definite deployment is connected. For any data,
analytics, data app, or pipeline request, use these tools directly: do not ask
the user to "use the connector," just use it. A caller only sees what their
Definite user is permitted to see.

## Find the right metric before writing SQL

Definite has a layered model. Discover through the top layers first so you use
certified metrics and the correct columns, instead of guessing against raw
tables.

1. **`resolve_ontology(term, context="")`**: call this first whenever the user
   names a code, categorical value, or short ambiguous term ("HI", "XXL",
   "missed"). Returns `{resolved, binding, value, confidence, alternatives}`.
   When `confidence` is `"ambiguous"`, show the alternatives and ask instead of
   guessing.
2. **`discover(query)`**: start every analytical question here. One call
   searches ontology business concepts and semantic models, measures, and
   dimensions together. A miss is a discovery miss, not proof that data is
   absent.
3. Drill down from the hits:
   - **`search_ontology(query)`** / **`describe_ontology_object(name)`**:
     business concepts with their aliases, guidance, and typed links down to
     measures, dimensions, models, tables/columns, scripts, docs, and URLs.
   - **`search_semantic(query)`**: find the certified metric/dimension and
     learn which raw columns to avoid.
   - **`list_semantic_models()`** / **`describe_semantic_model(name)`**: list
     models with their grain, then get the full definition (dimensions,
     measures, relationships, AI context).

## Querying data

- **`run_semantic_query(model, measures, dimensions, filters, limit)`**: prefer
  this whenever the thing you want is already in the semantic layer. Name
  dimensions and measures from a model and get grouped rows back, plus the
  generated `sql`. `filters` target dimensions; operators include `equals`,
  `not_equals`, `in`, `not_in`, `gt`, `gte`, `lt`, `lte`, `contains`, `is_null`,
  `is_not_null`.
- **`run_sql_query(sql)`**: row-level detail, ad-hoc joins, schema discovery.
  - Discover schema with `information_schema`, not `SHOW TABLES`.
  - Results are capped server-side at **500 rows and 1 MiB**; the default is
    100 rows. A larger `LIMIT` in your SQL is overridden, and the response sets
    `truncated: true`. Aggregate or paginate instead of asking for more.
  - For JSON columns use arrow syntax: `->` returns JSON, `->>` returns text;
    cast for math, e.g. `(d ->> 'price')::DOUBLE`.
  - A `403 "raw SQL access is disabled for this user"` is a per-user
    permission, not a transient error. Do not retry; fall back to
    `run_semantic_query` or ask the user to have an admin enable raw SQL.

## Updating the semantic layer and ontology

There are no dedicated MCP write tools for these; all writes go through
`run_cli` plus the workspace-file tools.

**Saves are atomic replace, not merge.** `semantic save` and `ontology save`
replace the whole definition in one transaction: any measure, dimension, or
link you leave out of the file is deleted. Never save a partial file to "add
one thing." Always pull the full current definition, edit it, and save it back.

Update a semantic model:

1. `run_cli(["semantic", "pull", "orders"])` writes `orders.yaml` into the
   workspace.
2. `read_workspace_file("orders.yaml")`, edit, then
   `write_workspace_file("orders.yaml", <full yaml>)`.
3. `run_cli(["semantic", "save", "-f", "orders.yaml"])`.
4. Check `warnings` in the output. Severity `unresolved` (the referenced parent
   does not exist yet) is advisory and the save succeeded; do not retry.
   Severity `missing` (the parent exists but the target is not in it) is a real
   problem to fix.

Also available: `semantic get <name>` prints the YAML to stdout in one call
(subject to the 64 KiB stdout cap); `semantic apply <dir>` saves every `*.yaml`
in a workspace directory (the GitOps path); a model file may include a
top-level `context:` list so one save applies the whole bundle including AI and
governance context.

Ontology writes work the same way: `ontology get <name>` for the round-trip
(there is no `ontology pull`), edit, `ontology save -f <file>`, and
`ontology delete <name> --yes`. Same replace semantics and warnings contract.

Ontology authoring vocabulary (stick to these lists; unknown relation types
silently degrade to `related_to` and lose resolver behavior):

- **Object kinds (6):** `entity`, `concept`, `metric_concept`,
  `dimension_concept`, `value_set`, `term`.
- **Relation types (12):** `alias_of`, `resolves_to`, `bound_to`,
  `derives_from`, `excludes`, `adds`, `materialized_from`, `joined_via`,
  `has_value_set`, `corrected_by`, `supersedes`, `related_to`. The resolver
  walks `alias_of`/`resolves_to` to redirect a term and `bound_to` to reach the
  physical binding.
- **Link target prefixes (10):** `ontology:`, `model:`, `dimension:`,
  `measure:`, `relationship:`, `table:`, `column:`, `script:`, `doc:`, `url:`.
- A concept with `links: []` is legal: business terms may exist before data
  backs them. Do not refuse to create one.
- Table and column targets must be schema-qualified without the `LAKE.` prefix
  (`main.orders`, not `LAKE.main.orders`); a `LAKE.` prefix is rejected.

Scope note: a write needs `fi:use` (to invoke `run_cli` and the workspace
tools) in addition to `semantic:read` (for the CLI's API calls). A token with
only `semantic:read` can query but cannot write.

## Building a data app (end to end)

The build (tsc + eslint + esbuild) runs inside the deployment, so you author
source files and the server compiles them. No node or npm on your side.

1. **`scaffold_data_app()`**: returns the starter template's editable files
   (`app.json` manifest + `src/` React components) plus authoring notes.
2. Edit the manifest's `resources.<key>.source.sql` and the components, then
   **`save_data_app(slug, files, mode="create")`**.
   - `mode="create"` (default) is create-only. `mode="update"` merges `files`
     over the app's stored source: send only the changed files; remove files
     with `delete_paths`.
   - On success you get the app record, its `/apps/<slug>` URL, and a
     per-resource SQL validation report.
   - On failure, `{ok: false, stage: ...}` tells you what to do:
     - `slug_conflict`: the slug exists. Pick a new slug, or ask the user
       before switching to `mode="update"`.
     - `build`: compiler output is in `error`. Fix the source and call again.
     - `not_found`: update against a slug that does not exist.
     - `source_conflict`: another session edited the app since you read it
       (optimistic concurrency via `expected_source_revision`). Re-pull with
       `get_data_app` and re-apply your changes.
     - `resource_validation`: a resource's SQL failed the dry-run; the stored
       app was not changed.
     - `manifest`: the manifest is invalid.
     - `result_size_confirmation`: your update removes or raises a `LIMIT`, or
       weakens a `WHERE`, sampling, or `required_filters`. Measure and explain
       the expected result scale, get explicit user approval, then retry with
       `confirm_result_size_increase=true`. Never set that flag proactively.
3. **`query_data_app_resource(slug, resource_key, limit)`**: spot-check the rows
   feeding each chart (default 50, max 500) without opening the app.
4. **`validate_data_app(slug)`**: dry-run every SQL resource (`SELECT ... LIMIT 0`)
   to catch unknown tables/columns and type mismatches any time.
- **`list_data_apps()`** / **`get_data_app(slug, include_source=True)`**: browse,
  or pull an existing app's source to edit it.

Validation never renders the app: `verification.render_verified` stays `false`
until a user opens the live app and confirms the modified views. Never claim an
app is visually verified; link the user to `/apps/<slug>` and ask them to
confirm.

Manifest contract: `version: 2`, each `resources.<key>.source = {type: "sql",
sql: "..."}` against lakehouse tables. Optional: `cache_ttl_hours` (top-level
or per resource, default 24), a `filterable` declaration per resource plus
`required_filters` when the server must reject unscoped requests. In
components, load with `useDataset`/`useJsonResource` and keep `enabled: false`
until the relevant view or filter is ready. Source caps: 200 files, 256 KiB per
file, 4 MiB total; split large apps into smaller modules. Do not include build
config, `runtime/`, `types/`, or `node_modules`: the server strips and overlays
those at build time.

**Date gotcha:** comparing a DATE column to a `::TEXT`-cast expression is a DuckDB
binder error that zeros out every KPI. Build date predicates from the picker's
`YYYY-MM-DD` `from`/`to` strings client-side instead.

## Integrations & automations

- **`list_integrations()`** / **`get_integration(id_or_name)`**: list or inspect
  integrations. Secrets are never returned.
- **`list_automations()`** / **`run_automation(pipeline_id)`**: list pipelines, or
  trigger a run and get back the run record.
- **`cancel_automation_run(run_id)`**: cancel one queued or running run without
  touching the pipeline's other runs.

## The `definite` CLI over MCP

Everything not covered by a dedicated tool (automation/script/agent CRUD,
semantic and ontology writes, transform projects, file loads, permission grants)
is available through the CLI.

- **`run_cli(args)`**: runs an allowlisted `definite` CLI command, authenticated
  as you, output as JSON. Call **`run_cli([])`** to get `definite --help`; every
  subcommand accepts `--help`.
  - Allowlisted groups: `run`, `semantic`, `ontology`, `transform`, `version`.
    The `run` group includes `app`, `load`, `drive get`, `maintenance`,
    `event-source`, and `permission` subcommands. Deploy/ops commands (`init`,
    `upgrade`, `admin`, ...) and interactive `run fi` are blocked, and the
    `--api-url` and `--token` flags are rejected.
  - Bounded: 120s timeout, output truncated past 64 KiB stdout / 16 KiB stderr,
    concurrent calls capped.
- **`write_workspace_file(path, content)`** / **`read_workspace_file(path)`** /
  **`list_workspace_files()`**: a private scratch dir that is the CLI's working
  directory. Use it for file-taking commands (`semantic save -f`,
  `run automation create <file>`, `transform apply <dir>`) and to read outputs
  back (`semantic pull`, `run app scaffold`). Limits: 1 MiB per write, 256 KiB
  per read, 500 files. The workspace is ephemeral: a pod restart clears it.

## Permissions & errors

- Reads respect your user's grants. Authoring a data app (`save_data_app`)
  requires the editor role; editing an existing app requires edit access on it.
- Raw SQL (`run_sql_query`) is a per-user permission; a 403 means it is
  disabled for this user, and no retry will succeed.
- API token scopes are enforced per tool. Login sessions and unscoped `def_`
  tokens pass everything; a scoped token errors with the missing scope named.
  The mapping: `query:read` (SQL, `discover`), `semantic:read` (semantic and
  ontology reads, `discover`), `docs:read`/`docs:write` (data apps),
  `integrations:read`, `pipelines:read`/`pipelines:run` (automations), and
  `fi:use` (`run_cli` and the workspace tools).
- A `401` means the connection token expired (sessions last 14 days): reconnect
  the connector or re-add the MCP with a fresh token. Long-lived `def_` API
  tokens do not expire with the session and are the only option on SSO-only
  deployments.

## Tool index

Discovery: `discover`, `resolve_ontology`, `search_ontology`,
`describe_ontology_object`, `search_semantic`, `list_semantic_models`,
`describe_semantic_model`
Querying: `run_semantic_query`, `run_sql_query`
Data apps: `scaffold_data_app`, `save_data_app`, `validate_data_app`,
`query_data_app_resource`, `list_data_apps`, `get_data_app`
Integrations & automations: `list_integrations`, `get_integration`,
`list_automations`, `run_automation`, `cancel_automation_run`
CLI & workspace: `run_cli`, `write_workspace_file`, `read_workspace_file`,
`list_workspace_files`
