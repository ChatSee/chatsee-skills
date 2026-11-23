---
name: remediation
description: Show an agent's ChatSee remediations with their portal status (Open, Proposed, In progress, Implemented), let the developer approve the ones waiting for approval, fold the approved (Implemented) rules into that agent's system prompt, then confirm them applied. The transparent, developer-invoked form of the closed loop, for prompt-level fixes. (Skill 2 of 4.)
---

# Remediation (skill 2)

Shows the agent's remediations as ChatSee shows them, lets the developer approve the ones
still waiting, and applies the approved rules to the target agent's **system prompt**:
transparent and developer-invoked, the developer sees the diff before anything is written.

## First: is ChatSee connected?
Use the ChatSee MCP tools (`list_my_agents`, `list_remediations`, `approve_remediation`,
`get_remediations`, `apply_remediations`, `ask_guardian`). If they are missing, or a call
fails with an authentication error, **stop and tell the developer to sign in** (`/mcp` → the
ChatSee server → **Authenticate**), then retry.

**Never search for API keys, `.env` files, environment variables or config files to get
around a missing sign-in,** and never read files outside the project.

## Where this sits in the loop
```
1 telemetry → [ChatSee classifies] → 4 ask → [a human approves] → ▶ 2 apply → 3 code fix
```

**This skill applies *approved* fixes, not detected errors.** Detection is automatic;
approval is a deliberate human decision, made in the ChatSee portal or here, by the developer,
one remediation at a time. An agent can carry dozens of incidents and still have nothing
approved.

## Use ChatSee's words for status
A remediation has a **status**, and it is the one the portal's Remediations screen shows.
Always use these words and the **FM code** (FM-004…), so the developer can find the same row
in the portal:

| Status (portal) | Means | What this skill offers (`next_step`) |
|---|---|---|
| **Open** | Logged; may or may not have a fix yet | Approve and apply, if it has a fix; otherwise nothing |
| **Proposed** | A fix is waiting for sign-off | Approve and apply |
| **In progress** | Being worked on, not signed off | Approve and apply |
| **Implemented** | Approved. The fix should be in effect | Apply, or nothing if it is already in the prompt |
| **Regressed** | Was Implemented, then marked as not working | Approve and apply again |

Two kinds are never applied to a prompt: a remediation **with no fix yet**, and a **Jira**
remediation (its work ships through the ticket). `list_remediations` marks both `none`.

**Approving = the portal's Approve & Apply.** It sets the status to **Implemented** in ChatSee, with
the same history and versioning as the portal button. It needs the **SRE or Admin** role, the
same as in the portal; `list_remediations` returns `can_approve` for the signed-in person.

**Nothing Implemented is not "healthy".** It means nothing is approved yet. Say how many are waiting
(Proposed, In progress, Open), and run **skill 4** to show what was actually detected.

## Which agent
Pass `agent_name` to every remediation tool. You can reach only agents the signed-in person
is assigned, the same as in the portal. If they have one agent, it is used automatically; if
several, the tool returns their list: call `list_my_agents` or ask which one, never guess.

## Finding and editing the prompt

**Every agent stores its prompt somewhere different, and you can read the codebase.** Find it,
edit it, verify it, acknowledge it. ChatSee leaves the edit to the developer's own tooling and
the MCP call is only the acknowledgement: **you are that tooling.**

| Where the prompt lives | How you apply the fix |
|---|---|
| A prose file (`.md`, `.txt`) | Edit the file: put the rules in the managed block (rules below) |
| A string literal in code (Python, JS/TS, Go, Ruby) | Edit the literal in place. If the prompt is computed (f-string, concatenation), change only the literal part and say what you did |
| A YAML/JSON/TOML key | Edit that key, keeping the file valid |
| Assembled from fragments at runtime | Work out which fragment carries the system instructions; if genuinely ambiguous, ask |
| A database row, a CMS | Write it back through whatever the app already uses |
| A prompt-management service (Langfuse, PromptLayer, Braintrust) | Create a **new version** through their API — do not mutate one in place |
| A vendor console (Copilot Studio, Agentforce, Bedrock) | Not on disk. Hand the developer the exact text and where to paste it |

### Rules for the edit

1. **Keep the markers.** Wrap the rules in `<!-- chatsee:failure-memory:start -->` /
   `<!-- chatsee:failure-memory:end -->`, or the nearest equivalent the format allows. If the
   block already exists, **replace** it rather than adding a second copy. If the format cannot
   carry markers (a database column, a console text box), say so.
