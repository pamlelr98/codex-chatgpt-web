# Codex ChatGPT Web

This project lets a Codex task use ChatGPT Web models while preserving the task's Codex-facing conversation contract and, when enabled, its tool access.

## Language

**Native Model**:
A Codex model whose turn is fulfilled by the official Codex backend.
_Avoid_: Default model, normal model

**Routed Web Model**:
A model selected from Codex whose turn is fulfilled by ChatGPT Web while remaining part of the owning Codex task.
_Avoid_: Proxy model, fake Codex model

**Full Harness**:
The interaction mode where a Routed Web Model can use the tools made available by its owning Codex turn.
_Avoid_: Full mode, automation mode

**Browser-only**:
The interaction mode where a Routed Web Model can answer from the Codex task context but cannot use the owning Codex turn's local tools.
_Avoid_: Read-only mode

**Turn Capability**:
Opaque authority scoped to one owning Codex turn that permits the corresponding Web turn to invoke only tools exposed by that turn.
_Avoid_: API key, session token

**Connector**:
The ChatGPT-visible integration that gives a Web turn access to its Turn Capability.
_Avoid_: Plugin, backend

**Generic API Gateway**:
A planned project feature that lets non-Codex clients use ChatGPT Web through an OpenAI-compatible API without joining the Codex-specific turn and MCP contract.
_Avoid_: Codex bridge, custom agent

**External Client**:
An application or agent that calls the Generic API Gateway and owns execution of any tools it exposes to the model.
_Avoid_: Provider, tool host
