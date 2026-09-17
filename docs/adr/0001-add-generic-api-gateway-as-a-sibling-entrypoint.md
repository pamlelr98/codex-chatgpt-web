# Add the Generic API Gateway as a sibling entrypoint

The project will add the Generic API Gateway as a separate local API entrypoint instead of turning
the existing Codex bridge into a generic server. This keeps the working Codex routing, Turn
Capability, broker, and MCP contracts stable while allowing non-Codex clients to reuse ChatGPT Web
execution through an OpenAI-compatible API. The new gateway will run on a separate local port and
will return client-supplied function calls to the client for execution rather than routing them into
the Codex tool runtime.
