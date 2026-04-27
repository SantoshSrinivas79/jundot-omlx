# AI Collaboration Guide for oMLX

Last updated: 2026-04-25

This guide is for AI coding agents working in this repository. Keep it current
whenever you change architecture, workflows, commands, dependencies, or known
test/runtime caveats. Treat stale guidance as a bug.

## Project Snapshot

oMLX is an Apple Silicon LLM inference server built on MLX. It exposes
OpenAI-compatible, Anthropic-compatible, Responses API, embeddings, reranking,
audio, admin-dashboard, model-management, cache, and MCP surfaces.

Primary goals:

- Serve local MLX-format models from one or more model directories.
- Support LLM, VLM/OCR, embeddings, rerankers, and audio models.
- Provide continuous batching and persistent tiered KV/prefix caching.
- Offer a web admin UI for model settings, downloads, benchmarks, auth, and
  integrations.
- Package as both a Python CLI and a macOS menubar app.

## High-Signal Entry Points

- `omlx/cli.py`: CLI entry point. `omlx serve` initializes settings, logging,
  cache config, MLX cache limits, and starts uvicorn.
- `omlx/server.py`: Main FastAPI app and most public API routes.
- `omlx/engine_pool.py`: Multi-model discovery, loading, LRU eviction, pinning,
  TTL/unload behavior, and engine selection.
- `omlx/engine_core.py`: Async generation loop and global MLX executor.
- `omlx/scheduler.py`: Continuous batching, mlx-lm `BatchGenerator` integration,
  prefix/SSD cache plumbing, grammar hooks, and many MLX safety patches.
- `omlx/model_discovery.py`: Detects model type and estimated size from model
  directories.
- `omlx/settings.py`: Global persisted settings in `~/.omlx/settings.json`.
- `omlx/model_settings.py`: Per-model persisted settings, profiles, and
  templates in `~/.omlx`.
- `omlx/admin/routes.py`: Admin panel routes and runtime settings updates.
- `omlx/api/*`: Pydantic request/response models and protocol helpers.
- `omlx/mcp/*`: Model Context Protocol configuration, clients, manager, tools,
  and execution.
- `packaging/`: macOS menubar app and DMG build flow.
- `tests/`: Large unit/integration suite with extensive mocks.

## Architecture Mental Model

Request flow for chat/completions:

1. FastAPI route in `omlx/server.py` validates protocol models from `omlx/api`.
2. Model id or alias is resolved through `EnginePool` and `ModelSettingsManager`.
3. `EnginePool.get_engine()` loads or reuses the right engine type.
4. LLM/VLM engines render chat templates and call `AsyncEngineCore`.
5. `EngineCore` adds a request to `Scheduler`.
6. `Scheduler.step()` runs on the global MLX executor and emits token outputs.
7. Server streaming/non-streaming helpers format OpenAI, Anthropic, or Responses
   API output, including usage/cache stats.

Important invariant: MLX GPU work is intentionally serialized through the
single global executor from `omlx/engine_core.py`. Do not introduce concurrent
MLX/Metal execution paths unless you deeply understand the existing crash
workarounds and can test them on Apple Silicon.

## Runtime And Settings

Common local run command:

```bash
.venv/bin/omlx serve --model-dir /Users/santosh/.omlx/models --host 127.0.0.1 --port 8000
```

Run a single downloaded model directory as the only/default model:

```bash
.venv/bin/omlx serve --model-dir /Users/santosh/.omlx/models/gemma-4-e2b-it-4bit --host 127.0.0.1 --port 8000
```

Preferred tuned command for this specific machine is documented in
`README.md` under "Recommended Start Command for This Mac". Use that recipe for
local runtime work unless the task explicitly needs a different model. The
machine is an Apple M1 MacBook Air with 8GB unified memory, and Gemma
`gemma-4-e2b-it-4bit` is the practical default. Avoid the previously persisted
`max_model_memory: 12GB` setting; prefer `4.5GB` for stable Gemma-only use or
`5GB` with at most two concurrent requests for short prompts.

Useful smoke checks:

```bash
curl -s http://127.0.0.1:8000/health
curl -s http://127.0.0.1:8000/v1/models
curl -sS http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"gemma-4-e2b-it-4bit","messages":[{"role":"user","content":"Say hello in one short sentence."}],"max_tokens":24,"temperature":0}'
```

Open the local web interface after server startup with:

```bash
open http://127.0.0.1:8000/admin
```

Admin chat is at `http://127.0.0.1:8000/admin/chat`.

Runtime settings are persisted. Passing CLI overrides such as `--model-dir`,
`--host`, or `--port` may update `~/.omlx/settings.json`. Be explicit about this
when changing or testing runtime behavior.

As of 2026-04-25, this machine has a repo-local `.venv` created with:

```bash
uv venv --python /Users/santosh/.local/bin/python3.11 .venv
uv pip install -e '.[dev]'
```

Optional structured-output grammar support is not included in that dev install.
Install it only when needed:

