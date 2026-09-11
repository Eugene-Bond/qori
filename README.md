# qori

> **Query Orchestration & Reasoning Infrastructure**

**qori** is an experimental framework for building explicit, observable
LLM workflows.

The project explores a simple idea: reasoning pipelines should be made
from small, inspectable primitives rather than hidden inside
increasingly complicated prompts or opaque agent runtimes.

qori is being designed around **pipelines**, **typed tools**,
**reasoning stages**, **execution traces**, and **deterministic
replay**.

> **Project status:** early design / experimental. The APIs shown below
> describe the intended direction of qori and should not yet be
> considered stable or production-ready.

------------------------------------------------------------------------

## Why qori?

LLM applications tend to start simple:

``` python
response = model.generate(prompt)
```

Then they accumulate retrieval, tools, retries, validation, planning,
state, observability, and evaluation.

Eventually, the interesting part of the application is no longer the
model call. It is the orchestration around it.

qori aims to make that orchestration explicit:

``` text
input
  │
  ▼
┌──────┐
│ plan │
└───┬──┘
    ▼
┌─────┐
│ act │──────► tools
└──┬──┘
   ▼
┌────────┐
│ verify │
└───┬────┘
    ▼
  output
    │
    ▼
  Trace
```

Every execution should be inspectable. Every tool boundary should be
typed. Every pipeline should be testable without requiring live model
calls.

## Design principles

-   **Provider agnostic** --- application architecture should not depend
    on a particular model vendor.
-   **Typed tool contracts** --- model-generated arguments should be
    validated before application code receives them.
-   **Observable by default** --- model calls, tools, state transitions,
    timings, and token usage belong in the execution trace.
-   **Composable reasoning** --- `plan`, `act`, `verify`, and `reflect`
    are explicit stages rather than conventions buried in prompts.
-   **Replayable execution** --- recorded model outputs should make
    pipeline logic reproducible during development and testing.
-   **Plain Python** --- pipelines should remain understandable without
    learning a large DSL.

------------------------------------------------------------------------

## Proposed API

The following API represents the current design direction.

``` python
from qori import Pipeline, Model, tool

pipeline = (
    Pipeline("research-agent")
    .plan(Model("default"))
    .act(tools=[search], max_steps=6)
    .verify(
        Model("default"),
        criteria="answer must be grounded in retrieved sources",
    )
)

result = pipeline.run(
    "Summarize the most important changes in the latest release."
)

print(result.answer)
print(result.trace.summary())
```

A pipeline is an ordered set of reasoning stages operating over shared
state.

The runtime records the execution as a `Trace`, allowing the same run to
be inspected, evaluated, compared, or eventually replayed.

------------------------------------------------------------------------

## Core concepts

  -----------------------------------------------------------------------
  Concept                             Purpose
  ----------------------------------- -----------------------------------
  `Pipeline`                          Composes model, reasoning, tool,
                                      and validation stages into an
                                      executable workflow.

  `Model`                             Provider-independent interface for
                                      model execution and configuration.

  `tool`                              Converts an application function
                                      into a schema-validated model tool.

  `State`                             Explicit data passed between
                                      pipeline stages.

  `Trace`                             Structured record of model calls,
                                      tool calls, state transitions,
                                      timings, and outputs.

  `Replay`                            Re-executes pipeline logic against
                                      recorded model outputs for
                                      deterministic testing.
  -----------------------------------------------------------------------

These concepts intentionally describe orchestration rather than a
particular model provider.

------------------------------------------------------------------------

## Reasoning stages

qori experiments with reasoning operations as first-class pipeline
primitives.

### `plan`

Produces or updates an execution plan.

``` python
pipeline.plan(model)
```

### `act`

Allows a model to execute registered tools against the current state.

``` python
pipeline.act(
    tools=[search, fetch],
    max_steps=6,
)
```

### `verify`

Evaluates the current result against explicit criteria.

``` python
pipeline.verify(
    model,
    criteria="claims must be supported by retrieved context",
)
```

### `reflect`

Optionally feeds verification results back into the pipeline before
producing the final output.

``` python
pipeline.reflect(model, max_attempts=2)
```

The goal is not to prescribe one reasoning strategy. These stages
provide a vocabulary for constructing and inspecting different
strategies.

------------------------------------------------------------------------

## Typed tools

Tools represent the boundary between probabilistic model output and
deterministic application code.

The intended interface uses ordinary Python types or schema models:

``` python
from pydantic import BaseModel
from qori import tool

class SearchArgs(BaseModel):
    query: str
    limit: int = 5

@tool(schema=SearchArgs)
def search(args: SearchArgs) -> list[str]:
    return index.search(
        args.query,
        limit=args.limit,
    )
```

