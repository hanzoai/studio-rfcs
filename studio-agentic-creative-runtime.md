# Hanzo Studio — the agentic multi-modal creative runtime

**Status:** proposed · **Scope:** studio + cloud (`apps.Wire`) + hanzo CLI
**One line:** Talk to Studio → the agent either runs a saved graph *or* writes+runs
arbitrary code in the sandbox → any modality (image · video · music · voice ·
design · deck) → assets land in one per-org store. Node editor is the advanced
view over the same substrate. This is a **composition of shipped apps, not a
rewrite.**

## Principle
One and only one way. Decomplect **control** (state: who/what/paid) from
**compute** (execute on CPU-sandbox or GPU). Never braid them. Reuse the cloud
`apps.Wire()` subsystems that already exist; build only the missing surface.

## Non-goals (explicit — do NOT do these)
- Do **not** rewrite the ComfyUI/Python execution engine in Go.
- Do **not** use gpython. `exec` already runs sandboxed CPython — that *is* the
  Python runtime. `go-embed-python` is unnecessary unless we later want light
  Python *inline* in a control node (`exec` covers it today).
- Do **not** rebuild a sandbox or a queue — `exec`/`functions`/`tasks` exist.

## Architecture — two planes, existing apps

**Control plane = the Go cloud binary (`apps.Wire`).** Owns all state, one binary:
| Concern | App |
|---|---|
| Tenancy / IAM / session / org-switch | `cli/studio.go` session + shared IAM |
| Durable job queue | **`tasks`** (`tasks.Embed`, fronts `tasks.hanzo.ai`) — the gpu-jobs backbone |
| Sandbox code execution | **`exec` / `functions`** — runs `python·node·go·deno·bash·container`, isolated on `code-exec-pool` |
| GPU dispatch / BYO-GPU | **`gpu`** + `cli/studio.go` render-backend supervision |
| Asset catalog / library / curation / billing | *new thin `studio` app* — ports `middleware/studio_home.py` library + `session.py` |
| Design systems / presets / templates | *new `studio` app* tables (per-org SQLite / Hanzo Base) |

**Data plane = where compute runs (stateless, no auth logic):**
- **GPU graphs** → Python ComfyUI engine (BYO-GPU or cloud GPU), dispatched via `tasks`.
- **Arbitrary agent code** → `exec` sandbox (`code-exec-pool`), any language, GPU-attachable.

**Security boundary:** the control plane never executes agent/user code inline.
Agent code *always* goes to `exec`; GPU graphs *always* go through `tasks`.

## Execution model (the agentic flexibility)
The copilot has exactly two verbs, both already-built backends:
1. `run(workflow, inputs)` → enqueue on `tasks` → GPU worker → outputs.
2. `sandbox.exec(lang, code, files?, gpu?)` → `exec` app → outputs.

Node graphs are one shape of (1). Anything a node can't express is (2) — the
agent writes PIL/torch/ffmpeg/librosa/pandas and runs it sandboxed. Outputs from
both land in the org asset store → surface in the chat-first home. That is "full
Python agentic flexibility," using infra that already exists.

## Multi-modal — engines already vendored in studio
| Modality | Engine (real, in-org / vendored) |
|---|---|
| Image / design | Qwen-Image-Edit, Flux, SDXL (proven on karma) |
| Video | `wan2.2`, `ltx`, WanVideoWrapper, LTXVideo, CogVideoX, HunyuanVideo, DiffuEraser, VideoHelperSuite |
| Music / audio | ACE-Step, `mugen` |
| Voice / TTS / realtime | `koe`, `tts`, `livekit`/`live`/`stream` |
| Deck / doc / site / app | agentic **code** flow via `exec` (HTML/React/Python → render) — the Claude-Design pattern |
| **Documents / office** | `exec` code flow with `python-docx` (.docx), `openpyxl` (.xlsx spreadsheets), `python-pptx` (.pptx), `pandas`/`csv`, `reportlab`/`weasyprint` (.pdf) → real files, no new service |

Each modality = node packs in the one engine (or an `exec` code flow), cataloged
by the one `studio` app. "All in one place" falls out of the composition.
Native office files (spreadsheets, docx, pptx, pdf) are the same `exec` flow with
the right libs — the agent writes the generator, `exec` runs it, the file lands
in the asset store.

## Connectors — Google Drive (and beyond) via `connectorruntime`
Reuse the cloud **`connectorruntime`** app (already present) + the Google Drive
API for a first-class connector so a user can **index, read, build, and edit
content in their own Drive**:
- **Index/read:** connector lists + pulls Drive files (Docs/Sheets/Slides + any
  binary) into the org asset catalog as references (not copies) — searchable,
  usable as inputs to any flow.
- **Build/edit/write-back:** an `exec` code flow (Drive API client) creates or
  edits Docs/Sheets/Slides in place and writes generated assets back to a chosen
  Drive folder. OAuth per-user (Drive scope), tokens in KMS, per-org.
- One connector interface; Drive is the first, others (Notion, S3, GitHub) drop
  in the same seam. The agent treats Drive content as just more inputs/outputs.

## Surfaces
- **Chat-first home (default):** monochrome Hanzo look, "What should we create?",
  design-system + template + model pickers, projects/assets below. Served by the
  `studio` cloud app. This is what most users see.
- **Advanced mode:** the ComfyUI node editor for people who build/edit flows —
  and it's editable *by chat* (the `copilot` already reads+mutates the graph).
- **CLI:** `hanzo studio` (login · generate · run · assets · pull), `hanzo
  design-sync` (registers a design-system package/tokens for the org — the
  `/design-sync` equivalent). `hanzo.sh` already installs the one binary.
- **Design systems / presets:** first-class, per-org, like Claude Design's
  Modernist/Classical/Nocturne set — but Hanzo-branded, seeded + user-created.

## Phased plan (reuse-first)
1. **Wire copilot → `exec` + `tasks`.** Give the copilot `sandbox.exec` and
   `run` tools backed by the existing `clients/exec` and `tasks` — unlocks the
   agentic flexibility with zero new runtime.
2. **Chat-first monochrome home** (rebuild `studio_home.html`) reading the
   already-populated `/v1/library`; templates from vendored packs.
3. **Thin `studio` cloud app** — port library/catalog/session/billing into
   `apps.Wire()` so state lives in the one Go binary (per-org SQLite).
4. **`hanzo studio` + `design-sync` CLI** on those APIs.
5. **Multi-modal templates** — surface Wan/LTX/ACE-Step/koe as first-class
   "create video / music / voice" flows.

## Status of the substrate (verified 2026-07-15)
`exec`/`functions` (python·node·go·deno·bash·container), `tasks` (durable,
embedded), `gpu` + BYO-GPU supervision, `code-exec-pool` (2 nodes) — all live.
The work is composition + the chat-first surface, not new runtimes.
