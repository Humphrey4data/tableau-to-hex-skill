# Tableau/Looker/Power BI → Hex Dashboard Migration Skill

An agent skill that migrates a Tableau, Looker, or Power BI dashboard to a [Hex](https://hex.tech) app with **matching queries, numbers, filters, colors, layout, and tooltips** — then proves the match with an automated screenshot-diff loop.

This file is for humans. The agent follows [SKILL.md](SKILL.md) (the procedure) and [reference.md](reference.md) (the commands).

Source support:

| Source | Status | Notes |
|---|---|---|
| **Tableau** (Cloud + self-hosted Server) | Battle-tested | Proven on a production migration |
| **Looker** (Looker-hosted + customer-hosted) | Documented equivalent | Easier in one way: the API returns the exact warehouse SQL behind every tile |
| **Power BI** (Fabric / Premium / PPU best) | Documented equivalent, hardest source | Extraction is clean (PBIR + TMDL via APIs), but DAX measures must be translated to warehouse SQL and proven equivalent; import-mode models without a reachable upstream warehouse can't be migrated at all |

## What it does, step by step

| Phase | What happens | Your involvement |
|---|---|---|
| 0. Setup | Installs/verifies the Hex CLI and a warehouse CLI, checks source BI access, asks the fidelity policy, suggests command allowlisting, front-loads the one-time logins, creates a resumable run manifest | Provide source credentials (Tableau PAT / Looker API key / Entra login) if not configured; answer one policy question; do the logins |
| 1. Extract | Pulls the ground truth from the source — Tableau: downloads the workbook (.twbx) and parses its XML for custom SQL, calculated fields, parameters, filters, layout, colors; Looker: fetches the dashboard, per-tile generated SQL, table calcs, filters, and vis config via the API; Power BI: fetches the report definition (PBIR) and semantic model (TMDL: DAX measures, relationships, M sources) via the Fabric APIs — and grabs rendered PNGs | Send screenshots of hover tooltips (they're invisible to every API); confirm the scope — which dashboards/pages to migrate, and whether tabbed siblings become one Hex app with tabs (default) or separate apps |
| 2. Validate | Re-runs every section's logic against the warehouse and reconciles with the source's numbers **before** building anything (for Power BI this includes the DAX-to-SQL translation) | None (agent asks only if the source is ambiguous) |
| 3. Build | Sends one comprehensive prompt to the Hex agent (`hex thread create --new-project`) with the validated SQL and exact styling spec, targeting a **Generative app** (not a classic notebook app, with tab navigation when the scope covers multiple dashboards) and verifying Hex complied | None — takes ~10–15 min |
| 4. Self-check loop | Screenshots the Hex app headlessly, diffs it panel-by-panel against the source PNG, re-verifies numbers and that every filter actually changes results, sends fix batches to the Hex agent; repeats until parity | One-time browser login for screenshots; expect 3+ rounds |
| 5. Sign-off | Presents an evidence pack: side-by-side images, per-panel parity table, explicit approximations, open questions | You accept or request changes |

## How to trigger it

The skill activates when you paste a Tableau view link, a Looker dashboard link, or a Power BI report link and ask for a migration. In a chat with your agent (Cursor, Claude Code, Codex, etc.):

> Migrate this Tableau dashboard to Hex: `https://<pod>.online.tableau.com/#/site/<site>/views/<Workbook>/<Dashboard>`

> Migrate this Looker dashboard to Hex: `https://<instance>.cloud.looker.com/dashboards/<id>`

> Migrate this Power BI report to Hex: `https://app.powerbi.com/groups/<workspace>/reports/<report>/<page>`

Phrases like "migrate/replicate/copy/rebuild this dashboard in Hex", "Tableau to Hex", "Looker to Hex", or "Power BI to Hex" all work. You can also invoke it explicitly (e.g. `@tableau-to-hex-migration` in Cursor).

## Installation

Copy this folder into your project's skills directory:

- **Cursor**: `.cursor/skills/tableau-to-hex-migration/` (project) or `~/.cursor/skills/...` (personal)
- **Claude Code**: `.claude/skills/tableau-to-hex-migration/`
- **Codex**: `.codex/skills/tableau-to-hex-migration/` (project) or `~/.codex/skills/...` (personal)

## Prerequisites

- **Hex workspace** with the CLI enabled and the **Hex agent (threads)** feature available — Phase 3 consumes workspace AI credits; without this feature the build path is blocked.
- **Tableau** (if migrating from Tableau): a Personal Access Token with permission to download the workbook. Tableau Cloud and self-hosted Server (2019.4+) both work.
- **Looker** (if migrating from Looker): API3 credentials (client ID + secret) for a user with access to the dashboard's folder and explores. Looker-hosted and customer-hosted both work.
- **Power BI** (if migrating from Power BI): a Microsoft Entra login (via Azure CLI) with Build permission on the semantic model and write access to the report; the `executeQueries` tenant setting enabled; Fabric capacity or Premium-Per-User strongly recommended (needed for TMDL export via API/XMLA and PNG export — without it the skill falls back to browser screenshots). DirectQuery/composite models migrate best; import-only models need their upstream warehouse tables to exist.
- **Warehouse CLI** for the engine your Hex data connection uses: Snowflake `snow`, BigQuery `bq`, Databricks `dbsqlcli`, Postgres/Redshift `psql` (any JSON-emitting SQL CLI can be plugged in).
- **macOS or Linux** (Windows via WSL). Python 3 with `playwright`, `pillow`, `pyyaml`, `jinja2` for the self-check loop.

## What to expect

- **It is a multi-hour, supervised task**, not one-shot magic: ~10–15 min initial build, then several 4–10 min correction rounds while the agent converges on parity.
- **Human touchpoints are few but real**: tooltip screenshots, a one-time Hex browser login, the fidelity-policy answer, the dashboard-scope answer (which dashboards/pages, tabs vs separate apps), and final visual sign-off.
- **Reduce "Allow" clicks up front**: the run issues hundreds of terminal commands, so at the start the agent will suggest (a) adding the repeated CLIs (`curl`, `python3`, `hex`, your warehouse CLI) to your agent's command allowlist — in Cursor via the terminal allowlist settings, in Claude Code via `/permissions` allow rules — and (b) doing all one-time browser logins (Hex CLI, screenshot profile, `az login`) back-to-back in Phase 0. Ten minutes of setup buys hours of unattended running; only allowlist these specific read/query commands, never destructive ones.
- **Known limits** (flagged up front by the skill): Tableau Stories, dashboard actions, row-level security, and user filters have no clean Hex equivalent — likewise Looker data actions and alerts, and Power BI RLS roles, drill-through actions, and bookmark navigation; refresh schedules, permissions, and publishing are out of scope. The deliverable is a draft Hex app plus a parity evidence pack.
- **Power BI takes longest**: DAX-to-SQL translation is validated measure by measure — budget roughly twice the correction rounds of a Tableau/Looker run.

## Safety notes

- The Tableau PAT / Looker API secret / Entra token lives only in your MCP config or session env vars; the skill never writes it to files or prompts, and reminds you to rotate it afterward.
- All run artifacts (spec, SQL, screenshots, manifest) live under `~/.tab2hex/<dashboard-slug>/`, so an interrupted migration is resumable.