Arguments should be validated **before** the underlying function
executes.

Tool calls should also become part of the execution trace automatically.

------------------------------------------------------------------------

## Trace

A `Trace` is intended to be the canonical representation of a qori
execution.

``` text
Trace
├── input
├── stages
│   ├── plan
│   │   └── model_call
│   ├── act
│   │   ├── model_call
│   │   └── tool_call
│   └── verify
│       └── model_call
├── output
├── usage
└── timing
```

Rather than treating observability as an external add-on, qori aims to
make execution metadata part of the runtime itself.

This should make it possible to answer questions such as:

``` python
trace.model_calls()
trace.tool_calls("search")
trace.stage("verify")
trace.usage
trace.duration
```

without reconstructing what happened from application logs.

------------------------------------------------------------------------

## Replay

LLM output is probabilistic. Pipeline logic does not have to be.

qori's proposed `Replay` abstraction separates the two.

A recorded trace can supply previously observed model responses while
the orchestration code executes normally:

``` python
from qori.testing import replay

trace = replay(
    "fixtures/research-run.json",
    pipeline,
)

assert trace.verify_passed
assert len(trace.tool_calls("search")) >= 2
```

This makes it possible to test orchestration, tool handling, validation,
and state transitions without repeatedly invoking a live model.

The longer-term goal is to support trace comparison as well:

``` text
baseline trace
      │
      ▼
    Replay
      │
      ├── state diff
      ├── tool-call diff
      ├── output diff
      └── evaluation
```

------------------------------------------------------------------------

## Architecture

qori is currently being designed around a small set of modules:

``` text
qori
├── models
├── pipeline
├── stages
├── tools
├── context
├── trace
├── replay
├── eval
└── observe
```

  Module            Responsibility
  ----------------- ---------------------------------------------------
  `qori.models`     Model interfaces and provider adapters
  `qori.pipeline`   Pipeline composition and execution
  `qori.stages`     `plan`, `act`, `verify`, and `reflect` primitives
  `qori.tools`      Typed tool definitions and execution
  `qori.context`    Context and state construction
  `qori.trace`      Execution traces and serialization
  `qori.replay`     Deterministic trace replay
  `qori.eval`       Assertions, evaluations, and regression testing
  `qori.observe`    Runtime instrumentation and exporters

------------------------------------------------------------------------

## Installation

qori is currently experimental and has not reached a stable public
release.

The intended installation interface is:

``` bash
pip install qori
```

Until an official package is published, examples in this README should
be treated as design documentation rather than a stable API.

------------------------------------------------------------------------

## Non-goals

qori does **not** aim to:

-   hide model behavior behind excessive abstraction;
-   turn every model call into an autonomous agent;
-   prescribe a particular model provider;
-   prescribe a vector database or retrieval architecture;
-   replace application-specific business logic;
-   pretend probabilistic systems are deterministic.

The framework should expose important decisions rather than conceal
them.

------------------------------------------------------------------------

## Roadmap

### Runtime

-   [ ] `Pipeline` execution model
-   [ ] `State` lifecycle
-   [ ] reasoning stages
-   [ ] structured model outputs
-   [ ] typed tool execution

### Observability

-   [ ] `Trace` representation
-   [ ] trace serialization
-   [ ] token and timing metadata
-   [ ] execution visualization
-   [ ] OpenTelemetry integration

### Testing

-   [ ] deterministic `Replay`
-   [ ] trace assertions
-   [ ] trace diffing
-   [ ] evaluation primitives
-   [ ] regression fixtures

### Ecosystem

-   [ ] provider adapters
-   [ ] reference pipelines
-   [ ] documentation
-   [ ] public package release

------------------------------------------------------------------------

## Security

Model output should always be treated as untrusted input.

Applications using qori should validate tool arguments, isolate
privileged operations, restrict model-accessible capabilities, avoid
exposing secrets through context, and perform threat modeling
appropriate to their deployment.

A framework can provide safer primitives, but it cannot make arbitrary
agent execution safe automatically.

------------------------------------------------------------------------

## Contributing

qori is currently in the design phase.

At this stage, useful contributions include:

-   API design proposals;
-   orchestration patterns;
-   trace and replay semantics;
-   provider-abstraction edge cases;
-   evaluation strategies;
-   minimal reproducible LLM workflows.

For significant changes, start with an issue or design discussion
describing the use case and the smallest abstraction needed to support
it.

------------------------------------------------------------------------

## License

A license will be selected before the first public release.

------------------------------------------------------------------------

```{=html}
<p align="center">
```
`<strong>`{=html}qori`</strong>`{=html}`<br/>`{=html}
`<sub>`{=html}Explicit infrastructure for observable LLM
reasoning.`</sub>`{=html}
```{=html}
</p>
```
