# InitBots Agent Context

## Purpose And Status

InitBots is an opinionated Python framework for creating and operating chatbot backends.

The product goal is to generate a small chatbot application with an OpenAI-compatible API, configurable model providers, static knowledge retrieval, optional memory, dynamic context connectors, controlled operations, and an integrated admin panel.

This repository is currently greenfield. The architecture, interfaces, commands, and layouts below describe the target state; they do not imply that those components already exist. Inspect the repository and its tests before making changes.

This file is internal guidance for coding agents. Keep user-facing installation, tutorials, and API documentation in the official website and documentation repository. Keep `README.md` in this repository concise.

## Repository Architecture

This repository targets one Python distribution and one public package, `initbots`. It includes the runtime, CLI, FastAPI server, configuration, model providers, context retrieval, extension interfaces, tools, memory, authentication, databases, OpenAI-compatible API, and optional admin panel.

The admin is an integrated framework component, similar in placement to Django's admin. It uses server-rendered Jinja templates, HTMX, and Tailwind, and is shipped as part of the Python distribution. It must remain optional at runtime and must not expose or edit Python source code.

The official website, documentation, examples, and agent skills live in separate repositories under `initbots-org`.

## Technology Defaults

Treat these choices as established defaults and architectural constraints unless a task explicitly changes them:

- Python
- FastAPI
- Pydantic
- SQLAlchemy and Alembic
- SQLite by default for relational data
- Chroma file-based vector database by default
- OpenAI-compatible Chat Completions API
- Jinja, HTMX, and Tailwind for the optional admin UI

## Product Invariants

- Generated applications must stay small.
- `init_bot()` must remain the canonical public entry point.
- Runtime logic should remain framework-independent where practical.
- Prefer explicit interfaces over hardcoded integrations.
- Default to local files, SQLite, and Chroma.
- Relational and vector databases must be able to scale independently.
- The integrated admin must remain optional and disabled by default.
- The admin must always require authenticated access.
- The admin must not expose or edit Python source code.
- Every admin mutation must be permissioned and audited.
- Write operations must be permissioned, bounded, and audited.
- Agent behavior must use a bounded tool loop rather than unrestricted autonomy.
- Web search must be disabled by default.
- The OpenAI-compatible Chat Completions API must remain the primary client interface.

## Configuration Model

Configuration has two layers:

- Deployment-owned settings come from `initbots.toml` and the environment. They include database URLs, data paths, network binding, admin exposure, authentication bootstrap, and secret references.
- Admin-managed runtime settings are stored in the configured relational database. They override corresponding TOML defaults and apply to new requests without disrupting in-flight work.

TOML and database records may contain environment-variable or external secret-provider references, but must not persist raw credential values.

Connector and operation configuration should use Pydantic models. The admin derives generic forms and validation from their JSON Schema rather than requiring extension-specific frontend code.

## Generated Application Contract

The target generated application should remain small:

```text
my-bot/
  initbots.toml
  app/
    main.py
    connectors/
    operations/
    prompts/
      system.md
  knowledge/
  data/
    app.sqlite
    chroma/
```

`app/main.py` should usually contain only:

```python
from initbots import init_bot

app = init_bot()
```

`init_bot()` returns the configured FastAPI application directly.

## Runtime Flow

The target default chat flow is:

1. Receive `POST /v1/chat/completions`.
2. Authenticate the request if API authentication is enabled.
3. Resolve deployment and runtime configuration.
4. Load the conversation and memory if enabled.
5. Retrieve static context from Chroma.
6. Resolve dynamic context through connectors.
7. Build the available tools and operations.
8. Call the configured model provider.
9. Execute allowed tool calls within configured limits.
10. Store messages, memory, and audit logs.
11. Return an OpenAI-compatible response.

## Extension Model

InitBots has three canonical extension concepts:

- `Connector`: reads dynamic context from an external system, such as fetching order status, querying Confluence, or loading a user profile.
- `Operation`: performs a controlled write or action, such as opening a ticket, changing a username, or canceling an order.
- `Tool`: exposes a connector, operation, or built-in capability to the model, such as static context search, a calculator, the current time, or optional web search.

Operations must be permissioned, logged, and bounded.

## Agent Harness

InitBots should provide a small agent-like harness, not an uncontrolled autonomous agent. Its default behavior must include:

- a bounded tool loop
- configurable maximum tool rounds
- explicit permissions
- audit logging
- web search disabled by default
- optional confirmation requirements for write operations

## Admin Panel

The admin is bundled with the distribution but mounted only when enabled in configuration or with `initbots run --admin`. The `--admin` and `--no-admin` flags override configuration for the current process.

Admin authentication is independent of whether client API authentication is enabled. The initial superuser is created with `initbots admin create-user`; subsequent users receive explicit permissions for administrative capabilities.

The target admin should expose:

- model and provider settings
- API authentication and client credential management
- admin users and permissions
- static knowledge documents and chunks
- Chroma ingestion and reindexing
- connectors and their runtime configuration
- operations, permissions, and confirmation policies
- conversations, memory, and audit logs
- debugging information for context and tool usage

Every mutation must record the authenticated actor and a redacted before-and-after representation in the audit log.

## Target CLI

The planned initial commands are:

```bash
initbots init my-bot
initbots run [--admin | --no-admin]
initbots ingest ./knowledge
initbots migrate
initbots admin create-user
initbots add-connector
initbots add-operation
initbots infra init docker-compose
```

## Target Source Layout

```text
src/
  initbots/
    admin/
      static/
      templates/
    auth/
    cli/
    config/
    connectors/
    context/
    db/
    infra/
    llm/
    memory/
    operations/
    runtime/
    server/
    templates/
    tools/
tests/
```

## Working Guidance

- Use `InitBots` for the product in prose. Use lowercase `initbots` for packages, commands, configuration, and URLs.
- Use `must` requirements as product or safety constraints. Treat `should` statements as preferred design choices.
- Verify the current code and tests instead of assuming a target component has been implemented.
- If code or tests conflict with this file, do not silently choose one as authoritative. Identify whether the task changes current behavior or durable architectural intent.
- Update this file when a task intentionally changes a repository-wide architecture decision, invariant, or public contract. Keep routine implementation details in code and tests.
- No development, test, lint, or build workflow has been established yet. Inspect project configuration for available commands, and document commands here only after the corresponding tooling exists.
- Add nested `AGENTS.md` files only when a subtree develops genuinely different commands or conventions.
