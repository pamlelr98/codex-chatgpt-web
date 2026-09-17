# Generic API Gateway

The Generic API Gateway is a planned additive feature that exposes ChatGPT Web execution to
non-Codex clients through an OpenAI-compatible local API. It is designed to support applications
such as translation tools and custom agents while leaving the existing Codex integration intact.

## Goals

- Keep the current Codex bridge and Full Harness behavior unchanged.
- Add a separate local API entrypoint for External Clients.
- Reuse the existing ChatGPT Web browser execution where practical.
- Support OpenAI Responses requests without requiring Codex-specific turn metadata.
- Preserve streaming response behavior.
- Preserve client-owned function calling: the client executes its own tools and returns tool output.

## Initial protocol surface

The first milestone supports:

- `POST /v1/responses`
- streamed and non-streamed responses
- input messages and instructions needed by ordinary Responses clients
- model selection for the ChatGPT Web routes exposed by the gateway
- function/tool definitions supplied by the External Client
- function calls returned to the External Client
- continuation after the External Client submits tool output

Additional compatibility endpoints, including Chat Completions, are outside the first milestone and
can be added later without changing the Codex bridge.

## Runtime boundary

```text
External Client
      │
      │ OpenAI Responses request
      ▼
Generic API Gateway
      │
      │ normalized Web turn
      ▼
ChatGPT Web browser execution
      │
      │ text / function call
      ▼
Generic API Gateway
      │
      ▼
External Client
      │
      ├─ renders text, or
      └─ executes its own tool and submits the result
```

The gateway owns protocol translation and ChatGPT Web execution. The External Client owns its tool
runtime. This differs from Full Harness, where a ChatGPT Web turn receives a scoped Turn Capability
and invokes tools belonging to the active Codex turn through MCP.

The new path therefore does not require:

- native Codex `thread_id` or `turn_id` metadata
- Codex continuation state
- a Turn Capability
- the `Codex Native2` connector
- the Codex turn broker or Codex MCP tool runtime

## Isolation from the Codex bridge

The Generic API Gateway runs on a separate local port and has its own request lifecycle. The
existing Codex-facing daemon continues to own Codex routing exactly as it does today.

Shared code should stay below the protocol boundary. Browser login state, model selection,
submission, response observation, and other provider-level behavior may be reused. Codex-specific
request parsing, turn identity, broker authority, and MCP execution remain on the Codex path.

This separation lets the new API evolve without making the existing Codex workflow depend on
generic-client compatibility behavior.

## Function-calling contract

For an External Client request containing tool definitions:

1. The gateway presents the tool definitions to the ChatGPT Web turn.
2. If the model requests a tool, the gateway emits the corresponding Responses function-call item.
3. The External Client executes the tool in its own runtime.
4. The External Client submits the tool result in the next Responses request or continuation.
5. The gateway resumes ChatGPT Web execution with that result.

The gateway must not execute arbitrary External Client tools itself and must not translate them into
Codex MCP calls.

## Example: translation application

```text
Translation app
      │ prompt + optional agent tools
      ▼
Generic API Gateway
      ▼
ChatGPT Web
      │
      ├─ translated text ───────────────► Translation app
      │
      └─ function call ─► Translation app tool runtime
                              │
                              └─ tool result ─► Generic API Gateway ─► ChatGPT Web
```

This supports both simple translation requests and agent-style workflows without making the
translation application participate in the Codex Full Harness contract.

## Compatibility principle

The public boundary is the API contract, not a specific External Client. A client that can speak the
supported Responses subset should not need project-specific knowledge beyond the gateway base URL
and available model names.
