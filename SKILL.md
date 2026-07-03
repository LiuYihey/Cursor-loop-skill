---
name: loop
description: >-
  Run a prompt or skill in this session on a recurring or variable interval
  (e.g. /loop 5m /foo).
disabled-environments:
  - cloud
---
# Loop

## Parse

Accept `/loop [interval] <prompt>`.

- Leading interval: `5m /foo`, `30s check status`, `2h run report`.
- Trailing interval: `check deploy every 5m`, `run tests every 10 minutes`.
- No interval: dynamic mode; the agent chooses the next delay after each run.
- Empty prompt: show `Usage: /loop [interval] <prompt>`.

Use intervals like `30s`, `5m`, `2h`, `1d`. Convert unit words to short units.

Use output-pattern wake to fire on each tick — see `references/primitives.md` for the host-agnostic spec.

## Fixed Schedule

```bash
while true; do
  sleep <seconds>
  echo 'AGENT_LOOP_TICK_<purpose> {"prompt":"<prompt>"}'
done
```

1. Check existing terminals for an already-running matching loop.
2. Start one background shell loop. If your host exposes output-pattern wake, arm it on the sentinel regex — see `references/primitives.md` §4. Otherwise, fall back to polling `read_output` once per tick.
3. Use a unique sentinel and a regex such as `^AGENT_LOOP_TICK_<purpose>`.
4. Smoke-check once to confirm clean startup.
5. Run the prompt once immediately after arming the loop.
6. The first sentinel should arrive only after the initial sleep, so startup does not double-run the prompt.
7. Track the PID so the agent can stop the loop if asked.
8. Briefly confirm: the interval, that the prompt already ran once, when the first tick will arrive, and that the loop will fire on each tick until stopped. On later ticks, give a short update of what changed. On stop, say the loop has stopped and why.

## Dynamic Schedule

The user wants the agent to self-pace. Decide what makes the next iteration worth running — a passage of time, or an observable event.

1. **Run the prompt now.**
2. **If the next run is gated on an event** (a git ref advancing, a log line matching, a file changing, a CI check completing), arm a background watcher that emits the sentinel only when the event fires — see `references/shells.md` for ready-to-paste watcher bodies. If the host supports output-pattern wake, register a wake on `^AGENT_LOOP_WAKE_<purpose>` (see `references/primitives.md` §4). Arm once; skip on later ticks if it's still running.
3. **At the end of the turn, arm a one-shot time-based wake**:

```bash
sleep <seconds>
echo 'AGENT_LOOP_WAKE_<purpose> {"prompt":"<prompt>"}'
```

   With a watcher armed, this is the **fallback heartbeat** — lean long so idle ticks aren't pure overhead. Without a watcher, this is the cadence — pick a delay based on when the result is worth checking again.

4. **On wake**, read the latest payload, execute its `prompt`, then re-arm the next heartbeat (and re-arm the watcher only if it exited). If both an output wake and a completion notification arrive, act on the output and ignore the completion.
5. **To stop**, kill any watcher PID and don't arm the next heartbeat.
6. Briefly confirm: that you're self-pacing, whether a watcher is the primary wake signal, what fallback delay you picked, and that the prompt already ran.

## Prompt Payload

Wake notifications include an output file path, not a submitted prompt. Put the prompt beside the sentinel, preferably as JSON. On wake, read the latest matching line and act on its `prompt`. The prompt may vary by tick.

## Guidance

- Title shell commands as `Loop <schedule>: <prompt>` (e.g. `Loop every 5m: check deploy status`).
- Adapt loop syntax to the user's shell (e.g. PowerShell `while ($true) { ... Start-Sleep }` on Windows). The examples above use bash.
- Prefer host output-pattern wake (see `references/primitives.md` §4) over OS cron when the agent needs wake notifications — stdout stays attached to the monitored task, and the wake fires the moment the sentinel lands.
- Use a unique sentinel per loop so unrelated output does not trigger notifications.
- Avoid noisy commands inside the loop.
- Do not create duplicate fixed loops or dynamic sleepers.
- If the user asks to stop, kill any tracked loop/sleeper PID, then await the shell task so its completion notification is consumed and does not wake the agent later. Do not schedule another dynamic wake.

## Reference Files

The supporting files in `references/` extend this skill with the universal contract and shell-specific templates. Read them on demand.

| File | Read this if… |
| --- | --- |
| `references/primitives.md` | Always — defines the 4 host-agnostic primitives (`spawn_loop`, `read_output`, `terminate`, `arm_wake`), the sentinel + payload contract, time parsing, capability detection, and cleanup discipline. |
| `references/shells.md` | The loop body is event-gated (dynamic schedule) or the user's shell is not bash. Provides ready-to-paste templates for sh, bash, zsh, PowerShell, fish, plus event-watcher variants. |

## Adding a Per-Host Mapping

If you want a per-host file that translates the four primitives into a specific agent's tool names, create `references/<your-host>.md` with this shape:

- One section per primitive, in the same order as `primitives.md` §1–§4.
- Each section shows a concrete tool call (or a short list of calls) that fulfils the primitive, plus a one-line note on the failure modes documented in the spec.
- A **Polling fallback** subsection under §4, describing what the host does if it cannot push-wake.
- A **Caveats** subsection at the end (Windows shell quirks, deduplication policy, output-buffer growth, etc.).

This skill itself stays host-agnostic; the per-host file is read by an agent only when it recognises itself in the filename.
