# Configuration and project guardrails

Read this reference only when the repair may change user-level Codex configuration or add project execution guidance.

## Publicly documented settings

The current OpenAI configuration reference documents:

- `model_auto_compact_token_limit`: token threshold that triggers automatic history compaction;
- `model_auto_compact_token_limit_scope`: `total` or `body_after_prefix`;
- `model_context_window`: tokens available to the active model;
- `model`, `model_reasoning_effort`, and `service_tier` as independent selections.

Source: <https://learn.chatgpt.com/docs/config-file/config-reference>

Project `AGENTS.md` files are loaded along the path from the project root to the working directory, with nearer instructions taking precedence. Source: <https://learn.chatgpt.com/docs/agent-configuration/agents-md>

## Mutation procedure

1. Resolve the exact active config path. Do not assume that every host uses the same home directory.
2. Show the proposed persistent changes and obtain explicit approval.
3. Copy the current file to a timestamped backup in an explicitly safe location.
4. Inspect the selected model's effective context window from current runtime telemetry when available.
5. Preserve the user's explicit model, reasoning effort, context-window preference, and service tier.
6. Change only keys needed by the observed failure.
7. Validate syntax with the installed Codex version, preferably strict config validation when supported.
8. Reload the app-server only when the setting requires it and the user has authorized the interruption.

## Threshold selection

Do not universalize `450000`. It was a local mitigation for an effective context window near `828400` tokens and a tool-heavy workload.

When the user has not chosen a threshold and trustworthy effective-window telemetry is available, use the following as a disclosed operational heuristic, not an OpenAI default. Without that telemetry, do not invent a threshold:

- begin evaluation around 50–60% of the effective model window for very tool-heavy, long-running tasks;
- leave more headroom when tools return large text, images, document extracts, or repeated web results;
- leave less aggressive compaction for short, text-only work;
- never set the threshold at or above the usable context window;
- avoid changing the maximum context window merely to force compaction.

Use `total` when the goal is to charge the complete active context against the threshold. Use `body_after_prefix` only when its carried-prefix semantics are intentionally desired.

## Version-specific features

Some installations expose a feature named `unbounded_connection_retries`. It is not a portable assumption.

Before changing it:

```powershell
codex features list
```

Only write `[features] unbounded_connection_retries = false` when the installed version advertises that feature and retry amplification is part of the observed problem. Do not invent or force unknown keys.

## Project AGENTS.md content

If a project file already exists, preserve it and add only non-conflicting stability guidance. Useful clauses include:

- preserve the user-selected model and reasoning effort;
- use targeted reads instead of reopening whole large files;
- keep model-visible tool output preferably below 4,000 tokens and normally below 8,000;
- split planning, editing, building, rendering, and QA into bounded stages;
- after about 10 tool calls or 10 minutes, write a durable checkpoint and end the turn;
- after a successful tool call, stop and report if about five minutes pass without substantive progress;
- do not create/fork/migrate a formal task solely for stability;
- validate long artifacts by counts, sizes, hashes, required fields, and samples instead of immediate full rereads.

Add domain-specific invariants only when the project itself requires them. Do not copy budget, manuscript, or presentation rules into unrelated projects.

## Rollback

Keep the backup until at least one same-task validation turn passes. If a config change causes parsing, startup, model-selection, or permission regressions, restore only the changed config file from that verified backup and reload once. Do not reset unrelated user configuration.
