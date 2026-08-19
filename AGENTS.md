# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Screenshot to Code: convert screenshots, mockups, Figma designs, and screen recordings into
working frontend code using AI. FastAPI + WebSocket backend, React/Vite frontend. This repo
is the OSS version; the hosted version lives on the `hosted` branch and talks to a separate
SaaS backend at `../screenshot-to-code-saas`.

## Commands

Python environment:

- Always use the backend Poetry virtualenv. Preferred invocation: `cd backend && poetry run <command>`.
- To activate directly: `cd backend && poetry env activate` (then run the printed `source .../bin/activate` command).

Backend (from `backend/`):

- Run server: `poetry run uvicorn main:app --reload --port 7001` (or `poetry run python start.py`)
- Run all tests: `poetry run pytest`
- Run a single test file: `poetry run pytest tests/test_screenshot.py`
- Run a single test: `poetry run pytest tests/test_screenshot.py::TestNormalizeUrl::test_url_without_protocol`
- Type check: `poetry run pyright`
- **Always run `pytest` and `pyright` after every backend code change.** Type-checking policy: no new warnings in changed files.

Frontend (from `frontend/`):

- Dev server: `pnpm dev` → `http://localhost:5173` (binds to `localhost` only — `127.0.0.1` will refuse the connection)
- Lint: `pnpm lint` (runs with `--max-warnings 0`; note there are pre-existing baseline errors, e.g. `@typescript-eslint/no-explicit-any` in `generateCode.ts`)
- Unit tests: `pnpm test` (Jest)
- E2E/QA test: `pnpm test:qa` (drives the real app in a browser; see `qa.test.ts`)
- Build: `pnpm build` (OSS) / `pnpm build-hosted` (hosted, `--mode prod`)

If a change touches both backend and frontend, run both sets of checks.

### Evals (`backend/evals/`, `backend/run_evals.py`)

- Input screenshots go in `backend/evals_data/inputs`, outputs in `backend/evals_data/outputs`
  (configurable via `EVALS_DIR` in `backend/evals/config.py`).
- Set `STACK`/`MODEL` in `backend/run_evals.py`, then run it with the relevant API key set.
- Rate outputs at the frontend's `/evals` route (1–4 scale); typically run 3x per model/prompt/stack
  combo and average.
- With `PROMPT_REPORTS_ENABLED=1` (+ `LOGS_PATH=...`), every LLM request/response, tool call, and
  final HTML is logged and browsable at `/evals/prompt-reports` — trust these over eyeballing the UI
  when debugging generation behavior.

## Environment / API keys

- At least one of `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY` is required, set in
  `backend/.env` (restart backend after editing) or via the in-app Settings dialog.
- `GEMINI_API_KEY` is effectively required for good results: it powers asset extraction and is
  required for video-input mode.
- `REPLICATE_API_KEY` (image generation/editing/background removal) only works via `backend/.env`,
  not the Settings dialog.
- `OPENAI_BASE_URL` for proxying OpenAI is disabled when `IS_PROD` is set.
- Other backend flags live in `backend/config.py`: `IS_DEBUG_ENABLED`, `DEBUG_DIR`,
  `PROMPT_REPORTS_ENABLED`, `LOCAL_ASSET_DIR`/`LOCAL_ASSET_BASE_URL`, `GENERATION_MAX_COST_USD`
  (hard per-variant spend ceiling — a run that would exceed it is aborted).
- Playwright Chromium powers the optional `screenshot_preview` agent tool; it's probed for
  availability at backend startup and only offered to the agent when present.

## Architecture

### Generation flow (backend)

The core endpoint is the `/generate-code` WebSocket (`backend/routes/generate_code.py`). It's built
as an explicit **middleware pipeline** (`Pipeline`/`Middleware`/`PipelineContext`, similar in shape
to an HTTP middleware stack) rather than one long function:

1. `WebSocketSetupMiddleware` — accept the connection, ensure cleanup.
2. `ParameterExtractionMiddleware` — parse/validate the incoming JSON params into `ExtractedParams`
   (stack, input mode, API keys, prompt, history, file state for edits, design system, etc.).
3. `StatusBroadcastMiddleware` — tell the frontend how many variants to expect.
4. `PromptCreationMiddleware` — build the LLM prompt messages (`prompts/pipeline.py`).
5. `CodeGenerationMiddleware` — select models per variant (`ModelSelectionStage`) and run generation
   for each variant concurrently (`AgenticGenerationStage`, one `agent.runner.Agent` per variant).
6. `PostProcessingMiddleware` — currently a no-op hook for post-generation work.

Each variant streams its own `status`/`chunk`/`toolStart`/`toolResult`/`setCode`/`variantComplete`/
`variantError` messages back over the same WebSocket, tagged with `variantIndex`, so the frontend can
render N variants generating in parallel from one connection.

