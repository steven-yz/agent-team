---
name: agent-team
description: Build autonomous multi-agent dev teams that collaborate across Codex threads using push-based communication. Use when the user wants to create a team (Manager/Dev/QA), set up autonomous spec--implement--verify--fix loops, or mentions multi-agent development. Proven through end-to-end testing. Triggers on "create a dev team", "set up agent team", "multi-agent development", or "$agent-team".
---

# Agent Team v5

Build a persistent 3-agent dev team that autonomously collaborates through `send_message_to_thread`. Agents are created ONCE and reused across versions. Communication is push-based (WHEN triggers), not poll-based.

## Proven design (tested 2026-07-30)

- **Never use `spawn_agent` for code tasks** -- file system isolation breaks output sharing
- **Default environment: local** -- Dev writes, QA reads directly. No git overhead.
- **Worktree mode: supported but requires commit+push/pull** -- detected from AGENTS.md convention
- **Agent init prompts MUST have WHEN triggers** -- without them agents stop and never notify peers
- **fix-verify loop closes Dev-QA autonomously** -- Manager only at endpoints (send spec, receive PASS)
- **QA must read `specs/{version}.md` directly** -- never infer criteria from context or file content
- **Clean output between versions** -- stale artifacts cause false PASS
- **Tool calls fail? Use `tool_search`** -- never hardcode alias names

## Environment detection

Before creating threads, determine environment:

1. Read `AGENTS.md` (if exists) -- look for "worktree" keyword
2. If worktree convention found -- use `environment.type = "worktree"`
3. Otherwise -- use `environment.type = "local"` (default, recommended)

**In local mode:** All agents share the project directory. Serial execution prevents conflicts (Dev finishes before QA starts).

**In worktree mode:** Each agent gets its own worktree path. Dev MUST commit + push before notifying QA. QA MUST pull before reading files.

## Role topology

```
User (project owner, says "go")
  |
  +-- Manager (single contact, spec + merge)
        |
        +-- Dev (implement)
        +-- QA (independent QA)
        +-- Release (merge + deploy + test)

Dev-QA loop is autonomous (Manager stays out)
```

User only talks to Manager. Dev and QA handle the fix-verify loop themselves.

## Three roles

| Role | Human role | Responsibility |
|------|-----------|---------------|
| Manager | Product Manager | Discuss requirements, write specs to `specs/{version}.md`, delegate, merge on PASS |
| Dev | Developer | Implement exactly what spec says. Write to EXACT path in spec. |
| QA | QA | Read `specs/{version}.md` directly. Verify ONLY against criteria in that file. Never infer. |

## Initialization workflow

1. Ask for team name (default: project name)
2. **Detect environment** using the rules above. Set `env_type = "worktree"` or `"local"`.
3. **Clean stale roster files**: If the repo contains any `roster.md` or `.agents/roster.md` from a previous team, delete or rename them. These files contain outdated thread IDs and will mislead agents.
4. **Get project context**: Use `create_thread` with `target.type = "project"` + `environment.type = env_type` + current project ID.
5. **Spawn agents**: For each agent, use the EXACT init prompt from [references/init-prompts.md](references/init-prompts.md). Fill in `{team}` and `{env-type}` and send verbatim.
6. Populate roster with actual thread IDs and worktree paths (if any)
7. Send each agent a boot message with the full roster inline (NOT via file -- worktrees cannot read uncommitted .agents/):

   To Manager:
   """
   BOOT-ROSTER for {team}:
   - You are Manager. Your thread ID: {manager_id}
   - Dev thread ID: {dev_id} -- send specs to this ID
   - QA thread ID: {qa_id} -- they will notify you on PASS

   Store these IDs. Do NOT read .agents/roster.md -- it may not exist in your worktree.
   """

   To Dev:
   """
   BOOT-ROSTER for {team}:
   - You are Dev. Your thread ID: {dev_id}
   - QA thread ID: {qa_id} -- notify them when implementation is complete
   - Manager thread ID: {manager_id} -- escalate only after 5 fix cycles

   Store these IDs. Do NOT read .agents/roster.md.
   """

   To Release (created with local environment for tool access):
   """
   BOOT-ROSTER for {team}:
   - You are Release Agent. Your thread ID: {release_id}
   - Manager thread ID: {manager_id} -- report deployment URL here
   - Main repo path: {main_repo_path}
   - Deploy command: {deploy_command}
   - Production URL: {production_url}

   Store these IDs. Do NOT read .agents/roster.md.
   """

   To QA:
   """
   BOOT-ROSTER for {team}:
   - You are QA. Your thread ID: {qa_id}
   - Dev thread ID: {dev_id} -- they will notify you when ready
   - Manager thread ID: {manager_id} -- notify them on PASS

   Store these IDs. Do NOT read .agents/roster.md.
   """

   To Release (created with local environment for tool access):
   """
   BOOT-ROSTER for {team}:
   - You are Release Agent. Your thread ID: {release_id}
   - Manager thread ID: {manager_id} -- report deployment URL here
   - Main repo path: {main_repo_path}
   - Deploy command: {deploy_command}
   - Production URL: {production_url}

   Store these IDs. Do NOT read .agents/roster.md.
   """
