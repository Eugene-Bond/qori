<h1 align="center">qori</h1>

<p align="center">
  <b>Observability and tracing for LLM-involved systems</b><br/>
  Detect where language models are at work, trace what they produce, and keep every step inspectable.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/status-experimental-orange" alt="Status">
  <img src="https://img.shields.io/badge/python-3.18%2B-3776ab" alt="Python">
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-Apache%202.0-blue" alt="License"></a>
</p>

---

**qori** is a lightweight framework for teams building with, or alongside, large language models. It provides a small, composable vocabulary for making LLM involvement explicit: which model touched which data, through which prompts and tools, and what came out the other end — without coupling a project to any single provider.

The project is intentionally early-stage. The goal is to explore what observability should look like when LLMs are treated as a core architectural primitive rather than a black box bolted on at the edge.

## Why qori?

AI is increasingly embedded in ordinary software and ordinary communication, but the LLM layer is usually hidden behind application-specific abstractions. qori aims to bring that layer into the open.

The framework is designed around a few principles:

- **Provenance first** — every model-generated artifact carries a record of how it was produced
- **Provider agnostic** — tracing and detection logic should not depend on one model vendor
- **Composable** — small primitives work independently and combine into larger workflows
- **Observable** — prompts, model calls, tool execution, and outputs are always inspectable
- **Replayable** — every run produces a trace that can be re-executed and asserted against in tests
- **Minimal surface area** — qori adds conventions without becoming the application itself

## Concept

qori models an LLM-involved system as a set of traced stages. Anything a model touches leaves a mark:

```text
input
  → context
  → model        ─┐
  → tool          ├─ trace: who, what, with which prompt, at what cost
  → model        ─┘
  → validation
  → output       (tagged with provenance)
```

The same trace format is used whether qori is orchestrating the pipeline itself or observing an existing one from the outside.

## Example API

> The API below illustrates the direction of the project and is not yet a stable public interface.

```python
from qori import Pipeline, Model, tool, trace

@tool
def search(query: str, limit: int = 5) -> list[str]:
    """Search application knowledge."""
    return index.search(query, k=limit)

pipeline = (
    Pipeline("researcher")
    .plan(Model("default"))
    .act(tools=[search], max_steps=6)
    .verify(criteria="answer cites at least two sources")
)

result = pipeline.run("Summarize the most important changes in the latest release.")

print(result.output)
print(result.trace.summary())
# 3 model calls · 2 tool calls · verify: passed · 2,314 tokens · 6.8s
```

Observing a system you don't control:

```python
from qori.detect import Provenance

report = Provenance.inspect(document)
print(report.llm_involvement)   # likely | unlikely | inconclusive
print(report.signals)           # structural and stylistic markers that informed the estimate
```

## Intended Modules

| Module | Purpose |
| --- | --- |
| `qori.models` | Unified model interfaces, retries, rate limits, cost tracking |
| `qori.prompts` | Reusable and versionable prompt definitions |
| `qori.tools` | Typed functions exposed to models, validated before execution |
| `qori.context` | Context construction and lifecycle management |
| `qori.pipeline` | Staged reasoning runtime (`plan` / `act` / `verify` / `reflect`) |
| `qori.trace` | Execution traces, provenance tags, replay, and diffing |
| `qori.detect` | Heuristics for estimating LLM involvement in untraced content |
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

1. Define the smallest useful set of tracing and provenance primitives
2. Establish provider-independent model and tool interfaces
3. Make replay and diffing of traces ergonomic
4. Explore reliable signals for LLM involvement in untraced content
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

- hide model behavior behind excessive abstraction
- prescribe a vector database, model provider, or deployment platform
- offer a definitive "AI or not" verdict — `qori.detect` produces estimates with stated signals, never certainties
- replace application-specific business logic
- promise deterministic behavior where the underlying model is probabilistic

## Roadmap

- [x] Design notes and module layout
- [ ] Core model abstraction
- [ ] Typed tool registration and execution
- [ ] Pipeline runtime with staged reasoning
- [ ] Trace serialisation, provenance tags, and replay
- [ ] Detection heuristics for untraced content
- [ ] Evaluation toolkit
- [ ] Reference applications
- [ ] Public package release

## Security

LLM applications introduce risks that a framework cannot eliminate automatically. Applications built with qori should treat model output as untrusted input, validate tool arguments, isolate privileged operations, and keep secrets out of model-visible context.

Detection results from `qori.detect` are probabilistic and should never be the sole basis for decisions about people.

## Contributing

qori is at an exploratory stage, so design discussion is currently more valuable than large implementation pull requests. Useful contributions include API design proposals, minimal reproducible workflow examples, provenance and tracing ideas, provider-abstraction edge cases, and documentation improvements.

Before proposing a large feature, open a discussion or issue describing the use case and the smallest abstraction that could support it.

## License

Apache 2.0 — see [LICENSE](LICENSE).

---

<p align="center"><sub><strong>qori</strong> — bringing LLM involvement into the open.</sub></p>
