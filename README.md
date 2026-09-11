<h1 align="center">qori</h1>

<p align="center">
  <b>Query Orchestration & Reasoning Infrastructure for LLM applications</b><br/>
  Explicit primitives for AI-native software: typed tools, staged reasoning, replayable traces.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/status-experimental-orange" alt="Status">
  <img src="https://img.shields.io/badge/python-3.18%2B-3776ab" alt="Python">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-Apache%202.0-blue" alt="License"></a>
</p>

---

**qori** is a framework for teams building with large language models. It provides a small, composable vocabulary for describing AI capabilities across applications, agents, and pipelines — without coupling projects to a specific model provider.

The project is intentionally early-stage. The goal is to explore what an application framework should look like when LLMs are treated as a core architectural primitive rather than an integration bolted on at the edge.

## Why qori?

Most LLM frameworks either hide too much (opaque chains you can't debug) or expose too little (raw API calls you orchestrate yourself). qori sits in between: a small, explicit runtime where every step of a reasoning pipeline is a typed, inspectable, replayable unit.

The framework is designed around a few principles:

- **AI-native by default** — models, tools, context, retrieval, and evaluation are first-class concepts
- **Provider agnostic** — application logic should not depend on one model vendor
- **Composable** — small primitives work independently and combine into larger workflows
- **Observable** — prompts, model calls, tool execution, and outputs are always inspectable
- **Replayable** — every run produces a trace that can be re-executed and asserted against in tests
- **Minimal surface area** — qori adds conventions without becoming the application itself

## Concept

A qori application is a pipeline of explicit reasoning stages operating over a shared state:

```text
input
  → context
  → plan
  → act        (tool calls)
  → verify
  → output
  ↳ trace      (everything above, serialised)
```

Stages are plain Python. Deployment, infrastructure, and product architecture stay in the developer's hands.

## Example API

> The API below illustrates the direction of the project and is not yet a stable public interface.

```python
from qori import Pipeline, Model, tool
from pydantic import BaseModel

class SearchArgs(BaseModel):
    query: str
    limit: int = 5

@tool(schema=SearchArgs)
def search(args: SearchArgs) -> list[str]:
    """Search application knowledge."""
    return index.search(args.query, k=args.limit)

pipeline = (
    Pipeline("researcher")
    .plan(Model("default"))
    .act(tools=[search], max_steps=6)
    .verify(criteria="answer cites at least two sources")
)

result = pipeline.run("Summarize the most important changes in the latest release.")
print(result.output)
print(result.trace.summary())
```

The same primitives are meant to cover simple model calls, deterministic workflows, retrieval-augmented generation, and multi-step agents.

## Intended Modules

| Module | Purpose |
| --- | --- |
| `qori.models` | Unified model interfaces, retries, rate limits, cost tracking |
| `qori.prompts` | Reusable and versionable prompt definitions |
| `qori.tools` | Typed functions exposed to models, validated before execution |
| `qori.context` | Context construction and lifecycle management |
| `qori.retrieval` | Retrieval and grounding primitives |
| `qori.pipeline` | Staged reasoning runtime (`plan` / `act` / `verify` / `reflect`) |
| `qori.trace` | Execution traces, replay, and diffing |
| `qori.eval` | Evaluations, assertions, and regression testing |

## Testing pipelines

Traces are plain JSON. A recorded run can be replayed with mocked model outputs so pipeline logic is tested deterministically:

```python
from qori.trace import replay

def test_researcher_cites_sources():
    trace = replay("fixtures/researcher-0412.json", pipeline)
    assert trace.verify_passed
    assert len(trace.tool_calls("search")) >= 2
```

## Status

**Experimental / pre-release.** Interfaces, naming, and package structure may change substantially before a first stable release.

Current priorities:

1. Define the smallest useful set of AI-native primitives
2. Establish provider-independent model and tool interfaces
3. Make structured generation and validation ergonomic
4. Build tracing and replay into the core execution model
5. Keep the framework small enough to understand end-to-end

## Installation

A public package is not available yet.

```bash
# Placeholder for a future release
pip install qori
```

Until a release is published, do not depend on qori in production systems.

## Non-Goals

qori does not aim to:

- hide important model behavior behind excessive abstraction
- prescribe a vector database, model provider, or deployment platform
- turn every LLM call into an autonomous agent
- replace application-specific business logic
- promise deterministic behavior where the underlying model is probabilistic

## Roadmap

- [x] Design notes and module layout
- [ ] Core model abstraction
- [ ] Typed tool registration and execution
- [ ] Prompt and context primitives
- [ ] Pipeline runtime with staged reasoning
- [ ] Trace serialisation and replay
- [ ] Retrieval interfaces
- [ ] Evaluation toolkit
- [ ] Reference applications
- [ ] Public package release

## Security

LLM applications introduce risks that a framework cannot eliminate automatically. Applications built with qori should treat model output as untrusted input, validate tool arguments, isolate privileged operations, and keep secrets out of model-visible context.

Security-sensitive deployments should perform their own threat modeling and review.

## Contributing

qori is at an exploratory stage, so design discussion is currently more valuable than large implementation pull requests. Useful contributions include API design proposals, minimal reproducible workflow examples, evaluation and observability ideas, provider-abstraction edge cases, and documentation improvements.

Before proposing a large feature, open a discussion or issue describing the use case and the smallest abstraction that could support it.

## License

Apache 2.0 — see [LICENSE](LICENSE).

---

<p align="center"><sub><strong>qori</strong> — explicit primitives for AI-native software.</sub></p>
