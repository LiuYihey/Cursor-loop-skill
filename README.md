# Cursor Loop Skill

A host-agnostic `/loop` skill. Drop it into any agent that exposes a background-process + output-wake primitive and the `/loop [interval] <prompt>` syntax starts working.

## Install

- **Cursor IDE**: copy `SKILL.md` and `references/` into `.cursor/skills/loop/` of your project (or your global skills dir).
- **Other agents**: read `SKILL.md` for the body, then read `references/primitives.md` to learn the 4-primitive contract. Write a `references/<your-host>.md` mapping those primitives to your host's tool names (shape described in `SKILL.md` §"Adding a Per-Host Mapping").

## Files

- `SKILL.md` — the skill body. The only file the agent runtime must load.
- `references/primitives.md` — the 4-primitive host-agnostic contract (`spawn_loop` / `read_output` / `terminate` / `arm_wake`), plus the sentinel + payload convention, time parsing, capability detection, and cleanup discipline.
- `references/shells.md` — ready-to-paste inner-loop templates for bash, sh, zsh, PowerShell, fish, plus event-watcher variants for dynamic schedules.

## License

See upstream Cursor's `loop` skill (`.cursor/skills/loop/SKILL.md` in any Cursor install) for the original. This repo adapts that skill to a host-agnostic shape.