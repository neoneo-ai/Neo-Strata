# Neo - the desktop layer over Strata

[English](README.md) · **中文** ([README.zh-CN.md](README.zh-CN.md))

<p align="center"><b>Run a 125-billion-parameter model on your own gaming PC — and drive it from one window.</b><br>
NVIDIA or AMD graphics card (12 GB or more) · Windows · free and open source</p>

<p align="center"><a href="https://github.com/Niko1221/Strata/releases/download/v0.1.10/Pagoda.mp4"><img src="https://raw.githubusercontent.com/Niko1221/Strata/main/docs/media/pagoda-preview.webp" width="720" alt="A voxel pagoda garden that the model Strata runs wrote, shown in the browser"></a><br>
<sub>A voxel pagoda garden, 1 shot prompt, running on an RTX 5070 with Strata (IQ3_S, 128K context) ·
<a href="https://github.com/Niko1221/Strata/releases/download/v0.1.10/Pagoda.mp4">full video (49 s)</a></sub></p>

## The speed you get

These are upstream Strata's own measurements on two ordinary gaming PCs. A token is about ¾ of a word.
**Writes answers** is how fast a reply appears in a short chat — 60 tokens per second is faster than you
can read. **Reads your prompt** is how fast it takes in what you send (here a 32K-token document, code
or chat history).

| Model size | RTX 5070 · writes | RTX 5070 · reads | RX 9070 XT · writes | RX 9070 XT · reads |
|---|---:|---:|---:|---:|
| **Q2_0** | 94 tokens/s | 2,650 tokens/s | 60 tokens/s | 1,160 tokens/s |
| **IQ2_XS** | 79 tokens/s | 2,090 tokens/s | 52 tokens/s | 1,110 tokens/s |
| **IQ3_XXS** | 62 tokens/s | 1,750 tokens/s | — | — |
| **IQ3_S** | 53 tokens/s | 1,620 tokens/s | — | — |
| **Coder** | 55 tokens/s | 2,180 tokens/s | 44 tokens/s | 1,420 tokens/s |

