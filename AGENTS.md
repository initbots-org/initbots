# InitBots Agent Context

## Purpose And Status

InitBots is an opinionated Python framework for creating and operating chatbot backends.

The product goal is to generate a small chatbot application with an OpenAI-compatible API, configurable model providers, static knowledge retrieval, optional memory, dynamic context connectors, controlled operations, and an integrated admin panel.

This repository is currently greenfield. The architecture, interfaces, commands, and layouts below describe the target state; they do not imply that those components already exist. Inspect the repository and its tests before making changes.

This file is internal guidance for coding agents. Keep user-facing installation, tutorials, and API documentation in the official website and documentation repository. Keep `README.md` in this repository concise.

## Repository Architecture

This repository targets one Python distribution and one public package, `initbots`. It includes the runtime, CLI, FastAPI server, configuration, model providers, context retrieval, extension interfaces, hooks, tools, memory, authentication, databases, OpenAI-compatible API, and optional admin panel.

The package is organised around a seam rather than around technologies: a framework-independent core of ports and the chat engine, adapters implementing those ports, and delivery shells over both. See `Target Source Layout`.

The admin is an integrated framework component, similar in placement to Django's admin. It uses server-rendered Jinja templates, HTMX, and Tailwind, and is shipped as part of the Python distribution. It must remain optional at runtime and must not expose or edit Python source code.

The official website and documentation share the `website` repository. Examples and agent skills live in their own repositories under `initbots-org`.

## Technology Defaults

Treat these choices as established defaults and architectural constraints unless a task explicitly changes them:

- Python
- FastAPI
- Pydantic
- SQLAlchemy and Alembic
- SQLite by default for relational data
- Chroma file-based vector database by default
- OpenAI-compatible Chat Completions API
- The `openai` client library, as the reference implementation of that format
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
- Streaming must be supported by the model provider port from its first implementation.
- The engine's internal message representation must not be the OpenAI wire format.
- `initbots.core` must not import web, database, vector store, or model provider libraries.
- A base install without optional extras must be able to run a bot.
- Connector failures must degrade. Operation failures must not.
- Confirmation must never block a request while waiting for a human.

## Configuration Model

Configuration has two layers:

- Deployment-owned settings come from `initbots.toml` and the environment. They include database URLs, data paths, network binding, admin exposure, authentication bootstrap, and secret references.
- Admin-managed runtime settings are stored in the configured relational database. They override corresponding TOML defaults and apply to new requests without disrupting in-flight work.

Both layers are resolved into an immutable settings snapshot at the start of each request. Runtime code must read that snapshot rather than global state. The snapshot is the mechanism behind the in-flight guarantee above.

TOML and database records may contain environment-variable or external secret-provider references, but must not persist raw credential values.

The `[model]` block is required. There is no implicit provider default, because a deployment must state which model it is spending money on. Missing or unresolvable provider credentials must fail at startup rather than on the first chat request.

The TOML key names are defined here rather than on the documentation site:

```toml
[app]
name = "my-bot"

[server]
host = "127.0.0.1"
port = 8000

[database]
url = "sqlite:///data/app.sqlite"

[vector]
path = "data/chroma"

[model]
provider = "openai"
name = "..."
api_key = "${OPENAI_API_KEY}"

[admin]
enabled = false
```

`[model]` accepts an optional `base_url`, which is how a single OpenAI-compatible adapter serves providers other than OpenAI.

Connector and operation configuration should use Pydantic models. The admin derives generic forms and validation from their JSON Schema rather than requiring extension-specific frontend code.

## Generated Application Contract

The target generated application should remain small:

