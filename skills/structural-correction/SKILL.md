---
name: structural-correction
description: Diagnose a monitored agent's deeper, code-level failures (context-window handling, tool wiring, control flow, memory growth) from the ChatSee Guardian's data, then propose a code fix for developer approval. For failures a prompt tweak can't fix — the counterpart to skill 2 (prompt-level). Always scoped to one named agent. (Skill 3 of 4.)
---

# Structural correction (skill 3)

Skill 2 applies *prompt-level* fixes; skill 4 *surfaces* insights to a human; skill 3 has
**Claude** read the Guardian's data for **one agent** and fix **structural** problems a
prompt can't — context-window/memory handling, tool-wiring, control-flow bugs.

## First: is ChatSee connected?
Use the ChatSee MCP tools (`list_my_agents`, `ask_guardian`, `list_remediations`,
`get_remediations`, `apply_remediations`). If they are missing, or a call fails with an
authentication error, **stop and tell the developer to sign in** (`/mcp` → the ChatSee server
→ **Authenticate**), then retry.

**Never search for API keys, `.env` files, environment variables or config files to get
around a missing sign-in,** and never read files outside the project.

## Where this sits in the loop
```
1 telemetry → [ChatSee classifies] → 4 ask → [a human approves] → 2 apply → ▶ 3 code fix
```

**Runs independently of the approval step** — it reads detected failures directly, so it
works as soon as classification lands. Nothing needs approving first.

⚠️ **The loop does not close here.** Skill 2 acknowledges its fixes back to ChatSee; this
skill has no equivalent, so a code fix is not visible to ChatSee and there is **no regression
tracking for code changes**. Say that plainly when handing a fix over.

## Always scope to one named agent
The Guardian sees every agent in the organization. Asked without an agent, it can answer
about a different one. So every question names the agent: call `ask_guardian` with
`agent_name`. The server checks that the person may see that agent and scopes the question
to it. Don't know the name? Call `list_my_agents`; if it lists several, ask which one —
never pick for them.

## Steps
1. **Pull** structural insights for the agent with `ask_guardian(question, agent_name)`.
   Ask for structural signatures specifically, e.g. *"Which failures in the last 7 days
   point at code rather than the prompt — truncated replies, failures that grow with
   conversation length, wrong or repeated tool calls, loops that don't end? Give error
   codes, counts and example sessions."* Follow up with narrower questions to drill in.
   The Guardian can take a minute or two when the window has little data.
2. **Triage strictly** — most failures are NOT structural. **Route on the failure's
   behaviour, never on its error-code prefix**: the same prefix appears on both kinds.
   Claim *structural* only when the behaviour points at code — replies truncated
   mid-sentence, failures that scale with conversation length, the wrong tool called, a loop
   that won't terminate. Anything fixable by telling the model to behave differently is
   **prompt-level → hand to skill 2**. When unsure, prefer skill 2.
3. **Locate** the cause in the agent's code.
4. **Propose a fix** as a concrete edit, citing the Guardian evidence.
5. **Apply only with developer approval** (propose-only by default).

`ask_guardian` fetches the Guardian's view of one agent. That is all it does — the diagnosis,
locating the cause in the code, and writing the fix are Claude's job, working from the
developer's actual repository.
