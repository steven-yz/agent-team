# agent-team

Build autonomous multi-agent dev teams for [Codex](https://openai.com/codex). One command creates a persistent 4-agent team (Manager / Dev / QA / Release) that autonomously collaborates through `send_message_to_thread`.

## Install

```bash
$skill-installer install --repo steven-yz/agent-team
```

Or manually: `git clone https://github.com/steven-yz/agent-team ~/.codex/skills/agent-team`

## Usage

```
$agent-team Create a dev team for my project
```

Auto-detects worktree vs local, creates 4 persistent threads, sends BOOT-ROSTER.

## Flow

```
Manager (spec) -> Dev (implement) -> QA (verify) -> Release (merge+deploy+test)
                   ^                  |
                   +--- FIX ----------+  (autonomous, max 5 cycles)
```

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Workflow, environment detection, initialization |
| `references/init-prompts.md` | Exact prompts for all 4 agents |
| `references/message-format.md` | Cross-thread message format |
| `assets/roster-template.md` | Registration table template |
| `assets/worklog-template.md` | Worklog format |

## Proven

8 rounds of real-world feedback on Vue 3 + Supabase project. Works in local and worktree environments.

## License

MIT
