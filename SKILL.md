---
name: codex-thread-stability-repair
description: Diagnose and safely recover an existing Codex Desktop task stuck in repeated reconnects, post-tool silence, failed context compaction, or long-context stalls (重连、卡死、压缩失败), while preserving the same task, model, and on-disk work. Use for evidence-backed repair and for ruling out quota before recovery; do not use for ordinary slow responses or generic network troubleshooting.
---

# Codex Thread Stability Repair

Recover the named Codex task without substituting a new task for the failed one. Start read-only, distinguish observation from diagnosis, and escalate mutations only when the evidence and user authorization justify them.

## Required outcome

- Keep the same task/thread id unless the user explicitly requests a new task.
- Preserve the user's selected model, reasoning effort, service tier, project files, and completed work.
- Identify the failure class from observable evidence.
- Apply the smallest recovery or persistent mitigation that addresses the observed failure.
- Leave the target task either demonstrably healthy and idle or clearly blocked with the original state preserved.

## Operating boundaries

- A request to diagnose authorizes read-only inspection, not process termination, configuration edits, compaction, task interruption, or external writes.
- Perform active-turn interruption, app-server restart, or manual compaction only from a separate healthy control task. If the target is the task currently running this skill, stop and ask the user to continue from an existing healthy task; do not make the target terminate its own control channel.
- Before interrupting an unfinished turn, state that its uncommitted response will be lost and obtain explicit approval unless the user already expressly authorized stopping that exact turn.
- A request to repair does not silently authorize persistent user-config or project-instruction changes. Preview those changes and obtain explicit approval before writing them.
- Obtain explicit approval immediately before terminating an app-server process or manually compacting a task. Identify other active local tasks when possible and state that the shared app-server restart will interrupt them, may reconnect slowly or fail to restart automatically, and can lose unflushed in-flight response state.
- Never create, fork, migrate, archive, or delete a formal task merely to reduce reconnects.
- Never kill every `codex` process. Resolve and re-check one exact app-server PID immediately before termination.
- Never claim quota throttling without a 429, `rate_limit_reached_type`, spend-control signal, or equivalent direct evidence.
- Never claim a permanent backend fix. This skill mitigates local state, context pressure, and retry amplification; it cannot prevent an upstream stream from closing.

## 1. Identify the exact target

Use the host's thread/task listing capability and match the user-visible title verbatim. Record:

- task id and host id;
- exact title and project/cwd;
- current status and active turn id;
- selected model and reasoning effort when visible.

If multiple tasks match materially, ask the user to choose. Do not infer from summaries alone and do not create a replacement task.

## 2. Audit before acting

Take one compact status snapshot. Prefer `wait_threads` with `timeoutMs: 0` for a local Codex task, then use a small `read_thread` page only when the snapshot lacks necessary evidence. Treat live app-server/thread status as authoritative, including waiting-for-approval or waiting-for-user-input states. Do not reconstruct authoritative lifecycle state from a truncated rollout page and do not repeatedly poll unchanged state.

When local rollout inspection is necessary, resolve the file whose filename ends with the exact thread id and extract only bounded event metadata: timestamps, event types, exact turn ids, token fields, settings, and sanitized error category/code. Do not load prompts, reasoning bodies, tool inputs/outputs, replacement history, credentials, URLs with query strings, or full absolute paths into model context. Set both line and byte/output caps, and disclose when the inspected window is truncated.

Do not classify a task as stalled merely because Ultra reasoning takes 30–90 seconds or the UI briefly says reconnecting. Stronger evidence includes at least one of:

- an explicit response-stream, TLS, transport, HTTP, or remote-compaction error;
- no substantive rollout event for about five minutes after a completed tool call, corroborated by unchanged token telemetry, no running external command, or an ignored cancel request;
- repeated identical token snapshots across about five minutes;
- an active task whose native interrupt fails or whose writer remains active after an acknowledged interrupt;
- an existing context already near or above its configured compaction threshold.

## 3. Classify the failure

| Evidence | Classification | Next action |
|---|---|---|
| 429 plus a named weekly/usage limit, `rate_limit_reached_type`, or spend-control signal | Quota or account limit | Classify and stop recovery; report the exact limit and reset information |
| Bare/transient 429 without quota metadata | Request or concurrency throttling | Use a bounded wait/retry policy; do not label it weekly quota or restart by default |
| Provider/WebSocket diagnostics fail outside the target task | Connectivity or endpoint path | Diagnose that path; do not blame context alone |
| Stream disconnected, TLS close, response-body decode failure, remote compact failed | Upstream response/compaction stream | Allow only the runtime's bounded retries; reduce context pressure when safe |
| Tool completed, then no events; native interrupt fails; writer remains active | Local writer/app-server state | Use exact-turn interrupt and, only if needed, exact-process recovery below |
| Context near/above threshold, large tool outputs, repeated whole-file reads | Context pressure | Add persistent guardrails; compact only when authorized and necessary |
| New events, token growth, or tool completion continue | Slow but progressing | Monitor with a bounded wait; do not restart |

