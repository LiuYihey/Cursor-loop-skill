---
name: loop-shells
description: Inner-loop body templates for the four most common shells. The loop body is just sleep, then print a sentinel line, then repeat — the agent's host wires the wake. Pick the template that matches the user's shell.
---

# Shell Templates

The loop body is the only piece of the loop skill that depends on the user's shell. All four templates below are semantically equivalent: sleep, then print a sentinel line, then repeat.

## bash / zsh  (default)

```bash
while true; do
  sleep <seconds>
  echo 'AGENT_LOOP_TICK_<purpose> {"prompt":"<prompt>"}'
done
```

- Use **bash** on Linux and macOS.
- **zsh** (default on macOS since Catalina) accepts the same syntax.

## sh (POSIX, Alpine / busybox)

```sh
while true; do
  sleep <seconds>
  printf '%s\n' 'AGENT_LOOP_TICK_<purpose> {"prompt":"<prompt>"}'
done
```

`echo` is not portable across all `sh` implementations (some expand backslash escapes, some do not). `printf '%s\n'` is.

## PowerShell  (Windows)

```powershell
while ($true) {
    Start-Sleep -Seconds <seconds>
    Write-Output 'AGENT_LOOP_TICK_<purpose> {"prompt":"<prompt>"}'
}
```

Notes:

- `while ($true)` with the dollar sign — PowerShell variable syntax.
- The single quotes are required to keep `$true` from being expanded if the body is embedded in a JSON string passed to a tool.
- Use `Start-Process -NoNewWindow powershell -ArgumentList '-Command', '<body>'` if the calling tool cannot background a long-running command directly.

## fish

```fish
while true
    sleep <seconds>
    echo 'AGENT_LOOP_TICK_<purpose> {"prompt":"<prompt>"}'
end
```

`end` closes the loop, not `done`.

## Dynamic schedule — event watcher  (any shell)

For event-gated loops, the sentinel is `AGENT_LOOP_WAKE_<purpose>` and the body fires the sentinel only when the event happens. The example below watches a git ref and fires when `main` advances:

```bash
while true; do
  sleep <fallback_seconds>          # e.g. 600s — lean long, see SKILL.md §Dynamic
  current=$(git rev-parse HEAD)
  last=$(cat /tmp/last_seen_ref 2>/dev/null || echo "")
  if [ "$current" != "$last" ]; then
    echo "$current" > /tmp/last_seen_ref
    echo 'AGENT_LOOP_WAKE_git-main {"prompt":"main advanced to '"$current"'"}'
  fi
done
```

Two more event patterns, ready to paste:

```bash
# Log-line watcher — fires when "ERROR" appears in app.log
tail -F /var/log/app.log 2>/dev/null \
  | awk '/ERROR/ { print "AGENT_LOOP_WAKE_app-errors {\"prompt\":\"error: " $0 "\"}"; fflush() }'

# File-mtime watcher — fires when a file changes
while true; do
  sleep <fallback_seconds>
  if [ "/tmp/watched" -nt /tmp/watched.mtime ]; then
    touch /tmp/watched.mtime
    echo 'AGENT_LOOP_WAKE_file-change {"prompt":"watched file changed"}'
  fi
done
```

The `tail -F | awk` form needs `fflush()` so the sentinel reaches the host's wake filter promptly, otherwise buffering can delay it by tens of seconds.

## Quoting the prompt

The sentinel is a single-quoted JSON object. If the prompt itself contains a single quote, switch the outer quotes to double and escape any inner double quotes:

```bash
echo "AGENT_LOOP_TICK_check {\"prompt\":\"deploy status: $(date)\"}"
```

The agent that wakes up parses the JSON and extracts the `prompt` field — quoting rules above are exactly the rules `jq` would require.