8. Tell user: "Talk to the Manager to start. Do NOT message Dev or QA directly."

## Per-version workflow

1. Manager writes spec to `specs/{version}.md` with numbered acceptance criteria
2. **Clean previous output**: Delete output directories/files from previous version
3. User says "go"
4. Manager reads roster, sends spec to Dev via `send_message_to_thread`
5. Dev reads spec, implements, commits (all changes in one commit), writes worklog, notifies QA
   - In worktree mode: push after commit, then notify
6. QA reads `specs/{version}.md` FIRST, verifies ONLY against its criteria, writes worklog
   - In worktree mode: pull first, then read files
7. PASS -> QA notifies Manager. FAIL -> QA notifies Dev (NOT Manager).
8. Dev fixes, re-notifies QA. Loop until PASS or max 5 cycles.
9. QA sends PASS directly to Release (cc Manager). Release merges, local smoke test, deploys, production smoke test, reports URL to Manager. Manager notifies user.

## Message format

All cross-thread messages use the format in [references/init-prompts.md](references/init-prompts.md) (Shared section). 

## Thread lifecycle

- Agents are created ONCE during initialization
- They persist across versions (same thread ID, same context)
- Manager reuses Dev and QA for every new version
- Only archive when user explicitly says "disband team" or project is complete

## Error handling for tool calls

If a tool call returns "not found":
1. Use `tool_search` with a query like "thread create send message"
2. Find the correct tool name from search results
3. Retry with the discovered name
4. Never guess tool names or try aliases

## Debugging

1. Read `.agents/roster.md` -- confirm all thread IDs are correct
2. Read `.agents/logs/manager.md` -- did Manager send the spec?
3. Read `.agents/logs/executor.md` -- did Dev implement? commit? notify QA?
4. Read `.agents/logs/verifier.md` -- did QA read the correct spec? use right criteria?
5. If a tool call failed: use `tool_search`, then retry
6. Go to the agent whose "Next" was never fulfilled and ask it directly

## Known pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| QA uses old ACs | PASS but wrong criteria | QA prompt: "Read specs/{version}.md DIRECTLY" |
| Stale output files | QA verifies old output | Clean output dirs before each version |
| Dev writes wrong path | File in wrong directory | Spec must include exact output path |
| worktree: stale reads | PASS but code broken | Dev push before notify; QA pull before verify |
| spawn_agent isolation | Output invisible | Never use spawn_agent for code tasks |
| Agent skips worklog | Can't trace | Init prompt enforces worklog |
| Thread archived mid-project | Messages fail | Unarchive with set_thread_archived |
| Tool name mismatch | Call fails | Use tool_search to discover correct name |
| stale roster committed to repo | Agent uses old thread IDs from previous team | BOOT-ROSTER overrides. Init prompt: ignore roster.md on disk. Cleanup step removes old files. |
| worktree: roster file unreachable | Agent cannot find peer IDs | Send roster as inline BOOT-ROSTER message, never via file |
| Agent does not call send_message_to_thread | Message only in final text, not sent | Add fallback: end response with ACTION REQUIRED prefix |
