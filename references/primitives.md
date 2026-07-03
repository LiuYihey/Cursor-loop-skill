---
name: loop-primitives
description: Host-agnostic primitive spec for the loop skill. Any agent that can spawn a background process, read its output, terminate it, and (optionally) wake on a regex match can implement the loop skill in SKILL.md. This file defines those four primitives, their inputs, outputs, and failure modes — without naming any specific agent host.
---

# Loop Primitives

The loop skill needs four host capabilities. Map them to your agent's tools once, then follow `SKILL.md` as written.

## Two views of the same contract

The four primitives can be described two ways — both are correct, just at different altitudes:

| # | Low-level (any background process) | Loop-skill view (purpose-flavored) | Purpose |
| --- | --- | --- | --- |
| 1 | `spawn_loop(template, vars, label) → Handle` | `arm_loop(interval, purpose, prompt)` | Start a **repeating** background timer that emits a wake carrying `prompt` every `interval`. |
| 2 | `arm_wake(source, regex, payload) → WakeId?` | `arm_watcher(condition, purpose, prompt)` | Start a background process that watches `condition` and emits a wake **only when the condition fires**. |
| 3 | `read_output(handle, since?) → bytes` | `read_wake_signal(task_id)` | Read the latest matching sentinel line plus the JSON payload that followed it. |
| 4 | `terminate(handle, signal?) → void` | `cancel(task_id)` | Stop the backgrounded task and **consume** any pending completion event so it cannot re-wake the agent later. |

The low-level names are the contract any host with a background-process tool can fulfil. The loop-skill names are an alias layer that names the four operations in the language of `/loop`. Both describe the same thing; pick whichever reads better in your host mapping.

`purpose` is a short string unique per loop in the same session. It distinguishes concurrent loops and is included in the sentinel so unrelated shells cannot trigger false wakes.

## 1. `spawn_loop(template, vars, label) → Handle`

Start a long-running background process that runs `template` (with `vars` substituted) and never returns on its own. Return a `Handle` the agent can later read or stop.

| Field   | Type    | Meaning                                                                            |
| ------- | ------- | ---------------------------------------------------------------------------------- |
| `id`    | string  | Opaque identifier the host uses internally (a shell id, a task id, a PID, etc.).   |
| `output_path` | string? | Where the host persists the process's stdout. May be shared or unique per id.   |
| `pid`   | int?    | OS process id, if the host exposes it.                                             |
| `label` | string  | Echo of the input `label`; useful for logs and duplicate detection.                |

**Failure modes.** Reject with a clear error if the template would block the foreground, or if a loop with the same `label` is already running. Do **not** silently start a duplicate — duplicates cause two ticks per interval and corrupt sentinel ordering.

## 2. `read_output(handle, since?) → bytes`

Read the accumulated stdout (and ideally stderr) of a running process. `since` is an optional offset for incremental reads; if omitted, return everything buffered so far.

**Failure modes.** Must not block the agent. If the process has exited, return whatever was buffered and mark the handle as done. Never raise on `EPIPE` — the loop may have been killed mid-line, leaving an incomplete final line in the buffer.

## 3. `terminate(handle, signal?) → void`

Stop the process.

- On Unix: send `SIGTERM` first, then `SIGKILL` after a short grace period (≈ 2s) if it is still alive. The inner `sleep` and any children must die with it — kill the process group, not just the leader.
- On Windows: use `taskkill /F /PID <pid> /T`. The `/T` flag kills the process tree so the inner `while` and any `sleep` children die together.

**Failure modes.** Must be idempotent. Calling terminate twice is a no-op, not an error. If the process has already exited, return success without raising. See *Cleanup Discipline* below for the mandatory drain step after terminate.

## 4. `arm_wake(source, regex, payload) → WakeId?`  *(optional)*

Register a wake condition: whenever a line matching `regex` appears in `source` (a `Handle` or a file path), inject `payload` as a new agent turn carrying the same conversation state.

| Field       | Type     | Meaning                                                                                |
| ----------- | -------- | -------------------------------------------------------------------------------------- |
| `regex`     | string   | Case-sensitive regex, anchored at line start by convention (use `^` in the pattern).   |
| `payload`   | string   | A free-form string or JSON object the agent should see as the new turn's user message.  |
| `source`    | Handle / path | What to watch.                                                                  |
| `debounce_ms` | int?  | Suppress repeats within this window. Defaults to 5000.                                  |

