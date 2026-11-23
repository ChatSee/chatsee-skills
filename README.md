# ChatSee Skills for Claude Code

**Monitor your AI agent and fix its failures without leaving Claude Code.**

[ChatSee](https://chatsee.ai) watches a deployed AI agent's real conversations and its
**Guardian** finds what goes wrong: policy violations, incomplete answers, tool errors, loops.
This plugin connects Claude Code to ChatSee so you can ask what is failing, apply the fixes
ChatSee has approved to your agent's prompt, and get code-level fixes proposed, all from your
editor. **Requires a ChatSee account.**

## What you get

Four skills and one MCP server, working as one loop:
**set up telemetry → ask what's wrong → fix the prompt → fix the code → repeat.**

| Skill | What it does |
|-------|--------------|
| **`telemetry-setup`** | Adds the ChatSee SDK to your agent's code so each conversation is reported to ChatSee. |
| **`guardian-query`** | Ask the Guardian in plain English: top errors, clusters, anomalies, sentiment. |
| **`remediation`** | Shows ChatSee's remediations with their status, lets you approve the waiting ones, and writes the approved rules into your agent's system prompt. |
| **`structural-correction`** | Finds code-level failures (truncation, wrong tool calls, loops) and proposes a code fix for you to approve. |

Every step is developer-invoked and transparent: you ask, you see the evidence and the diff,
and you approve the change. Nothing is applied silently.

The MCP server provides six tools: `list_my_agents`, `ask_guardian`, `list_remediations`,
`approve_remediation`, `get_remediations` and `apply_remediations`.

## Install

In Claude Code:

```
/plugin marketplace add ChatSee/chatsee-skills
/plugin install chatsee-skills@chatsee
```

Then sign in: run `/mcp`, choose **chatsee**, and select **Authenticate**. Your browser opens
on ChatSee: sign in, click **Continue**, then **Allow**. Claude Code sees only the agents your
ChatSee account can see. There is no ChatSee API key to set.

## Try it

- *"Set up ChatSee telemetry for my agent"*
- *"What are the top errors for my agent in the last 24 hours?"*
- *"Show my ChatSee remediations and apply the approved ones"*
- *"Find the code bug behind my agent's truncated replies"*

## What this plugin runs and sends

The plugin adds four skills (instructions for Claude, in Markdown) and one remote MCP server.
It contains **no scripts, no hooks and no analytics**: nothing runs automatically, and nothing
is collected about you or your Claude Code sessions.

| Part | What it does | What leaves your machine, and where to |
|------|--------------|----------------------------------------|
| **ChatSee MCP server** (`https://app.chatsee.ai/mcp/`) | Answers Guardian questions and lists, approves and acknowledges remediations | You sign in with your ChatSee account (OAuth, in the browser). Each tool call sends its arguments to ChatSee over HTTPS: your question, the agent name, remediation IDs and their status. Access is limited to the agents your account can see. |
| **`telemetry-setup`** | Edits **your agent's** code to add the `chatsee-ai` SDK (from PyPI), and writes the agent's ChatSee IDs to your project's `.env` | Only after you deploy that change: your agent sends each of its user interactions (prompt, answer, tool calls, metadata) to ChatSee over HTTPS with certificate checks on. The skill does not send anything itself. |
| **`guardian-query`** | Asks the Guardian about your agent's errors | The question and agent name, through the MCP server above |
| **`remediation`** | Shows ChatSee's proposed prompt fixes, applies the ones you approve to your prompt file, then tells ChatSee they were applied | Approvals and "applied" acknowledgements, through the MCP server. A local record is written to `.chatsee/applied.jsonl` beside the prompt file. |
| **`structural-correction`** | Diagnoses code-level failures from the Guardian's data and proposes a code change | Questions through the MCP server. The code change is proposed in your editor and applied only if you approve it. |

**Your source code is not uploaded by the plugin.** Claude reads and edits files in your
project locally, and only when you ask. What reaches ChatSee is what the table above lists.

- Privacy policy: <https://www.chatsee.ai/privacy-policy>
- Terms of service: <https://www.chatsee.ai/terms-of-service>
- Support and security issues: **contact@chatsee.ai** (please report security issues
  privately, not in a public issue).

## License

Apache License 2.0; see [`LICENSE`](LICENSE) and [`NOTICE`](NOTICE). The license covers this
repository only; ChatSee's hosted services are provided under ChatSee's Terms of Service.