2. **Show the diff before writing.** Non-negotiable — this edits what the agent says to every
   customer. Write only after the developer agrees.
3. **Verify the file still loads.** Re-parse the YAML or JSON, check the code still compiles.
   A prompt file that no longer parses takes the agent down.
4. **Keep a local record.** Append one line of JSON to `.chatsee/applied.jsonl` beside the
   prompt file: the time, the agent name, the prompt file and the FM codes applied.
5. **Then acknowledge** with the `apply_remediations` tool, or ChatSee still believes the fix
   was never applied.

## Tools
- `list_remediations(agent_name)` — every remediation with `fm_code`, `title`,
  `status_label`, `next_step`, `reason`, the rule text (`fix`) and `can_approve`.
  **Read-only**: call it as often as you like.
- `approve_remediation(failure_memory_id, agent_name)` — the portal's Approve & Apply for ONE
  remediation; it becomes **Implemented**. **State-changing, in the portal.** Only after the
  developer's explicit yes for that remediation (Steps, below).
- `get_remediations(mode, agent_name)` — the **Implemented** rules only, as `skills_markdown` (each
  rule labelled with its FM code) plus the `failure_memory_ids` you confirm later. `mode` is
  `all` (default) or `pending`.
- `apply_remediations(failure_memory_ids, details, agent_name)` — acknowledge rules that are
  now in the prompt. Refuses anything not Implemented. **State-changing.**

## Two separate states
1. **Status** (the portal's, table above): has a person approved it? Only **Implemented** may be applied.
2. **In the prompt or not** (`in_prompt`): has a coding agent put it in this agent's prompt?
   Behind it are three delivery states you may see in raw output:

| Delivery state | Means |
|---|---|
| **Created** | Approved, never fetched by a coding agent. |
| **Delivered** | Fetched. Says nothing about whether it was applied. |
| **Applied** | Confirmed in the agent's prompt (`in_prompt: true`). |

Delivery states are **not** portal statuses; never present "Delivered" or "Applied" as the
remediation's status. Fetching with `get_remediations` moves a rule `Created → Delivered`, so
its `pending` means *"never fetched"*, not *"waiting to be applied"*; this skill says **new**
instead and **defaults to `all`**. `list_remediations` changes nothing.

## Steps
1. **Show.** Call `list_remediations` (with `agent_name` if needed). Show a table, newest
   first: FM code · title · status (portal words) · what can be done (from `next_step`):
   - `apply`: "Implemented, ready to apply"
   - `done`: "Implemented, already in the prompt"
   - `approve_and_apply`: "<status>, needs approval"
   - `none`: the `reason` (no fix yet / Jira)

   Add one line of counts, e.g. "2 Implemented (1 already applied) · 1 Proposed · 1 Open, no fix".
2. **Let the developer choose.** Offer: apply the Implemented ones, and approve any of the
   `approve_and_apply` ones. If `can_approve` is false, say that approving needs an SRE or
   Admin (`approval_note`) and continue with the Implemented ones only.
3. **Approve one at a time, only on an explicit yes.** For each remediation the developer
   wants approved, show its `fix` text and ask: *"Approve FM-003 in ChatSee? It becomes Implemented."*
   Call `approve_remediation` only after a clear yes for **that** remediation. Never approve
   because the developer said "apply the fixes", never approve several on one vague yes, and
   never approve on your own initiative. Report each result; if one is refused, show the
   tool's message and carry on with the rest.
4. **Fetch the Implemented rules.** Call `get_remediations` with `mode: "all"`. The managed block in
   the prompt holds **every Implemented rule**: the ones already applied stay, newly approved
   ones join. If the developer does not want an Implemented rule in effect, that is a decision
   for the portal (mark it Regressed), not something to drop silently here.
5. **Apply.** Find the prompt, show the diff, and after the developer agrees, write the rules
   into the managed block following the rules for the edit above.
6. **Confirm.** Once the rules are truly in place, call `apply_remediations` with the
   `failure_memory_ids` and a note saying what changed.
   ⚠️ **State-changing:** records each one as in the prompt (`Applied`).
7. **Close.** Call `list_remediations` again and show the updated table, so the developer
   sees the same statuses the portal now shows.