```bash
uv pip install -e '.[dev,grammar]'
```

## Testing Baseline

Fast smoke subset known to pass after environment setup:

```bash
.venv/bin/python -m pytest tests/test_config.py tests/test_model_settings.py tests/test_api_utils.py tests/test_openai_models.py -q
```

Observed result on 2026-04-25: `308 passed`.

Default test collection works:

```bash
.venv/bin/python -m pytest --collect-only -q
```

Observed collection on 2026-04-25:

- `3909` total tests
- `3855` selected by default
- `54` deselected by `pytest.ini` markers

Full default suite baseline on 2026-04-25:

- `3816 passed`
- `30 skipped`
- `54 deselected`
- `9 failed`

Known baseline failures at that time:

- `tests/test_accuracy_benchmark.py::TestAccuracyBenchmarkRequest::test_all_valid_benchmarks`
  expects 12 benchmarks but current request contains 16.
- `tests/test_admin_api_key.py::TestListModelsSettings::test_list_models_includes_all_model_settings_fields`
  is missing `preserve_thinking` and `turboquant_skip_last` in the admin model
  settings response/test expectations.
- `tests/test_admin_profiles_api.py::test_all_model_settings_fields_classified`
  needs `preserve_thinking` classified or excluded.
- `tests/test_boundary_snapshot_store.py::TestBoundarySnapshotSSDStore::test_cleanup_all_drains_queue`
  leaves one `_boundary_snapshots/req-1` child behind.
- Five `tests/test_grammar.py` tests fail without optional `xgrammar`.

When changing code, run the most specific affected tests first, then a broader
subset. Do not claim the suite is clean unless you have run it after your
changes and accounted for the baseline failures.

## Development Practices For AI Agents

- Prefer `rg` and `rg --files` for navigation.
- Keep edits focused. This codebase has many hardware-specific workarounds and
  protocol compatibility edges.
- Do not revert unrelated user or generated changes.
- Use `apply_patch` for manual file edits.
- Add or update tests near the behavior you changed.
- Include `# SPDX-License-Identifier: Apache-2.0` on new Python source files.
- Avoid adding new dependencies unless the project already uses that ecosystem
  or the change clearly requires it.
- Be careful with persisted user settings under `~/.omlx`; tests should prefer
  temp dirs and mocks.
- For server work, consider auth, OpenAI error shape, Anthropic compatibility,
  streaming and non-streaming behavior, usage fields, disconnect handling, and
  alias resolution.
- For engine/scheduler/cache work, consider active requests, abort paths,
  `mx.synchronize()`, `mx.clear_cache()`, cache corruption recovery, and model
  ownership cleanup.
- For admin work, update route models, templates, i18n JSON files, and tests
  together when user-facing fields change.
- For model settings work, update all affected surfaces: dataclass, persistence,
  admin request/response payloads, profile field classification, UI, and tests.

## Public API Surfaces To Preserve

Main endpoints in `omlx/server.py`:

- `GET /health`
- `GET /api/status`
- `GET /v1/models`
- `GET /v1/models/status`
- `POST /v1/models/{model_id}/unload`
- `POST /v1/chat/completions`
- `POST /v1/completions`
- `POST /v1/messages`
- `POST /v1/messages/count_tokens`
- `POST /v1/responses`
- `GET /v1/responses/{response_id}`
- `DELETE /v1/responses/{response_id}`
- `POST /v1/embeddings`
- `POST /v1/rerank`

Audio routes live in `omlx/api/audio_routes.py` and are included when optional
audio dependencies are available:

- `POST /v1/audio/transcriptions`
- `POST /v1/audio/speech`
- `POST /v1/audio/process`

Admin routes live under `/admin` in `omlx/admin/routes.py`.

## Model And Cache Notes

Model discovery accepts both:

- A parent directory containing model subdirectories.
- A direct model directory containing `config.json`.

The server supports two-level organization folders and HuggingFace cache
snapshot layouts. LoRA/PEFT adapter dirs are skipped.

Cache behavior is centered around paged SSD cache plus block-aware prefix
cache. Memory KV cache is managed by mlx-lm `BatchGenerator`; oMLX cache code
handles prefix reuse and SSD persistence. VLMs also use a vision feature cache.

## Packaging Notes

The macOS app is not Electron. It is a PyObjC/rumps menubar app packaged via
venvstacks.

Packaging entry points:

- `packaging/build.py`
- `packaging/venvstacks.toml`
- `packaging/omlx_app/*`

Packaging requires macOS, Apple Silicon, Python 3.11+, and venvstacks.

## Keeping This Guide Updated

Update this file in the same change whenever you:

- Add, remove, or rename major modules or endpoints.
- Change setup, dependency, build, packaging, or test commands.
- Change baseline test expectations or known failures.
- Add new persistent settings files or runtime side effects.
- Discover a sharp edge future agents should know before editing.

Future agents should read this file before making changes, then verify any
claims that might have gone stale.