RTX 5070 (12 GB) with a Ryzen 5 7600 and 64 GB RAM; RX 9070 XT (16 GB) with a Ryzen 9 3900X and 47 GB
RAM. Q2_0 was measured on engine 0.1.36, the other rows on 0.1.26 (4K answers, 32K prompts). A card
with more VRAM is faster: an RTX 3090 (24 GB) should write about 100-140 tokens per second. Full tables
from the same runs: Strata's
[DETAILS.md](https://github.com/Niko1221/Strata/blob/main/docs/DETAILS.md#speed-measured),
[speed of each size](https://github.com/Niko1221/Strata/blob/main/docs/MODELS.md#how-fast-is-each-size),
[community results](https://github.com/Niko1221/Strata/blob/main/docs/COMMUNITY_BENCHMARKS.md).
Neo does not re-measure them: it ships the pinned engine source (today `v0.1.43.1`) and shows the
numbers the running engine reports.

## What Neo adds

A window over that engine: process supervision (**no auto-restart on crash** — it records the exit code
plus one diagnosis line and waits for you to press Start), status / log / chat / monitor pages, plus two
extra jobs — "install a model" and "connect a coding agent". It talks to Strata only through upstream's
existing command-line arguments, environment variables and HTTP endpoints, and **never touches a single
upstream file**. The default is `127.0.0.1` with no API key; token, LAN address and agent config files
are all opt-in.

Upstream is [Niko1221/Strata](https://github.com/Niko1221/Strata) (the `upstream` remote of this repo).
Engine source only ever comes from a GitHub tag: it is pinned in `neo/engine_ref.json` (today
`v0.1.43.1` = `a0116b2782a5b82622358c04a06496485d12b2d6`).

> Full design doc (with measured numbers): [docs/NEO_DESKTOP.md](docs/NEO_DESKTOP.md)
> (Chinese: [docs/zh/NEO_DESKTOP.zh-CN.md](docs/zh/NEO_DESKTOP.zh-CN.md)).
> Development conventions: [AGENTS.md](AGENTS.md).

## Quick start

### Strata already on this PC? Point Neo at it — nothing gets copied

```bash
python -m neo.engine_setup --set-root D:\Work\Strata   # remember the directory once (neo/state/launcher.json)
python -m neo.manager                                   # open the window
```

That is the whole setup. From then on every Neo command and the window use that directory. Per-run
alternatives: `python -m neo.manager --root D:\Work\Strata`, or `set NEO_STRATA_ROOT=D:\Work\Strata`.
In the window it is the Connect page's "use an external engine directory" box. Neo runs an existing
install **without writing anything into it** — its own state stays in `neo/state/`. If that tree's HEAD
is not the pinned tag, Neo says so and leaves it alone; moving its HEAD takes an explicit `--align`.
Deploying (below) is not how you use an existing install — it is how you ship the windowed exe into
that tree so other people can double-click it.

### Nothing installed yet? Neo drives upstream's installer

```bash
python -m neo.engine_setup --ensure                     # shallow-clone the pinned source into strata/ (gitignored)
python -m neo.wizard --check                            # what this PC has + which model fits (read-only, installs nothing)
python -m neo.wizard --family coder --source auto       # runs upstream setup.py --yes --no-start
python -m neo.manager                                   # the window
```

What the engine needs (upstream's numbers): a 12 GB+ NVIDIA/AMD card, 32 GB RAM (64 GB runs every
size), about 80 GB of free disk, Windows 10 / 11. The venv, dependencies, the engine binary and the
model download are all upstream `setup.py`'s job — Neo only runs it and reads its model tables.

### Which directory Neo uses

One rule, `neo/paths.py::strata_root()`: `--root` > `NEO_STRATA_ROOT` > the remembered root
(`neo/state/launcher.json`) > the app directory itself (when Neo sits inside a Strata checkout) >
`<app>/strata` > none of them, and then it says "no engine here" and the Connect page offers to install
one. The tests follow the same rule (`ENGINE` in `neo/tests/_bootstrap.py`), so "tests green" means
"the tests ran against the root Neo will really use".

## For developers (this repo)

```bash
python -m neo.engine_setup --ensure      # pinned engine source into strata/ (gitignored)
python -m neo.wizard --check             # read-only: this PC + which model to install
python -m neo.tests.run_all --quick --skip-network
python -m neo.webcheck
```

### Tests

| How to run | Measured (2026-10-14, same machine) |
|---|---|
| `python -m neo.tests.run_all --quick --skip-network` | 9.6 s, 435 tests, 86 parallel units |
| `python -m neo.tests.run_all --skip-network` | 16.3 s, 523 tests, 101 parallel units |
| `python -m neo.tests.run_all` (includes the real-clone test) | the clone unit is network-bound — measured 6.8 s and 145.3 s for that one unit on the same machine; the 30 s budget is held with `--skip-network` |

`run_all` splits every module into test classes and runs them in parallel (`--jobs`, default
`min(4, CPU cores)`); a failing class is re-run alone once, so a flaky failure caused by ports or load
is reported separately from a real one. Measurement probes live in `neo/tests/probes/` (`probe_cost.py`:
the cost of starting a process and reaching ready; `supervisor_events.py`: the supervisor's event
timeline).

### Pages

`neo/web/` is two `<script>` files sharing one global scope, so a duplicate top-level declaration
leaves the later file never executing — the window opens and every click is dead.
`python -m neo.webcheck` parses them in load order as one program (~1 s), and `build_exe` runs it
before packaging.

## Deploying onto a Strata tree (optional)

```bash
python -m neo.deploy --to D:\Work\Strata            # first see what would be written (dry-run)
python -m neo.deploy --to D:\Work\Strata --apply    # write: only neo/** + NEO-START.bat + NeoStrata.exe
python -m neo.build_exe                             # build the windowed exe (runs webcheck first; reports every missing package at once)
```

The deploy tree is a git checkout of upstream Strata, and Neo's files there are all untracked. Gates:
its HEAD equals the pin, tracked files are clean, and no `NeoStrata.exe` from that tree is running.
A version change swaps three things together — `NeoStrata.exe`, the `neo/` source and the `neo/web/`
pages. Details in [docs/NEO_DESKTOP.md](docs/NEO_DESKTOP.md).