Returns a `WakeId` the agent can cancel. Returns `null` if the host does not support push-wake — in that case the agent polls `read_output` instead. See *Two Delivery Models* below for the polling equivalent.

The wake primitive is what makes the loop skill feel **real-time**. Without it, the agent must poll, which is acceptable for intervals ≥ ~30s but wasteful for tighter cadences.

## Sentinel + Payload Contract

Every wake is a single line of the form:

```
AGENT_LOOP_TICK_<purpose> {"prompt":"<the prompt to run on wake>"}
```

for fixed-schedule loops, or:

```
AGENT_LOOP_WAKE_<purpose> {"prompt":"<the prompt to run on wake>"}
```

for event-gated or dynamic-schedule loops.

- The sentinel is the regex the host matches against. Agents SHOULD anchor it (`^…`) to avoid false positives from inside-script echoes.
- The JSON payload is the actual instruction the agent must execute on wake. The host delivers the line; the agent parses the JSON and runs the `prompt` field as if the user had typed it.
- `purpose` is uppercased, ASCII, no spaces by convention (example: `AGENT_LOOP_TICK_DEPLOY_CHECK`). Kebab-case (`AGENT_LOOP_TICK_check-deploy`) is equally valid — the only constraint is uniqueness across concurrent loops.

The regex to arm is `^AGENT_LOOP_(TICK|WAKE)_<purpose>`. Sentinels must be **unique per loop** — if two loops share `AGENT_LOOP_TICK_check`, one loop's tick can wake the other.

## What "Wake" Means

A wake is a host event that returns control to the agent **along with the matched line**. The agent's contract on wake is:

1. Read the JSON `prompt` field.
2. Execute it as if the user had typed it.
3. Re-arm the next tick (fixed schedule) or the next heartbeat (dynamic schedule).
4. Briefly report what changed since the last tick — or, on the first tick, that the loop is armed, when the next tick will arrive, and how to stop it.

A wake is **not** a stop signal. To stop a loop, the user must say so explicitly; the agent then runs `terminate(handle)`.

## Two Delivery Models

Hosts differ in **how** they deliver a wake. The skill itself does not care; the per-host files document each:

- **Push (Cursor-style)**: the host watches the backgrounded process and pushes a wake event to the agent when the sentinel matches. The agent receives the line without polling.
- **Pull (polling-style)**: the host buffers stdout; the agent must call `read_output` (or `BashOutput` / equivalent) periodically to discover the line. There is no push.

Both models are equivalent in behaviour. The pull model just means the agent has to spend one tool call per check.

## Why a JSON Payload, Not Just a Sentinel

The skill carries the prompt **beside** the sentinel instead of letting the host inject it, because the host does not know what the user wants re-run. A single sentinel line with the prompt inline means:

- The same task can be re-purposed mid-loop (the agent edits the payload before re-arming).
- The same code path handles fixed and dynamic schedules.
- A wake carries enough context that the agent can re-derive state from the transcript, not from a side channel.

## Cleanup Discipline

When stopping a loop, the host may still emit a completion event (Cursor does). The agent MUST consume that event — by calling `terminate(handle)` and then `read_output(handle)` one final time with a short timeout — before declaring the loop stopped. Otherwise the next `/loop` from the user can spuriously wake on the dead task's exit.

This is why `terminate` is listed as idempotent and why `read_output` must not block: the final drain is a one-shot poll, not a wait.

## Time parsing

`parse_interval(s) → seconds` must accept:

| Input                         | Seconds |
| ----------------------------- | ------- |
| `30s`, `30 sec`, `30 seconds` | 30      |
| `5m`, `5 min`, `5 minutes`    | 300     |
| `2h`, `2 hr`, `2 hours`       | 7200    |
| `1d`, `1 day`                 | 86400   |
| bare integer                  | treated as seconds |

Reject ambiguous inputs. Reject zero and negative — a zero-interval loop pegs the CPU and produces no useful ticks.

## Capability detection

At the start of a `/loop` turn, the agent should check what it has:

| Has                            | Can do                                                  |
| ------------------------------ | ------------------------------------------------------- |
| `spawn_loop` + `read_output`   | Fixed schedule with polling.                            |
| + `terminate`                  | Also stop a running loop on user request.               |
| + `arm_wake`                   | Push-wake — the only way to feel real-time < 30s.       |
| None of the above              | Tell the user the host can't run a loop; suggest OS cron as a fallback. |