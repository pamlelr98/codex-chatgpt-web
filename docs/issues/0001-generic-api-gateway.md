# Generic API Gateway for External Clients

## Problem Statement

The project currently lets Codex use ChatGPT Web through a local Responses bridge, but that path is tied to Codex-specific turn metadata, Turn Capabilities, broker state, and MCP execution. Users who want to connect another application or custom agent to the same ChatGPT Web execution path cannot do so through a stable OpenAI-compatible API without participating in the Codex Full Harness contract.

The existing Codex workflow is already in active use and must remain stable. The new capability therefore needs to be additive: External Clients should gain a dedicated API entrypoint without turning the current Codex bridge into a generic server or changing its routing behavior.

## Solution

Add a Generic API Gateway as a sibling local service for External Clients. The gateway exposes an OpenAI Responses-compatible API and reuses the existing ChatGPT Web browser execution components where practical, while keeping Codex-specific request parsing, turn authority, broker behavior, and MCP execution on the existing Codex path.

The first milestone supports `POST /v1/responses`, streamed and non-streamed responses, ChatGPT Web model selection, and client-owned function calling. When an External Client supplies tools, the model may request a function call; the gateway returns that function call to the External Client, which executes the tool and submits the tool result back through the Responses API. The gateway does not redirect External Client tools into the Codex MCP runtime.

The Generic API Gateway runs on a separate local port and has its own request lifecycle so failures or compatibility changes in this new feature do not alter the currently working Codex integration.

## User Stories

1. As an External Client user, I want to call ChatGPT Web through a local OpenAI-compatible API, so that I can use my ChatGPT Web account from applications other than Codex.
2. As a translation application user, I want to send translation prompts through the Generic API Gateway, so that the application can use ChatGPT Web as its model backend.
3. As a custom-agent developer, I want to point my agent at a local `/v1/responses` endpoint, so that I can reuse an existing Responses-compatible client implementation.
4. As a Codex user, I want the current Codex bridge to remain unchanged, so that adding the Generic API Gateway does not break my existing workflow.
5. As a Codex user, I want Codex to keep using its current local route, turn metadata, Turn Capability, broker, tunnel, and MCP behavior, so that the new feature does not weaken or alter the Full Harness contract.
6. As an External Client user, I want the Generic API Gateway to run on a separate local port, so that it can coexist with the Codex bridge without route collisions.
7. As an External Client user, I want to select an available ChatGPT Web model through the API, so that the client can choose the intended execution mode.
8. As an External Client user, I want non-streamed Responses requests to work, so that simple applications can receive a complete response in one result.
9. As an External Client user, I want streamed Responses requests to work, so that interactive applications can display model output incrementally.
10. As an External Client user, I want the gateway to accept normal Responses input messages and instructions, so that I do not need to construct Codex-specific task envelopes.
11. As an External Client user, I want requests to work without Codex `thread_id` or `turn_id` metadata, so that my application does not need to emulate Codex internals.
12. As an External Client user, I want requests to work without a Turn Capability, so that my application does not need a Codex-owned authorization token for model execution.
13. As an External Client user, I want requests to work without the `Codex Native2` connector, so that model inference is independent of the Codex MCP connector.
14. As an External Client developer, I want to send function/tool definitions using the Responses contract, so that the model can participate in my application's agent workflow.
15. As an External Client developer, I want model-requested function calls returned as Responses function-call items, so that my application can execute them using its own tool runtime.
16. As an External Client developer, I want to submit function-call outputs back to the gateway, so that the ChatGPT Web turn can continue after my application executes a tool.
17. As an External Client developer, I want tool execution to remain owned by my application, so that the gateway does not gain arbitrary access to application-specific tools.
18. As a Codex user, I want External Client tools kept out of the Codex MCP runtime, so that Generic API requests cannot accidentally invoke Codex tools.
19. As an External Client user, I want ChatGPT Web browser login state to be reused where safe, so that I do not need a second independent ChatGPT login flow solely for the gateway.
20. As a project maintainer, I want browser-level execution behavior shared below the protocol boundary, so that the Codex bridge and Generic API Gateway do not duplicate fragile ChatGPT Web automation logic.
21. As a project maintainer, I want Codex-specific parsing and authority to remain isolated on the Codex path, so that generic-client compatibility does not spread Codex assumptions into the new API surface.
22. As a project maintainer, I want malformed External Client requests to fail at the Generic API boundary, so that invalid input does not open or corrupt unrelated browser turns.
23. As an External Client user, I want unsupported models to fail explicitly, so that the gateway never silently routes my request to a different model.
24. As an External Client user, I want browser or model failures represented through the API contract, so that my application can distinguish request failures from successful responses.
25. As an External Client user, I want cancellation or client disconnects to terminate only the request I own, so that unrelated Codex and External Client turns remain unaffected.
26. As a project maintainer, I want the Generic API Gateway to have its own request lifecycle and activity accounting, so that service management can distinguish it from Codex-owned turns.
27. As a project maintainer, I want the gateway bound to loopback by default, so that the new API is not unintentionally exposed to the network.
28. As an External Client developer, I want the public contract to depend on the supported Responses subset rather than a named client application, so that translation tools, custom agents, and other compatible consumers can use the same gateway.
29. As a translation application developer, I want simple translation requests to work without function calling, so that ordinary inference remains a first-class use case.
30. As an agent application developer, I want multi-round function calling to work, so that the model can request a tool, receive its result, and continue producing a final answer.
31. As a project maintainer, I want compatibility work for future endpoints such as Chat Completions to be addable later, so that the first milestone stays focused on the Responses API.
32. As a project maintainer, I want regression coverage proving the existing Codex bridge still behaves as before, so that shared browser changes do not accidentally alter the production Codex path.

