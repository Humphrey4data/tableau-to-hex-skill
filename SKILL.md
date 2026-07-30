---
name: tableau-to-hex-migration
description: Migrate a Tableau, Looker, or Power BI dashboard to a Hex app that matches it exactly — same queries, results, filters, colors, layout, and tooltips. Use when the user pastes a Tableau dashboard/view link, a Looker dashboard link, or a Power BI report link and asks to migrate, replicate, copy, or rebuild it in Hex, or mentions "Tableau to Hex" / "Looker to Hex" / "Power BI to Hex".
---

# Tableau/Looker/Power BI → Hex Dashboard Migration

Given a Tableau, Looker, or Power BI dashboard URL, produce a Hex app that is a faithful copy: same underlying queries and numbers, same filters, same colors/layout/labels, same tooltips. The build is driven by the **Hex agent** (`hex thread` CLI); source logic is extracted from the **ground truth** — Tableau's workbook file (.twbx), Looker's dashboard + query APIs, or Power BI's report definition (PBIR) + semantic model (TMDL/DAX) — and validated against the **same warehouse Hex queries** before it is handed to Hex.

All command snippets referenced below live in [reference.md](reference.md) — read it before starting Phase 1.

**Adaptivity rule**: the procedure is environment-agnostic; only the leaf commands vary. Phase 0 pins four environment choices — **source BI** (Tableau / Looker / Power BI), warehouse engine (Snowflake / BigQuery / Databricks / Postgres-Redshift), source hosting (Tableau Cloud vs self-hosted Server; Looker-hosted vs customer-hosted; Power BI capacity/licensing tier), and OS (macOS vs Linux) — and every snippet in reference.md is either shared or labeled per branch. Follow your branch exactly; don't mix branches mid-run. (The Tableau + Snowflake + macOS branch is the most battle-tested; the others are documented equivalents.) Phases 2–5 are source-agnostic: Phase 1 is the **source adapter**, and whatever the source, it must produce the same artifact — a written spec with ground-truth queries, calc definitions, parameters/filters, colors, layout, and a rendered PNG.

## Preconditions and non-goals

Confirm these before Phase 1, and tell the user about anything that applies:

