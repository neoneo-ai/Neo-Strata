# Neo — the Strata desktop manager

> **English (default).** 中文版本：[docs/zh/NEO_DESKTOP.zh-CN.md](zh/NEO_DESKTOP.zh-CN.md)
> README: [README.md](../README.md) (English) · [README.zh-CN.md](../README.zh-CN.md) (中文)

Neo is the desktop layer over Strata: one window that starts, stops, logs and connects models.
When the service crashes, Neo does **not** restart it — it records the exit code and one line of
diagnosis and waits for the user to press "Start" (2026-10-13). It lives in `neo/`, talks to Strata
only through the command-line arguments, environment variables and HTTP endpoints upstream already
has, and **does not modify any existing file in the repository** — so upstream updates never
conflict with Neo.

Progress: **M1 is done** (desktop shell + process supervision + status / log / chat / monitor
pages; the supervisor does not auto-restart, see "Supervisor behaviour"). The M2 and M3 features
(install wizard and download-source probe, LAN access, tokens, Claude Code / Codex wiring, engine
source tree) are implemented and are described with measurements in the sections below. M4
(llama.cpp engine adapter) has not started.

## How it starts

```
python neo/install_neo.py          # installs pywebview (into Strata's .venv; into the current Python if there is no .venv)
NEO-START.bat                      # Windows double-click / command-line start; neo\NEO-START.vbs when there is no console
python neo/manager.py              # run directly; --engine mock uses the mock engine, --no-webview uses the browser
NeoStrata.exe                      # the built single-file exe — double-click it inside the Strata root (see "Building the exe")
```

pywebview uses the Edge (WebView2) backend. When the machine has no WebView2 runtime, or pywebview
is not installed, `neo/manager.py` falls back to the browser UI — the console service is standard
library, the features are identical, there is just no native window. On that path the process sits
in the background with nothing on screen, so the "a second instance is already running" message has
to say exactly who is running (see "second instance" below).

**QuickEdit freezes the console**: when running with a console (`.bat` / `python -m neo.manager`),
one click inside the window enters selection mode; once the console buffer is selected, the thread
writing to stdout parks there — Neo's output goes through that thread, and the window reads as "Not
responding". Three fixes: turn QuickEdit off in the console properties; use `neo\NEO-START.vbs` (no
console); use `NeoStrata.exe` (the windowed build has no console at all).

### Double-click flashed and closed (fixed)

Double-clicking `NEO-START.bat` made the window appear and vanish. Two measured causes:

- The launcher ran `python neo\manager.py`. `neo/manager.py` opens with package-relative imports, so
  running it as a script raises `ImportError: attempted relative import with no known parent
  package` and the process exits immediately. The launcher now always goes through `-m neo.manager`
  (`.bat` / `.vbs` / `.ps1` all changed), and `manager.py` itself recognises the `neo` package, so
  both spellings start (measured: `python neo\manager.py --help` and `python -m neo.manager --help`
  both exit 0).
- On a non-zero exit code the `.bat` used to close the window immediately, so the reason was never
  visible. It now keeps the window open, prints the exit code and the last 15 lines of
  `neo\state\logs\manager.log`, and waits for a key press; the `.bat` starts with `chcp 65001` so
  Chinese log lines no longer turn into mojibake in the console.

