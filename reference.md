# Tableau → Hex Migration: Command Reference

Snippets are bash/zsh (macOS or Linux; OS-specific branches are labeled where they exist). Set these once per session (Tableau values from your agent's Tableau MCP config if one exists — locations in §5 — or from the user; server/site come from the dashboard URL `https://<server>/#/site/<site>/views/...`, with no `/site/` segment on a Tableau Server default site):

```bash
export RUN_DIR=~/.tab2hex/<dashboard-slug>             # run directory (SKILL.md Phase 0); mkdir -p "$RUN_DIR"
export TAB_SERVER="https://<pod>.online.tableau.com"   # Tableau Cloud pod, OR self-hosted Server URL (https://tableau.acme.com)
export TAB_SITE="<site>"                               # site contentUrl; "" (empty) for a Tableau Server default site
export TAB_PAT_NAME="..."                              # PAT name (Tableau Server needs 2019.4+ for PATs)
export TAB_PAT_SECRET="..."                            # PAT secret — env only, never in files (SKILL.md Phase 0)
# REST API version: read it from the server, don't pin it (it advances; a stale pin looks like an auth bug)
export TAB_API=$(curl -s "$TAB_SERVER/api/2.4/serverinfo" -H "Accept: application/json" \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['serverInfo']['restApiVersion'])")
```

Warehouse branch — set `WH` to the engine of the **Hex data connection** (not your personal preference; validation must run where Hex runs), smoke-test it, and record it in the manifest:

```bash
# Snowflake:
export WH=snowflake; export SNOW_CONN="..."      # `snow connection list`; smoke: snow sql -c "$SNOW_CONN" -q "select 1"
# BigQuery:
export WH=bigquery                                # gcloud auth login; smoke: bq query --nouse_legacy_sql 'select 1'
# Databricks SQL:
export WH=databricks                              # dbsqlcli configured (or Statement Execution API); smoke: dbsqlcli -e 'select 1'
# Postgres / Redshift:
export WH=postgres; export PGURI="postgres://..." # smoke: psql "$PGURI" -c 'select 1'
# Other engines (Trino, Athena, DuckDB/MotherDuck, ...): same pattern — any CLI that takes a SQL
# string and prints JSON rows; add your command to §8 and to the CMDS table in §9.
```

Dialect rule: the twb custom SQL is **already** in this engine's dialect (it ran against this warehouse in Tableau) — never translate it, and write every validation query and the Hex build prompt in the same dialect.

## 1. REST API sign-in

Build the payload from env with python (keeps the secret off the `curl` argv — terminal transcripts capture full command text):

```bash
cd "$RUN_DIR"   # the ~/.tab2hex/<slug>/ run directory from SKILL.md Phase 0
python3 -c "
import json, os
json.dump({'credentials': {'personalAccessTokenName': os.environ['TAB_PAT_NAME'],
                           'personalAccessTokenSecret': os.environ['TAB_PAT_SECRET'],
                           'site': {'contentUrl': os.environ['TAB_SITE']}}}, open('signin.json','w'))"
curl -s -X POST "$TAB_SERVER/api/$TAB_API/auth/signin" \
  -H "Content-Type: application/json" -H "Accept: application/json" \
  -d @signin.json > tab_auth.json && rm signin.json
export TOKEN=$(python3 -c "import json;print(json.load(open('tab_auth.json'))['credentials']['token'])")
export SITEID=$(python3 -c "import json;print(json.load(open('tab_auth.json'))['credentials']['site']['id'])")
```

Error `401001` = PAT invalid/expired → user must regenerate the PAT. Note the secret may contain a leading `/` — quote it carefully. When the migration is done, remind the user to revoke/rotate the PAT.

Tableau Cloud vs self-hosted Server: the sign-in, `serverinfo`, workbook download, and image endpoints are identical on both — only `TAB_SERVER`/`TAB_SITE` differ (Server default site uses an empty `contentUrl`). If a Server admin has disabled PATs, fall back to `{"credentials":{"name":..., "password":..., "site":{...}}}` in the same payload.

Recovery when `401001` (or the Tableau MCP server is in an `error` state):

1. Ask the user to regenerate: Tableau Cloud → avatar → **My Account Settings** → **Personal Access Tokens** → name it (reuse the existing `PAT_NAME`) → **Create Token** → copy the secret (shown once). Users typically paste only the secret — keep the existing `PAT_NAME`.
2. If a Tableau MCP server is configured, update `PAT_VALUE` in its `tableau` block with the new secret (verbatim, quoted — it often starts with `/` and contains `==:`); config location per agent in §5. Then re-export `TAB_PAT_SECRET` for this shell session.

```bash
python3 - <<'EOF'
import json, pathlib
p = pathlib.Path.home() / ".cursor" / "mcp.json"   # <- your agent's MCP config path (see §5)
cfg = json.loads(p.read_text())
cfg["mcpServers"]["tableau"]["env"]["PAT_VALUE"] = "<NEW_SECRET>"   # verbatim
p.write_text(json.dumps(cfg, indent=2) + "\n")
print("updated", cfg["mcpServers"]["tableau"]["env"]["PAT_NAME"])
EOF
export TAB_PAT_SECRET="<NEW_SECRET>"
```

3. Re-run the sign-in above and confirm a `token` comes back before continuing. The MCP server only re-reads `PAT_VALUE` after the agent app reloads, so drive Phase 1 through the REST API with the exported `TAB_PAT_SECRET`.

## 2. Find and download the workbook

The dashboard URL `.../#/site/<site>/views/<WB_CONTENT_URL>/<VIEW>` gives the workbook `contentUrl`.

```bash
# workbook LUID by contentUrl
curl -s "$TAB_SERVER/api/$TAB_API/sites/$SITEID/workbooks?pageSize=1000" \
  -H "X-Tableau-Auth: $TOKEN" -H "Accept: application/json" > wbs.json
python3 - <<'EOF'
import json
TARGET = "<WB_CONTENT_URL>"   # the /views/<THIS>/... segment of the dashboard URL
for wb in json.load(open('wbs.json'))['workbooks']['workbook']:
    if wb['contentUrl'] == TARGET:
        print(wb['id'], wb['name'])
EOF

# download .twbx and extract the .twb XML
curl -s "$TAB_SERVER/api/$TAB_API/sites/$SITEID/workbooks/$WB/content" \
  -H "X-Tableau-Auth: $TOKEN" -o "$RUN_DIR/wb.twbx"
mkdir -p "$RUN_DIR/wb_x" && cd "$RUN_DIR/wb_x" && unzip -o "$RUN_DIR/wb.twbx"
# a .twb may download directly (not zipped) — check with `file "$RUN_DIR/wb.twbx"`
```

## 3. Parse the .twb XML

Custom SQL per datasource (the ground-truth queries):

```python
import xml.etree.ElementTree as ET
root = ET.parse("wb.twb").getroot()
for ds in root.iter('datasource'):
    for rel in ds.iter('relation'):
        if rel.get('type') == 'text' and rel.text and 'select' in rel.text.lower():
            print('=== DS:', ds.get('caption') or ds.get('name'))
            print(rel.text[:2000])
```

Calculated fields (incl. worksheet-level table calcs — only visible here):

```python
for ds in root.iter('datasource'):
    for col in ds.iter('column'):
        calc = col.find('calculation')
        if calc is not None and calc.get('formula'):
            print(col.get('caption') or col.get('name'), '=', calc.get('formula'))
```

Dashboard → worksheet mapping, parameters, filters, colors:

```python
# sheets on each dashboard, in layout order
for dash in root.iter('dashboard'):
    print('DASHBOARD:', dash.get('name'))
    for z in dash.iter('zone'):
        if z.get('name'): print('  sheet:', z.get('name'))

# parameters live in the 'Parameters' datasource: <column> with <members>/range
# filters: <filter> elements under each <worksheet>
# colors: <style-rule element='mark'> / <encoding attr='color'> under worksheets
```

Metadata API (GraphQL) alternative for custom SQL + lineage (no table calcs):

```bash
curl -s "$TAB_SERVER/api/metadata/graphql" -H "X-Tableau-Auth: $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query":"{ customSQLTablesConnection(filter: {}) { nodes { name query downstreamDatasources { name luid } } } }"}' > csql.json
```

## 4. Rendered images for visual reference

```bash
# view LUID: GET /sites/$SITEID/workbooks/$WB -> views list
curl -s "$TAB_SERVER/api/$TAB_API/sites/$SITEID/views/$VIEW_LUID/image?resolution=high" \
  -H "X-Tableau-Auth: $TOKEN" -o "$RUN_DIR/tableau_view.png"
```

Tooltips are not included in image exports — request screenshots from the user.

## 5. Tableau MCP (optional)

MCP server entry (enables live view-data queries from chat). Config location depends on the agent: Cursor `~/.cursor/mcp.json`; Claude Code project `.mcp.json` or `claude mcp add`; other agents per their docs. Same shape everywhere:

```json
"tableau": {
  "command": "npx",
  "args": ["-y", "@tableau/mcp-server@latest"],
  "env": {
    "SERVER": "https://<pod>.online.tableau.com",
    "SITE_NAME": "<site>",
    "PAT_NAME": "<pat name>",
    "PAT_VALUE": "<pat secret>"
  }
}
```

(Use an absolute `command` path — e.g. `/opt/homebrew/bin/npx` on Apple-silicon macOS — if the agent's launcher doesn't inherit your PATH.) This MCP config is the only sanctioned place (besides session env vars) to store the PAT — never write it into the repo, the spec, or the run manifest.

## 6. Hex build prompt template

Write to a file, then `hex thread create --new-project "$(cat prompt.txt)" --json`. Adapt this skeleton — the more exact the prompt, the fewer fix rounds. The first line pins the app type: there is no CLI flag for it, and without this demand the Hex agent usually builds a classic notebook+app (wrong target — see SKILL.md Phase 3):

```
Build this as a GENERATIVE APP (App builder -> Generative app), not a classic notebook app:
the entire dashboard must render inside the generative app, with SQL cells as its data sources.

Replicate our Tableau "<NAME>" dashboard. Use the "<Hex data connection name>"
data connection (<engine>); write all SQL in <engine>'s dialect.

DATA SOURCES (use these exact tables/columns):
1. <db.schema.table> — columns: <col list>. Notes: <grain, dedup rules, excluded IDs, bake lag>.
2. ...

GLOBAL: dark-and-light-mode readable; number formats <from the spec>; auto-scale all chart axes (do NOT force zero);
<business-calendar rules from the spec, if any — e.g. fiscal-quarter starts, workday calendars>.

HEADER: title "<...>"<, subtitle from the spec if Tableau shows one, e.g. a refresh timestamp>.
Filters (real input params wired into every SQL cell via jinja) — omit this block entirely if the spec recorded none:
- <param name>: options [...], default <...>
- ...

SECTION 1 — <title>:
<layout, metrics with formulas (paste validated SQL), labels verbatim, colors (#hex), tooltips (fields + labels), formats>

SECTION 2 — ...

TOOLTIPS: <per panel: trigger, fields, labels, format; make long lists scrollable>
```

Everything in `<>` comes from the Phase 1 spec — don't inherit another dashboard's calendar, header, or formats.

## 7. Hex CLI workflow

```bash
# create (record thread_id + project url in the run manifest immediately)
hex thread create --new-project "$(cat "$RUN_DIR/prompt.txt")" --json   # -> thread_id, project url

# poll (FOREGROUND only; backgrounded hex loses PATH, and on macOS also Keychain auth)
sleep 250; hex thread get $THREAD_ID --json | python3 -c "
import sys,json; d=json.load(sys.stdin)
print('status:', d.get('status'))
r=d.get('response') or ''
print(str(r)[-1400:] if r else '(working...)')"
# repeat until status IDLE (done) or ERROR (crashed)

# iterate
hex thread continue $THREAD_ID "<numbered fix list>... Do not change anything else." --json

# inspect result
hex project export $PROJECT_ID -o "$RUN_DIR/app.yaml"
# grep for labels/colors/SQL; check jinja param wiring; find orphaned SQL cells
```

### Inspect a Generative app's rendered React code (visual check without a screenshot)

The export YAML **must** contain a non-empty `genAppFiles` list — that is the proof the build produced a Hex **Generative app** (a React app rendered client-side), which is the required target (SKILL.md Phase 3 step 4; if it's missing, Hex built a classic notebook app — send the rebuild continuation before doing anything else). You can read the generative app's source to verify titles, section order, colors, chart types, and which SQL cell/column each chart reads. Dump the files, then read `App.js` (layout + wiring), `lib/theme.js` (color map), and `components/*` (axis/legend/tooltip behavior):

```python
import yaml, os
d = yaml.safe_load(open("app.yaml"))          # run from $RUN_DIR
os.makedirs("genapp", exist_ok=True)
for f in d.get("genAppFiles", []):
    open("genapp/" + f["path"].replace("/", "__"), "w").write(f.get("contents") or "")
```

`App.js` maps each chart to a data cell via `useHexData("<cellStaticId>")` and to an INPUT via `useHexInput("<inputStaticId>")`, so you can trace every panel back to its SQL and parameters. Still request a user screenshot for true pixel parity (fonts, spacing, rendered axis scaling).

Thread statuses: `RUNNING` (wait), `IDLE` (done — read `response`), `ERROR` (crashed — `hex thread continue` with "verify and complete" message; partial work usually landed).

Other useful commands: `hex project open <id>` (sets context for cell commands), `hex cell list`, `hex cell update <cell_id> -f file` (cell IDs change on every `hex project import`!), `hex run <project_id>`.

## 8. Warehouse validation pattern

Write long queries to a file first, then run them with the `WH` branch pinned in the preamble:

```bash
# Snowflake (most battle-tested branch)
snow sql -c "$SNOW_CONN" -f "$RUN_DIR/check_section1.sql" --format json | python3 -c "
import sys,json
for r in json.load(sys.stdin): print(r)"

# BigQuery
bq query --nouse_legacy_sql --format=json "$(cat "$RUN_DIR/check_section1.sql")"

# Databricks SQL
dbsqlcli -e "$RUN_DIR/check_section1.sql"        # or POST /api/2.0/sql/statements

# Postgres / Redshift
psql "$PGURI" -f "$RUN_DIR/check_section1.sql" --csv
```

Compare each output against the Tableau screenshot values. Only after they match, paste that SQL into the Hex prompt. When the Hex agent later edits SQL, re-export the YAML and re-validate the query it actually shipped — always through the same `WH` branch.

## 9. Functional filter/parameter test (prove filters actually change results)

**Skip this whole section if the Phase 1 spec recorded "no filters/parameters"** — a workbook with no `Parameters` datasource and no quick filters is a legitimate outcome, not something you missed.

Deps: `python3 -m pip install pyyaml jinja2`.

Hex SQL cells use **Jinja** with `{% if 'All' not in x %} ... {% endif %}` filter blocks and `{{ x | array }}` / `{{ scalar }}` substitutions. Render with the real Jinja2 engine — hand-stripping `{% %}` blocks or blindly regex-replacing `{{ }}` silently corrupts the WHERE clause (e.g. turns `segment in ({{ segment|array }})` into `segment in ('% of $')`) and gives false failures. Provide the `array` filter Hex uses (quote + comma-join a list):

```python
import yaml, subprocess, json, jinja2

def render(sql, params):
    env = jinja2.Environment()
    env.filters["array"] = lambda xs: ", ".join("'" + str(v).replace("'", "''") + "'" for v in (xs if isinstance(xs, list) else [xs]))
    return env.from_string(sql).render(**params)

import os
WH = os.environ.get("WH", "snowflake")
CMDS = {  # engine -> argv producing JSON rows on stdout (keep the retry loop for ANY engine)
    "snowflake":  lambda q: ["snow", "sql", "-c", os.environ["SNOW_CONN"], "-q", q, "--format", "json"],
    "bigquery":   lambda q: ["bq", "query", "--nouse_legacy_sql", "--format=json", q],
    "databricks": lambda q: ["dbsqlcli", "--output", "json", "-e", q],
    "postgres":   lambda q: ["psql", os.environ["PGURI"], "-t", "-A", "-c",
                             f"select json_agg(t) from ({q}) t"],
}

def run(sql):  # returns list of dict rows; retries transient warehouse errors
    import time
    for i in range(6):  # e.g. Snowflake intermittently throws 000904/001003 on IDENTICAL valid SQL
        p = subprocess.run(CMDS[WH](sql), capture_output=True, text=True)
        if p.returncode == 0 and p.stdout.strip().startswith(("[", "{")):
            return json.loads(p.stdout)
        time.sleep(3 + i)
    raise RuntimeError((p.stdout + p.stderr)[:200])

d = yaml.safe_load(open("app.yaml"))
src = next(c["config"]["source"] for c in d["cells"]
          if c["cellType"] == "SQL" and c["cellLabel"] == "<cell label>")

# build the param dicts from the app's actual INPUT cells (names, defaults, option lists
# all come from the YAML / Phase 1 spec — never invent them):
base = {"<param>": ["All"], ...}                 # every param at its default
one  = dict(base, **{"<param>": ["<option>"]})   # one param narrowed to a specific option

def val(rows):  # engines disagree on result-column casing (Snowflake "C", others "c")
    r = rows[0]; return int(r.get("C", r.get("c")))

n_all = val(run(f"select count(*) c from ({render(src, base)}) t"))
n_one = val(run(f"select count(*) c from ({render(src, one)}) t"))
bad   = val(run(f"select count(*) c from ({render(src, one)}) t where <filter_column> <> '<option>'"))
assert n_one < n_all and bad == 0, f"filter dead/leaky: all={n_all} narrowed={n_one} leak={bad}"
```

Assertions per filter type:
- **Value filter working** ⇔ narrower input ⇒ strictly fewer rows AND zero rows violate the filter. Identical counts ⇒ the param is dead.
- **Branch/toggle param** (`{% if <param> == '<option>' %}` etc.): render two option values and diff the produced SQL — the group-by column or measure must differ; then run both and confirm both return valid, differently-shaped results.
- **End-to-end**: `hex project run <project_id> -i <param>=<option> --json` (per hex skill) runs the logic with the override and must finish without error.

Gotchas: (1) wrap as `select count(*) c from (<rendered sql>) t` — most engines accept a trailing `order by` inside an aliased subquery, so do NOT try to regex-strip it (you'll mangle window-function `over(order by ...)`); BigQuery may warn about it — remove the *outermost* `order by` only if the engine rejects it. (2) keep the `run()` retry loop for any engine's transient errors (Snowflake's `000904`/`001003` on identical valid SQL is one instance). The shape of the assertions is what matters: strictly-fewer rows, zero leaked rows, and visibly different SQL per toggle branch.

## 10. Screenshot Hex yourself + visual diff vs Tableau (primary parity check)

You CAN screenshot the Hex app without the user: Playwright + the **system Chrome** + a persistent profile. One-time interactive login, headless forever after. This is what catches transform/shape mismatches that SQL diffs and code reads pass silently — for charts, a wrong transform (a cumulative line that should climb rendered as a flat per-day line: same data, same labels, totally different picture); for tables/heatmaps, wrong fill scales, unreadable text on dark fills, and empty-vs-null cell treatment.

Setup (once per machine): `python3 -m pip install playwright pillow`.

- **macOS**: no `playwright install` needed, but you **must** use `channel="chrome"` (system Chrome) — playwright's downloaded Chromium gets its GPU/network processes SIGKILLed by Gatekeeper (symptom: `GPU process exited unexpectedly: exit_code=9` then SEGV).
- **Linux**: `playwright install chromium` and drop the `channel="chrome"` argument — plain Chromium works. Everything else (persistent profile, scroll-and-stitch) is identical.

First run only — headed login (session persists in the profile dir). On a headless-only host (remote Linux / CI) run this step once on any machine with a display, then copy the whole profile directory to the headless host — the session travels with it:

```python
from playwright.sync_api import sync_playwright
import time
URL = "https://app.hex.tech/<workspace>/hex/<project-slug>/draft/app"   # draft editor; /app/<id>/latest works ONLY if published
PROFILE = "/Users/<user>/.hex-screenshot-profile"
with sync_playwright() as p:
    ctx = p.chromium.launch_persistent_context(PROFILE, headless=False, channel="chrome",
                                               viewport={"width":1800,"height":1400})
    page = ctx.pages[0] if ctx.pages else ctx.new_page()
    page.goto(URL, wait_until="domcontentloaded", timeout=90000)
    while "/login" in page.url: time.sleep(3)      # tell the user to log in once (~5 min timeout)
    ctx.close()
```

Every round — headless capture of the FULL app as ONE stitched PNG (matches Tableau's full-height export; do not settle for viewport screenfuls — panels hide at screenful boundaries and you'll mis-flag them as missing):

```python
from PIL import Image
import io
with sync_playwright() as p:
    ctx = p.chromium.launch_persistent_context(PROFILE, headless=True, channel="chrome",
                                               viewport={"width":1800,"height":1400})
    page = ctx.pages[0] if ctx.pages else ctx.new_page()
    page.goto(URL, wait_until="domcontentloaded", timeout=90000)   # networkidle NEVER fires (websockets)
    page.wait_for_timeout(10000)
    try: page.get_by_text("App builder", exact=True).first.click(timeout=5000)   # unpublished app: editor opens on Notebook
    except Exception: pass
    page.wait_for_timeout(20000)          # generative app boots in an iframe (static.hexoutputs.tech / about:srcdoc)

    # 1) find the scroll container INSIDE the app iframe (full_page=True only grabs the outer
    #    viewport, and a giant 12000px viewport renders nothing but a spinner)
    frame = info = None
    for f in page.frames:
        if "hexoutputs" in f.url or f.url == "about:srcdoc":
            info = f.evaluate("""() => { let best=null;
                for (const el of document.querySelectorAll('*'))
                    if (el.scrollHeight > el.clientHeight+200 && el.clientHeight>400)
                        if (!best || el.scrollHeight>best.sh){ el.setAttribute('data-shot-scroller','1');
                            best={sh:el.scrollHeight,ch:el.clientHeight}; }
                return best; }""")
            if info: frame = f; break
    sh, ch = info["sh"], info["ch"]

    # 2) measure the app's sticky header (title+filter bar) — it repaints in every tile and
    #    would overwrite ~175px of content at each stitch boundary if not handled
    sticky_h = frame.evaluate("""() => { const sc=document.querySelector('[data-shot-scroller]'); let h=0;
        for (const el of sc.querySelectorAll('*')) { const cs=getComputedStyle(el);
            if ((cs.position==='sticky'||cs.position==='fixed') && el.getBoundingClientRect().top<300)
                h=Math.max(h, el.getBoundingClientRect().bottom - sc.getBoundingClientRect().top); }
        return Math.ceil(h); }""")

    # 3) iframe's box on the page (crops out the Hex editor chrome/sidebar)
    for el in page.query_selector_all("iframe"):
        cf = el.content_frame()
        if cf and (cf == frame or "hexoutputs" in (cf.url or "")): box = el.bounding_box(); break

    # 4) scroll in overlapping steps, crop each tile to the iframe, drop the sticky band, stitch
    step, full, pos, first = ch - sticky_h - 20, Image.new("RGB", (int(box["width"]), sh), "white"), 0, True
    while True:
        actual = frame.evaluate("(y)=>{const el=document.querySelector('[data-shot-scroller]');el.scrollTop=y;return el.scrollTop;}", pos)
        page.wait_for_timeout(2500)
        shot = Image.open(io.BytesIO(page.screenshot()))
        tile = shot.crop((int(box["x"]), int(box["y"]), int(box["x"]+box["width"]), int(box["y"]+box["height"])))
        if first: full.paste(tile, (0, int(actual))); first = False
        else:     full.paste(tile.crop((0, sticky_h, tile.size[0], tile.size[1])), (0, int(actual)+sticky_h))
        if actual < pos: break            # bottom reached
        pos += step
    full.save("hex_full.png")             # run from $RUN_DIR; full app height, no chrome, no gaps
    ctx.close()
```

Compare — normalize both full images to the same width, slice into aligned crops, and read the pairs:

```python
# Tableau PNG: §4 view-image endpoint (?resolution=high&maxAge=1) -> tableau_view.png (tall); run from $RUN_DIR
from PIL import Image
for name in ["tableau_view", "hex_full"]:
    im = Image.open(f"{name}.png").convert("RGB")
    im = im.resize((1400, int(im.size[1]*1400/im.size[0])))
    for i in range(0, im.size[1], 1150):
        im.crop((0, i, 1400, min(i+1150, im.size[1]))).save(f"{name}_{i//1150:02d}.png")
```

Then **read the image pairs** and diff panel by panel with the fixed per-panel checklists from SKILL.md's discrepancy scan — **charts** against the plotted-series checklist (chart type + marks; x-axis dimension/range/tick format; y-axis unit/tick range; series count/legend/colors; curve geometry — the fingerprint of the underlying transform; headline + endpoint values; number formats), and **tables/heatmaps** against the tabular checklist (fill scale type/steps/palette; scale domain and centre; text colour incl. auto-inversion on dark fills; headers and sort order; empty vs null vs zero cells; triangle vs rectangle shape; column fit; cell formats). Judge only from what is rendered; matching titles or annotation labels prove nothing about the panel's content. Sections won't align 1:1 across slices — match panels by their section headers, not by slice index. Feed every mismatch into the Phase 4 fix batch, and after the Hex thread goes IDLE, **re-run this capture + diff from scratch** — never trust that a fix landed until you see it in the new screenshots. Loop until the diff is clean (or only documented trade-offs remain).
