---
name: guardian-query
description: Ask the ChatSee Guardian, in plain English from Claude Code, what's happening in production for a monitored agent — top error codes, error clusters, anomalies, sentiment. Read-only, developer-driven. Name an agent to scope the answer to it. (Skill 4 of 4.)
---

# Guardian query (skill 4)

Lets a developer ask the Guardian — from Claude Code, without the ChatSee portal — what's
happening in production. Read-only.

**Skill 3 vs. skill 4 (same data, different driver):** skill 4 = the *developer* asks and
reads; skill 3 = *Claude* consumes the same data and proposes fixes.

## First: is ChatSee connected?
Use the ChatSee MCP tools (`list_my_agents`, `ask_guardian`, `list_remediations`,
`get_remediations`, `apply_remediations`). If they are missing, or a call fails with an
authentication error, **stop and tell the developer to sign in** (`/mcp` → the ChatSee server
→ **Authenticate**), then retry.

**Never search for API keys, `.env` files, environment variables or config files to get
around a missing sign-in,** and never read files outside the project.

## Where this sits in the loop
```
1 telemetry → [ChatSee classifies] → ▶ 4 ask → [a human approves] → 2 apply → 3 code fix
```

**If it reports nothing detected:** classification has not finished yet (see skill 1) or the
agent has no telemetry. Not evidence of a healthy agent.

**Detecting an error does not create a fix.** Between this skill and skill 2 sits a human
approval step: a person (SRE or Admin) reviews ChatSee's suggested fix and approves it,
which makes it **Implemented**. That happens in the portal (Remediations → Approve & Apply) or in
skill 2, one remediation at a time on the developer's explicit yes. Nothing bridges the two
automatically, and that gate is deliberate: skill 2 rewrites a production system prompt.
So when the developer wants something fixed:
- **prompt-level** → skill 2: it shows each remediation's status (Open, Proposed,
  In progress, Implemented), lets them approve the waiting ones, and applies the Implemented ones
- **code-level** → skill 3, which needs no approval because it only proposes

Say which of the two you are recommending and why, rather than jumping straight to a fix.

## How to ask
Use the ChatSee MCP tools directly. They travel on the connection the developer signed
in with, so nothing needs a key:

- `list_my_agents` — which agents this person can see. Call it first when you're unsure
  which agent they mean.
- `ask_guardian(question, agent_name)` — the question, scoped to one agent by name.

Scoping follows the signed-in person's ChatSee permissions, exactly as the dashboard
does. A non-admin with several agents must name one; if they don't, the tool returns
their agent list — pick the one they meant and ask again rather than guessing. Admins
may omit `agent_name` to ask about the whole organization.

## Steps
1. If the agent isn't obvious, call `list_my_agents`.
2. Call `ask_guardian` with the question and `agent_name`.
3. Present the answer concisely. Answers may contain `<redirect index="…">` UI anchors;
   leave them out. Pass the returned `session_id` back to `ask_guardian` to ask a follow-up
   in the same Guardian conversation.
4. To fix something: hand off to skill 2 (prompt-level) or skill 3 (structural).

## What it can answer
- **List agents** ("what agents can you see?") — identify a target by name.
- **Top error codes** ("top error codes last 24h") — codes + human names + counts.
- **Anomalies / quality** ("unusual patterns last 7 days") — error rate, escalations.
- **Sentiment** ("how is sentiment trending?").
- **Follow-ups** in the same conversation, via `session_id`.

A Guardian answer usually takes 10–60 seconds; wider time windows take longer.

**Quote the error codes; sanity-check totals.** Error codes and their counts are reliable.
Interaction totals and the Guardian's own tallies ("4 patterns" followed by a list of 3) can
be off, so check them before repeating them.
