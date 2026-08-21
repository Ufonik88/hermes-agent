## Summary

Fixes two `hermes doctor` issues that surfaced repeatedly on this install.

### 1. model.default 'deepseek/deepseek-v4-flash' is vendor-prefixed but model.provider is 'command-code'

The config is **valid** — `command-code` is an aggregator-style gateway
whose catalog is exclusively vendor/model slugs (deepseek/..., Qwen/...,
zai-org/..., MiniMaxAI/..., nvidia/..., xiaomi/..., stepfun/...). The model
works; doctor's vendor-slug allowlist simply omitted this provider, so it
warned on every run and suggested switching to openrouter.

Fix: add `command-code` to the `providers_accepting_vendor_slugs` set in
`hermes_cli/doctor.py`, next to the other aggregators (openrouter,
kilocode, opencode-zen, deepinfra, ...).

No config change was needed (and changing provider to openrouter would have
broken the working model).

### 2. state.db is large — enable sessions.auto_prune in config.yaml

`sessions.auto_prune` was **already enabled** (`true`) in this config, with
retention 90 days, vacuum after prune, and a `last_auto_prune` marker in
the DB. The advisory is stale: it fires whenever logical size exceeds 1 GiB
and suggests enabling something that is already on.

Fix: `collect_state_db_stats` now reports `auto_prune_enabled` from the
user's config.yaml, and the oversized-DB advisory switches to a concrete
reclaim action when pruning is already enabled:

- auto_prune off → "consider enabling sessions.auto_prune ..."
- auto_prune on  → "sessions.auto_prune is enabled; run 'hermes sessions
  prune --older-than 90' to reclaim space now"

Files: `hermes_cli/doctor.py`, `hermes_state.py`.

## Verification

- `python3 -m pytest tests/hermes_cli/test_doctor.py` — 56 passed
  (includes the vendor-slug acceptance tests for named custom providers)
- Render-helper simulation with the real config values:
  - auto_prune=true + 1.5 GB logical size → warn message shows the prune
    action, not the enable suggestion
  - auto_prune=false + large DB → original enable suggestion preserved
- `python3 -m py_compile hermes_cli/doctor.py hermes_state.py` — clean

## Notes

- The CommandCode provider's real model list was taken from this install's
  `~/.hermes/config.yaml` (providers.command-code.models).
- No config.yaml change is required for either fix.
