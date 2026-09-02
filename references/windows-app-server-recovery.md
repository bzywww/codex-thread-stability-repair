# Windows app-server recovery

Read this reference only for a local Windows Codex task whose bounded stop request failed, or when a supported manual compaction operation is required.

## Safety gate

Before termination, tell the user:

- which exact task is stuck;
- that reconnect may be brief or prolonged and automatic restart can fail;
- which other active local tasks share the app-server when that can be determined, and that they will be interrupted;
- that already flushed project files and the task id are intended to be preserved, while unflushed in-flight response state can be lost and preservation is not guaranteed.

Obtain explicit approval immediately before the process mutation.

Run this recovery from a separate healthy control task. If the target is the current task, stop and ask the user to switch to an existing healthy task before continuing.

## Resolve the exact process

Preflight the actual host commands first. Confirm the installed Codex CLI and required shell are available, and use `codex app-server --help` or generated current-version schemas rather than assuming flags or methods.

First list candidates read-only:

```powershell
Get-Process -Name codex -ErrorAction SilentlyContinue |
  Select-Object Id, ProcessName, StartTime
```

For one candidate, compare its command line, executable identity, parent, and start time inside PowerShell. Use a task-specific variable name such as `$targetPid`; do not reuse PowerShell's reserved `$PID` variable. Never print raw command lines or full executable/parent paths into model-visible output because unrelated Codex commands may contain prompts, paths, tokens, or other sensitive values.

Perform the comparisons inside the shell and expose only a compact record containing `Pid`, `StartTime`, `NameMatches`, `ExecutableMatches`, `IsAppServer`, and `ParentIdentityMatches`. Do not emit the compared raw strings.

Require every identity check below to be positively available and agree:

- process still exists;
- name is `codex.exe`;
- executable path is the active Codex installation;
- command line identifies `app-server`;
- parent and start time are consistent with the active desktop instance.

If any field is unavailable or identity changed, block termination and re-resolve. Never use a wildcard or terminate every Codex process. A PID alone never proves task association because one app-server may host several tasks.

Approval is bound to the verified PID and identity. If the PID, path, command line, parent, or start time changes before termination, discard the stale approval, re-resolve, and ask again.

## Terminate and reload

After approval and a final in-command identity check, terminate only the exact PID. The terminating tool call may be aborted if it uses that app-server, but do not rely on that behavior as evidence. After the desktop reconnects:

- resolve the new app-server PID and start time;
- confirm the target task is idle or its old turn is interrupted;
- confirm persistent config and project instructions still exist;
- do not automatically restart the user's substantive workload.

## Manual compaction

Manual compaction changes the active conversational context through lossy summarization and may not be practically reversible. Require an on-disk checkpoint and explicit user approval that acknowledges that loss risk.

1. Inspect help and the installed app-server protocol instead of assuming a command or flag exists. Generate a schema only after disclosing its temporary filesystem write. Use `--experimental` only when current `--help` advertises it:

```powershell
codex app-server generate-json-schema --out <temporary-directory>
```

2. Search the generated schema for `thread/compact/start` and inspect its current parameter schema.
3. Prefer the host's native compact action. A literal `/compact` sent as an ordinary delegated prompt is not equivalent.
4. Ensure the target is idle and use the existing host/UI compact operation whenever available.
5. If a separate client is unavoidable, use only a documented single-writer handoff: positively release the desktop writer, verify exclusive ownership, invoke the advertised compact method, then cleanly release ownership before the desktop resumes.
6. If exclusive ownership cannot be positively established, block manual compaction; never open a competing writer against a desktop-owned rollout.
7. Wait for a completed context-compaction item/turn and a return to idle.

An accepted empty RPC result only means compaction started. It is not completion evidence.

## Failure handling

Errors such as stream disconnection, TLS close without completion, response-body decode failure, or remote-compaction failure implicate the upstream response path. Let the installed runtime perform only its built-in bounded retries. If those retries fail:

- stop manual retries;
- preserve the idle/interrupted thread and all files;
- report the sanitized error code/category and attempt count;
- do not claim quota exhaustion unless separate quota evidence exists.

## Post-recovery verification

Use the original task id for one text-only marker test, followed by at most one read-only tool test. Confirm the model, reasoning effort, loaded `AGENTS.md`, task completion, idle state, and pre/post context scale. Stop once these checks pass.
