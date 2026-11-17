# Azure AI Agent Backend — architecture case study

A written walkthrough of a FastAPI control plane for Azure AI Foundry agents.
The implementation is kept private; this repository documents the architecture,
the decisions behind it, and the operational lessons.

## The problem

Azure AI Foundry gives you agents, threads, vector stores and tools, but a product
needs a single service that creates them, wires them together, and streams the
result to a front end. Doing that ad hoc in every client couples UI code to SDK
details and makes memory, files and tool configuration impossible to reuse.

This service sits between clients and the Foundry SDK and exposes one consistent
HTTP surface.

## Shape of the system

```
Client (web / other services)
        |  HTTP + SSE
        v
FastAPI control plane
  |-- agent registry        create / list / configure agents
  |-- thread manager        conversation state
  |-- tool manager          file search, code interpreter, Azure AI Search,
  |                         custom function calling
  |-- knowledge manager     vector stores, file upload, agent <-> KB links
  '-- memory store          long-term memory across conversations
        |  Azure AI Foundry SDK
        v
Azure AI project (models, indexes, files, memory preview)
```

## Decisions worth calling out

- **One control plane, many agents.** Agents are created and configured through
  API calls instead of being hard-coded, so the same service supports several
  use cases without redeployment.
- **Streaming first.** Chat returns SSE so UIs can render tokens as they arrive;
  the service owns the SDK streaming loop, not the client.
- **Knowledge is a first-class resource.** Vector stores and files are managed
  separately from agents, then linked. That keeps one knowledge base reusable
  across multiple agents.
- **Tools are configurable, not compiled in.** File search, code interpreter,
  Azure AI Search and function calling are toggled through the tool manager.
- **Memory is opt-in.** Long-term memory uses the Foundry memory preview behind
  a flag, so a deployment can stay stateless until it is wanted.
- **Modern Python packaging.** `uv` for dependency management, a single
  container image, and a health endpoint for orchestration.

## API surface (summary)

| Area | Operations |
|---|---|
| Agents | create, list, configure instructions/models |
| Threads | create, list, continue conversations |
| Chat | send message, stream response (SSE) |
| Tools | attach/detach file search, code interpreter, AI Search, functions |
| Knowledge | create vector store, upload file, link to agent |
| Health | liveness/readiness |

## Stack

Python 3.11+ · FastAPI · Azure AI Foundry SDK · Azure AI Search ·
`uv` · Docker · CORS enabled for browser clients.

## Deployment notes

Runs as a container behind a reverse proxy. Configuration is entirely
environment-driven: the Foundry project endpoint plus optional Azure AD
credentials. No secrets are stored in the repository.

## Related work

- [AutFin](https://github.com/GreatHavoc/AutFin) — autonomous multi-agent
  financial research system.
- [Optional Medium](https://github.com/GreatHavoc/optional-medium) — browser
  extension that opens Medium links on a mirror.
