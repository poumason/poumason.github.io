
---
title: How using Claude Code Agent SDK to programming
date: 2026-03-31 22:40:11
categories: AI
tags:
    - Claude Code
    - Skills
    - Agents
---
Claude code is powerful agent for coding. I want to create skills and using Claude code to execute, but i need it run on container. how can i do?
<!-- more-->

Claude code provider Agent SDK for deveploper to easy create powerful agent with exist skills or plugins.



```
import anyio
import os
from claude_code_sdk import query, ClaudeCodeOptions

from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, ResultMessage

# Redirect to the local router
os.environ["ANTHROPIC_BASE_URL"] = "http://127.0.0.1:8000"
os.environ["ANTHROPIC_AUTH_TOKEN"] = ""
os.environ["ANTHROPIC_MODEL"] = "Qwen3.5-9B-MLX-4bit"

async def main():
    # Agentic loop: streams messages as Claude works
    async for message in query(
        prompt="execute skill 'hello-world'",  # Example prompt to execute a skill
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Edit", "Glob", "Skill"],  # Tools Claude can use
            permission_mode="acceptEdits",  # Auto-approve file edits
            cwd='./'
        ),
    ):
        # Print human-readable output
        if isinstance(message, AssistantMessage):
            for block in message.content:
                if hasattr(block, "text"):
                    print(block.text)  # Claude's reasoning
                elif hasattr(block, "name"):
                    print(f"Tool: {block.name}")  # Tool being called
        elif isinstance(message, ResultMessage):
            print(f"Done: {message.subtype}")  # Final result



if __name__ == "__main__":
    anyio.run(main)

```

# References
- [Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
- [Agent Skills in the SDK](https://platform.claude.com/docs/en/agent-sdk/skills)
- [Run Claude Code programmatically](https://code.claude.com/docs/en/headless)
- [claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python)
- [claude-code-router](https://github.com/musistudio/claude-code-router)