- **Works for**: dashboards of charts, tables, and heatmaps whose datasources are reachable from a Hex warehouse connection (Tableau custom SQL or published sources; Looker explores/Looks on a warehouse connection Hex can also reach; Power BI DirectQuery/composite models over such a warehouse, or import models whose upstream source tables live in one), with quick filters and parameters (Looker: dashboard filters and listen-mapped tiles; Power BI: slicers and report/page/visual-level filters).
- **Power BI hard blockers — check before promising anything**: (a) an **import-mode model with no reachable upstream warehouse** (data lives only inside the pbix/semantic model) has nothing for Hex to query — stop and tell the user; (b) measures are **DAX, not SQL** — every measure must be *translated* to warehouse SQL and proven equivalent, which makes Phase 2 the dominant cost and complex DAX (nested `CALCULATE` context transitions, heavy time intelligence) a per-measure judgment call; (c) licensing walls — XMLA read and export-to-file need Fabric capacity or Premium-Per-User, and the `executeQueries` REST API needs its tenant setting enabled. Budget roughly 2× the Tableau/Looker correction rounds.
- **Hex workspace requirements**: the CLI must be enabled for the workspace, and Phase 3 depends on the **Hex agent (threads)** feature, which consumes the workspace's AI credits and may be disabled or exhausted on some plans. Verify early — `hex thread create` failing with a credits/feature error means the whole build path is blocked until the workspace admin enables it (there is no free fallback with comparable quality).
- **No clean Hex equivalent — agree a fallback with the user up front**: Tableau Stories; dashboard actions (filter/highlight/URL/set actions); row-level security and user filters; user-specific data blending.
- **Out of scope**: refresh schedules, permission migration, publishing/embedding, alert subscriptions. The deliverable is a draft Hex app at visual/numeric parity (Phase 5 evidence pack).
- A container usually holds **several dashboards** (a Tableau workbook, a Power BI report's pages, a Looker folder). After Phase 1 lists them, **ask the user two scope questions** and record the answers in the spec:
  1. **Which to migrate** — all of them, only the linked one, or a subset. Default to only the linked one if they don't care.
  2. **One app or one app with tabs** — if the selected dashboards are **tabs of the same logical report** (a Tableau workbook published with tabs, a Power BI report's pages), the default is **one Hex Generative app with a tab per dashboard** (generative apps support tab navigation; say so in the build prompt and keep each tab's SQL cells scoped to its own dashboard). Only genuinely unrelated dashboards get separate apps (one thread each).

  Whatever the grouping, never mix panels from a dashboard that isn't in scope into another dashboard's tab.

## Progress checklist

```
- [ ] Phase 0: Setup + ground rules (tooling, fidelity policy, run manifest)
- [ ] Phase 1: Extract the source spec (Tableau: twbx + XML; Looker: dashboard/query APIs; Power BI: PBIR + TMDL/DAX; + rendered PNGs)
- [ ] Phase 2: Validate every query against the warehouse (the source's numbers as truth)
- [ ] Phase 3: Build the Hex app with `hex thread create --new-project`
- [ ] Phase 4: Self-check loop (screenshot diff, YAML, numbers) until parity
- [ ] Phase 5: Deliver the parity evidence pack, get user sign-off
```

## Phase 0 — Setup and ground rules

Check what's installed; install only what's missing.

1. **Hex CLI** (required — this drives the whole build; macOS/Linux, Windows via WSL):
   ```bash
   hex --version || curl -fsSL https://hex.tech/install.sh | bash
   hex auth status || hex auth login          # opens browser; confirms workspace + user
   ```
2. **Source BI access** (required — pick your source branch):
   - **Tableau**: a Personal Access Token for the Tableau site — works for both **Tableau Cloud** and self-hosted **Tableau Server** (Server needs 2019.4+ for PATs; branch notes: reference.md §1). If a Tableau MCP server is configured for your agent, its `env` block already has `SERVER`, `SITE_NAME`, `PAT_NAME`, `PAT_VALUE` — reuse those values for the REST API (config locations per agent: reference.md §5). Otherwise ask the user for a PAT (Tableau: My Account Settings → Personal Access Tokens). PATs expire after ~15 days of non-use. If sign-in returns `401001` (also shows up as the Tableau MCP server stuck in an `error` state), the PAT is dead — recovery steps: reference.md §1.
   - **Looker**: **API3 credentials** (client ID + secret; user's Admin → Users → Edit → API keys, or ask a Looker admin) plus the instance base URL. Works for Looker-hosted and customer-hosted instances alike via `POST /api/4.0/login` (reference.md §11). If a Looker MCP is configured (managed MCP server or the MCP Toolbox `--prebuilt looker`), its env already has `LOOKER_BASE_URL`/`LOOKER_CLIENT_ID`/`LOOKER_CLIENT_SECRET` — reuse those. The API user needs access to the dashboard's folder and explores (API calls inherit their content permissions — a tile you can't see in the UI won't return SQL either).
   - **Power BI**: a **Microsoft Entra token** — easiest via the Azure CLI: `az login --allow-no-subscriptions` then `az account get-access-token` per API audience (reference.md §12); a service principal also works but skips row-level security, which changes the numbers you validate against. The user needs Build permission on the semantic model and read+write on the report (the Fabric `getDefinition` API oddly requires write). Also verify the licensing preconditions above **now**, and check whether Microsoft's Power BI MCP servers are available (remote Fabric-hosted or local `@microsoft/powerbi-modeling-mcp`) — useful accelerators, not required (reference.md §12).
3. **Warehouse CLI** (required for validation): determine which engine the Hex data connection uses — ask the user or check the Hex workspace's data connections — then set the `WH` branch and smoke-test it (reference.md preamble): Snowflake `snow`, BigQuery `bq`, Databricks `dbsqlcli`/Statement API, Postgres/Redshift `psql`. For any other engine (Trino, Athena, DuckDB/MotherDuck, ...), follow the same pattern: any CLI that takes a SQL string and returns JSON rows plugs into §8/§9 unchanged. **All validation SQL and everything you feed the Hex prompt must be in this engine's dialect** — Tableau's twb custom SQL and Looker's generated SQL already are (they ran against that warehouse), so never translate those; Power BI DAX is the exception that must be translated into this dialect (Phase 2). Never validate against a different engine than the one Hex will query.
4. **Source MCP** (optional, nice for live queries from chat): Tableau MCP config in reference.md §5; Looker managed MCP server or MCP Toolbox config in reference.md §11; Power BI remote/local MCP config in reference.md §12. The REST API paths work without any of them.
5. **Fidelity policy — ask the user now, not at Phase 4**: if the source dashboard turns out to be internally inconsistent (two panels disagreeing, a label that doesn't match its formula), should the copy **replicate the inconsistency faithfully (default)** or fix it in flight? The answer defines what "parity" means for the whole run; record it in the spec.
6. **Run manifest — create it first, update it as IDs appear**: a migration spans hours and multiple sessions; `thread_id`/`project_id` must not live only in shell scrollback. Use a stable run directory that survives reboots — `~/.tab2hex/<dashboard-slug>/` (not `/tmp`) — for the spec, saved SQL, PNGs, and a `manifest.json` with: dashboard URL, source IDs (Tableau: workbook + view LUIDs; Looker: dashboard ID + tile query IDs; Power BI: workspace + report + semantic-model IDs), spec path, source PNG paths, `thread_id`, `project_id`, Hex draft URL, and the environment branch (source BI, `WH` engine + connection name, hosting flavor, OS). **Resuming a partial migration** = read the manifest, re-pin the same environment branch, `hex project export` to see current state, continue the existing thread with `hex thread continue` — never start a second thread for the same app.
7. **Secret handling**: the source secret (Tableau PAT / Looker client secret / Entra token) lives only in your agent's MCP config (locations: reference.md §5, §11, §12) and session env vars — never in the repo, the spec, the run manifest, or the Hex prompt. Pass it to `curl` via a payload file, not inline `-d` (terminal transcripts capture full command text; reference.md §1, §11). Remind the user to revoke/rotate it when the migration is done.
8. **Time budget**: this is a multi-hour task, not a one-shot. Expect ~10–15 min of thread time for the initial build, ~4–10 min per correction round plus export/validate/screenshot time, and **3+ correction rounds** (dense dashboards take more). Heavy validation queries may need a larger warehouse.
9. **Approval fatigue — allowlist commands and front-load logins (tell the user this now, not mid-run)**: the run issues hundreds of terminal commands over hours, and every "Allow once" click makes the user the bottleneck (worse: an unattended run stalls at the first prompt).
   - **Suggest allowlisting the repeated CLIs** in the agent's settings for this session/project: `curl`, `python3`, `unzip`, `hex`, the warehouse CLI (`snow` / `bq` / `dbsqlcli` / `psql`), and `az` (Power BI branch). Where: **Cursor** → Settings → Terminal command allowlist (or per-prompt "Always allow" for that command); **Claude Code** → `/permissions` → add allow rules like `Bash(hex:*)`, `Bash(curl:*)` (or `permissions.allow` in `.claude/settings.json`); **Codex** → approvals policy in config. Scope rules to these specific binaries — do NOT suggest blanket auto-approval of all commands, and never allowlist destructive ones (`rm`, `git push`, etc.).
   - **Front-load every interactive login into Phase 0, back-to-back, while the user is present**: `hex auth login` (browser), the one-time headed browser login for the Hex screenshot profile (§10), `az login` (Power BI), and any MCP OAuth flows. Each is once-per-machine; batching them here means Phases 1–4 run unattended instead of stalling on a surprise pop-up. macOS only: the first `hex` call may trigger a Keychain prompt — have the user click "Always Allow" so it doesn't recur.

## Phase 1 — Extract the source spec

Goal: a written spec of every section before any Hex work. Do not build from a screenshot alone — the calculation logic must come from the source's ground truth (Tableau: the workbook file; Looker: the query APIs; Power BI: the PBIR report definition + TMDL model). Every branch ends at the same final spec step.

### Tableau branch

1. **Parse the URL**: `https://<server>/#/site/<site>/views/<WorkbookContentUrl>/<ViewName>` → workbook content URL + view name. (Tableau Server **default site** URLs omit the `/site/<site>` segment: `https://<server>/#/views/<WorkbookContentUrl>/<ViewName>` — use an empty `TAB_SITE`.) The view name may be a **dashboard or a single worksheet** — after downloading the workbook, list its dashboards (reference.md §3), show the list to the user and confirm the scope (all / the linked one / a subset — see Preconditions), then scope strictly to the chosen dashboards' worksheets.
2. **Sign in to the REST API** with the PAT, look up the workbook LUID, and **download the .twbx** (`/workbooks/<luid>/content`). Unzip it; the `.twb` inside is XML. (Commands: reference.md §1–2.)
3. **Parse the .twb XML** (reference.md §3) to extract:
   - **Dashboards → worksheets**: which sheets are on the target dashboard, their layout order and sizes. Ignore hidden sheets and QA/validation scaffolding sheets.
   - **Datasources → custom SQL**: every `<relation type='text'>` is a full SQL query. Save each to the run directory — these are the ground-truth queries.
   - **Calculated fields**: `<column>` elements with `<calculation formula='...'>`. Worksheet-level table calcs (RUNNING_SUM, WINDOW_*) also live here — this is the only place to see them; the Metadata API does not expose them.
   - **Parameters** and their allowed values — each one becomes a Hex input parameter. **If the workbook has no `Parameters` datasource and no quick filters, that is a legitimate outcome**: record "no filters/parameters" in the spec and skip Phase 4 step 3 / reference.md §9 entirely.
   - **Filters** per worksheet (fields + members) and **color encodings**.
4. **Capture visual reference**: download rendered images of the dashboard and key sheets via the REST view-image endpoint (reference.md §4), and ask the user for screenshots of hover tooltips (tooltips render client-side and are not in the image export).
5. **Check for dbt (or other transform) models**: if a custom SQL query reads a table built by a dbt project you have access to, read the model too — it often contains semantic details (business calendars, metric bundling, completeness lags) the Tableau SQL silently relies on.

### Looker branch

(Commands: reference.md §11. Looker's API hands you more than Tableau's — the *exact generated warehouse SQL* per tile — so extraction is mechanically easier, but table calcs hide in a different place; see step 3.)

1. **Parse the URL**: `https://<instance>/dashboards/<id>` — a numeric `<id>` is a user-defined dashboard (UDD); `<model>::<slug>` is a LookML dashboard (element addressing differs; both work with the same endpoints). `/looks/<id>` is a single saved Look — treat it as a one-tile dashboard.
2. **Sign in and fetch the dashboard**: `POST /api/4.0/login` with the API3 credentials, then `GET /dashboards/<id>` → title, `dashboard_filters` (name, dimension, default value, per-tile listen mapping), `dashboard_layouts` (grid position/size per tile), and `dashboard_elements` (title, `query_id`, `look_id`, `merge_result_id`, `vis_config`).
3. **Pull ground truth per tile**:
   - **Generated SQL**: `GET /queries/<query_id>/run/sql` returns the exact warehouse SQL Looker runs — already in the warehouse's dialect, exclusions and calendar logic included. Save one file per tile; these go into Phase 2 verbatim.
   - **The trap — post-SQL layers**: the SQL does **not** include table calculations, custom fields (both live in the query's `dynamic_fields` JSON), pivots, or row totals — Looker applies those client-side after the SQL returns, exactly like Tableau's worksheet table calcs. Read `GET /queries/<query_id>` and translate every `dynamic_fields` entry into SQL (window functions) or a Hex-side transform; a tile whose SQL matches but whose table calc was dropped will pass the number check and fail the screenshot diff.
   - **Reference values**: `GET /queries/<query_id>/run/json` — the rows Looker actually displays; this is the comparison target for Phase 2 (no OCR from screenshots needed).
   - **Merged-results tiles** (`merge_result_id` set, `query_id` null): fetch the merge query to get the source query IDs, extract each, and replicate the join in SQL for Hex.
   - **Chart config**: `vis_config` on each element holds the chart type, series colors, axis options, and value formats — the Looker equivalent of the twb's color encodings.
4. **Capture visual reference**: create a dashboard render task (`POST /render_tasks/dashboards/<id>/png`), poll until `success`, download the PNG (reference.md §11). Ask the user for hover-tooltip screenshots — same rule as Tableau, tooltips render client-side.
5. **Check the semantic layer**: LookML plays the role dbt models play in the Tableau branch — if a dimension/measure's meaning is unclear from the generated SQL, read its LookML definition (via the Metadata endpoints, the LookML git repo, or the Looker MCP's `get_dimensions`/`get_measures`). Business calendars, bake lags, and metric bundling live there.

### Power BI branch

(Commands: reference.md §12. No .pbix binary parsing needed — the Fabric APIs return the report as readable JSON and the model as TMDL. The extraction is easy; the **DAX→SQL translation in Phase 2 is where the work lives**.)

1. **Parse the URL**: `https://app.powerbi.com/groups/<workspaceId>/reports/<reportId>/<pageId>` → workspace + report IDs (`groups/me` = My workspace — ask the user to publish to a real workspace first). The URL pins one **page**, but a report usually has several — after step 2 lists them, ask the user which pages to migrate (all / the linked one / a subset — see Preconditions).
2. **Get the report definition** (layout ground truth): Fabric `POST .../reports/<id>/getDefinition` → base64 parts in **PBIR** format (`definition/pages/*/visuals/*/visual.json` — visual type, bound fields/measures, formatting incl. colors; page-order and filter files) or **PBIR-Legacy** (one `report.json`; each visual's `config` is a JSON *string* inside the JSON — double-parse it). This is the twb-XML equivalent: sections, visuals, slicers, report/page/visual-level filters, bookmarks, custom tooltip pages.
3. **Get the semantic model** (calc ground truth): the report's `definition.pbir` names its semantic model → export it as **TMDL** (Fabric semantic-model `getDefinition`, the local modeling MCP, or XMLA via Tabular Editor CLI). From TMDL pull: every **DAX measure**, calculated columns, relationships, RLS roles, and each table's **M `partition` source** — the M source reveals the upstream warehouse table (and for import mode, whether one exists at all — the hard-blocker check).
4. **Pull reference values per visual**: run each visual's measure/grouping as a DAX `EVALUATE SUMMARIZECOLUMNS(...)` via the `executeQueries` REST API (tenant setting required; 100k-row cap) or the MCP Execute-Query tool. These rows are the Phase 2 comparison target — the displayed truth, RLS included.
5. **Capture visual reference**: `exportToFile` (PNG/PDF) if the workspace is on capacity; otherwise screenshot app.powerbi.com with the §10 Playwright machinery (one-time login, persistent profile — same as the Hex side). Ask the user for hover-tooltip screenshots; also extract any **tooltip pages** (hidden pages flagged as tooltips in PBIR) — those are replicable from the definition, unlike hover defaults.
6. **Check upstream transforms**: heavy Power Query (M) steps are that model's dbt — if the M does real transformation (merges, pivots, custom columns), replicate it in the warehouse SQL, and prefer reading any actual dbt/warehouse models the M reads from.

### All branches — final step

**Write the spec** to the run directory: one block per dashboard section with exact title, subtitle, metrics + formulas, filter wiring, colors (hex codes from the twb / `vis_config` / `visual.json`), tooltip fields and labels, time ranges, number formats, and "as of" date logic. Record the fidelity-policy decision and any preconditions flagged above.

## Phase 2 — Validate queries against the warehouse

For every section, before telling Hex anything:

1. Rewrite the section's logic as a standalone query in the warehouse's own dialect (start from the custom SQL / calculated fields — already in that dialect — not from guesswork). **Looker branch shortcut**: the per-tile generated SQL from Phase 1 runs verbatim — your work is folding the `dynamic_fields` layers (table calcs, custom fields) into it, not reconstructing the query. **Power BI branch — this is the expensive phase**: translate each DAX measure to warehouse SQL measure by measure (filter context → WHERE/GROUP BY; `CALCULATE` modifiers → scoped CTEs; time intelligence → date-table joins), validating each against its `executeQueries` result before composing visuals from them. Never batch-translate: one wrong measure poisons every visual built on it.
2. Run it with the warehouse CLI (reference.md §8) and compare against the numbers the source displays — Tableau: the screenshot values (or view data via MCP); Looker: the `run/json` rows saved in Phase 1; Power BI: the `executeQueries` rows saved in Phase 1. Match to within rounding.
3. If numbers differ, debug **here** — never ship an unvalidated formula to Hex. Check these classes of cause:
   - row/entity exclusions (internal IDs, test accounts) applied upstream in Tableau;
   - business-calendar logic vs calendar days (fiscal quarters, workday/holiday calendars);
   - metric bundling in the source model (one category silently folded into another);
   - fields duplicated across a finer grain that need `MAX()` not `SUM()`;
   - per-metric completeness lags ("bake") — a metric complete only through `latest_snapshot - N`;
   - timezone of date boundaries (UTC vs local);
   - source snapshot staleness (Power BI import mode): the model's rows are from its **last refresh**, the warehouse is live — compare against data as of the refresh time, and record the difference under the fidelity policy.
4. Keep each validated query in the run directory — they go verbatim into the Hex build prompt.

## Phase 3 — Build the Hex app

Use the Hex agent, not manual cell authoring (browser automation into Hex's editors is unreliable and `hex project import` regenerates all cell IDs).

**Target app type: Hex Generative app** (App builder → Generative app — a React app Hex renders client-side), NOT a classic notebook+app. Generative apps give pixel-level layout control (custom KPI cards, formula rows, hover tooltips, side-by-side groupings) that classic app builder cells cannot express, and the whole Phase 4 loop (genAppFiles code reads, iframe screenshots) assumes one. `hex thread create` has **no app-type flag** — the build prompt's wording is the *only* control, so the prompt must demand a Generative app explicitly (the template in reference.md §6 opens with this) and Step 4 below verifies Hex complied.

1. **Write one comprehensive build prompt** to a file (template: reference.md §6). It must open by demanding a **Generative app**, and include: every data source with exact schema.table and column names, the validated SQL per section, exact titles/subtitles/labels, hex color codes, layout (rows/columns of cards and charts), tooltip contents, number formats, filter parameters with their option lists and jinja wiring, and dark/light-mode readability. If the spec scopes multiple dashboards as tabs of one report, demand **tab navigation** with one tab per dashboard (names and order verbatim) and list each tab's sections separately.
2. **Create**: `hex thread create --new-project "$(cat prompt.txt)" --json` → record `thread_id` and project URL **in the manifest**.
3. **Poll in the foreground** (reference.md §7): `sleep 250` then `hex thread get <thread_id> --json`, repeat. Runs take 4–10 minutes. Do not background the poll — backgrounded `hex` calls lose PATH, and on macOS also Keychain auth.
4. **Verify the app type before any parity work**: `hex project export <project_id> -o app.yaml` and check for a non-empty `genAppFiles` list. If it's missing, Hex built a classic notebook+app — do NOT start Phase 4 on it. Send `hex thread continue <thread_id> "You built this as a classic notebook app. Rebuild it as a Generative app (App builder → Generative app): move the entire dashboard into the generative app, keeping the SQL cells as data sources. Do not change any queries."` and re-verify after the thread goes IDLE.
5. **If status is `ERROR`**: the run died partway. Send `hex thread continue <thread_id> "Your previous run errored partway. Verify and complete: <bullet list>"` — partial changes usually landed.

## Phase 4 — Self-check loop (repeat until parity)

The loop is driven by the **screenshot diff** and runs without user involvement:

```
repeat:
  1. screenshot Hex (headless, reference.md §10) + re-download the source PNG (Tableau view-image / Looker render task / Power BI exportToFile-or-Playwright)
  2. diff the image pairs panel-by-panel (+ steps 1–3 below: YAML, numbers, filters)
  3. zero discrepancies (or only documented trade-offs)? -> exit loop, go to Phase 5
  4. send ONE fix batch via `hex thread continue`, wait for the thread to go IDLE
  (re-screenshot — never assume a fix landed; verify it in pixels)
```

1. **Export the app**: `hex project export <project_id> -o app.yaml`. Grep/py-inspect the YAML for: exact titles and labels, color codes, SQL logic, filter options, tooltip fields, formatting helpers. This is faster and more reliable than browser inspection.
2. **Verify numbers**: re-run the app's SQL (copied from the YAML) via the warehouse CLI and compare with the source's values again — the Hex agent sometimes "simplifies" a query.
3. **Verify filters/parameters are real AND actually change results** — *skip this step entirely if Phase 1 recorded "no filters/parameters"*. Every source filter/slicer/parameter must be a Hex INPUT with the same options/default, referenced via jinja in the SQL — but grepping `{{ param }}` only proves it's *present*, not that it *works*. Run a functional diff for each one (harness: reference.md §9):
   - **Value filters** (categorical, date range, rolling window): render the cell's SQL with (a) the default and (b) a specific value using the real Jinja2 engine (Hex uses Jinja — do NOT hand-strip `{% if %}` blocks, that silently corrupts the query), run both via the warehouse CLI, and assert the output **changes in the right direction**: a narrower filter returns fewer rows, and every returned row matches the filter. If (a) and (b) are identical, the param is dead — send a fix.
   - **Branch/toggle params** (a dimension selector, a measure switch): these live inside `{% if param == '...' %}` branches, so confirm each option renders a **different, valid** query (different group-by column or measure) — not just the default branch. Test at least two option values per toggle.
   - **End-to-end option**: `hex project run <project_id> -i <param>=<value> --json` runs the logic with an overridden input; use it to confirm the app materializes without error under a non-default filter.
   Watch for the classic failure: a display-only dropdown that drives nothing, or a panel pointed at a source that lacks the filter's column (it silently won't respond).
4. **Verify visually — screenshot BOTH sides yourself and compare the images** (this is the primary parity check; full pipeline: reference.md §10):
   - **Source PNG**: Tableau — REST view-image endpoint (`/views/<luid>/image?resolution=high&maxAge=1`); Looker — dashboard render task (reference.md §11); Power BI — exportToFile or Playwright (reference.md §12). Same call as Phase 1's visual-reference step; re-download fresh each round.
   - **Hex PNG**: screenshot the app with Playwright using a persistent profile (`~/.hex-screenshot-profile`); browser per OS: macOS system Chrome via `channel="chrome"`, Linux plain Chromium (reference.md §10). The FIRST time, launch headed and have the user log in once; the session persists, so every later round is fully headless and automatic. (Headless-only host, e.g. remote Linux/CI: do the one-time login on a machine with a display and copy the profile directory over — reference.md §10.) If the app is unpublished, the `/app/<id>/latest` URL shows a lock screen — open the draft editor URL and click **App builder** instead. Capture the **FULL app as one stitched PNG**: scroll the app's inner scroll container inside its iframe in overlapping steps, crop each tile to the iframe (drops the editor chrome), drop the sticky header band from tiles 2+, and stitch — viewport screenfuls are NOT enough (panels hide at screenful boundaries and get mis-flagged as missing; `full_page=True` and giant viewports both fail on Hex). **Multi-tab apps**: repeat the capture once per tab (click the tab inside the iframe first), and pair each tab's PNG with its own source dashboard's PNG (each Tableau tab is its own view LUID; each Power BI page exports separately).
   - **Compare**: slice the tall source PNG into screenful-sized crops, then read the Hex and source image pairs side by side and diff every panel — charts against the plotted-series checklist, tables/heatmaps against the tabular checklist (both below). Seeing the rendered pixels is the only check that reliably catches transform/semantics mismatches the SQL and code diffs miss (e.g. a cumulative line vs a per-day line: same data, same labels, totally different picture).
   - The `genAppFiles` code read (React `App.js`, `lib/theme.js`) is still useful to trace WHICH cell/column a wrong panel reads once the screenshot shows a discrepancy — but it is a debugging aid, not the verification.
5. **Send fixes**: `hex thread continue <thread_id> "<numbered fix list>"`. Keep each round to one coherent batch; be surgical ("change X to Y", with exact strings/colors/values) and always end with "do not change anything else".
6. **Clean up orphans**: after replacing a section, check the YAML for SQL cells whose `resultVariableName` is no longer referenced and have the agent delete them.

**Discrepancy scan (run this every round, then auto-fix what doesn't match).** The scan and the parity checklist below say "Tableau" / "worksheet" / "twb" for brevity — read them as *your source* / *its panels (tiles, visuals)* / *its ground-truth artifact* (twb XML, Looker query body + `vis_config`, PBIR `visual.json`). Explicitly walk each axis below, list the deltas, and — unless a delta is a deliberate, documented trade-off or covered by the fidelity policy — send the fix to the agent automatically rather than asking the user:

```
- Layout: is every Tableau worksheet on the dashboard present, in the same section order and side-by-side grouping? (Compare the screenshot pairs; agents commonly DEFER dense panels — cohort triangles, latency matrices — silently.)
- Plotted series — CHART panels (judge ONLY from what is rendered, never from titles/labels/annotations): for EVERY chart, extract these fields from BOTH screenshots and require them to be equal:
    1. chart type + marks (line/area/bar, stacking, reference lines, negative handling)
    2. x-axis: dimension, range, and tick format (calendar date vs day-of-month vs cohort index vs category)
    3. y-axis: measure unit and tick range ($ vs % vs count vs ratio)
    4. series: count, grouping/legend, and per-series color
    5. curve geometry: monotonic climb / flat oscillation / smoothness / boundedness / resets — same data under a different transform (raw, cumulative, rolling, share-of-total, index, %-change) produces visibly different geometry, and the axes + geometry are the only proof of which one is plotted
    6. headline + endpoint values
  Titles, legends, and static annotations can all match Tableau while the plotted series is wrong — any field above differing means the panel needs a fix, even if every label agrees.
- Tabular & heatmap panels (tables, crosstabs, highlight tables, cohort triangles) — the parallel checklist:
    1. fill scale: stepped vs continuous, number of steps, palette, and direction
    2. scale domain and centre (diverging midpoint at 0? at the mean? fixed range?)
    3. text-colour rules on filled cells, incl. auto-inversion on dark fills (unreadable text = fail)
    4. row/column headers verbatim, and their sort order
    5. empty vs null vs zero cell treatment (blank / '-' / 0 render differently and mean different things)
    6. shape: cohort triangle vs full rectangle (future/unbaked cells blanked, not zero-filled)
    7. all columns visible without horizontal scrolling at app width
    8. cell number formats and totals/subtotal rows
- Numbers: re-run each panel's SQL and compare to Tableau; flag any metric-column mismatch, denominator scope, or timezone (UTC vs local) that shifts values.
- Parameters: does every Tableau parameter exist as a Hex input with the same options/default? (Agents drop toggles like dimension selectors and measure switches — rebuild them and wire them, don't hardcode one branch.) Skip if Phase 1 recorded none.
- Filters: is each filter actually wired into EVERY panel Tableau applies it to? Watch for a panel pointed at a source that lacks the filter's dimension (it silently won't respond) — reconcile or document the trade-off. Skip if Phase 1 recorded none.
- Unbaked periods: does the app exclude today / partial days and apply each metric's bake lag exactly as Tableau does?
- Formats & colors: number formats and hex codes match the twb (don't eyeball).
```

Parity checklist to satisfy before sign-off:

```
- [ ] Every number matches Tableau (same filters applied) to within rounding
- [ ] Every chart passes the plotted-series checklist (marks, x-axis, y-axis unit, series, curve geometry, endpoints)
- [ ] Every table/heatmap passes the tabular checklist (fill scale, domain/centre, text colour, headers, empty-cell treatment, shape, column fit)
- [ ] All filters/parameters present, same options, same defaults, actually wired into every panel they drive (or "none" recorded in the spec)
- [ ] Every Tableau worksheet is present (no silently deferred panels), same section order/grouping
- [ ] Section titles, subtitles, axis labels, legends match verbatim
- [ ] Colors match (pull exact codes from the twb, don't eyeball)
- [ ] Tooltips exist wherever Tableau has them, same fields and labels
- [ ] Time ranges and "as of" dates match, incl. per-metric bake lags and Exclude-Today
- [ ] Number formats match
- [ ] Layout matches: section order, side-by-side groupings, alignment
- [ ] Multi-tab scope: every tab present, names and order verbatim, each tab diffed against its own source dashboard
- [ ] Readable in both dark and light Hex themes
```

## Phase 5 — Parity sign-off (evidence pack)

Done is defined by user acceptance of an evidence pack, not by the agent declaring victory. Assemble and present:

1. **Side-by-side images** from the final Phase 4 round: the stitched Hex PNG next to the source PNG (or aligned crop pairs for tall dashboards).
2. **Per-panel parity table**: one row per source panel (worksheet / tile / visual) — panel name | numbers match (Y/N + checked value) | visual match (Y/N) | notes.
3. **Approximations and trade-offs**, explicitly listed: anything that cannot be replicated (precondition items), fidelity-policy decisions taken (inconsistencies replicated or fixed), fonts/spacing differences inherent to Hex.
4. **Open questions** for the user (ambiguous source behavior, panels that need a business judgment).
5. **Links and handles**: Hex draft URL, `project_id`, `thread_id` (for future fix rounds), and the run-manifest path.
6. **Post-migration reminders**: rotate/revoke the source credentials (Tableau PAT / Looker API key / Entra session); publishing, scheduling, and permissions are out of scope — point the user to Hex's publish flow.

## Hard-won gotchas

**Hex tooling**

- **The Hex agent silently defaults to a classic notebook+app** if the prompt doesn't demand a Generative app — there is no CLI flag for app type (`hex thread create`/`app run` can't force it), so it must be in the prompt text, and `genAppFiles` in the export YAML is the proof it happened (Phase 3 step 4).
- **Agent threads error transiently** — resend with `hex thread continue`; work usually partially landed. Export the YAML to see what survived.
- **`hex project import` regenerates every cell ID** — never cache cell IDs across an import; re-list with `hex cell list`.
- **Poll threads in the foreground** — backgrounded `hex` calls lose PATH, and on macOS also Keychain access (auth silently fails).
- **Screenshots (reference.md §10)**: on macOS, playwright's own downloaded Chromium gets its GPU process SIGKILLed (Gatekeeper) — always `channel="chrome"`; on Linux, plain Chromium works (same scroll-and-stitch logic). Hex keeps websockets open so `networkidle` never fires — use `domcontentloaded` + poll the `canvas/svg` count. `full_page=True` captures only the viewport and a giant viewport renders just a spinner — scroll the iframe's inner container and stitch tiles (handle the sticky header, it overwrites content at stitch boundaries). Unpublished apps show a lock screen on `/app/<id>/latest` — use the draft editor + App builder tab.

**Tableau extraction**

- **Tooltips don't appear in any API/image export** — get them from user screenshots and replicate as hover tooltips (scrollable if long).
- **Metadata API vs twb**: the GraphQL Metadata API gives custom SQL + upstream tables quickly, but only the .twb XML has worksheet-level table calcs, dashboard layout, and color encodings.
- **Don't pin the REST API version** — read it from the unauthenticated `serverinfo` endpoint (reference.md preamble).

**Looker extraction**

- **`run/sql` omits the post-SQL layers**: table calcs and custom fields (`dynamic_fields`), pivots, and row totals are applied client-side after the SQL returns. A tile can pass the SQL/number check and still render wrong — always fold `dynamic_fields` into the Hex SQL (window functions) or chart config, and let the screenshot diff catch what slipped.
- **UDD vs LookML dashboards**: numeric IDs vs `model::slug` — same endpoints, different element addressing. A LookML dashboard's source of truth is a YAML file in the LookML repo; prefer reading it when you have repo access.
- **Merged-results tiles** have `merge_result_id` instead of `query_id` — extract every source query and replicate the merge as a SQL join.
- **Render tasks are async**: create → poll `/render_tasks/<id>` until `success` → download `/results`. A 202 from `/results` means still rendering, not failure.
- **Filters are dashboard-level with per-tile listen mappings** — a tile that doesn't listen to a filter is the Looker version of Tableau's "panel the filter silently doesn't drive"; read the mapping instead of assuming every filter hits every tile.
- **API permissions mirror UI permissions**: if the API user can't see a folder/explore, the SQL and data endpoints return errors — fix access first, don't debug the calls.

**Power BI extraction**

- **DAX→SQL is the cost center, not extraction**: translate and validate one measure at a time against `executeQueries`; a measure that's "roughly right" fails silently in every visual that uses it.
- **PBIR vs PBIR-Legacy**: new format is a folder of per-visual JSON files; legacy is one `report.json` where each visual's `config` is a JSON **string** inside the JSON — parse twice.
- **Implicit measures**: visuals can aggregate bare columns ("Sum of X") with no measure in the model — the aggregation lives in `visual.json`'s field bindings, not TMDL.
- **Auto date/time**: hidden `LocalDateTable_*` hierarchies make Year/Quarter/Month drill-downs work without a date table — replicate with date functions, don't hunt for a table that doesn't exist.
- **RLS changes the truth**: `executeQueries` under a user enforces row-level security; under a service principal it doesn't. Validate under the same identity whose numbers the user expects, and record which.
- **Licensing walls surface mid-run**: `getDefinition` needs report *write* permission; XMLA/`exportToFile` need capacity/PPU; `executeQueries` needs its tenant setting. Hit them in Phase 0, not Phase 4 — and the Playwright screenshot path (§10) is the capacity-free fallback for visual reference.
- **Custom tooltip pages are real pages** (hidden, tooltip-flagged in PBIR) — extract and replicate them; only default hover tooltips need user screenshots.

**Data semantics**

- **Negative series plotted as absolute values**: keep labels/tooltips signed if Tableau does.
- **Axis scaling**: Tableau usually auto-scales; explicitly tell Hex not to force zero-based axes.
- **Duplicated-grain fields** (a monthly value repeated per row) need `MAX()` not `SUM()` — cross-check magnitudes against Tableau before shipping.
- **Per-metric bake lags and business calendars** are the most common source of "numbers almost match" — pull them from the dbt models, not from guesses.

## Additional resources

- All commands, XML-parsing snippets, prompt template, and MCP config: [reference.md](reference.md)
- Human-facing overview (install, trigger phrases, expectations): [README.md](README.md) — point the user here if they ask how the skill works
