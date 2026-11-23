# Reference: an instrumented agent turn

The pattern to mirror when adding ChatSee telemetry to an agent. It is an ordinary
Anthropic-SDK agent with a tool-use loop; the ChatSee parts are the tracker calls. Adapt it to
the developer's own agent and framework rather than copying it whole.

```python
import os
import anthropic
from chatsee import ChatseeTracker

# Construct once per process or session, not per turn.
tracker = ChatseeTracker(
    agent_id=os.environ["CHATSEE_AGENT_ID"],
    tenant_id=os.environ["CHATSEE_TENANT_ID"],
    api_base_url=os.environ["CHATSEE_ENV"],   # never default this
    verify_ssl=True,                           # the SDK defaults to False: always pass True
    redaction_enabled=False,
)


def run_turn(client: anthropic.Anthropic, model: str, system: str, user_msg: str) -> str:
    """One user -> assistant exchange, tool calls included, reported to ChatSee as one turn."""
    tracker.start_turn(
        user_message=user_msg,
        system_prompt=system,
        metadata={"model": model, "code_version": os.environ.get("GIT_SHA")},
    )
    messages = [{"role": "user", "content": user_msg}]
    try:
        while True:
            resp = client.messages.create(
                model=model, max_tokens=4096, system=system, tools=TOOLS, messages=messages
            )
            messages.append({"role": "assistant", "content": resp.content})
            tool_uses = [b for b in resp.content if b.type == "tool_use"]
            if not tool_uses:
                answer = "".join(b.text for b in resp.content if b.type == "text").strip()
                break
            results = []
            for tu in tool_uses:
                out = call_tool(tu.name, tu.input)            # the agent's own tools
                tracker.log_tool_call(tu.name, tu.input, result=out)
                results.append({"type": "tool_result", "tool_use_id": tu.id, "content": str(out)})
            messages.append({"role": "user", "content": results})
    except Exception as e:
        # Report the failure and still end the turn: a broken turn is data the Guardian wants,
        # and telemetry must never hide the agent's own error.
        tracker.log_exception(e)
        tracker.end_turn(bot_message=f"(error: {e})")
        raise

    tracker.end_turn(bot_message=answer)   # sends the whole turn to ChatSee
    return answer
```

After a turn, `tracker.session_id` is set when ChatSee accepted the interaction. No
`session_id` means it did not land: check the ids and `CHATSEE_ENV`.
