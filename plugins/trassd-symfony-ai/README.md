# trassd-symfony-ai

Skills and agents for the **Symfony AI** stack — AI Bundle setup, the Platform
component & model bridges (OpenAI / Anthropic / Gemini / VertexAI / Voyage / …),
building Agents with tool calling, Chat + memory, Store / RAG with vector
databases, the MCP bundle & servers, Mate, and multi-agent orchestration.

This is a [Claude Code](https://claude.com/claude-code) plugin. Its skills
trigger automatically when relevant, and its agents become available to the
`Agent` tool.

## Skills

| Skill | Covers |
|-------|--------|
| `symfony-ai-bundle-setup` | Installing & configuring the AI Bundle (ai.yaml); fast-track flow |
| `symfony-ai-platform` | Platform component, model bridges, model catalogs, invoking models |
| `symfony-ai-agents-tools` | Agents, tool calling, dynamic tools, human-in-the-loop, multi-agent orchestration |
| `symfony-ai-chat-memory` | Chat component, conversation history, memory, context compression |
| `symfony-ai-store-rag` | Store component, vector stores (local/SQLite/MongoDB/Supabase/S3), RAG pipelines |
| `symfony-ai-mcp` | MCP Bundle, building & wiring MCP servers/tools |
| `symfony-ai-mate` | Mate dev tool: project integration and writing extensions |

## Agents

| Agent | When to use |
|-------|-------------|
| `symfony-ai-code-reviewer` | Review AI Bundle config, platform/agent/tool wiring, chat/memory, and store/RAG setup against the Symfony AI docs. |

## Installing

This plugin is published through the **trassd** marketplace. Add the marketplace
(by local path or, once published, its git repo), then install:

```
/plugin marketplace add <git-repo-of-the-trassd-marketplace>
/plugin install trassd-symfony-ai@trassd
```

## License

MIT © Andreas van Hulst (see the marketplace `LICENSE`).
