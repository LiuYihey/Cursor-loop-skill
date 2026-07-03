# Loop Skill

A host-agnostic port of Cursor IDE's built-in `/loop` skill, documented around a 4-primitive contract so it runs on any agent that exposes a background process + output-pattern wake (Cursor, Claude Code, Codex, OpenCode, etc.).

## Install

```bash
# Project install (interactive; the CLI detects which agents you have)
npx skills add LiuYihey/Cursor-loop-skill
```

Other common variations:

```bash
# Global install to a specific agent, non-interactive
npx skills add LiuYihey/Cursor-loop-skill -a trae -g -y
npx skills add LiuYihey/Cursor-loop-skill -a claude-code -g -y
npx skills add LiuYihey/Cursor-loop-skill -a codex -g -y

# Preview the skills in this repo without installing
npx skills add LiuYihey/Cursor-loop-skill --list
```

Flags: `-a <agent>` target a specific agent, `-g` install globally to your user dir, `-y` skip prompts, `--all` install to every detected agent. See [`vercel-labs/skills`](https://github.com/vercel-labs/skills) for the full reference.

## Usage

Once installed, the agent understands:

```
/loop 5m check the deploy status
/loop 30s ping the test endpoint
/loop run the build every 10 minutes
```

- **Leading interval** — `/loop <interval> <prompt>` (fixed cadence).
- **Trailing interval** — `/loop <prompt> every <interval>` is also accepted.
- **No interval** — `/loop <prompt>` puts the agent in dynamic mode (self-paced).
- **Empty prompt** — prints `Usage: /loop [interval] <prompt>`.

## Files

- `SKILL.md` — the skill body. The only file the agent runtime must load.
- `references/primitives.md` — the 4-primitive host-agnostic contract (`spawn_loop` / `read_output` / `terminate` / `arm_wake`), plus the sentinel + payload convention, two delivery models (push / pull), time parsing, capability detection, and cleanup discipline.
- `references/shells.md` — ready-to-paste inner-loop templates for bash, sh, zsh, PowerShell, fish, plus event-watcher variants for dynamic schedules.

## Origin

Extracted from the `loop` skill that ships built into Cursor IDE, redocumented around a 4-primitive contract rather than Cursor's `Shell` tool. Cursor users don't need to install this to get `/loop` — it is already built in. Install this version if you want the generic primitive spec, or if you want to override Cursor's built-in with this version.