**Prompt construction** (`backend/prompts/`): `prompts/plan.py` decides a `construction_strategy`
(`update_from_history`, `update_from_file_snapshot`, or fresh `create`) based on generation type and
what state is available, then `prompts/pipeline.py` dispatches to `prompts/create/` (image/text/video
input) or `prompts/update/` (from chat history, or from a snapshot of the current file — used for
select-and-edit). `prompts/system_prompt.py` and `prompts/design_system.py` assemble the system prompt;
`prompts/message_builder.py` defines the `Prompt` message-list type.

**Agent loop** (`backend/agent/`): `agent.runner.Agent` is a thin subclass of `agent.engine.AgentEngine`,
which runs one variant's full generation as a tool-calling loop against a provider-specific session
(`agent/providers/{openai,anthropic,gemini}.py`, chosen via `agent/providers/factory.py`). Tools are
declared once in `agent/tools/definitions.py` (`create_file`, `edit_file`, `generate_images`,
`remove_backgrounds`, `edit_images`, `extract_assets`, `screenshot_preview`, `retrieve_option`) and
executed via `agent/tools/runtime.py` (`AgentToolRuntime`). `agent/state.py` tracks the in-progress
HTML file across tool calls; `EmptyOutputError` / `BudgetExceededError` in `engine.py` are the two
"legitimate failure" exceptions the loop can raise (no file produced; spend ceiling hit).

**Models** (`backend/llm.py`): the `Llm` enum is the single source of truth for every callable
model+reasoning-effort combination, each mapped to a provider (`MODEL_PROVIDER`) and, for OpenAI, an
API name + reasoning effort (`OPENAI_MODEL_CONFIG`). `backend/routes/model_choice_sets.py` defines
which model combos get used per input-mode/generation-type/available-keys combination (see
`ModelSelectionStage` in `generate_code.py`) — variants cycle through a short model list so N variants
map to a fixed rotation. The frontend's `frontend/src/lib/models.ts` `CodeGenerationModel` enum must
be kept in sync with this.

**Assets & images**: `asset_extraction.py` extracts real logos/images/icons from the source screenshot
so the model reuses them instead of hallucinating replacements (Gemini-only capability);
`uploaded_assets/` stores user-uploaded images content-addressed as `asset_<sha256[:24]>.png` and
serves them back to the model as local asset URLs; `image_generation/` wraps Replicate for
generate/edit/remove-background tool calls.

### Frontend architecture

- `App.tsx` owns the overall generation flow: builds `GenerationRequest`s (`lib/prompt-history.ts`),
  opens the WebSocket via `generateCode.ts`, and routes incoming messages into two Zustand stores:
  `store/project-store.ts` (commits, variants, per-variant agent-event streams, head pointer — the
  project/version-history state) and `store/app-store.ts` (transient UI state: app phase, select-and-edit
  mode, update instructions/images).
- **Commits & variants** (`components/commits/`, `design-docs/commits-and-variants.md`,
  `design-docs/variant-system.md`): each generation round produces a "commit" with N parallel
  "variants" (one per model); `head` tracks which commit is currently shown. Updates operate on the
  currently-selected variant's history.
- `Stack` (`lib/stacks.ts`) must stay in sync with the backend's `prompts/prompt_types.py` `Stack`.
- `components/unified-input/` is the multi-tab input surface (upload screenshot / paste URL / import
  code / text-to-code); `components/select-and-edit/` implements click-an-element-to-edit; `components/evals/`
  is the UI for the eval-rating and prompt-report workflows described above.
- `IS_RUNNING_ON_CLOUD` (`config.ts`) and `pnpm dev-hosted`/`build-hosted` (`--mode prod`) are the
  hooks for hosted-vs-OSS behavior differences within this same codebase.

## Conventions

- Prefer triple-quoted strings (`"""..."""`) for multi-line prompt text; for interpolated multi-line
  prompts, prefer a single triple-quoted f-string over concatenated string fragments.
- Backend model additions: add to `Llm` in `llm.py` (+ `MODEL_PROVIDER`, and `OPENAI_MODEL_CONFIG` if
  OpenAI), then mirror the entry in `frontend/src/lib/models.ts`.

## Hosted

The hosted version is on the `hosted` branch and connects to a separate SaaS backend codebase at
`../screenshot-to-code-saas`.

## Cursor Cloud / cloud environment notes

Dependencies are refreshed automatically on startup (`poetry install` in `backend/`, `pnpm install`
in `frontend/`); no manual install is needed. Cursor Cloud environment setup should run
`bash /agent/repos/screenshot-to-code/scripts/cursor-cloud-install.sh` (it `cd`s to the repo root
itself, so it works regardless of the startup working directory).

Non-obvious caveats:
- `poetry` is installed under `~/.local/bin` and is on PATH for interactive shells (`.bashrc`) but not
  necessarily for non-interactive scripts; use the full path `~/.local/bin/poetry` if `poetry` is not found.
- The Poetry virtualenv commonly resolves to a newer Python (e.g. 3.12) than the `^3.10` pin in
  `pyproject.toml` — that's expected, just use `poetry run`.
- `pnpm install` prints an "Ignored build scripts (esbuild, puppeteer)" warning — harmless; dev/build/test
  all work without approving builds.
