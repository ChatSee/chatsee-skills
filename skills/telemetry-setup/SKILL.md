---
name: telemetry-setup
description: Instrument a deployed AI agent's code so each of its user interactions (prompt, answer, tool calls) streams to the ChatSee Guardian, using the official chatsee-ai SDK. Use when a developer wants their production agent monitored by ChatSee, or says telemetry/tracing/monitoring is not set up. This edits the agent's own source; the subject is its real end-user conversations, not the Claude Code session. (Skill 1 of 4.)
---

# Telemetry setup (skill 1)

**This skill edits the developer's agent code.** It finds the agent's turn loop and adds four
SDK calls so every real conversation is reported to ChatSee, where the Guardian analyzes it and
skills 2–4 can act on it.

**Subject = the agent's real end-user conversations**, never the Claude Code session.

## First: is ChatSee connected?
Use the ChatSee MCP tools (`list_my_agents`, `ask_guardian`, `list_remediations`,
`get_remediations`, `apply_remediations`). If they are missing, or a call fails with an
authentication error, **stop and tell the developer to sign in** (`/mcp` → the ChatSee server
→ **Authenticate**), then retry.

**Never search for API keys, `.env` files, environment variables or config files to get
around a missing sign-in,** and never read files outside the project.

## Where this sits in the loop
```
▶ 1 telemetry → [ChatSee classifies] → 4 ask → [a human approves] → 2 apply → 3 code fix
```

Nothing downstream exists until this is done and traffic has flowed.

## What you are installing

Four calls, wrapped around one turn of the agent's existing loop:

```python
from chatsee import ChatseeTracker

tracker = ChatseeTracker(agent_id=AGENT_ID, tenant_id=TENANT_ID,
                         api_base_url=CHATSEE_ENV, verify_ssl=True)

tracker.start_turn(user_message=user_msg, system_prompt=system_prompt,
                   metadata={"model": model, "code_version": git_sha})
tracker.log_tool_call(tool_name, tool_input, result=tool_result)   # once per tool call
tracker.log_exception(exc)                                         # on failure
tracker.end_turn(bot_message=answer)                               # sends the turn
```

The agent is identified by **`agent_id` + `tenant_id`**; no key is needed. Sending fails
**open**: if ChatSee is unreachable the agent keeps serving users. A full worked example is in
[references/instrumented-agent.md](references/instrumented-agent.md).

## Steps

1. **Find the turn loop.** Locate where a user message enters and a final response leaves —
   the function handling one exchange. In a raw SDK agent that is the `while` loop around
   `messages.create`; in a framework it is the handler or callback that returns the reply. If
   there are several entry points, instrument the one real users hit.

2. **Add the SDK** to the agent's dependencies: the `chatsee-ai` package from PyPI, installed
   into the environment *the agent* runs in. Tell the developer what you are adding.

3. **Get the ids from ChatSee, don't invent them.** If the agent's config already has
   `agent_id`, `tenant_id` and an environment, use those.

   **If the developer pastes the portal's "Connect Source" snippet**, take only `agent_id`,
   `tenant_id` and the environment (or `api_base_url`) from it, not the snippet itself (see
   *Getting the ids*). Then call `list_my_agents` and check that the pasted `agent_id` is
   listed and the `tenant_id` matches. If either is off, say so before writing anything — a
   typo here sends telemetry nowhere, with no error.

   Otherwise call the **`list_my_agents`** tool: it returns `tenant_id`, the SDK
   `environment` this ChatSee serves, and the agents the signed-in person can reach, each with
   its `agent_id`. If it lists several agents, ask which one this code is — never pick. If the
   agent isn't listed, it hasn't been added to ChatSee yet (see *Getting the ids*). **Never
   write a default environment** — a plausible guess silently sends production traffic to the
   wrong place. If `list_my_agents` returns no `environment`, ask.

   Write the three values to the project's `.env` as `CHATSEE_AGENT_ID`, `CHATSEE_TENANT_ID`
   and `CHATSEE_ENV`. None of them is a secret.

   ⚠️ **Check `.env` is gitignored before you finish.** It usually holds the agent's model API
   key. Add `.env` to `.gitignore` if it is missing. If the agent has no way to read `.env`
   yet, `python-dotenv` is the normal choice; don't make the developer export variables by hand.

   **Do not tell the developer to export the model key in their shell**: anything started from
   that shell inherits it, including Claude Code, which would then bill that key instead of
   their subscription. A file the agent reads does not have that problem.

4. **Wrap the loop** with the four calls above. Construct the tracker once per process or
   session, not per turn.

5. **Leave the system prompt where it is.** Skill 2 can edit a prompt wherever it lives, so do
   not restructure someone's agent as a precondition for using the product. Mention a separate
   `.md`/`.txt` prompt file only as an option: it can be changed without a code deploy.

6. **Offer to verify — do not just run their agent.** Running an agent spends money on their
   model provider, puts a real interaction into their ChatSee data, and **fires their agent's
   tools for real** — which might send an email, charge a card, or write to a production
   database. You cannot tell from the code whether it is safe.

   So finish the edit, then say what you would run and what it will touch, and let them
   decide. Run it yourself only if they say yes, or if they already asked you to test it.

   What to look for when it is run: a successful send sets `tracker.session_id`. **No
   `session_id` means it did not land** — check the ids and the environment.

7. **Then set expectations and stop.** Classification is asynchronous: it can take from a few
   minutes to over an hour. Until it finishes the Guardian answers *"no errors have been
   detected"*. **Do not re-instrument on the strength of that answer** — if `session_id` came
   back, the transport works. Say so, and give the rough wait.

## Details that matter

⚠️ **`verify_ssl` defaults to `False`** in the SDK, and it suppresses the warning, so TLS
certificates would go unchecked on real prompt traffic. Always pass `verify_ssl=True`.

**PII redaction: leave it off for now.** Pass `redaction_enabled=False`. Turning it on today
only prints a warning on every turn; mention to the developer that redaction exists.

**Always send `metadata`.** A free-form dict the SDK forwards as is. At minimum `model` and
`code_version` (a git sha), so "did this start when we switched models or shipped that build?"
can be answered later.

**The environment for ChatSee production (`app.chatsee.ai`) is `poc`.** If `list_my_agents`
reports `prod` or `production`, write `CHATSEE_ENV=poc`: the SDK has no `prod` alias and would
treat the word as a URL, so telemetry would go nowhere with no error.

## Getting the ids

Signed in with ChatSee, `list_my_agents` gives them (step 3). For an agent that doesn't
exist in ChatSee yet, someone adds it first: ChatSee portal → **Settings → Add agent** → name
it, type **interaction** + **custom** → Add. The ids are created there.

If the portal's "Connect Source" snippet passes `user_id`, `tenant_name` or `api_key`, leave
those out: `agent_id` + `tenant_id` are the only required arguments of `ChatseeTracker`.

The SDK is Python-only. A TypeScript or Go agent has to send interactions to ChatSee's REST
API directly.

## SDK version

Written against `chatsee-ai` **0.12.1**. That version also accepts two optional arguments
worth using:
- `start_turn(..., user_id=...)` — attribute a turn to a real end user.
- `log_tool_call(..., duration_ms=...)` — per-tool latency.