## Implementation Decisions

- The Generic API Gateway is an additive sibling entrypoint, not a replacement or generalization of the existing Codex bridge.
- The existing Codex bridge remains the owner of Codex routing, native passthrough, Codex turn identity, Turn Capabilities, broker behavior, tunnel integration, and Codex MCP execution.
- The Generic API Gateway runs on a separate loopback port with an independent request lifecycle.
- The first supported public endpoint is `POST /v1/responses`.
- The first milestone supports both streamed and non-streamed Responses behavior.
- The first milestone supports the input and output shapes required for ordinary text turns and function calling.
- The gateway accepts External Client requests without native Codex thread or turn metadata.
- The gateway does not require a Turn Capability or the `Codex Native2` connector.
- The gateway exposes ChatGPT Web model routes appropriate to the Generic API surface and rejects unsupported model names explicitly.
- Shared implementation should remain below the protocol boundary. Browser login/session ownership, model selection, prompt submission, response observation, and other ChatGPT Web provider-level behavior may be reused by both entrypoints.
- Codex-specific request parsing, context compilation, continuation rules, compaction authority, tool broker state, and MCP execution remain isolated to the Codex path.
- External Client tool definitions are presented to the ChatGPT Web turn as model-callable tools.
- A model-requested function call is returned to the External Client using the Responses function-call contract.
- The External Client owns execution of its tools and returns function-call output through a later Responses request or continuation.
- The Generic API Gateway does not execute arbitrary External Client tools and does not translate those calls into Codex MCP calls.
- The public compatibility boundary is the supported Responses contract rather than compatibility logic for one named application.
- The gateway should preserve explicit failure behavior rather than silently falling back to another provider, model, interaction mode, or the Codex tool path.
- Loopback-only binding remains the default network boundary for the new service.
- The design follows ADR 0001: the Generic API Gateway stays a sibling entrypoint so the current Codex integration can evolve independently from generic-client compatibility.

## Testing Decisions

- The primary test seam is the Generic API Gateway's HTTP Responses boundary. Tests should send requests as a real External Client would and assert externally observable HTTP, JSON, SSE, function-call, continuation, cancellation, and error behavior.
- Prefer one high-level seam over tests that directly target internal protocol translation helpers. Internal tests are justified only for behavior that cannot be observed reliably through the HTTP contract.
- The gateway tests should substitute or control ChatGPT Web execution below the public API boundary where deterministic browser execution is required. Tests should not depend on implementation details such as private helper calls or internal object layout.
- Non-streamed tests should verify complete Responses output for a normal text request.
- Streaming tests should verify the externally visible SSE event sequence and terminal response semantics.
- Function-calling tests should verify that client-supplied tools can produce a function-call item without invoking the Codex tool runtime.
- Continuation tests should verify that a submitted function-call output can resume the model workflow and produce a later answer.
- Isolation tests should verify that External Client requests do not require Codex turn metadata, Turn Capabilities, or the `Codex Native2` connector.
- Negative tests should verify explicit rejection of unsupported models, malformed tool definitions, invalid continuation data, and other invalid Responses inputs.
- Cancellation tests should verify that disconnecting or cancelling an External Client request does not cancel unrelated gateway or Codex turns.
- Regression tests should continue exercising the existing Codex `/v1/responses` bridge so shared lower-level browser changes cannot silently change Codex behavior.
- Prior art exists in the project's server-level Responses tests, Responses bridge streaming/function-call tests, ChatGPT Web harness tests, and browser worker contract tests. New tests should reuse those styles at the highest applicable seam.
- Good tests assert public behavior and security boundaries rather than private implementation structure. A refactor that preserves the HTTP contract should not require broad test rewrites.

## Out of Scope

- Replacing or refactoring the existing Codex bridge into a generic gateway.
- Changing Codex's `openai_base_url` integration behavior for this feature.
- Changing Full Harness Turn Capability semantics, Codex broker semantics, or the `Codex Native2` MCP contract.
- Executing arbitrary External Client tools inside the Generic API Gateway.
- Routing External Client function calls into Codex MCP tools.
- Requiring a specific External Client such as a particular translation application or agent framework.
- Chat Completions compatibility in the first milestone.
- Generic multi-provider routing comparable to a full provider aggregation service.
- Adding Claude, Gemini, or other non-ChatGPT-Web providers as part of this feature.
- Exposing the Generic API Gateway publicly on the network by default.
- Reworking ChatGPT account authentication or bypassing product authentication, access controls, or usage limits.

## Further Notes

The motivating use case is an External Client such as a translation application or custom agent that already understands an OpenAI-compatible API. Simple translation should require only the gateway base URL and model name. Agent-style clients should additionally be able to provide tools, receive function calls, execute those tools locally, and return their outputs.

This spec intentionally preserves the current Codex integration as a separate production path because it is already in active use. Sharing is expected at the ChatGPT Web execution layer, not at the Codex protocol or authority layer.

The implementation should treat the existing Generic API Gateway design document and ADR 0001 as architectural constraints for this feature.