```text
my-bot/
  initbots.toml
  .env.example
  app/
    main.py
    hooks.py
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

`build_bot()` returns the same configured bot without the HTTP layer, for callers that are not serving requests over HTTP, such as workers or chat platform adapters. `init_bot()` is `build_bot()` plus the server. The bot is also reachable as `app.state.bot`.

The two-line `main.py` means the application contains no wiring code. It does not mean there is no configuration.

## Runtime Flow

The target default chat flow is:

1. Receive and validate `POST /v1/chat/completions`.
2. Resolve the settings snapshot for this request.
3. Authenticate the request and resolve the calling principal if API authentication is enabled.
4. Load the conversation and memory if enabled.
5. Build the prompt from `prompts/system.md`, memory, retrieved static context, and any always-injected connector context.
6. Assemble the toolset for this request, filtered by the principal's permissions.
7. Run the bounded tool loop, up to the configured maximum number of rounds:
   1. Call the configured model provider.
   2. Leave the loop if the model requested no tool calls.
   3. Authorize each requested call against policy.
   4. Leave the loop with a confirmation request if a call requires confirmation it has not received.
   5. Execute the allowed calls and audit each one.
   6. Append the results and continue.
8. Finalize the response with an explicit stop reason: `done`, `round_limit`, `needs_confirmation`, or `blocked`.
9. Store messages, audit records, and usage.
10. Update memory.
11. Return the response, or close the stream.

Configuration is resolved before authentication because authentication is itself configured.

Streaming is not a variant of this flow. Steps 7 through 9 must be able to emit incremental events, and step 9 must persist after the stream completes rather than before the response begins.

## Context Resolution

Static knowledge and connectors are both retrievable context, but they default differently:

- Static knowledge retrieval is eager. The retriever runs during step 5 and its results are injected into the prompt.
- Connectors are model-invoked. They are exposed as tools and run inside the loop in step 7, only when the model asks for them.

The defaults follow cost and consequence. Knowledge retrieval is local and almost always relevant. A connector call reaches an external system, often carries per-user data, and is frequently irrelevant to a given turn.

Both defaults must be overridable per source. A knowledge collection may be exposed as a search tool instead of being injected, and a connector may declare that it is always injected.

Connector failures degrade: a failing or timed-out connector is omitted from context and recorded, and the turn continues. Operation failures do not degrade. They surface to the model and to the audit log.

## Extension Model

InitBots has three canonical extension concepts:

- `Connector`: reads dynamic context from an external system, such as fetching order status, querying Confluence, or loading a user profile.
- `Operation`: performs a controlled write or action, such as opening a ticket, changing a username, or canceling an order.
- `Tool`: exposes a connector, operation, or built-in capability to the model, such as static context search, a calculator, the current time, or optional web search.

Operations must be permissioned, logged, and bounded.

Hooks are an escape hatch rather than a fourth extension concept, and must not be presented as one in user-facing material.

## Agent Harness

InitBots should provide a small agent-like harness, not an uncontrolled autonomous agent. Its default behavior must include:

- a bounded tool loop
- configurable maximum tool rounds
- explicit permissions
- audit logging
- web search disabled by default
- optional confirmation requirements for write operations

Confirmation works across turns rather than by blocking. When an operation requires confirmation it has not received, the loop stops, the pending operation is persisted, and the response carries the `needs_confirmation` stop reason describing the proposed call. A later request either approves or discards that pending operation. The framework must never wait in-request for a human.

## Hooks

Hooks let an application adjust the pipeline without replacing a component. They are discovered from `app/hooks.py`, by the same mechanism that discovers connectors and operations.

The target hook points are exactly these five:

- `on_request`
- `before_model`
- `after_model`
- `before_tool`
- `after_tool`

The set is deliberately small, and new hook points should be resisted. A need that no hook serves usually indicates a missing port rather than a missing hook.

Hooks must not be able to bypass permissions, confirmation policy, or audit logging.

## Model Providers

`ModelProvider` is a port with `complete` and `stream` methods and a capabilities descriptor. Streaming belongs in the port from its first implementation; it cannot be added later without reworking the tool loop.

The OpenAI-compatible Chat Completions format is a wire format rather than a vendor. One adapter with a configurable `base_url` is expected to serve OpenAI, self-hosted OpenAI-compatible servers, and the hosted providers that expose the same format. Providers with native formats get their own adapters.

The engine's internal message representation must not be the OpenAI wire format. Requests are translated from the OpenAI-compatible API into internal types at the HTTP edge, and from internal types into the provider's format inside the adapter. Two translations are the cost of keeping the client wire format from becoming the framework's domain model.

Adapters are resolved through a registry of module paths and imported lazily. Importing `initbots` must not import any provider SDK. A missing optional dependency must fail with the install command that fixes it.

Custom providers are supported without registration through an import path in configuration, such as `mycorp.llm:GatewayProvider`.

The capabilities descriptor exists so the engine can degrade honestly instead of branching on provider names. A configuration that requires tools from a provider that does not support them must fail at startup, not per request.

The default model identifier must live in exactly one constant. Model identifiers are deprecated on schedules this project does not control.

## Packaging

The distribution ships one package. Optional provider dependencies are installable extras, such as `initbots[anthropic]`.

Extras change installed dependencies, never installed modules. Every adapter module ships in every install; only its third-party imports are conditional.

Two properties must be enforced by tests rather than by convention:

- `initbots.core` must not import FastAPI, SQLAlchemy, Chroma, or any provider SDK.
- A base install with no extras must import, boot, and serve a chat completion against a fake provider.

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
    core/            ports, engine, domain types, policy
    adapters/
      llm/
      vectorstore/
      db/
      memory/
    admin/
      static/
      templates/
    auth/
    cli/
    config/
    connectors/
    context/
    infra/
    operations/
    server/
    templates/
    tools/
tests/
```

`core/` holds the ports and the chat engine and imports no web, database, vector store, or provider library. `adapters/` holds one implementation per port. `server/`, `admin/`, and `cli/` are delivery shells over the same engine.

Nothing belongs in this package that is not a port, the engine, an adapter for a port, or one of those three shells.

## Working Guidance

- Use `InitBots` for the product in prose. Use lowercase `initbots` for packages, commands, configuration, and URLs.
- Use `must` requirements as product or safety constraints. Treat `should` statements as preferred design choices.
- Verify the current code and tests instead of assuming a target component has been implemented.
- If code or tests conflict with this file, do not silently choose one as authoritative. Identify whether the task changes current behavior or durable architectural intent.
- Update this file when a task intentionally changes a repository-wide architecture decision, invariant, or public contract. Keep routine implementation details in code and tests.
- No development, test, lint, or build workflow has been established yet. Inspect project configuration for available commands, and document commands here only after the corresponding tooling exists.
- Add nested `AGENTS.md` files only when a subtree develops genuinely different commands or conventions.