More than one class may be present. Report the direct trigger separately from contributing factors.

When connectivity remains a plausible class and the installed CLI provides it, use the read-only network/provider checks in `codex doctor` rather than a generic ping. If the necessary diagnostic or runtime evidence is unavailable, report it as unavailable instead of inferring a result.

## 4. Apply persistent mitigation

Read [references/configuration-and-guardrails.md](references/configuration-and-guardrails.md) before editing user config or a project `AGENTS.md`.

Core invariants:

- back up the exact config file before editing;
- preserve model, reasoning effort, service tier, and context-window choice unless the user asks to change them;
- derive any compaction threshold from the active model window and observed workload instead of hard-coding one user's value;
- detect version-specific feature flags before writing them;
- merge with an existing project `AGENTS.md`; never overwrite unrelated project instructions;
- bound tool-visible output, whole-file rereads, turn duration, and retry loops;
- checkpoint durable work to disk before risky recovery.

Apply persistent changes only when repeated failures, context growth, oversized tool output, or retry amplification supports them. For a single isolated upstream disconnect, prefer recovery and observation over speculative config mutation.

Changing an auto-compaction threshold does not prove that an already oversized persisted task was compacted. Verify the target separately.

## 5. Release a stuck turn

1. Resolve the exact active thread and turn ids, then prefer the host's dedicated cancel/interrupt operation. Feature-detect the installed app-server's `turn/interrupt` or equivalent method when it is not exposed directly.
2. Do not use a normal follow-up message as cancellation; it may merely queue behind the stuck writer and consume more context.
3. After an accepted native interrupt, wait up to about 60 seconds for the exact turn to become interrupted/completed and the thread to become idle.
4. If no native interrupt exists, it fails, or the writer remains active, proceed only through the platform recovery reference.

On Windows, read [references/windows-app-server-recovery.md](references/windows-app-server-recovery.md) before inspecting or terminating a process or using manual compaction.

## 6. Manual compaction branch

Use manual compaction only when all are true:

- the task is idle or its stuck turn has been safely interrupted;
- important work is already on disk or otherwise recoverable, verified by task-specific evidence such as expected file existence plus size/hash/timestamp or a clean completed write event;
- context pressure is evidenced, not guessed;
- the current installed protocol or UI explicitly supports manual compaction;
- the user approved the context-changing operation, possible brief reconnect, and the fact that compaction is lossy summarization that may not be practically reversible.

Prefer a native compact command or protocol operation. Do not send the literal text `/compact` as an ordinary delegated model prompt. Respect the runtime's own retry cap; do not wrap failed compaction in an unbounded manual loop.

Success requires an observable compaction-completed item/turn and a return to idle. A request being accepted is not success.

## 7. Validate at low cost

Do not send a normal model test before compacting an already oversized task; the test itself can consume the full old context.

After recovery:

1. Run one same-task text-only test that forbids tools and returns a fixed marker.
2. If it succeeds, run at most one read-only tool test such as retrieving the current directory.
3. Verify the same task id, completed turn, idle status, expected model/effort, loaded project instructions, and absence of a new reconnect loop.
4. Compare only like-for-like runtime metrics. Label `last_token_usage.input_tokens / model_context_window` as last-request input utilization; do not equate it with a `total` or `body_after_prefix` compaction metric unless the runtime exposes the matching scoped value.
5. Stop testing once these invariants pass.

## 8. Report precisely

Lead with whether the original task is healthy, idle, active, interrupted, or blocked. Include:

- exact target task id and title;
- direct failure evidence and contributing factors;
- files/settings changed and their backups;
- whether an app-server restart or compaction occurred;
- validation marker, duration, and pre/post token scale;
- exact quota fields and sanitized error code/category, including explicit null/absent results when relevant;
- residual risk and the next safe user action.

Do not describe an idle validation state as background work still running.

Treat loaded project instructions as verified only when runtime evidence such as `instructionSources`, injected `AGENTS.md` world state, or equivalent host metadata names the expected file. File existence alone is not proof that the active turn loaded it.