**A second instance** (for example one left over from a previous run) shows a modal "Neo is already
running" and exits — that box has to be dismissed, it is not a crash. When the port is held by a
service left over from a previous run, the new instance adopts it (the status page says "adopted the
service already running").

The mutex is per **root directory** (`mutex_name_for(root)` in `neo/manager.py`): one Neo per Strata
directory, two directories do not block each other; only a second instance on the same directory is
blocked. To say *who* is running, the second instance reads `neo/state/instance.json` (written by
`state.record_instance()`: pid / root / port / UI URL / window-or-browser, cleared by `atexit` on
exit):

```
A Neo is already running: process 47308, root D:\Work\Strata.
Its UI is at http://127.0.0.1:8083/
To close it: end process 47308 in Task Manager. To bypass this check: add --force.
```

On the browser path the record carries `mode="browser"`, so the second line becomes "it has no
window — the UI is in a browser: …" plus the command that installs pywebview. When the recorded
process is already gone (killed from Task Manager, power loss), the message is the other one: "…but
it is gone — this record has been cleared", and the stale record is removed
(`state.instance_alive()` asks with `OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION)` — no messages
sent, the process is not touched). Measured 2026-10-12 on Windows 11: with `NeoStrata.exe` running,
the second instance's log contains exactly the text above.

## Files

| File | What it does |
| --- | --- |
| `neo/manager.py` | Desktop shell: single-instance mutex (per root), window geometry persistence, pywebview container (**no `js_api`**), browser fallback, `--force` |
| `neo/supervisor.py` | Process supervision: start, liveness, exit diagnosis, event timeline (**no auto-restart**; backoff and circuit breaker removed 2026-10-13) |
| `neo/console_server.py` | Console service (standard library `http.server`): the pages + JSON API; the landing point of every action in the UI, including `POST /neo/open` / `/neo/open-log` |
| `neo/state.py` | State landing point `neo/state/` (launch settings, window geometry, logs, `instance.json`), atomic write + `.bak`; inside the exe the landing point is chosen by `pick_state_dir()` |
| `neo/paths.py` | Path resolution: `safe_resolve()` does not follow Windows reparse points (the `.SyncRoot` of a OneDrive sync root), depth capped at 64 segments; `app_dir()` / `web_dir()` / `is_frozen()` handle both the exe and the source layout; `strata_root()` resolves the engine root with one rule (see "How the root is resolved"), `app_root()` / `engine_checkout()` |
| `neo/console_io.py` | Console output gate: 400 characters per line, 30 lines per second, identical lines folded inside 5 s, control characters stripped; only wraps streams whose `isatty()` is true; a 2–60-line `write()` passes as one whole block (`BURST_LINES` lines/s); `quiet_gui_logging()` silences pywebview/pythonnet internal logging; drops lines when the real stream cannot be written instead of queueing them; `NullStream` covers the windowed build where `sys.stdout` is `None` |
| `neo/ui_watchdog.py` | UI-thread watchdog: probes the window thread with `evaluate_js`; when it does not answer within 3 s, writes the frame each Python thread is currently sitting in to the log |
| `neo/jobobj.py` | Windows Job Object: when Neo ends, its children (the engine, the vision encoder) end with it |
| `neo/engines/` | Engine adapter layer: `strata.py` (discovers configs, builds the command line), `base.py`, `mock.py` (tests) |
| `neo/netprobe.py` | Download-source probe: connects to `huggingface.co` / `hf-mirror.com` / `www.modelscope.cn` / `github.com` over TCP 443 and times them, results cached; the first three feed the "which source to download from" suggestion, `github.com` is only for the engine source tree (it does not change upstream's source selection) |
| `neo/tokens.py` | Keys and binding: `sk-neo-<32 hex>`, masking, keys cached per base, this machine's LAN addresses, the firewall command (suggested, never executed) |
| `neo/wizard.py` | Install wizard: four status steps + builds the `setup.py --yes --no-start [--family X --model Y]` command + runs it in the background writing `logs/wizard.log`; `run_state()` says whether a run is in progress |
| `neo/engine_setup.py` | Engine source-tree wizard: per the pin in `engine_ref.json`, `git clone --depth 1` into `strata/` or unpacks a zip whose digest matched; `plan()` returns one of seven states, then hands off to `setup.py`; CLI `python -m neo.engine_setup --check \| --ensure [--zip file] [--json]` |
| `neo/engine_ref.json` | The pinned engine: `url` / `tag` / `sha` / `zip_sha256` (+ byte count and a one-line note). A bump edits this one file |
| `neo/deploy.py` | Version-swap script: writes the dev tree's `neo/` + `NEO-START.bat` + `NeoStrata.exe` into the deploy tree after passing three gates (engine / paths / process), then verifies SHA-256 per file and runs `webcheck` and the quick suite in place; dry-run by default, `--apply` writes |
| `neo/models.py` | Install catalogue: reads `setup.py`'s `MODELS` / `FAMILIES` tables with `ast` (never imports or runs it) + this machine (RAM / GPU / free disk) + a by-RAM recommendation + installed state (fits / tight / no); CLI `python -m neo.models [--json]` (root defaults to `neo/paths.py::resolve_root()`) |
| `neo/connectors.py` | Recipes for the four agents (Claude Code / Codex CLI / OpenCode / OpenClaw) and merged writes: back up to `.neo-bak` first, touch only Neo's own section, leave the file alone when it cannot be parsed |
| `neo/connect.py` | Wiring CLI: `python -m neo.connect [agent] [--persist] [--lan] [--json]` |
| `neo/web/` | Console pages: status / models / chat / monitor / logs / settings / connect |
| `neo/webcheck.py` | Page-script health check: parses the pages as one program in `index.html` load order (cross-file top-level name collisions, brace balance, wiring at top level); the gate before `build_exe` ships; CLI `python -m neo.webcheck` |
| `neo/NeoStrata.spec`, `neo/build_exe.py`, `neo/neo_entry.py` | Build into a single `NeoStrata.exe` (see "Building the exe") |
| `neo/tests/` | 523 tests (see "Tests"): `_bootstrap.py` provides `REPO` and `ENGINE` (the engine tree is what `paths.strata_root()` resolves) and `server_ready` (when the engine's real service cannot import, the tests that use the real `serve/server.py` skip and name the missing package); `test_paths.py` guards the root rule, `test_engine_setup.py` guards pin / clone / zip digest / dirty / external engine dir, `test_deploy.py` guards the deploy gates, `test_harness.py` guards the 30-second time standard and that skip rule |
| `neo/requirements-neo.txt`, `neo/install_neo.py` | Neo's own dependencies, never added to upstream `requirements.txt` |

Neo's runtime state lives entirely in `neo/state/`, which is gitignored.

## Supervisor behaviour

```
idle ──start──> starting ──/health answers──> running
  │                        │
  │  load timeout          │  child exits unexpectedly
  ▼                        ▼
idle (records the exit code and one line of diagnosis, waits for the user to press "Start")
  │
  └── a live service appears on the port ──> external (adopt)
```

- **Neo never auto-starts anything**: not after the user presses "Stop" (`intent=stop` is a hard
  switch), and not after a crash or a load timeout. **The backoff restart and the circuit breaker
  were removed on 2026-10-13** (they were "backoff 5 / 15 / 45 s; after 3 crashes inside 10 minutes
  the breaker trips to `halted`; the page shows a single Retry button"). The reason: the benefit was
  tiny and the cost was measured. Model reload belongs to upstream (see "engine reload is not Neo's
  job" below), so a restart at Neo's layer is duplicated work. Every cost was measured — that path
  was the slowest cell in the whole suite (`TestCrashRestart` 14.7 s, about 80% of the 18.3 s full
  run); when an engine dependency was missing it ran the entire backoff table (the full run went from
  18.3 s to 5 minutes); and the breaker made the status page write "Not started" while a live service
  was on the port. A crash now does three things: record the exit code (the "Last exit" row on the
  status page), give one line of diagnosis, and write "stopped, press Start to run it again" into the
  events. That also fixed an old bug: the supervision thread parked in the port-watching layer
  (`_watch_external`) and could not leave it, so pressing "Retry" after a breaker did nothing; that
  layer now recognises "the user pressed Start again" (`_start_seq`), and after a crash pressing Start
  really starts a new process (`TestCrashStops.test_start_after_a_crash_starts_again`).
- **Liveness has three layers**: the port is listening (TCP) → `/health` answers (the service is
  alive) → `loaded` (the model is loaded). Loading a model takes minutes; during that time Neo shows
  "starting", which is not treated as a failure.
- **Probe cadence: process death is checked at `poll_s`, `/health` is asked at `probe_s`** (changed
  2026-10-13). Previously every round probed, and one probe cost up to TCP 1.2 s + `/health` 4 s +
  `/metrics` 4 s, while `proc.poll()` was only looked at between rounds — so a crashed child was only
  visible after the current probe round finished. Measured 2026-10-13 on this machine
  (`neo/tests/probes/probe_cost.py`, timing each layer after killing the service): once the listener
  is killed Windows black-holes the port (SYN is unanswered, not RST), so one probe cost
  1.21 + 2.02 + 2.02 = 5.25 s; what the user saw was "the process is already dead, the status page
  still says running, and the auto-restart started 5 seconds late" (auto-restart existed then; it is
  gone now, and this measurement is about how long death takes to become visible). The TCP timeout was
  cut from 1.2 s to 0.25 s (connecting to a live loopback listener takes 0.00 s, same script), which
  moved death detection from 2.06 s to **0.02 s** and start-to-ready from 1.82 s to **0.61 s**
  (`neo/tests/probes/supervisor_events.py`: it reads the supervisor's own event timestamps).
- **Engine reload is not Neo's job**: `serve/server.py` restarts the engine itself
  (`StrataEngine.restart`, issue #344); Neo only shows a line in the events and does not intervene.
- **A live service already on the port is adopted**: if the user started Strata with `START-HERE.bat`
  first, Neo recognises it and does not start a duplicate. The same applies when Neo cannot start it
  (missing configuration): the port-watching layer hands control back to the main loop to adopt the
  live service (`_watch_external` returns True while `intent=run`). Previously there was only the
  "running (external)" display without the hand-back, so Neo stayed stuck in "external" and pressing
  Start never adopted. After the 2026-10-13 TCP cut from 1.2 s to 0.25 s, `test_adopt_shows_config_log`
  exposed it: those 1.2 s were exactly the window given to an external service to boot, and once the
  window was gone the path was broken.
- **"Ours" vs "someone else's" service is decided through the process tree, not by comparing pids**:
  the python that starts the service spawns another process to run the script, so the listener on the
  port is usually Neo's grandchild. Measured 2026-10-12 in `D:\Work\Strata`: NeoStrata 40856 →
  NeoStrata 46380 → `.venv\Scripts\python.exe` (trampoline) 46984 → 45868 bound to 8080, the last two
  with identical command lines; starting that venv python.exe alone, the start call returns 42016 and
  the process actually running the script is its child 42388. `jobobj.is_descendant` exists for
  exactly this: written as "if the listener is not my child, it is somebody else's service", it would
  classify the very service Neo is talking to as foreign and kill it.
- **An exit gets an actionable line**: the tail of the log is scanned for `out of memory` /
  `failed to map` / `page file` / `already in use` / `driver is too old` and friends, with wording
  aligned to `docs/TROUBLESHOOTING.md`; when nothing matches, the line is "the service stopped: read
  the log for the cause, or press Start to run it again".

## Console API

Binds `127.0.0.1` by default; port = Strata port + 3 (8080 → 8083), and when it is taken it tries
+4..+8 in order. A request carrying `Origin` must point at this machine; a `Host` that is not a
local address is always 403.

```
GET  /                      the console page (static assets are always no-store)
GET  /neo/status            status snapshot: state / pid / listening / healthy / loaded / metrics / events
GET  /neo/models            startable models (the root's strata-*.json, most recently used first)
GET  /neo/log?tail=N        tail of the service log
GET  /neo/config            POST /neo/config     launch settings (stored in neo/state/launcher.json, the key is never echoed back)
POST /neo/start /stop /retry /model
POST /neo/open              {"page":"chat|monitor"} opens that page in a browser (executed by the shell)
POST /neo/open-log          opens the service log with the system default handler
GET  /neo/net?fresh=1       download-source connectivity (TCP 443) and the source suggestion; cached in neo/state/netprobe.json
GET  /neo/lan               this machine's LAN addresses, the UI URLs reachable from them, the firewall hint, current token state
POST /neo/token             {"action":"new|reveal|unset|forget"}: new mints a key and writes neo/state/launcher.json;
                            reveal hands the current key back verbatim to the page (for "copy token"; read-only, and only
                            for requests whose peer address is this machine — everywhere else only the mask);
                            unset clears the key on the service (the cache stays); forget deletes it from the cache.
                            new and reveal answer with the full key, every other action only with the mask
GET  /neo/connect           the wiring recipes for the four agents (reads configs, writes nothing)
POST /neo/connect           {"agent":"claude|codex|opencode|openclaw"} writes the agent config file (backup first)
GET  /neo/wizard            install wizard: four status steps + the setup.py command about to run + log tail
POST /neo/wizard            runs setup.py --yes --no-start in the background (with --family X --model Y when one is named)
GET  /neo/catalog           installable catalogue: setup.py's MODELS / FAMILIES tables read with ast + this machine
                            (RAM / GPU / free disk) + the by-RAM recommendation + installed state + the install in
                            progress + the download-source row (probe verdict, the four options' timings and command
                            prefixes/suffixes, the one that was remembered)
GET  /neo/catalog?fresh=1   same, and re-probes on a background thread (the page reads the cache, never blocked by a slow site)
POST /neo/install           {"family":"qwen","model":"IQ3_S","source":"modelscope"} runs
                            setup.py --yes --no-start --family qwen --model IQ3_S in the background; output goes to
                            neo/state/logs/wizard.log; an item that is not in the menu, or an install already running,
                            is refused with the reason
POST /neo/source            {"source":"probe|huggingface|modelscope|mirror"} remembers the download source in
                            neo/state/source.json and returns the same catalogue; an unknown source returns one
                            explanatory line, writes nothing, starts no subprocess
GET  /neo/engine            the engine source tree's state (the "engine source tree" card on the connect page):
                            the pin's tag / sha / archive URL, whether this machine has git, whether github.com is
                            reachable (reads the probe cache, does not go online), where strata/ is, whether HEAD
                            matches the pin, plan()'s seven states and the next action, the clone command about to
                            run, the three-step status, the tail of engine.log, whether a run is in progress
POST /neo/engine            {"action":"install","zip":"D:/x.zip"}: fetches the source per the pin on a background
                            thread (wait=False — one clone takes tens of minutes and must not block the request
                            thread), then hands off to setup.py (setup=True; Neo does not install the venv /
                            dependencies / engine / models itself); host and key are passed down from the supervisor;
                            output goes to neo/state/logs/engine.log; an unknown action returns one explanatory line
                            and runs nothing; a dirty tree or a plain directory is refused before anything is touched
GET  /i18n.js               the language/theme script (no-store, same as /app.js)
GET  /neo/upstream?what=…   the upstream panel's three blocks (models / status / metrics), read by Neo on the page's
                            behalf: a browser on 8083 hitting 8080 directly is blocked by CORS, and the API key never
                            leaves Neo
GET  /neo/remote            the remote-access command line: address, model and context computed and filled in, the
                            key is only a placeholder — the copy action fills it via /neo/token reveal
GET  /neo/model/settings    one model's runtime parameters (context / KV / vision / VRAM reserve) + the options per
     ?model=ID             field + a VRAM estimate + whether an override config exists + whether a restart is
                            needed; with no model named it uses the first config; a named model that is not found
                            lists the catalogue instead of silently switching to another model
POST /neo/model/settings    writes the override config neo/state/configs/strata-<id>.neo.json (not one byte of the
                            upstream file changes); out-of-range and unknown keys are refused with an explanation;
                            {"action":"reset"} deletes the override, so the next start goes back to what setup.py wrote
```

## Connect page (M2/M3): install models, open the LAN, wire four agents

The page is the "Connect" tab in the window's top bar; the command lines are `python -m neo.wizard`
and `python -m neo.connect`. **The default path does not change**: no user file is written, the
service binds `127.0.0.1`, no key; every item on this page only moves when the user names it.

### "Download and install a model" on the model page (`neo/models.py`)

The catalogue is not copied into Neo: `neo/models.py` reads `setup.py`'s `MODELS` / `FAMILIES` tables
with `ast` (it never imports or runs them; the cache invalidates when the file changes), installed
state comes from scanning the root's `strata-*.json` (config present, model file missing is recorded
as "half installed"), and the machine comes from `GlobalMemoryStatusEx` (the same algorithm as
`setup.py`'s `ram_gb()`), `nvidia-smi` (not asked twice within 60 s; if an AMD card cannot be
queried, it says so) and `shutil.disk_usage` for free disk.

The recommendation is by RAM, using the tiers from `docs/AI_SETUP.md` step 3 and `docs/MODELS.md`'s
"Pick by RAM": 32 GB → Coder IQ1_M, 48 and 64 GB → IQ2_XS, 96 GB and above → IQ3_S. Fits/tight/no
follows `setup.py --check`'s rule: required memory at or below this machine's RAM = fits, up to 8 GB
over = tight, more than that = no; the two unsloth tiers are streaming loads and are judged against
their own floor (48 GB).

Pressing "Install" = `POST /neo/install` = running
`setup.py --yes --no-start --family X --model Y` in the background, output into
`neo/state/logs/wizard.log`; while the model page is open the page refreshes the catalogue and the
log tail every 5 s. Neo never downloads anything itself: downloading, verifying and writing
`strata-*.json` are all `setup.py`'s job; an item that is not in the menu, or an install already
running, is refused by the endpoint with the reason and no subprocess is started. The command line
takes the same road: `python -m neo.models` (root defaults to `neo/paths.py::resolve_root()`;
`--json` for the machine-readable form) shows the catalogue,
`python -m neo.wizard --family coder --model IQ1_M` installs.

The row under the catalogue is the **download source**: `auto (by probe)` / `Hugging Face official` /
`ModelScope (China)` / `hf-mirror`, and choosing one is remembered by `POST /neo/source`
(`neo/state/source.json`) while the command preview changes immediately. All four options go through
openings upstream already has; Neo does not change a line of `setup.py`: official and ModelScope are
values of `--source`, the mirror is `HF_ENDPOINT=https://hf-mirror.com` in the subprocess environment
(`setup.py` reads it itself; `--source` has no mirror entry, so the mirror comes without `--source`).
The "re-probe" button = `GET /neo/catalog?fresh=1`, measured on a background thread, the page reads
the cache. Measured 2026-10-13 on this PC: when `auto` suggests ModelScope the command is
`python setup.py --yes --no-start --source modelscope`; when the mirror is chosen it is
`HF_ENDPOINT=https://hf-mirror.com python setup.py --yes --no-start`; both lines are written into the
head of `wizard.log`.

Measured 2026-10-13 on this PC (Windows 11, Python 3.13, 94 GB RAM, RTX 5090 D v2 with 24 GB VRAM,
487 GB free on D:): 10 catalogue items, recommendation `--family qwen --model IQ3_S` (83.6 GB
download, needs about 62 GB RAM); the already-installed Swift 1.5 IQ3_XXS is marked "installed" in
the catalogue and its button reads "installed".

### Download-source probe (`neo/netprobe.py`)

TCP 443 connect timings against `huggingface.co` / `hf-mirror.com` / `www.modelscope.cn`, results
cached in `neo/state/netprobe.json` (`GET /neo/net?fresh=1` re-measures). Measured 2026-10-12 on this
PC (Windows 11, home network): `huggingface.co` 196 ms, `hf-mirror.com` 4052 ms,
`www.modelscope.cn` 36 ms, all three reachable, so the suggestion keeps the default source and the
`setup.py` command carries no `--source`.

Neo **does not change upstream's source selection**; it only translates the one the user picked into
parameters upstream already accepts: `--source modelscope` / `--source huggingface`, and the mirror via
`HF_ENDPOINT` (see above). The probe verdict only feeds the `auto` tier: when `huggingface.co` is
unreachable and ModelScope or the mirror is reachable, it suggests that one and says why; when all
three are reachable it adds no `--source` and uses `setup.py`'s default source. The user can always
override the suggestion and pick one directly.

### Install wizard (`neo/wizard.py`)

Four steps: environment (root and `setup.py`) → models (`strata-*.json`) → network (probe verdict) →
security (binding and key). "Start install" runs
`[HF_ENDPOINT=https://hf-mirror.com] python setup.py --yes --no-start [--source ...] [--host ...]
[--api-key ...]` (the bracketed environment is only added when the mirror is chosen, see above),
output into `neo/state/logs/wizard.log`, and the page reads the tail every 5 s. `--no-start` is
mandatory: the service belongs to the supervisor, `setup.py` must not start one that occupies the
port. Measured 2026-10-12: running a stub `setup.py` that only prints its arguments, the subprocess
took 4.4 s (cold start of the venv python), exit code 0, and both the command line and the output
landed in `wizard.log`. `--check` only probes and installs nothing, `--dry-run` only prints the
command that would run.

### LAN and token (`neo/tokens.py`)

- The key shape is `sk-neo-<32 hex>` (`secrets.token_hex(16)`); in the UI it is normally only shown
  masked: `sk-neo-xxxxxxxx… (39 characters)`.
- **One-click copy**: "copy token" on the connect page puts the currently active full key on the
  clipboard (`navigator.clipboard`, falling back to `execCommand("copy")` when the permission is
  missing — the same road as "copy service address" and "copy command"). The full key only ever
  appears in the read-only input box on the token row; when the mask changes (key replaced, key
  cleared) the page clears that box immediately, so what gets copied is the current key. Fetching it
  goes through `POST /neo/token {"action":"reveal"}`: read-only, writes no file, and only answers
  requests whose **peer address** is this machine (127.0.0.1/::1) — when the console is bound to
  `0.0.0.0`, a request arriving from the LAN only gets the mask.
- The key used for each base is stored in `neo/state/agent_keys.json` (in two buckets, saved /
  minted), so switching the binding does not require minting a new one.
- When bound to `0.0.0.0` it lists this machine's LAN addresses and the `http://<ip>:<port>` UI URLs,
  and gives a Windows firewall command that is **suggested, never executed**
  (`netsh advfirewall firewall add rule ... port=<port>`). Binding to the LAN without a key is judged
  "not ready" by `wizard.py`, and `neo/manager.py` also prints a warning line.

### What the token covers, and how the monitor page gets its data (`/strata/` in `neo/console_server.py`)

The token only governs the model: every `POST` (`/v1/*`, `/settings`, `/config`, `/load`, `/unload`,
`/slots/*`) and the model listings (`/models`, `/v1/models`) require the key on **any** address. The
monitoring read-only endpoints (`/metrics`, `/metrics/prometheus`, `/status`, `/slots`, `/props`,
`/config`, `/settings`, `/mcp`, `/api/requests`, `/v1/status`) require the key **as well** once one is
set — upstream has no loopback exemption for them.

Neo does not touch upstream here: the loopback exemption patch that had been applied to
`serve/server.py` was reverted on 2026-10-13, and the deploy tree's (`D:\Work\Strata`, a checkout of
the upstream repository) `serve/server.py` is back to upstream's original (391,688 bytes; `git status`
no longer shows a modified tracked file). The monitor page gets its data through Neo's own read-only
channel: the console forwards upstream's page and panel endpoints from its own port, the key is added
by Neo server-side, and the page never sees the key. The monitor URL on the status page is therefore
`http://127.0.0.1:8083/strata/#monitor`; the chat page still connects straight to upstream
(`http://127.0.0.1:8080/`) — chat needs `POST`, and Neo does not spend the user's tokens on the chat
page.

Channel rules (`STRATA_READ_PATHS` / `STRATA_READ_PREFIXES` in `neo/console_server.py`):

| Rule | Detail |
| --- | --- |
| Only GET and HEAD are forwarded | `POST /strata/*` is always 405, so writes and inference cannot leave |
| Whitelist | `""`, `/api-monitor`, `/health`, `/metrics`, `/metrics/prometheus`, `/status`, `/slots`, `/props`, `/config`, `/settings`, `/mcp`, `/api/requests`, `/models`, `/v1/models`, `/v1/status`; prefixes `/web/`, `/fonts/`, `/slots/` |
| Inference is not forwarded | Everything under `/v1/*` except `/v1/models` and `/v1/status` is always 405 (`GET /strata/v1/chat/completions` is 405 too) |
| Upstream not up | 502, saying clearly that upstream did not answer |
| Per-request limits | Response at most 8 MB, upstream request timeout 5 s |
| Does not shadow Neo's own routes | `/strata` and `/strata/` both map to the upstream root (`""`); Neo's `/neo/*` routes keep working |

Upstream's page issues its requests with relative paths, so the `/strata/` prefix naturally carries
the page, its js/css and its data requests together. `/api-monitor` and `/api/requests` only exist
when the upstream config has `api_monitor` on; when it is off, the channel forwards the 404 through
unchanged.

Measured 2026-10-13 (mock engine with a key set, `tools/_strata_channel_check.py`, 23 checks all
pass): direct upstream `/metrics` without a key 401; through the channel 200 with all four sections
`live`/`totals`/`hardware`/`hardware_static` present and no key in the response; upstream's page and
`web/app.js`, `web/tokens.css` all 200; `POST /strata/settings` 405,
`POST /strata/v1/chat/completions` 405, `GET /strata/v1/chat/completions` 405, `POST /strata/load`
405; upstream's `/v1/models` with a key 200, without a key 401. At the browser layer
(`tools/_strata_page_check.py`, headless Edge, 10 checks all pass): the same upstream page says
"API key needed" in the DOM when reached directly, and through the channel that line is gone, the
monitor tab is active, `renderMonitor` runs to completion, and the status badge lights up.

Regression tests: `TestStrataChannel` in `neo/tests/test_console.py` (fake upstream; covers the page
and its resources, the server-side key, query strings, write operations, inference, path rejection,
upstream down, not shadowing Neo's own routes, path-constant consistency) and
`test_pages_monitor_goes_through_the_console` in `neo/tests/test_engines.py`;
`neo/tests/test_upstream_api.py` watches the inverse — "Neo does not depend on an upstream patch":
the exemption patch markers must not appear in upstream files, and the monitor page and those
read-only endpoints must still be there.

The rule for adding a console endpoint: if `Handler` calls `self.server.X(...)`, then
`ConsoleServer.start()` must have a matching `httpd.X = self.X` — the httpd object is what
`self.server` is, and the console's methods are attached one by one. A missing attachment does not
crash in the page: the AttributeError is swallowed by socketserver's `handle_error`, the connection
is simply closed, `curl` only sees `Empty reply from server`, and the page shows a fetch that spins
forever (the engine card on 2026-10-13: both attachments were missing). `test_console.py` now pins
two things: every name used as `self.server.X(` in the source must be in `start()`'s attachment list,
and a real HTTP round trip of `GET /neo/engine` (calling the method directly cannot catch this).

### Recipes for the four agents (`neo/connectors.py`)

| Agent | Landing point | The block Neo writes |
| --- | --- | --- |
| Claude Code | `~/.claude/settings.json` | In `env`: `ANTHROPIC_BASE_URL` (**without `/v1`**), `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_MODEL`, plus four variables that switch off telemetry/experimental features |
| Codex CLI | `~/.codex/config.toml` + `neo.config.toml` | `[model_providers.neo]` (`base_url` **with** `/v1`, `wire_api = "responses"`, `requires_openai_auth = false`, `stream_idle_timeout_ms = 1200000`) + `[profiles.neo]` |
| OpenCode | `~/.config/opencode/opencode.json` | `provider.neo` using `@ai-sdk/openai-compatible`, model ids prefixed with `neo/`, output cap `window // 4` (capped at 32000), compaction reserve `min(max(0.3×window, 4000), 15000)` |
| OpenClaw | `~/.openclaw/openclaw.json` | `models.providers.neo` (`api = "openai-completions"`) + `agents.defaults.model.primary = neo/<id>`; it also sets `memory.search.provider` to `none` and `env.shellEnv.enabled` to `false`, otherwise it goes looking for a real OpenAI key |

Rules for writing files (each one watched by `test_connectors.py`):

- **Merge, never overwrite**: only Neo's own section is touched, every other user key stays as it was.
- **Back up first** to `<file>.neo-bak`; a TOML/JSON that cannot be parsed is **left untouched** and a
  warning is returned.
- **Idempotent**: re-running replaces Neo's own section, it does not stack duplicate blocks;
  `GET /neo/connect` reads and never writes.
- Directories honour `CLAUDE_CONFIG_DIR` / `CODEX_HOME` / `XDG_CONFIG_HOME` / `OPENCLAW_HOME`; the
  tests use exactly those four variables to put all four agent configs into a temporary directory and
  never touch real user configuration.
- Claude Code only honours env passed through `--settings`, and `--settings` does not read
  `~/.claude`, so the recipe also carries a PowerShell / POSIX `$env:` / `export` block. When a key is
  present, the 11 variables that must be unset are listed in the recipe (`ANTHROPIC_API_KEY`,
  `CLAUDE_CODE_OAUTH_TOKEN`, `ANTHROPIC_UNIX_SOCKET`, the six Foundry/Bedrock/Vertex/AWS/Google
  Cloud/Mantle switches and their base_url/resource) — leaving any one of them will send the request
  to the real Anthropic.

Command-line examples (measured 2026-10-12; with `--key` given explicitly, user configuration is not
read):

```
python -m neo.connect list
python -m neo.connect claude --port 8080 --key sk-neo-... --model qwen3.8-flash-next
python -m neo.connect codex --persist --lan
python -m neo.wizard --check
```

## Engine source tree: installed by pin (`neo/engine_setup.py`, `neo/engine_ref.json`)

Once Neo and Strata were split apart, the engine source no longer lives in Neo's repository; Neo
fetches it by a **pinned tag**. `neo/engine_ref.json` is a JSON with four things: repository URL, tag,
commit SHA, and the SHA-256 of the archive zip. Today's pin is `v0.1.43.1` =
`a0116b2782a5b82622358c04a06496485d12b2d6` (the deploy tree `D:\Work\Strata` has exactly that HEAD).

**Source is only taken from a GitHub tag.** Mirrors like ModelScope serve a repository's current
files, not the pinned revision — that is exactly what `#908` in upstream `setup.py`'s `model_source()`
says. Mirrors are only used for downloading models (see "Download-source probe"), never for source.

Two roads:

```
git clone --branch v0.1.43.1 --depth 1 --single-branch <url> strata/
<url>/archive/refs/tags/v0.1.43.1.zip          (the fallback when this machine has no git)
```

A shallow clone lands on a detached HEAD, which is what we want: the pin is a commit SHA, not a
branch. The zip is packed by GitHub on demand and has **no official digest**, so `zip_sha256` is
computed on Neo's side and pinned: on 2026-10-13 the `v0.1.43.1` archive zip was downloaded —
21,786,355 bytes, 3415 entries — the local SHA-256 came out
`9e37f07e87473210ac98ee6041e6651c02e1c473bf75eae65191840fa2b80768`, and that string is what is written
into `engine_ref.json`; before unpacking, Neo recomputes it and unpacks nothing if it does not match.

**The directory is called `strata/`, not `engine/`.** In upstream `setup.py`'s `engine_dir()`,
`engine/` is the directory of the **engine binaries** (`/engine/` in the root `.gitignore` is that
one), so the source tree must not take that name. `strata/` is in the root `.gitignore`: it is cloned,
it does not enter Neo's repository. In the deploy tree that line is not in `.gitignore` (that is an
upstream tracked file) but in `.git/info/exclude`, added by `neo/deploy.py`.

`plan()` is a pure function (no network, no writes) with seven states: `ready` (HEAD == pin and git
is clean), `clone`, `zip_copy`, `need_zip`, `checkout_pin` (HEAD is not the pin), `dirty` (someone
edited it → refuse, Neo never touches the user's changes), `not_a_git_tree`. A zip unpacks without a
`.git`, so HEAD cannot be checked; Neo therefore records a provenance note in `neo/state/engine.json`
(it can only be written when the digest matched the pin at install time), and `ready` accepts that
note.

After the source is in place Neo installs nothing: the venv, dependencies, engine binaries and models
all belong to upstream `setup.py` (`neo/wizard.py`'s `--yes --no-start` road). In the UI it is the
"engine source tree" card on the connect page (`GET /neo/engine`, `POST /neo/engine {"action":"install"}`);
on the command line:

```
python -m neo.engine_setup --check
python -m neo.engine_setup --ensure --no-setup
python -m neo.engine_setup --ensure --zip D:\Downloads\Strata-0.1.43.1.zip
```

`netprobe.py` therefore measures one more host, `github.com` (443, reachability only, like the model
sources). It does not feed the "which source to download models from" suggestion — that suggestion only
looks at huggingface / hf-mirror / modelscope.

### How the root is resolved (`strata_root()` in `neo/paths.py`)

One rule, shared by the program and the tests (`ENGINE` in `neo/tests/_bootstrap.py` is what it
computes):

```
--root  >  NEO_STRATA_ROOT  >  the application directory itself  >  <app dir>/strata  >  the application directory (with "no engine here yet" in the reason)
```

Rules 3 and 4 only apply when the directory contains both `setup.py` and `serve/server.py`; the first
two are the user naming a directory, so they are used as given with no marker check (if it cannot
start, the start step says so plainly — Neo never silently swaps the directory). When the exe sits in
the Strata root (the install every user has today), rule 3 hits first and the resolved result is
byte-for-byte identical to what it was before this rule existed.

### Bumping: what changes when the engine moves a version

1. Upstream publishes a new tag → edit `tag` and `sha` in `neo/engine_ref.json`;
2. download `archive/refs/tags/<tag>.zip`, compute its SHA-256 locally and put it in `zip_sha256`
   (record the byte count as well);
3. in the deploy tree `git checkout <tag>` (or `git pull --ff-only`), then
   `python -m neo.deploy --to <deploy tree>`;
4. re-run the full test suite, and update the numbers in this section and in "Tests" with the new
   measurements.

## Model settings page, language and theme (`neo/model_settings.py`, `neo/gguf_meta.py`, `neo/web/i18n.js`)

The model settings page changes **one model's launch parameters**: four fields — context length
(`--max-context`), KV type (`--kv`), KV resident units (`--kv-resident`), VRAM reserve
(`--vram-reserve-mib`) — plus the image switch (`--vision` and the `vision` section). It is the first
block inside the "Settings" page; the left rail no longer has a separate tab (merged 2026-10-13): not
one control id moved, and `showPage("settings")` pulls out both the "model settings" and the "launch
settings" blocks in one call (`if (name === "settings") { loadConfig(); loadModelSettings(); }`).
Writing is done as an **override config**: the `strata-*.json` written by `setup.py` is read-only,
changes go to `neo/state/configs/strata-<id>.neo.json`, and at start `config_path_for()` points at
whichever file exists; deleting the override returns to the original. How each of the four numbers is
derived is shown on the page: KV VRAM = resident units × bytes per unit per layer × layer count, with
the per-unit-per-layer byte figures from `bench/results/2026-09-27-kv-q4/README.md` (q4_0 576 B, int8
1,056 B) and the layer count read from the GGUF header (`neo/gguf_meta.py`, header only, the model is
never loaded).

Three traps, each guarded by a test:

- **Turning images off takes two actions together**: `serve/server.py:2348` (#1322) adds `--vision`
  back whenever the `vision` section is still in the config, so deleting only the flag is the same as
  not turning it off. Neo deletes the flag and empties the section at the same time, keeps the original
  section in `neo_vision_saved`, and restores it verbatim when images are turned back on (Neo never
  guesses the mmproj path — it only hands back the user's own).
- **`--kv-resident` carries a value**: deleting only the flag leaves the number as a stray positional
  argument, and the engine will treat it as the model file. When it is deleted, the value goes with it.
- **The page sends `"on"`/`"off"` strings**: `bool("off")` is True, so the strings are matched
  literally (`on/true/yes/1/开`, `off/false/no/0/关`); anything unrecognised is an error, never a guess.

The override config = upstream's file + this change, so keys identical to upstream are not written
into it; when what this change wants already equals upstream, the existing override is deleted (back to
the original) rather than kept as a useless copy. A model id that is named but not recognised is never
silently swapped for another model.

Language and theme live only in the page — neither Neo nor upstream stores them: `neo/web/i18n.js`
keys off `data-i18n` (static text is swapped block by block) and `t("中文")` in `app.js` (dynamic
text), with the memory keys `neo.lang` / `neo.theme` in localStorage. **With no stored preference the
page is word-for-word what it always was** (Chinese, dark theme) — i18n.js changes no text. The whole
of i18n.js is wrapped in one IIFE exporting only `window.NEO_I18N` (the namespace app.js reads):
`<script src="/i18n.js">` and `<script src="/app.js">` are two classic scripts in one global scope, so
top-level names collide with app.js and the browser refuses to run app.js at all when parsing it (see
"Every click was dead and the status stayed 'Not started'"). Theme only swaps CSS variables:
`style.css` overrides the `:root` block with `html[data-theme="light"]`, components always use
`var(--…)`, so switching themes changes no component rule (a test in `neo/tests/test_ui.py` measures
exactly that: no hard-coded colours outside `:root` and `html[data-theme="light"]`).

## Shape of the model config (`strata-*.json`)

The config is written by `setup.py`; Neo only reads it and turns it into `serve/server.py`
command-line arguments, changing not one line of upstream. Measured 2026-10-12 in `D:\Work\Strata`
(all four configs recognised, `installed=True`):

```
{"exe": "engine/strata.exe", "cwd": "D:\\Work\\Strata",
 "args": ["--native", "D:\\Work\\Strata-data\\models\\swift-IQ3_XXS\\...gguf",
          "--pack", "D:\\Work\\Strata-data\\packs\\swift-iq3_xxs",
          "--max-context", 202072, "--mtp", "...", "--gpu", "0", ...],
 "model_name": "swift-1.5-iq3_xxs", "port": 8080, "tokenizer": ..., "log": ..., "vision": {...}}
```

- **The model file is inside `args`, under `--native`** (then `--pack`, `vision.model`); **there is no
  `path` key**; the context is under `--max-context` (then `--context`). `model_file_of()` /
  `context_of()` / `arg_value()` in `neo/engines/strata.py` accept both shapes (flat list and the old
  dict).
- The early version that only understood `path` + a dict `args` could never compute a model file from
  real configs, and the start error it printed was **the empty message "model file missing: "**
  (`cfg["path"]` was empty to begin with) — measured in `manager.log` on 2026-10-12.
- Engine executable: the config's `exe` key wins (relative paths expand against `cwd`, matching the
  comment in `serve/server.py`); if absent, the root's `strata.exe` / `strata-hip.exe` /
  `strata-sycl.exe`.
- Whatever is missing is written out one item per line (`missing_parts()`), and given to the user
  verbatim when a start is refused — no more empty messages.
- Model directories default to `<root>-data` next to Strata (measured `D:\Work\Strata-data`:
  `models` / `mtp` / `packs`).
- When no model is named, a config that **can start** is picked (`default_config()` in `base.py`: the
  named one → the first installed one → the first one); only when none of them can start is the first
  returned, so the error message is about something real.

## Logs and console output: four flood sources, each measured

Flooding the window turned out to have four independent sources, each fixed on its own; every number
and every wording below comes from a measurement (reproduction scripts
`neo/state/repro/crlf.py`, `neo/state/repro/flood.py`).

**1. `\r` in `manager.log` grew with the number of writes (fixed).**
`state.log()` used to clean the log by reading the tail back in text mode and rewriting it: a `\n`
written out becomes `\r\n`, and reading it back and writing it again makes `\r\r\n` — every write adds
one more `\r` to every earlier line (measured: after 6 consecutive writes the first line carried 6
`\r`). In a console each `\r` returns the cursor to the start of the line, so the line is redrawn once
more, which looks like the same sentence repeating. Writes are now binary and cleaning only removes
`\r` without limiting line count (history is not lost). Measured: after 8 consecutive writes the file
contains 0 `\r`.

**2. A single line of several hundred thousand characters used to pass through every layer intact
(fixed).** pywebview/pythonnet's property-chain recursion log and `RecursionError` messages are lines
like that. Every layer is now bounded:

| Layer | Limit | Measured |
| --- | --- | --- |
| Console stdout/stderr (`neo/console_io.py`) | 400 characters per line, 30 lines per second, identical lines folded into one count within 5 s, control characters (`\r`, BEL, ANSI sequences) replaced by spaces; only streams whose `isatty()` is true are wrapped; a 2–60-line `write()` passes as one whole block | 5000 `.SyncRoot` lines of 360k characters (1.8 billion characters total) pushed through the gate → out came 441 characters, 2 lines, longest line 419 characters, 0 `\r` |
| `neo/state/logs/manager.log` | each line trimmed to 400 characters marked "original length N"; trimmed to the last 200 KB before writing | the same 5000 giant lines → file 196 KB, 454 lines, longest line 419 characters, 0 `\r` |
| `GET /neo/log` | at most 60 KB / 600 lines per call, lines already trimmed | after the flood, the response was 539 characters |
| The log page (`neo/web/app.js`) | fetches 60000 characters at a time, renders at most 400 lines, does not redraw when content is unchanged | the page is no longer wedged by one giant line |
| The tail printed by `NEO-START.bat` | 15 lines, control characters replaced by spaces, 200 characters per line | measured 15 lines, longest 77 characters, Chinese intact |
| Uncaught exceptions | traceback at most 40 lines / 4000 characters; `RecursionError` is not formatted into a traceback (thousands of frames, the message itself is hundreds of thousands of characters, formatting it recurses once more) — only one line pointing at pywebview and sync directories | a 90019-character exception message lands in the log as a 416-character line |

**5. The gate used to cut whole blocks the machine was about to read (fixed).** The gate trims by line
(30 lines/s), and `neo/manager.py` installs it at import time, which produced two measured side
effects: the 30-line JSON of `python -m neo.connect --json` lost 26 lines (the page received half a
JSON and `json.loads` reported "Expecting value: line 31 column 1"); and the `StringIO` swapped in by
`redirect_stdout` in tests was treated as a console and wrapped, so unittest's traceback was folded
into "the same line repeated N times" and it was impossible to tell which test failed. Two rules now:
`install()` only wraps streams whose `isatty()` is true (pipes, files and `StringIO` are never wrapped
— flooding only happens on a console); and a 2–60-line `write()` passes as one whole block, charged
against the "whole-block allowance" (`BURST_LINES` = 60 lines/s), with the excess dropped as before.
Measured: 40 lines in one write all pass; two 40-line blocks in one second, the second is dropped for
exceeding the allowance; a 500-line one-line-per-write flood is still compressed to under 120 lines
(`test_logging.py`'s `test_one_print_block_passes_whole`, `test_burst_allowance_is_also_capped`,
`test_rate_cap_drops_excess`, `test_install_leaves_a_pipe_alone`).

Reading the log in `.bat` uses `Get-Content -Encoding UTF8`: Windows PowerShell 5.1 decodes files with
the system ANSI code page by default, so a UTF-8 Chinese log becomes mojibake (measured before/after
adding `-Encoding UTF8`: `default` produced mojibake bytes, `utf8` produced
"stopped (will not auto-restart)").

**3. `.SyncRoot` is external noise, not in the repository.** Two independent paths print that name:

- pywebview's WinForms backend logging property chains through pythonnet
  (`window.native.ActiveControl...SyncRoot...GenericSansSerif`; that `SyncRoot` is .NET's
  `ISynchronizeInvoke.SyncRoot`). The gate must be installed **before** `import webview`: logging's
  `StreamHandler` captures whatever `sys.stderr` object exists at that moment, so installing late
  means it cannot govern it. The order in `neo/manager.py` is `console_io.install()` →
  `state.install_excepthook()` → only then import webview.
- Windows reparse points: `Path.resolve()` follows the reparse point of a OneDrive sync root,
  `os.readlink()` answers `.SyncRoot` for such a point, and realpath concatenates all the way back
  until `maximum recursion depth exceeded`. `safe_resolve()` in `neo/paths.py` now does no filesystem
  walk and caps depth at 64 segments, with `is_reparse()` recognising them; `state.py`, `manager.py`,
  `console_server.py`, `install_neo.py` and `engines/strata.py` all use it. Measured: a path built
  from 5000 `.SyncRoot` segments is truncated to under 64 segments, no recursion.

**4. Advice for users**: when Neo or Strata (including the model directory) sits in OneDrive or
another sync directory, move it out of the sync scope, or point `--root` at a plain directory. Window
output is already trimmed; the complete log is `neo/state/logs/manager.log`.

## Clicking made the window "(Not responding)" (fixed, pywebview 6.2.1 / WinForms + WebView2)

"(Not responding)" in the title bar means the window thread is not pumping messages. That thread is
pywebview's own STA thread (`create_window` in `platforms/winforms.py`: `ThreadStart(create)` +
`ApartmentState.STA`); the Python main thread just spins in `thread.Join(500)`. The full path of one
click in the page through pywebview's js_api bridge:

1. Click a button → WebView2 `PostWebMessageAsJson` → `on_script_notify` at
   `platforms/edgechromium.py:225` runs **on the window thread**, and the `print` inside it writes
   stdout on that same thread;
2. `js_bridge_call` in `util.py:247` throws the `Shell` method onto a new thread (that part does not
   block the window);
3. when the method returns, `window.evaluate_js(...)` sends the result back to the page →
   `self.webview.Invoke(...)` at `platforms/edgechromium.py:137` **posts synchronously to the window
   thread**, then `semaphore.acquire()` waits for it to finish;
4. for the window thread to run that delegate it must acquire the GIL. If it cannot get the GIL for a
   long time, or is stuck on a `print` against the console, the message loop stops: every js_api call
   stops returning and the window ignores all clicks.

**Main fix: remove the bridge, all actions go through the service.** `window_kwargs()` in
`neo/manager.py` has **no `js_api`**; the window is only a container. Every action in the page
(`neo/web/app.js`) is an HTTP call to the console service — including "open the chat / monitor page"
(`POST /neo/open`) and "open the log file" (`POST /neo/open-log`), which are executed by the `Shell`
attached to the service (`_shell_action` in `neo/console_server.py`). Not one line of Python runs on
the click path; the window thread does one thing: pump messages. The browser UI uses the same set of
endpoints, so the two paths are now completely identical. `TestClickPathIsHttpOnly` in
`neo/tests/test_ui.py` watches this — if `js_api` appears in the window kwargs, or `window.pywebview`
appears in the page code, the test goes red.

Measured 2026-10-12 (Windows 11 / pywebview 6.2.1 / the packaged `NeoStrata.exe`, mock engine, port
8780): window open for 95 seconds (the page polls over HTTP every 2 s), `Responding=True`, 0 "no
answer" entries in `manager.log`; `GET /neo/status` 8.7 ms, `POST /neo/start` 4.5 ms, `POST /neo/open`
65.3 ms (a browser really opened), `POST /neo/open-log` 146.7 ms (the log file really opened). The same
exe placed in `D:\Work\Strata` running `--no-start`: window `Responding=True`, the model page lists 4
models, `POST /neo/open` returns 200, log has 0 "no answer".

The three places that used to feed this trap are kept — they guard "print on the window thread and the
GIL", and the watchdog probe and pywebview's own threads still take that road:

| Place | Before | Now | Measured |
| --- | --- | --- | --- |
| `console_io.BoundedStream.write` | queued waiting for `_lock`; when a thread was parked on a real stream write (console selected by QuickEdit, full pipe), the window thread hung with it | returns immediately if the lock is not available; the line is dropped and counted | with another thread holding the lock, `write()` returns immediately (the test asserts < 0.5 s) and the lines are counted in "console cannot be written, dropped N lines" |
| `state.log()` | every log line read the whole tail (up to 200 KB), cleaned it line by line and rewrote the whole file, holding the GIL throughout | appends one line in the normal case, trims once only when `LOG_MAX` is exceeded | 20 log lines = 20 append writes, 0 `\r` in the file; 6 threads writing 90 lines concurrently, nothing lost, nothing interleaved |
| `Shell.stop()` | `sup.stop()` did `thread.join(timeout=10.0)`, so the call hung for 10 seconds and the button that `await`s it spun the whole time | `join_s=0.5`; the remaining wind-down is finished by the supervision thread itself, and the status page updates through its 2-second poll | pressing Stop answers within 0.5 s |

One more thing fixed on the way: pywebview 6.2.1 **does not have** `window.get_geometry()` (measured:
`geometry` is not in `window.py`). `neo/manager.py` used to call it every 5 seconds, each call raising
`AttributeError` swallowed by `except Exception` — window geometry had never been saved. Geometry is now
taken from the `events.resized` / `events.moved` events (`Event.set` in `event.py` starts a thread for
the callback; the callback only records two numbers in a dict: no file access, no touching the window
thread), and a background thread stores it every 5 seconds. Measured: after one windowed start,
`neo/state/window.json` contains real coordinates (`x=20, y=261, w=1280, h=860`).

**The watchdog `neo/ui_watchdog.py`**: every 2 seconds it probes the window thread through pywebview's
own road — `window.evaluate_js("1")` only returns when the window thread pumps messages. Silence beyond
3 seconds counts as stuck, and the frame each Python thread is currently in (`sys._current_frames()`)
is written line by line into `neo/state/logs/manager.log`; at least 30 seconds between two snapshots.
The probe only runs on background threads, at most 4 in flight. Measured with a window for 1 minute
(the page polls over HTTP every 2 s): 0 "no answer" in the log, process `Responding=True`; in tests,
freezing `evaluate_js` immediately produces "no answer" in the log plus the frame `neo-ui-probe` is
sitting in.

## The chat and monitor pages were blank (fixed)

Clicking "Chat" / "Monitor" in the window left a 1062x677 blank embedded frame; the status pill said
"Not started" while the service on 8080 was answering normally (`/health` 200, `/neo/status` 200).

Measured 2026-10-12 in a real open window (pywebview 6.2.1 / WebView2, page `http://127.0.0.1:8083/`):
the page's resource list contained only `style.css`, `icon.svg` and `app.js` — **not one
`/neo/status` request**: the page had never started polling. The cause was in `neo/web/app.js`: the body
of `num()` was cut off after its opening, so `render` and every button wiring fell inside its body, the
final `poll()` at the end of the file became a `ReferenceError`, `last` stayed null, `ensureFrame()`
returned immediately, and the two embedded frames' `src` stayed at `about:blank`. Leading indentation
does not reveal this kind of damage (the swallowed code was written flush left anyway); brace depth
does.

The fix was to close `num()`, returning wiring and polling to the top level. Re-measured under the same
probe (source run with `--engine strata --no-start`, console on 8084, the service the one already
running on 8080): `/neo/status` every 2 seconds (200, 2 ms), status pill "Running", chat frame
`src=http://127.0.0.1:8080/`, monitor frame `src=http://127.0.0.1:8080/#monitor`; after 5 seconds both
frames were 1062x677 with a cross-origin document inside. The same iframe rendered the complete Strata
monitor page in a headless Edge screenshot (Speed / VRAM / PCIe cards and the request table all
present); `127.0.0.1:8080` sends no `X-Frame-Options` and no CSP, so embedding is not blocked.

Regression test: `TestAppJsTopLevelWiring` in `neo/tests/test_ui.py` uses a brace-depth scan
(`js_depth_lines`, skipping comments, strings, template literals and regex literals) to pin that
`render`, `poll`, `ensureFrame` and `num` are at top level, that the file's final `poll();` and
`setInterval(poll, 2000);` are at top level, and that braces balance. Remove `num()`'s closing brace and
these tests go red immediately.

## Status cards were all "—" and the requests row read [object Object] (fixed)

The user's window screenshot on 2026-10-12: the service was running (status pill "Running", context
202,072), and of the six cards **generation speed / decode step / VRAM / RAM / running requests** were
five "—", with the "Context" card's subtitle reading "requests [object Object],[object Object]".

Measured 2026-10-12 on this machine (RTX 5090 D v2 / Windows 11 / Python 3.13.15,
`serve/server.py --engine mock --port 28181`, read after one `/v1/chat/completions`): the top level of
`/metrics` has only nine keys — `engine`, `live`, `requests`, `requests_kept`, `totals`, `hardware`,
`hardware_static`, `history`, `time` — and every number the cards want lives inside the sections:

| Section | What is inside |
| --- | --- |
| `live` | the current request: `state`, `queued`, `phase`, `prompt_tokens`, `generated`, `tok_s`, `tok_s_mean` (with `--batch` also `running`, `slots`, `waiting`) |
| `totals` | cumulative: `requests`, `prompt_tokens`, `reused`, `output_tokens`, `prompt_ms`, `decode_ms` |
| `hardware` | sampled once per second: `gpu_util`, `gpu_mem_used`, `gpu_mem_total` (bytes), `gpu_temp`, `gpu_power`, `cpu` (percent), `ram_used`, `ram_total` (bytes), `disk_read_mb`, `disk_write_mb` |
| `hardware_static` | `gpu_name`, `gpu_count`, `gpu_note`, `cpu_name`, `cores`, `threads` |

`neo/engines/base.py` used to pick from 41 flat key names (`tokens_per_s`, `decode_step_ms`,
`gpu_used_mib`, `ram_used_gb`, …) at the **top level**: upstream has never published any of those names,
and of the keys the cards wanted only `requests` existed — and it is the **record list** of the last 12
requests; interpolating a list into a template string is `[object Object],[object Object]`.

The fix has two layers: `flatten_metrics()` in `base.py` reads values from the real sections and converts
units (bytes → MiB / GB, `decode_ms / output_tokens` → ms/step, `live.running` or `live.state` → number
of running requests); keys upstream does not have simply do not appear and the page shows "—". In
`app.js` every card goes through `num()` / `count()`, and `count()` answers "—" for lists and objects, so
the page layer cannot produce `[object Object]` either.

Measured after the fix (`python -m neo.console_server mock 28190`, headless Edge screenshot 1500x420):
VRAM **24079 / 24455 MiB · NVIDIA GeForce RTX 5090 D v2**, RAM **70.4 / 93.6 GB**, running requests
**0** (queued 0), context **32,768**, requests **0**; generation speed and decode step are "—" when
idle — with nothing generating, no fake 0 is invented.

Regression tests: `TestMetricsFlatten` in `test_engines.py` (8, fed the payload from that real
response), `test_status_metrics_are_numbers` in `test_supervisor.py` (real process; no lists or objects
in the values), `test_metrics_sections_neo_flattens_still_exist` in `test_upstream_api.py` (upstream
renames ring this first), and `TestStatusPageMetricCards` in `test_ui.py` (cards only pass through
`num()` / `count()`; things like `m.requests` must not be interpolated directly into a template string).

## Every click was dead and the status stayed "Not started" (fixed)

Symptom: the window opens, the page is fully laid out, the status pill stays "Not started" forever,
every button is dead, and the embedded chat/monitor frames are blank.

Root cause: `neo/web/index.html` loads two scripts in sequence — `<script src="/i18n.js">` and
`<script src="/app.js">`. Two classic scripts **share one global scope**, and i18n.js declared
top-level names that collide with app.js: `const t` (app.js has `const t = (zh, en) => …`),
`function applyLang` (app.js calls it at the end of its top level), `const LANGS`, `const LANG_KEY`,
`const THEME_KEY`, `let theme`. The collision is reported when app.js is parsed
(`Identifier 't' has already been declared`), so **app.js does not execute a single line**: `poll()`
is never called, `setInterval` is never registered, the buttons have no handler, and the status pill
stays at its initial value. The page looks normal (the HTML is static), every click is dead.

Fix: i18n.js is wrapped in one IIFE, exporting only `window.NEO_I18N` (the namespace app.js reads);
no top-level name is left.

Regression test: `TestNoGlobalScopeCollision` in `neo/tests/test_ui.py` — it extracts the top-level
declared names of the two scripts and asserts the intersection is empty, and asserts i18n.js has no
top-level declarations. Add a top-level `const t` to i18n.js and the test goes red immediately.

## The service was running and the pill in the corner said "Start" (fixed)

The user's words: the model is already started, so the button should be "Stop". In the screenshot the
status pill read "Running" and the service row read "Running · model ready", yet among the three
buttons in the top-right corner **Start was the bright blue primary button and Stop was grey**.

Measured 2026-10-12 (`python -m neo.console_server mock 28210`, headless Edge screenshot 1500x120,
`/neo/status` answering `state=running`, `healthy=true`, `loaded=true`):

- `neo/web/style.css` had **not one `:disabled` rule** — a disabled button looked exactly like an
  enabled one, and the blue of `button.primary` was hard-wired onto "Start" (the class shipped in
  `index.html`, never swapped by the page), so a running page read as "you should press Start".
- The disable logic itself was right (`btn-start` is disabled in running/starting, `btn-stop` only
  disabled in idle); it was purely a styling problem.

Fix: the primary blue follows "the button that can be pressed right now" — in `app.js`'s `render()`,
`running / starting / stopping / external` light up **Stop**, and when stopped they light up **Start**;
`style.css` gained `button:disabled` (`opacity .42` + `not-allowed`) and `button.primary:disabled`
(back to grey background, grey text), so a disabled button is obvious at a glance.

Measured after the fix (same probe, same screenshot size): with `state=running` the top-right corner
shows **Stop** in bright blue and **Start** greyed out.

Regression test: `TestHeaderActionButtons` in `test_ui.py` (3) — the CSS must contain
`button:disabled` and `button.primary:disabled`; both `btn-start` and `btn-stop` must
`toggle("primary")`; `btn-stop`'s disable condition must not mention `running` / `external` /
`stopping` (when it is running, it must be stoppable).

## The model page's "Start it": what happens when you press it while running

The user asked: does "Start it" in the model list require the service to be stopped first?

**No, you don't stop it first.** `set_model` (`neo/supervisor.py:156`) is "record the new model +
`stop()` + `start()`" — Neo stops the current service itself and restarts it with the new model.
Measured 2026-10-12 (real mock process, `TestSetModel` in `neo/tests/test_supervisor.py`): calling it
while running changes the pid, the state returns to running, and `last_exit` is empty — a model switch
the user asked for is not an unexpected exit, so the "last exit" row on the status page does not light
up because of it.

The cost is that **requests in flight get cut** (`stop()` kills the whole child-process tree once the
graceful-exit window is over). The button used to just say "Start it", identical whether running or
stopped, so nothing about the consequence was visible. Now it is spelled out:

- current model: button "In use", disabled;
- service running (running / starting / stopping / external): button "Switch to it", hover text "will
  first stop the current service (requests in flight will be cut), then start again with «model
  name»";
- stopped: button "Start it", hint "start the service with «model name»"; models whose file is missing
  are always disabled.

Regression tests: `TestModelPageSwitch` in `test_ui.py` (4) and `TestSetModel` in `test_supervisor.py`
(1).

## The window was blank (fixed)

Symptom: the window opens, the title bar is normal, the content area is blank.

Root cause: `neo/manager.py` used to read the page files from `Path(__file__).parent / "web"`. After
packaging, `__file__` points inside the PyInstaller archive, so `web/` is not next to it; the page
files are in the archive, and the console service's `WEB` pointed at a nonexistent directory — every
request 404, the window blank.

Fix: `neo/paths.py::web_dir()` — `sys._MEIPASS` (the PyInstaller extraction directory) wins, otherwise
`neo/web` next to the source.

## Packaging: `NeoStrata.exe` (fixed)

`python -m neo.build_exe` (spec `neo/NeoStrata.spec`) → `dist/NeoStrata.exe` (measured 2026-10-12:
7,181,111 bytes, 6.85 MB). The spec bundles `neo/web` (the page files) and `neo/engine_ref.json`
(the pinned source version) as data files; `neo.paths.web_dir()` and `neo.engine_setup.ENGINE_REF`
both read them from `sys._MEIPASS`. The exe is built with `--name NeoStrata` and is designed to sit in
the Strata root directory (the root resolution rule above is exactly for that).

```
python -m pip install pyinstaller      # only needed when packaging; users don't install it
python -m neo.build_exe                # produces NeoStrata.exe
python -m neo.build_exe --dest D:\Work\Strata
```

One single-file exe from `neo/NeoStrata.spec`, **designed to be dropped into the Strata root directory
and double-clicked**. Measured 2026-10-12 on Windows 11 / Python 3.13.15 / PyInstaller 6.22.3:
**16.8 MB**; after double-click the window appears (title Neo Strata), `Responding=True`,
`/neo/status` reaches `state=running` (`served_model=qwen3.8-flash-next`), and
`neo/state/instance.json` records `mode="window"`.

- **The exe's "root directory" is the directory the exe itself sits in** (`default_root()` in
  `neo/manager.py`; when running from source it is the directory above `neo/`). State, logs and window
  geometry all land in `<root>\neo\state\` (when the exe sits in a non-writable directory it falls back
  to `%APPDATA%\Neo Strata` — `pick_state_dir()` in `state.py`).
- **The exe carries no models, no engine, no torch / numpy**: Neo only starts `serve/server.py` with
  command-line arguments (`neo/engines/strata.py`); models, engine and `.venv` all live on the Strata
  side. Bundling them only makes the exe bigger and the versions mismatched.
- **The exe does carry pywebview and its WinForms backend** (`collect_submodules("webview")` in the
  spec: pywebview picks its backend by platform at runtime, and PyInstaller cannot see those imports)
  and the console pages `neo/web/` (read from the unpack directory at runtime, `paths.web_dir()`).
- **If the build environment is missing packages, packaging is refused and every missing package is
  reported at once** (`build_exe.missing_build_deps()`): with pywebview missing,
  `collect_submodules("webview")` is an empty list, PyInstaller silently succeeds, and the resulting
  exe has no window. Measured from the deployed exe's own log on 2026-10-12 19:58:
  `pywebview unavailable (No module named 'webview'), falling back to the browser interface` — the user
  sees no interface at all. The gate used to stop one package at a time, so in an environment with
  neither package installed (the deploy tree's `.venv` is exactly that) the message did not even mention
  pywebview, and you hit it again after installing pyinstaller; now one message lists every missing
  package with its install command (`python -m pip install -r neo/requirements-neo.txt`); if you really
  only want the browser interface, pass `--no-webview`. Measured both builds: with pywebview
  **13.2 MB** (73 `webview.*` modules bundled, including `webview.platforms.edgechromium` and
  `winforms`), without it **8.8 MB**.
- **If the page-script check fails, no package is produced** (`build_exe.web_gate()` →
  `neo/webcheck.py`): the pages are this exe's entire interface, and per-file syntax checks cannot see
  cross-file name collisions. Measured 2026-10-13: the deployed exe (13,981,433 bytes) was built
  successfully, installed, and its interface was dead (see "Every click was dead and the status stayed
  'Not started'"). Now `python -m neo.build_exe` checks `neo/web/` before invoking PyInstaller, and when
  it is not clean it names the two files, the two lines and the colliding name, and tells you to run
  `python -m neo.webcheck` first.
- **The exe is the windowed build (`console=False`), it has no console**: the QuickEdit path is bypassed
  entirely (see "How it starts"). In the windowed build `sys.stdout` / `sys.stderr` are `None`;
  `console_io.NullStream` catches them and all output goes to `neo/state/logs/manager.log`.
- **The exe needs Strata's `.venv`**: after packaging, `sys.executable` *is* `NeoStrata.exe`, so using
  it as "Python" means Neo keeps launching Neo itself. `python_executable()` in `state.py` therefore
  only accepts `<root>\.venv\Scripts\python.exe` and a system `python` inside the exe, and if neither
  exists it prints one line: "no Python found for running Strata: this exe needs the .venv in the Strata
  directory — run START-HERE.bat once in the Strata directory".
- The entry point is `neo/neo_entry.py`: the top of `neo/manager.py` uses intra-package relative
  imports, and when PyInstaller analyses a module inside the package as a top-level script it cannot
  resolve the parent package, so those imports get flagged "module not found"; hence an explicit script
  that imports the package, with `pathex` pointing at the repo root.
- **Killing it means killing two processes**: a PyInstaller single-file exe runs a bootloader process
  first, then starts the application process after unpacking. Measured 2026-10-12: after `Stop-Process`
  killed the bootloader (pid 42780) the application process (pid 26312) was still there, with service
  port 28180 and console port 28183 both open; killing the application process closed both ports. Neo's
  Job Object governs the Strata service Neo started, not this parent-child pair. Closing the window (or
  pressing "Stop" in the window and then closing) goes through `atexit` and both processes exit; if you
  kill it in Task Manager, kill both `NeoStrata.exe` entries.
- Packaging artifacts (`NeoStrata.exe`, `neo/dist/`, `neo/build/`) are not in the repository
  (`neo/.gitignore`).

## The deploy tree: Neo never changes upstream (`D:\Work\Strata`)

The tree that gets deployed is **a checkout of the upstream Strata repository** (`origin` =
`github.com/Niko1221/Strata`; measured 2026-10-13 at v0.1.43 / `7695859a`), not the development repo.
The rules:

- **Nothing is ever committed in the deploy tree**, and no second remote is added to its `origin` —
  upstream updates must stay a plain `git pull`.
- **Neo never modifies upstream files.** Everything Neo brings along is untracked: `NeoStrata.exe`,
  `neo/`, `NEO-START.bat`. At deploy time `neo/deploy.py` appends those three plus `strata/` (the
  engine source tree) and `strata-*.json.bak` (the model-config backups upstream's `setup.py` leaves on
  the user's machine — upstream's `.gitignore` only goes as far as `/strata-*.json`, and a pattern
  ending in `.json` does not match `.json.bak`) to the deploy tree's `.git/info/exclude` — git's local
  ignore file, part of no commit — so `git status` only ever shows upstream's own files, and Neo has
  touched no tracked file (upstream's `.gitignore` is a tracked file, hands off). On 2026-10-13 the
  local exemption patch previously applied to `serve/server.py` was removed (the old file is backed up
  in `neo/state/backups/server.py.v0.1.43.neo-patched.bak`, 393,602 bytes) and the upstream file
  restored (391,688 bytes). The gap left by removing the patch is filled by Neo itself: the monitor page
  now reads through the `/strata/` read-only channel (see "What the token covers").
- The check: `git -C D:\Work\Strata status --short --untracked-files=no` shows **no modified tracked
  files** (Neo's files are untracked by design and must not count as "dirty" — `git_dirty(...,
  tracked_only=True)` in `neo/engine_setup.py` exists for exactly this);
  `git rev-list --count HEAD..origin/main` and `origin/main..HEAD` are **33 and 0** — the pull is a pure
  fast-forward.
- **Version changes go through `python -m neo.deploy --to D:\Work\Strata`** (dry-run by default, `--apply`
  writes). Three gates, and if any fails nothing is written: ① engine — the deploy tree's HEAD must
  equal the pin in `neo/engine_ref.json` and tracked files must be clean; ② paths — only what is on
  Neo's manifest is written (`neo/**`, `NEO-START.bat`, `NeoStrata.exe`); `neo/state/`, `__pycache__/`,
  `build/`, `dist/` are not carried over; ③ process — if `NeoStrata.exe` from the deploy tree is
  running, refuse (the files are locked, a write would land half-done). After writing, every file's
  SHA-256 is verified, then `python -m neo.webcheck` and the quick suite run right there. First real
  run 2026-10-13 (dry-run, `--no-tests`): engine gate OK (HEAD `a0116b2782a5` == pin), process gate
  blocked by the pid 7276 recorded in `instance.json`, manifest new 7 / same 26 / update 30, and it
  reported an extra `neo/test_ui.py` sitting in the deploy tree (left by an old sync, `--prune` would
  delete it) — the trap of "only `.py` synced, `neo/web/` missed" is exactly how it surfaced.
- **Deploying = swapping `NeoStrata.exe` + syncing `neo/`'s source and pages**. The exe bundles the
  development tree's `neo/`; running source in the deploy tree (`python -m neo.manager`,
  `python -m neo.models`, running tests) uses the deploy tree's `neo/`, so both must be kept current.
  Measured 2026-10-13, this trap was hit: only `.py` was synced and `neo/web/` was not, so the deploy
  tree's source-run pages were still the dead interface with the `t` / `applyLang` collision —
  `python -m neo.webcheck` reported it on the spot in the deploy tree. After syncing,
  `python -m neo.webcheck` in the deploy tree is clean and the quick suite passes all 326 tests (using
  its own `.venv`).
- Packaging intermediates (`neo/build/`, `neo/dist/`) never enter the deploy tree.
- **Console printing**: any CLI `main()` that prints Chinese calls `wide_stdout()` first (the one shared
  copy lives in `neo/cli.py`), reconfiguring stdout to UTF-8 (`errors="replace"`). Measured 2026-10-13:
  `python -m neo.engine_setup --ensure` crashed at the print step on a cp936 console
  (`UnicodeEncodeError: 'gbk' codec can't encode character '\ufffd'`) — git's output decoded as UTF-8
  yields the replacement character `U+FFFD`, which GBK cannot encode. Now those CLIs print that line as
  `git : fatal: ...` on a cp936 console and exit 1, without crashing. Measured the same day, a second
  mode: when output goes into a pipe (`| Select-Object`, redirect to a file, capture in tests)
  `sys.stdout.encoding` falls back from the console's 65001 to the locale's gbk, GBK bytes are written,
  and the side reading as UTF-8 sees mojibake (`python -m neo.wizard --check` is fine in the console and
  garbled behind a pipe; the same command with `-X utf8` is fine). Since 2026-10-13 the `main()` of
  `wizard` / `models` / `model_settings` / `connect` / `start` calls it too;
  `neo/tests/test_deploy.py::TestWideStdoutShared` pins "the seven CLIs share the same function, and
  there is exactly one `reconfigure` in `cli.py`", and
  `neo/tests/test_model_settings.py::TestCliRootAndConsole` runs `main()` on a cp936 byte console and
  requires the text read back to be UTF-8 Chinese.

### The version-change walk (`neo/deploy.py` on the command line)

```
python -m neo.deploy --to D:\Work\Strata                      # plan only, writes nothing
python -m neo.deploy --to D:\Work\Strata --json               # the same plan, for machines to read
python -m neo.deploy --to D:\Work\Strata --no-tests           # plan without the test step
python -m neo.deploy --to D:\Work\Strata --apply              # really write
python -m neo.deploy --to D:\Work\Strata --apply --prune      # really write, and delete Neo files in the target that the source no longer has
# other switches: --from <dev tree> (default: the repo root the script sits in), --exe <NeoStrata.exe>, --ref <engine_ref.json>
```

The order is a fixed four steps, and you read each step's output before taking the next:

1. **dry-run**: `--to <deploy tree>`. Check the three gates, how many new / update / same entries the
   manifest has, which files the target has extra, and which lines are missing from
   `.git/info/exclude`. This step writes nothing; a non-zero exit code means a gate failed.
2. **stop the running exe**: the process gate looks at two things — the pid recorded in
   `neo/state/instance.json`, and any `NeoStrata.exe` in the Windows process table whose image path
   lands inside this deploy tree. Writing while a process holds the files locked produces a half exe, so
   this gate cannot be bypassed. Close the window (or `taskkill /PID <pid> /T /F`) and continue.
3. **`--apply`**: each file is written to a temp file in the same directory, its SHA-256 verified, then
   swapped in with `os.replace`; missing ignore lines are appended to `.git/info/exclude` (append only,
   never duplicated; dry-run only reports); after writing, the whole manifest's digests are verified
   again, and `python -m neo.webcheck` plus the quick suite run **inside the deploy tree** (`--no-tests`
   skips only the tests, never webcheck).
4. **afterwards**: `python -m neo.webcheck` in the deploy tree, the quick suite,
   `git -C <deploy tree> rev-parse HEAD` equals the pin, and open the window to look at the engine card
   on the Connect page.

All gates must be green before `--apply`; the script does not accept "the user says it's fine". A target
tree with no git, a HEAD that is not the pin, modified tracked files, an exe running — all four cases
write **not one byte** and say which gate stopped it.

## Tests: `python -m neo.tests.run_all`

Tests are split by test class, run in parallel (`--jobs`, default `min(4, CPU cores)`), each class
running in a separate process (unittest's `stream` is not thread-safe, so parallelism is process-level;
a failing class is re-run alone once, and the report says "re-run alone passed").

- `--quick` (17 modules that don't start a real engine process) is for running during development;
  the full suite is run before deploying.
- **Time standard (2026-10-13): running tests must report time, and the wall time of each unit and of
  the whole suite is capped at 30 seconds** (`BUDGET_S` in `neo/tests/run_all.py`, `--timeout` defaults
  to it; over-budget units are reported `OK(over budget)` and the whole run is judged a failure).
  Over budget means optimize, not raise the timeout.
- Measured 2026-10-13 on the same machine: `--quick --skip-network` **9.6 s / 435 tests / 86 units**;
  `--skip-network` **17.1 s / 523 tests / 101 units** (the two slowest units 5.7 s
  `test_models.TestCatalog`, 5.1 s `test_supervisor.TestCrashStops`).
- The one real-clone test (`test_engine_setup.TestRunAndPage`) is network-dependent: measured 6.8 s and
  145.3 s on the same machine the same day. The full run keeps it out with `--skip-network` (that is how
  the 30-second standard is held); to see it, run that class alone with a relaxed `--timeout`.
- When the real service cannot start (the `.venv` in the engine root lacks upstream's packages), the
  tests that use the real `serve/server.py` skip themselves and say which package is missing
  (`server_ready` in `neo/tests/_bootstrap.py`: it reads imports with ast, milliseconds, no process
  started). Measured 2026-10-13: with `jinja2` missing, that process dies at import, the supervisor
  restarts it through the backoff again and again, tests run out their waits, and the full run went from
  16.3 seconds to 5 minutes — this skip rule was added for exactly that.

```
python -m neo.tests.run_all --quick --skip-network   # during development: 17 modules, 9.6 s
python -m neo.tests.run_all                          # before a version change: 20 modules, includes the clone test
python -m neo.tests.run_all --no-split --jobs 1      # the old way: modules in series
```

523 tests (Windows 11, Python 3.13.15, real processes started, no GPU needed, no installed model
needed, real user configuration never touched; the only test that really uses the network is the clone
path in `test_engine_setup.TestRunAndPage`). The mock-server path saves the 3.5 seconds the real
`serve/server.py --engine mock` needs before it first answers `/health` — process start is 0.27 s in
both cases. `TestStartAndReady.test_start_to_running` uses the real `serve/server.py --engine mock`:
what it verifies is "upstream's ready line really lands in the log", which is the Neo/upstream seam and
cannot be vouched for by a fake service.

All the cost sits in the modules that start real processes and build real git trees, and the combined run
is slower than the sum of its parts — measured (2026-10-13, same machine): `test_engine_setup` and
`test_deploy` alone are 8.4 s and 5.0 s, and inside the full suite they took tens of minutes — each of
their cases does a `git init` + one commit in a temp directory, and when Windows Defender scans the temp
directory a single `git init` takes tens of seconds. That is why these two live in the quick suite during
development and the full suite is reserved for before a version change. Waits that measurements changed
(2026-10-13): `neo_console_info` probes TCP once first (0.05 s) before sending the HTTP request, so a
port nobody listens on no longer waits out a 0.8 s timeout; the console thread runs
`serve_forever(poll_interval=0.05)`, and `stop()` went from 0.51 s to 0.06 s. In the supervisor tests
`poll_s` 0.25→0.05, `probe_s` 0.4→0.1, `wait()` asks every 0.05 s — waiting on a condition, not on a
number of seconds: `test_connect_api` 39.6→6.7 s, `test_console` 34.7→9.1 s, `test_supervisor`
70.3→56 s. The same units serialised add up to 66.1 s, and 19 modules in one `python -m unittest` run
took 1067.1 s = 17.8 minutes (492 tests, the pre-parallelism number). After class-level parallelism the
two slowest classes: `TestCrashRestart` 29.3→**7.0** s, `TestExternalService` 42 s (flaky under
concurrency)→**2.6** s — deleting a test case would not have helped; what was slow was that "noticing
the process died" waits for a full probe round. (That cell later became `TestCrashStops`: 5.1 s after
the backoff was deleted. The slowness was fixed on the way: `_wake` stayed set once raised, so the
`_sleep` in the port-watching layer returned immediately and that layer became a busy-wait that probes
over and over, the supervision thread holding the GIL and slowing the status page's polling in tests;
`_sleep` now clears the flag when it wakes, and the same cell went from 12.2 s to 5.0 s, measured
2026-10-13 on this machine.)

### What each module pins

- `test_supervisor`: start to ready, event records, **no auto-restart** after an unexpected exit (exit
  code + one diagnosis + "stopped, press Start to bring it back" in the event), pressing Start after a
  crash-and-stop really starts a new process, a user stop is not undone by a relaunch, taking over a
  service already running, diagnosis keywords, and **the status page's metrics are real numbers** (real
  process: `requests` is a cumulative count, no lists or objects among the values).
- `test_engines`: config discovery and ordering (the `touch()` semantics of `setup.py`), start commands
  using only parameters upstream already has, refusing to start when the model file is missing, the
  bootstrap state when there is no config, the mock adapter; **the shape of real configs** (flat `args`,
  model under `--native`, context under `--max-context`, no `path` key), **model present and engine exe
  missing must be flagged and the start refused**, and when no model is named a config that can start is
  picked; **the four `/metrics` sections flattened into card keys** (`TestMetricsFlatten` fed the real
  payload: VRAM 24028 / 24455 MiB, RAM 70.6 / 93.6 GB, `requests` is 1 not a list, no invented
  generation speed when idle); **the monitor page address goes through the console**
  (`test_pages_monitor_goes_through_the_console`: with a `console_port`, `pages["monitor"]` is
  `http://<host>:<console>/strata/#monitor`; without one it goes straight to upstream; the chat page
  always goes straight to upstream).
- `test_upstream_api`: the `serve/server.py` parameters Neo uses (`--engine/--config/--port/--host/
  --api-key/--gpu`), the fields of `/health`, the **four sections of `/metrics` and the keys inside
  them** (`live` / `totals` / `hardware` / `hardware_static`), the key only gating inference routes,
  the read-only monitoring endpoints answering loopback peers, and Neo's patch markers staying absent.
  Measured 2026-10-13: the four `/metrics` sections and the keys inside them (`live` / `totals` /
  `hardware` / `hardware_static`, plus `cpu` / `ram_used` / `mem_used` in `serve/telemetry.py`), the chat
  and monitor page files, setup.py's M2 parameters, and upstream's `requirements.txt` untouched. When
  upstream renames something, this test rings first — not the user's Neo failing to start.
  **`test_the_monitor_channel_needs_no_upstream_change` watches the reverse**: `serve/server.py` must not
  contain the local-exemption patch markers (`loopback_peer`, `_authorized(self, monitor: bool = False)`,
  `_authorized(monitor=True)`), while the monitor page and those read-only endpoints must still be there —
  the monitor page reads through Neo's `/strata/` channel, not through a patched upstream.
- `test_console`: the status endpoint, static pages served `no-store`, an engine that cannot start
  staying stopped with a reason (no auto-restart), foreign Origin and foreign Host 403, settings persisted
  without echoing the API key; **the page's "open" buttons go through the service** (`POST /neo/open` with
  a page, `POST /neo/open-log`, 503 when there is no shell, foreign Origin 403 with the shell never
  called); **the second-instance message** (window mode names the pid and the interface address, browser
  mode says "no window" and gives the pywebview install command, and when the recorded process is gone it
  says "it is already gone" and clears the stale record); **the model-settings group**
  (`TestModelSettingsEndpoint`, 10): the form shape of `GET /neo/model/settings`, `?model=` accepting both
  id and file name, the first config when nothing is named, a named-but-unknown model listing the menu and
  **writing nothing**, `POST` writing the override config from the exact form `app.js` sends (`--max-context`
  and `--vision` land, an unpaired `--kv-resident` is removed), out-of-range answers `ok:false` with an
  explanation and **the config bytes untouched**, `action:"reset"` deleting the override with the upstream
  file untouched, things that need a restart answering `restart_required` with the pid; **the `/strata/`
  read-only channel group** (`TestStrataChannel`, 9, against a fake upstream): the page and its js/css all
  proxied, panel endpoints answered with the key Neo attaches server-side and the response carrying no key,
  query strings passed through verbatim, `POST` always 405, `/v1/chat/completions` never proxied, paths
  outside the whitelist never proxied, upstream down → 502, `/strata` not shadowing Neo's own `/neo/*`, and
  the channel's path constants matching the monitor address `engines/base.py` produces.
- `test_ui` (65): the gate's `write()` returning immediately when another thread holds the lock, counting
  the dropped lines and saying so, normal output untouched when nobody contends; `state.log()` appending in
  the normal case (20 lines = 20 writes, 0 `\r`), trimming only past `LOG_MAX` and keeping the tail, 6
  threads writing 90 lines concurrently with nothing lost and nothing interleaved; the watchdog not
  starting without a window, thread snapshots bounded and carrying thread names, "no answer" plus the
  `neo-ui-probe` frame in the log when `evaluate_js` is frozen and no log line when the window answers
  normally; `window_kwargs()` containing no `js_api` (x/y only when coordinates exist), `app.js` containing
  no `window.pywebview` and carrying the `/neo/open` and `/neo/open-log` routes; **`app.js`'s braces
  balancing with `render`/`poll`/`ensureFrame`/`num`/`count` at top level, and the trailing `poll();` and
  `setInterval(poll, 2000);` at top level** (the measured incident where the embedded frames sat at
  `about:blank` was code swallowed into a function body); **the six status cards passing only through
  `num()` / `count()`** (things like `m.requests` must not be interpolated straight into template strings;
  `count()` answers "—" for lists and objects); **packaging is refused when the build environment lacks
  packages**, with the install command given, `--no-webview` really reaching `build(allow_no_webview=True)`,
  and both packages missing listed in one message (`test_build_names_every_missing_dep_at_once` — the
  deploy tree's `.venv` is exactly such an environment); **the language/theme group**
  (`TestLanguageAndTheme`, 9): `i18n.js` syntax balanced, `BLOCKS`/`PHRASES`/`ATTRS` complete with
  `applyLang`/`applyTheme`/`init` at top level, every `data-i18n` key in `index.html` present in `BLOCKS`
  (a missing key means that block never switches), `data-i18n` never wrapping a control (wrapping one
  erases the dropdowns and buttons), every `t("中文")` literal in `app.js` present in `PHRASES`, the
  Chinese `placeholder`/`title` strings in the HTML present in `ATTRS`, the two top-bar buttons and
  `.nav-prefs` present in HTML and CSS, and **no hard-coded colours in any rule outside `:root` and
  `html[data-theme="light"]`** (one component rule hard-coded `#fff` and became a white patch in the light
  theme); **the two `<script>` files must not declare the same top-level name** (top-level names chained in
  `index.html`'s load order — the incident where the status stayed "Not started" and every click was dead
  was `t` colliding with `applyLang`).
- `TestWebCheck` inside `test_ui` (8): `neo/webcheck.py`'s check and the packaging gate — a clean real tree
  passes `build_exe.web_gate()`; removing i18n.js's IIFE makes the gate refuse and name the two files, the
  two lines and the colliding name; when node exists the concatenated program is handed to V8 and the
  report quotes the browser's own words (`already been declared`), and when node is absent the check must
  not pretend it ran; a missing brace is reported, a missing top-level `poll();` and `setInterval(poll` is
  reported, a `<script>` pointing at a missing file is reported, and more than 40 top-level names must be
  collected (otherwise this check is not running at all).
- `test_logging` (25): 0 `\r` in the log after 8 consecutive writes (the CRLF-amplification regression), a
  200,000-character single line trimmed with its original length marked, cleaning the tail without losing
  history; `tail_text` stripping `\r` and control characters, capping line count and marking "N lines
  omitted"; the gate bounded under an 1.8-billion-character flood, reporting what was folded and how much
  was dropped, normal output untouched; **the 40 lines of one print passing as one whole block** (the CLI's
  JSON not sliced in half), **a second 40-line block inside one second dropped for exceeding the
  allowance**, a 500-line one-line-per-write flood still compressed under 120 lines, **`install()` wrapping
  only streams whose `isatty()` is true** (pipes and `StringIO` untouched) and being idempotent;
  `clean_line` behaviour; `RecursionError` never formatted into a traceback, ordinary tracebacks capped in
  lines and total length, the excepthook installing; `safe_resolve` truncating a 5000-segment `.SyncRoot`
  path to under 64 segments, relative paths and `..` behaving, `is_reparse` answering False for a normal
  directory; `instance.json` recorded / read back / cleared, only `mode` patched without erasing `root`, a
  record whose process is gone counting as stale.
- `test_model_settings` (38): one test for each of the three traps (turning images off must delete the
  section too, `--kv-resident` must be deleted together with its value, `"off"` is not truthy); plus the
  shape of `view`'s form (`model_id`/`config_name`/`fields`/`options`/`estimate`/`overridden`/
  `restart_required`), override configs landing in `neo/state/configs/` with the upstream file's bytes
  untouched, `clear` returning to the original, keys identical to upstream never written into the override
  (that copy being deleted), `config_path_for` pointing at whichever file exists, the VRAM estimate (q4_0 16
  units = 128 MiB, int8 same shape = 234 MiB, KV residency off = 0), the GGUF header (a synthetic file and
  the real `qwen3.8-flash-mlx-32b-ikd-q4.p1.gguf`'s 48 layers, corrupt and truncated files not raising,
  reading the header never exceeding 1 MiB).
- `test_netprobe` (12): the injected `connect` recording address family / port / timeout, all four targets
  going over 443; all reachable recommending the default source, HF down + ModelScope up recommending
  `--source modelscope`, only the mirror up only mentioning `HF_ENDPOINT` without switching source, nothing
  reachable recommending nothing; `github.com` only feeding the "engine source tree reachable" column and
  never the model-source recommendation; cache read/write with a corrupt file falling back to empty,
  `probe_cached` not touching the network, `describe` stating the recommendation.
- `test_tokens` (16): the key shape `sk-neo-` + 32 hex characters with 100 generated all distinct, the mask
  never containing the full key, a cache hit on the same base returning the same key with the file storing
  plaintext (the interface only ever echoes the mask), `forget_key` deleting the cache, the firewall hint
  absent for loopback and present for `0.0.0.0` with the port, `base_for` mapping `0.0.0.0` to this
  machine's address, `looks_like_key` accepting both `sk-neo-` and `sk-` shapes.
- `test_connectors` (26): the four agents' landing points honouring `CLAUDE_CONFIG_DIR`/`CODEX_HOME`/
  `XDG_CONFIG_HOME`/`OPENCLAW_HOME`; **merging** (the user's `permissions` / `[model_providers.other]` /
  `provider.other` / `models.providers.other` all intact after Neo wires in); **idempotence** (run twice,
  one provider block, the values inside following a new port); **backup first** (`.neo-bak` holds the
  pre-wiring content); **broken files untouched** (truncated JSON answering "your file was not modified",
  bytes identical); Claude's base **without `/v1`** and its unset list containing `ANTHROPIC_API_KEY` /
  `CLAUDE_CODE_OAUTH_TOKEN`; Codex's `wire_api = "responses"` and `requires_openai_auth = false` with the
  profile layer; OpenCode's `neo/` prefix and `output_limit = 32000` (window 131072) with the compaction
  reserve; OpenClaw's `api = "openai-completions"` and `primary = neo/<id>` with `memory.search` and
  `shellEnv` switched off; with no key the unset list is empty and a "create a key first" hint appears;
  both the PowerShell and the POSIX command blocks present.
- `test_wizard` (26): the command is `setup.py --yes --no-start` with the port passed along, `--no-start`
  mandatory (the service belongs to the supervisor), `--host` only present with `--host 0.0.0.0` and
  carrying the key, `--source` added only when modelscope was probed, never changed when the user named
  hf, the source placed before `--yes`; among the four steps "LAN without a key" counts as not ready, and
  a missing `setup.py` is stated; a real run (a stub `setup.py` in a temp directory) lands its command line
  and output in `logs/wizard.log`, and nothing runs when `setup.py` is missing.
- `test_connect_api` (28): `GET /neo/net` not touching the network (fed from cache), `GET /neo/lan`
  reporting the binding and mask with no firewall hint on loopback, and giving LAN addresses plus a
  port-carrying firewall command when bound to `0.0.0.0`; `POST /neo/token` landing the generated key in
  `launcher.json`, in the key cache and with the supervisor, the response carrying the full key and the
  mask; `{"action":"reveal"}` handing back **the same key** with the bytes of `launcher.json` and the key
  cache untouched, plainly saying "no token" when there is none, and only the mask for non-loopback peers
  (`api_key` returning an empty string); `unset` clearing the key on the service while the cache keeps it,
  `forget` deleting the cache, a bad action writing nothing and generating no key; `GET /neo/connect`
  returning all four recipes and **writing no files**, `POST /neo/connect` writing into a temp directory
  and leaving backups, an unknown agent answering an explanation; `GET /neo/wizard` returning the four
  steps with the probed source recommendation; `GET /neo/catalog` listing the menu and this machine;
  `POST /neo/install` running the print-only fake `setup.py` in a temp directory (command line and output
  into `wizard.log`), refusing items not in the menu / not runnable in the plan / already running, without
  starting a subprocess; the download-source row: four options present, the remembered source entering the
  install command, the mirror only adding `HF_ENDPOINT` and no `--source`, an unrecognised source refused
  and `source.json` not written; `?fresh=1` not holding up the probe; **the five read endpoints write not
  one file**.
- `test_connect_cli` (8): `list` naming the four agents and each one's config file, an unknown agent
  returning 2 and listing the choices; `--json` emitting the whole recipe (base, env, unset list, commands,
  the PowerShell and POSIX blocks) and **creating no directories**, and with `--key` given writing no key
  cache; `--base` overriding `--host`/`--port` (it is what lands in `config.toml`'s `base_url`); the key's
  origin stated, with a reminder when the service has no key; `--persist` writing only the two files in the
  temp directory, each with a `.neo-bak` holding the user's original content, and running twice leaving
  exactly one `[model_providers.neo]` block.
- `test_paths` (14): `strata_root()`'s five-step order (a directory with `setup.py` + `serve/server.py`
  beats `<app>/strata`; `<app>/strata` beats the app directory itself; `NEO_STRATA_ROOT` beats both;
  `--root` beats the environment variable); the root the CLI resolves is the same directory the
  `Manager` uses; **the CLI's default root is not "one level above neo/"** (after the repo split that
  parent is not the engine root; `--check` must not print a command pointing at a non-existent
  `<repo root>/setup.py`); `--root` overriding the default; the report naming which rule matched.
- `test_engine_setup` (14): the pin file's four fields, cloning the pinned tag (a local `file://` tree
  carrying a tag), **the digest gate** (a zip with a recomputed SHA-256 unpacks; a zip that does not match
  is refused and the target directory stays empty), a missing `engine_ref.json` stated, a missing zip
  telling you to run `--fetch`, `--fetch` only downloading and checking the digest without unpacking,
  `--ensure` unpacking after a hit, a source tree with `setup.py` and `serve/server.py` saying "no need to
  fetch", the clone command carrying `--branch <tag> --depth 1` and **not containing the user's
  credentials**, the digest gate refusing a zip whose content changed, and the clone's return code
  landing in the log.
- `test_deploy` (29): one test per gate — engine (HEAD == pin and clean to pass; a HEAD that is not the
  pin blocks; dirty tracked files block; **no git does not count as clean**; **the deploy tree's own
  untracked Neo files do not count as dirty**), paths (`neo/state/`, `__pycache__/`, `build/`, `dist/`
  never carried; the exe not in the source manifest; every file classified new / update / same; extra
  files in the target reported), process (a missing exe blocks, a running instance blocks); dry-run
  writes not one file; `--apply` verifies SHA-256 file by file, refuses a tree whose gates fail,
  `--prune` deletes only "Neo files the source no longer has"; `--json` is JSON and states whether gates
  passed; the report names the gates and the counts; omitting `--to` is an error; a pin that does not
  match the target tree blocks and is reported. The local-ignore entry (`.git/info/exclude`) has six
  tests: missing lines appended in one pass, the second run writing not one byte (no duplicates), dry-run
  only reporting, a non-git target answering "nowhere to write", and the appended lines making
  `git check-ignore -v` really treat `strata/` and `strata-q2_0.json.bak` as ignored (the backup entry is
  a file upstream's `setup.py` writes; upstream's `.gitignore` only covers `/strata-*.json`); the
  `--apply` test also verified upstream's `.gitignore` byte-for-byte unchanged. One console-printing
  test: stdout swapped for a cp936 byte stream (`BytesIO` + `TextIOWrapper`, `reconfigure`-able like a
  real console), printing a line containing `U+FFFD` — without widening it throws `UnicodeEncodeError`,
  after widening, reading it back as UTF-8 gives `git : fatal: ...`.
- `test_harness` (7): two rules about the tests themselves — the time standard (`run_all.BUDGET_S` is 30
  seconds, `--timeout`'s default is that value, every run must report seconds and the "over budget" line)
  and the skip rule (when the real engine service cannot import, `server_ready` names the missing package
  and points at `requirements.txt`; pure-stdlib imports never count as missing; a directory with no engine
  says "no engine"; the current engine root must either start or give a reason — "cannot start, no
  explanation" is not allowed).
- `TestEngineEndpoint` in `test_console` (6): `GET /neo/engine` returning the 13 keys the page reads, the
  three step names, the two routes wired inside `console_server.py`, an unrecognised action refused,
  `install` running on a background thread (`wait=False` — a click must not block the request thread for
  tens of minutes) with `setup=True` (the source tree handed to `setup.py`, Neo never installs by hand),
  host and key passed down from the supervisor; netprobe and `run_engine` both faked — not one 443
  connection, not one clone. The two tests added 2026-10-13 pin the wiring itself: `start()` must attach
  to the httpd every name the Handler uses via `self.server.X(...)` (miss one and socketserver swallows
  the exception in `handle_error`, the connection closes, the page only sees a failed fetch — the engine
  card was spinning, `curl` reported `Empty reply from server`), and **a real HTTP round trip** of
  `GET /neo/engine` (previously only `console.engine_status()` was called directly, bypassing the httpd,
  so a missing attachment never rang).
- `test_manager_cli` (4): the root `Manager` uses is the root `resolve_root()` computes; `--root`
  overriding it; the report naming the rule; the root written into `launcher.json` is that root.
- The Connect page checks inside `test_ui`: the page's five rows and the two buttons, `app.js`'s
  `renderConnect` reading the fields of `/neo/wizard` and `/neo/connect`, the token button's
  `{"action":"reveal"}`, the copy button's `navigator.clipboard` with a `textarea` fallback, the
  `lan-key-value` row, the four agent cards' copy commands, the download-source row reading
  `/neo/wizard`'s `source_options` and `source_recommendation`, the install button's
  `POST /neo/install` with `model_id` and `source`, the wizard log's `POST /neo/open-log` with
  `wizard.log`; **the token's full string never appearing in the HTML** (the page only gets the mask, the
  full string only from a loopback `reveal`); **the install button's debounce** (the button disabled
  before the request goes out, `renderConnect` not re-enabling it, `renderConnect` called after the
  response, the button re-enabled in the `finally` branch).

## Manual verification on the exe path

The exe path is verified by hand (2026-10-12, Windows 11): `NeoStrata.exe --engine mock` opens a window →
`/neo/status` reaches `running`, `Responding=True`, `neo/state/instance.json` records `mode="window"`;
opening a second instance puts the message with pid / root / interface address / `--force` in the log;
closing the window makes `atexit` erase `instance.json`. Re-verified after the js_api bridge was removed:
window open for 95 seconds with `Responding=True`, 0 "no answer" in the log, `POST /neo/open` and
`/neo/open-log` both answering 200 and really opening the page / file; the same exe placed in
`D:\Work\Strata` running `--no-start`: the model page lists 4 models.

The five Connect endpoints were verified one by one inside the exe (2026-10-12,
`NeoStrata.exe --engine mock --port 28180`, console 28183): `/neo/status` `state=running` `healthy=True`
`pid=6912`; `/neo/lan` says "bound to loopback only (127.0.0.1)", `api_key_masked` empty, `lan_bases`
giving `http://172.18.240.1:28180` and `http://192.168.0.165:28180`; `/neo/net` with all three targets
reachable (huggingface.co 228 ms / hf-mirror.com 382 ms / www.modelscope.cn 67 ms), recommended source
`huggingface`; `/neo/connect` with all four recipes, `base=http://127.0.0.1:28180`, model `mock`, context
32768; `/neo/wizard` with all four steps OK, the command
`...\python.exe ...\setup.py --port 28180 --yes --no-start`. On the command line,
`python -m neo.wizard --check` and `python -m neo.connect list` print in full on the same machine (the
gate no longer drops whole blocks — see "Logs and console output", item 5).

The "copy token" path was verified one step at a time on a mock shell (2026-10-13, Windows 11 /
Python 3.13, `ConsoleServer` + `MockEngine`, Neo's state directory pointed at a temp directory): at the
start `/neo/lan` says there is no key; `POST /neo/token` produces `sk-neo-23fb810c80e7629c12f0d26dd16c7f6e`
(39 characters), `/neo/lan` then reports only `sk-neo-23fb810c… (39 characters)`; `{"action":"reveal"}`
hands back the same key, twice in a row; when the peer is not this machine (calling
`make_token(..., loopback=False)` directly) `api_key` returns an empty string and only the mask; after
`unset`, `launcher.json` holds an empty string while the key cache still holds 1 entry; in the pages `/`,
`/app.js`, `/style.css` the strings `lan-copy-key`, `lan-key-value`, `{"action":"reveal"}` and `.keyval`
are all present.

Token coverage was verified in the exe too (2026-10-13, Windows 11,
`NeoStrata.exe --engine mock --port 28190 --console-port 28193`; exe 13.3 MB, `instance.json`
`mode="window"`, the window really up): `POST /neo/token` produces `sk-neo-cb2cc50c…` (39 characters) →
`POST /neo/stop` → `POST /neo/start`, and the service log's command line becomes
`... serve/server.py --engine mock ... --port 28190 --api-key sk-neo-cb2cc50c606104ccabdaec5a7da32588`.
After that, on this machine `GET /metrics` without a key answers **200** with all five groups
`live`/`totals`/`hardware`/`hardware_static`/`requests` (exactly what the monitor page's cards need),
`/status` and `/slots` answer 200 without a key, `Host: evil.example.com` **401**,
`POST /v1/chat/completions` without a key **401** / with a key **200**, `GET /v1/models` without a key
**401**, `POST /settings` without a key **401**, `/health` reports `api_key=true`; after
`{"action":"unset"}` the `api_key` in `/neo/lan` is empty. `/api/requests` is 404 on this machine — that
route only exists when `svc.api_monitor` is on, which has nothing to do with authentication.

Force-killing the exe (`Stop-Process -Force`) does not run `atexit`: `neo/state/instance.json` stays, and
the supervisor process and the service process each stay alive (measured: pid 16536 listening on 28193 and
pid 42336 listening on 28190 left behind) — you have to take the ports and `instance.json` together;
closing the window normally clears all three on its own.

## Measured pitfalls on pywebview 6.2.1

- `create_window()` does not accept `min_width` / `min_height` / `confirm`: the window parameters are
  `min_size=(w, h)` and `confirm_close`; `private_mode` and `storage_path` belong to `webview.start()`.
  A wrong parameter throws `TypeError` at startup; `neo/manager.py` recognises that exception and falls
  back to a minimal parameter set.
- `webview.start(icon=...)` with an SVG makes the Windows backend's `System.Drawing.Icon.Initialize` throw
  `ArgumentException`, crashing on the window thread (Neo's icon is an SVG, so no icon is passed).
- With `confirm_close=True`, `window.destroy()` pops a confirmation dialog and blocks: the page's "quit"
  turns `confirm_close` off before destroying.
- When a real page opens, pywebview writes a string of WinForms font-recursion internal logs to stderr
  (`window.native.ActiveControl...GenericSansSerif`); window function is unaffected, it is noise.

## End-to-end check

`python neo/tests/manual_end_to_end.py` really starts the desktop shell once (mock engine, service port
28080, console 28083): ready in 2 seconds (`state=running`, `healthy=true`, `max_context=32768`), the
status / model / log endpoints all answering; then the shell process is killed to simulate being ended by
Task Manager, and 3 seconds later service port 28080 is closed — the Job Object took the Strata service
with it, no leftover process. The browser fallback path was verified the same way (`--no-webview`, console
28093, `state=running healthy=true`, port closed after killing the shell).

## Known boundaries

- **The window is a container; all actions go through the service.** The page has no Python bridge
  (`js_api` removed, see "Clicking made the window '(Not responding)'"); the window thread only pumps
  messages. The watchdog only writes logs and does not fix the window.
- **The windowed run is not covered by tests** (pywebview needs a real window), so the window path is
  guarded by three layers: `neo/webcheck` (static checks of page scripts and endpoints), the
  `TestClickPathIsHttpOnly`-style tests (pinning "no bridge, all actions through the service"), and
  `neo/ui_watchdog.py` (writes a log when the window thread stops pumping messages).
- **The agent connector writes real config files** (`~/.claude/settings.json`, Codex's
  `config.toml`); it only merges keys and never overwrites the file, and backs up to
  `neo/state/backups/` first.
- **The model download is a detached subprocess**; the window only shows the tail of the log.
- **The `/strata/` proxy only forwards GET/HEAD and only the fixed path menu**; it does not forward
  inference routes, and the page carries no API key.
- **The engine source tree is pinned to a tag**; it is cloned/extracted only after the SHA-256 matches
  the pinned value.
- Changing the port needs a Neo restart to take effect (M1 has no hot switch).
- The chat and monitor pages are embedded upstream pages. **The monitor page needs no key on this
  machine**: the read-only endpoints like `GET /metrics` are reached through Neo's `/strata/` channel (see
  "What the token covers"), and the window's cards keep producing numbers after a token is set. The chat
  page needs a key — "Generate token" on the Connect page exists for exactly that; the key is shown in
  full only on the one generation, after that the UI only has the mask (the "Copy token" button hands back
  the current one).
- Opening the LAN, generating a token and wiring the four agents all live on the Connect page; **the
  default is still `127.0.0.1` only, with no key**.
- The firewall command is **given, never executed** — the user runs it with administrator rights; without
  it, other machines on the LAN cannot connect.
- The wiring recipes write the agents' config files, and Neo **does not check whether the agent is
  installed**: with the agent absent the config file is still written correctly and works once the agent
  is installed; when an agent changes its config format (Claude / Codex / OpenCode / OpenClaw all move
  fast), Neo follows the reference implementation's shape, and a format change means the recipe must
  change with it (`test_connectors.py` is what watches that shape).
- When the wizard runs `setup.py` it does not stream output; the page reads the tail of
  `logs/wizard.log` every 5 seconds, and only one install may run at a time.
- With no model installed the onboarding page shows (no `strata-*.json` in the root); the Connect page's
  wizard brings `setup.py`'s checks and install process into this window.
- `neo/state/` (runtime data) and `neo/state.py` (the module) share a name: running a script from inside
  the `neo/state/` directory makes `import neo.state` hit the directory instead of the module (measured:
  `python neo\state\repro\hook.py` reports `AttributeError: module 'neo.state' has no attribute
  'install_excepthook'`). Run Neo's code from the repository root; the repro scripts under
  `neo/state/repro/` insert the root into `sys.path` themselves.

## For maintainers

- The design doc is the source of truth for behavior; code and doc are updated together.
- Measured numbers must carry the machine and date they were measured on; no conclusions without
  measurement.
- Upstream changes go through the fork (`neoneo-ai/Neo-Strata`, renamed `strata-upstream`);
  `neo/tests/test_upstream_api.py` watches upstream's shape, and Neo's patch markers must stay absent.
  The fork carries **no Neo code** - two copies of the same code drift, and they had: measured 2026-10-14,
  that tree's `README.md` was Neo's README (upstream's own README had been overwritten by it, and it still
  said the supervisor auto-restarts a crash, a behavior removed 2026-10-13). Commit `9c4d68f1` deleted
  `neo/`, `docs/NEO_DESKTOP.md` and `NEO-START.bat` there - 69 tracked files - and put `README.md`,
  `AGENTS.md` and `.gitignore` back to `v0.1.43.1`. Measured after the removal: `git ls-files` in that tree
  has 0 paths containing `neo`, and `git diff v0.1.43.1 HEAD` is 3 upstream-side doc files
  (`docs/AI_SETUP.md`, `docs/DETAILS.md`, `docs/MODELS.md`); `serve/`, `setup.py`, `src/` and `include/`
  already matched the pin. Neo's code and docs live in this repository only.
- Docs style: plain words, measured numbers with the machine they were measured on, no conclusions
  without measurement.



