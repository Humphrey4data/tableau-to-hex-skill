# Tableau → Hex Dashboard Migration Skill

An agent skill that migrates a Tableau dashboard to a [Hex](https://hex.tech) app with **matching queries, numbers, filters, colors, layout, and tooltips** — then proves the match with an automated screenshot-diff loop.

This file is for humans. The agent follows [SKILL.md](SKILL.md) (the procedure) and [reference.md](reference.md) (the commands).

## What it does, step by step

| Phase | What happens | Your involvement |
|---|---|---|
| 0. Setup | Installs/verifies the Hex CLI and a warehouse CLI, checks Tableau access, asks the fidelity policy, creates a resumable run manifest | Provide a Tableau PAT if none is configured; answer one policy question |
| 1. Extract | Downloads the Tableau workbook (.twbx) via the REST API and parses its XML for the ground-truth custom SQL, calculated fields, parameters, filters, layout, and colors; grabs rendered PNGs | Send screenshots of hover tooltips (they're invisible to every API) |
| 2. Validate | Re-runs every section's logic against the warehouse and reconciles with Tableau's numbers **before** building anything | None (agent asks only if the source is ambiguous) |
| 3. Build | Sends one comprehensive prompt to the Hex agent (`hex thread create --new-project`) with the validated SQL and exact styling spec | None — takes ~10–15 min |
| 4. Self-check loop | Screenshots the Hex app headlessly, diffs it panel-by-panel against the Tableau PNG, re-verifies numbers and that every filter actually changes results, sends fix batches to the Hex agent; repeats until parity | One-time browser login for screenshots; expect 3+ rounds |
| 5. Sign-off | Presents an evidence pack: side-by-side images, per-panel parity table, explicit approximations, open questions | You accept or request changes |

## How to trigger it

The skill activates when you paste a Tableau view link and ask for a migration. In a chat with your agent (Cursor, Claude Code, etc.):

> Migrate this Tableau dashboard to Hex: `https://<pod>.online.tableau.com/#/site/<site>/views/<Workbook>/<Dashboard>`

Phrases like "migrate/replicate/copy/rebuild this Tableau dashboard in Hex" or "Tableau to Hex" all work. You can also invoke it explicitly (e.g. `@tableau-to-hex-migration` in Cursor).

## Installation

Copy this folder into your project's skills directory:

- **Cursor**: `.cursor/skills/tableau-to-hex-migration/` (project) or `~/.cursor/skills/...` (personal)
- **Claude Code**: `.claude/skills/tableau-to-hex-migration/`

## Prerequisites

- **Hex workspace** with the CLI enabled and the **Hex agent (threads)** feature available — Phase 3 consumes workspace AI credits; without this feature the build path is blocked.
- **Tableau**: a Personal Access Token with permission to download the workbook. Tableau Cloud and self-hosted Server (2019.4+) both work.
- **Warehouse CLI** for the engine your Hex data connection uses: Snowflake `snow`, BigQuery `bq`, Databricks `dbsqlcli`, Postgres/Redshift `psql` (any JSON-emitting SQL CLI can be plugged in).
- **macOS or Linux** (Windows via WSL). Python 3 with `playwright`, `pillow`, `pyyaml`, `jinja2` for the self-check loop.

## What to expect

- **It is a multi-hour, supervised task**, not one-shot magic: ~10–15 min initial build, then several 4–10 min correction rounds while the agent converges on parity.
- **Human touchpoints are few but real**: tooltip screenshots, a one-time Hex browser login, the fidelity-policy answer, and final visual sign-off.
- **Known limits** (flagged up front by the skill): Tableau Stories, dashboard actions, row-level security, and user filters have no clean Hex equivalent; refresh schedules, permissions, and publishing are out of scope. The deliverable is a draft Hex app plus a parity evidence pack.

## Safety notes

- The Tableau PAT lives only in your MCP config or session env vars; the skill never writes it to files or prompts, and reminds you to rotate it afterward.
- All run artifacts (spec, SQL, screenshots, manifest) live under `~/.tab2hex/<dashboard-slug>/`, so an interrupted migration is resumable.
