<p align="center">
  <img src="assets/logo.png" alt="Quadro Logo" width="420">
</p>

<p align="center">
  A governed coordination substrate for multi-agent LLM systems
</p>

<p align="center">
  <a href="https://github.com/ESgarbi/quadro/actions/workflows/ci.yml"><img src="https://github.com/ESgarbi/quadro/actions/workflows/ci.yml/badge.svg" alt="CI"></a>
  <a href="https://github.com/ESgarbi/quadro/actions/workflows/lint.yml"><img src="https://github.com/ESgarbi/quadro/actions/workflows/lint.yml/badge.svg" alt="Code Style"></a>
  <a href="https://github.com/ESgarbi/quadro/actions/workflows/security.yml"><img src="https://github.com/ESgarbi/quadro/actions/workflows/security.yml/badge.svg" alt="Security"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-yellow.svg" alt="License: MIT"></a>
  <a href="https://www.python.org/"><img src="https://img.shields.io/badge/python-3.11%2B-blue.svg" alt="Python 3.11+"></a>
</p>

> [!IMPORTANT]
> **Status: archived.** This is a complete, functional v0.1 reference implementation of the Reactive Governed Blackboard pattern. It is preserved as a design and pattern reference rather than a maintained library. All examples and the full test suite run as documented.

---

## Contents

- [The coordination gap](#the-coordination-gap)
- [Quickstart](#quickstart)
- [The Board](#the-board)
- [The Chief](#the-chief)
- [Governance](#governance)
- [Sagas — declarative work inside a stage](#sagas--declarative-work-inside-a-stage)
- [Lifetime — the Sponsor](#lifetime--the-sponsor)
- [Cost projection — the Estimator](#cost-projection--the-estimator)
- [How this relates to existing work](#how-this-relates-to-existing-work)
- [Reference implementation](#reference-implementation)
- [Pattern reference](#pattern-reference)
- [Contributing](#contributing)

---

## The coordination gap

Enterprise processes have structure for a reason. When an order gets fulfilled, an invoice approved, or a document cleared through legal review, someone needs to be able to answer:

> *"What state is this in right now, and who is responsible for it?"*

When that question can't be answered, the problem usually isn't the agents. It's the coordination layer they run in.

In a multi-agent system, every agent implicitly needs three questions answered. **What should I work on?** Something has to decide which agent gets which task and in what order; left implicit, agents duplicate work, block each other, or idle while work sits waiting. **When should I start?** A writing agent can't start before research finishes. When sequencing constraints live inside individual agent prompts, they are fragile, untestable, and invisible. **When am I done?** Without a shared, enforced definition of terminal state, agents finish and nothing happens, because nothing downstream reliably knows they finished.

Quadro's answer has three parts: a durable shared surface (the **Board**) that holds the state of all work, a coordinator (the **Chief**) that reacts to changes in that state and dispatches the next action, and a governed state machine that enforces which transitions are legal. The agent's questions become structural properties of the system instead of logic buried in prompts.

Together these form the **Reactive Governed Blackboard**. Quadro is a reference implementation of that pattern. For how it sits against LangGraph, Temporal, and the 2025 blackboard revival, jump to [How this relates to existing work](#how-this-relates-to-existing-work).

Lifetime — *should the runtime still be working at all?* — is a separate concern, governed by an external signal called the **Sponsor**. The Sponsor sits outside the reactive pattern and answers to whatever authority fits the deployment: a goal predicate, a deadline, a token budget, an open CRM ticket, an HTTP endpoint. See [Lifetime — the Sponsor](#lifetime--the-sponsor).

### Activity is not progress

Most multi-agent frameworks treat agent activity as the default state. Agents loop and call tools, and the system counts as "working" as long as they're busy. The cost of that assumption is that activity and progress become indistinguishable. An agent burning tokens in a reasoning loop looks the same as an agent making a decision that matters, and there is no structural signal for "everything that can be done right now is being done."

Quadro inverts this. The Chief cannot speculate, explore, or act without a governed task driving it, so when the system is healthy the coordinator spends most of its time dormant. A sleeping Chief means one of two things: all dispatchable work has been dispatched and agents are executing, or there is no work to do. Both are fine. There is no third state where tokens burn without a governed task behind them. If the Chief wakes and finds nothing to act on, telemetry records the no-op, so wasted wakes show up in the numbers instead of hiding in overhead.

I call this rhythm the *sleep pattern*. Early observations from running Quadro pipelines suggest that longer uninterrupted sleep intervals between decision cycles correlate with fewer redundant dispatches. Formal analysis is planned (see [TODO item 13](TODO.md)).

---

## Quickstart

```bash
git clone https://github.com/esgarbi/quadro.git
cd quadro
pip install -e ".[dev]"
```

The package is not on PyPI; install extras from the clone as needed:

```bash
pip install -e .                   # substrate only
pip install -e ".[maf]"            # Microsoft Agent Framework adapter
pip install -e ".[langchain]"      # LangChain / LangGraph adapter
pip install -e ".[anthropic]"      # Anthropic SDK adapter
```

The substrate has zero LLM-framework dependencies. Adapter packages (`quadro_maf`, `quadro_langchain`, `quadro_anthropic`) live as siblings to the core and are imported by user code, never by the substrate. Any other LLM SDK plugs in through a custom reasoner adapter; [`examples/minimal/`](examples/minimal/) shows the shape in 30 lines against the bare OpenAI SDK.

Run the deterministic examples (no API key needed):

```bash
python examples/cooperation/main.py
python examples/ordering_minimal/main.py
pytest
```

Run the LLM-backed examples (requires `OPENAI_API_KEY` or `ANTHROPIC_API_KEY`):

```bash
python examples/newsroom/main_pipeline.py
python examples/ordering/main_pipeline.py
python examples/anthropic_minimal/main.py        # Claude reasoner
python examples/estimator/main.py                # cost projection demo
```

Watch the live Board UI while an example runs:

```bash
# board.db is created automatically when an example runs
python -m quadro.ui newsroom.db --open
```

The Board tab shows the live Kanban. The Costs tab attributes token spend per stage and per task, and renders dollar amounts alongside tokens when pricing is configured on the runtime — see [Cost projection — the Estimator](#cost-projection--the-estimator).

```python
from types import SimpleNamespace

from quadro import ChiefAgent, QuadroRuntime, WorkerAgent
from quadro.board.backends.sqlite import SqliteBoardBackend
from quadro.sponsor import GoalSponsor

runtime = QuadroRuntime(SqliteBoardBackend()).with_profiles(
    profile_resolver={"mywork": "fast"},
)
bc = runtime.client

def do_work(context, board_fn):
    task = context["payload"]["task"]
    board_fn("board.update_task", {
        "task_id": task["task_id"],
        "to_status": "COMPLETE",
    })
    return "done"

worker = (
    WorkerAgent.builder("worker_1", bc)
    .capability("mywork")
    .at("a2a://workers/1")
    .execute(do_work)
    .wakes("a2a://chief")
    .build()
)
worker.register()

chief = ChiefAgent.builder(bc).at("a2a://chief").build()
bc.post_task("mywork", "do something useful")

# Quadro's lifetime is governed by a Sponsor. GoalSponsor is the drop-in
# "run until this predicate is true" shape; see the Lifetime section below
# for richer authorities (deadline, budget, external systems).
runtime.sponsor(
    GoalSponsor(lambda s: all(t["status"] == "COMPLETE" for t in s["tasks"]))
).run(SimpleNamespace(chief=chief))
```

The `fast` profile allows `IN_PROGRESS → COMPLETE` directly. The default `review_required` profile enforces a review step (see [Governance](#governance)). The Chief's default routing matches `task_type` to `capability`, so simple pipelines need no custom policy.

For richer pipelines with multi-step LLM reasoning, validation, audit capture, retry/deadline modifiers, or compensation rollback, see [Sagas](#sagas--declarative-work-inside-a-stage).

---

## The Board

An LLM agent is a function: it runs, produces output, and ceases to exist. Where shared state lives between invocations is left entirely to the developer, and most teams end up scattering it across agent contexts, message threads, and whatever databases happen to be reachable.

The **Board** is a single durable surface that holds the current state of every task, assignment, and result. An agent wakes, reads what it needs, does its work, writes back, and exits. The Board is the only memory between invocations.

This imposes a discipline called **Hydration**: the agent's working context is assembled from the Board's current state at invocation time. Same Board state, same context, every time — no implicit queries, no context window that grows with the age of the process.

## The Chief

The **Chief** is the coordinator. It never executes tasks. It reads the Board, decides what should happen next, writes those decisions back, dispatches workers, and sleeps.

No agent sends the Chief a message. Agents write to the Board, and when the Board changes, a signal carrying no data wakes the Chief. The signal means only: *the board changed; look at it.* The Chief opens the Board, sees everything in flight, acts on all of it in a single pass, and goes back to sleep. If there's nothing to do, it says so and sleeps again.

The design follows from a simple observation: a coordinator that never executes tasks has no reason to stay awake between decisions.

<img src="assets/ombudsman_sleeping.png" alt="The Chief asleep between decision cycles" width="250">
<BR>
<BR>

> This is not polling, which wakes on a timer whether or not anything changed. It is not event callbacks, where the coordinator handles one event at a time and always reasons from a fragment. The Chief wakes only when the board actually changed, reads the full state, and acts on all of it.

<img src="assets/ombudsman_sleeping_pattern.png" alt="The sleep pattern over time: long dormant intervals between short decision cycles" width="300">

## Governance

A durable Board and a reactive Chief aren't enough if the state machine is implicit. Most systems have one anyway: a `status` column, an enum, some if-statements. It works until a bug moves a task directly from `IDEATING` to `PUBLISHED`, or two agents independently transition the same task in different directions.

A **Lifecycle Profile** is a formal contract for a class of work: a set of valid state transitions declared at startup. The Board rejects illegal transitions mechanically, before any application code runs — a `TransitionError`, not a silent overwrite. Every transition emits an immutable event into an append-only log, so the audit trail writes itself.

When an agent crashes mid-task, the **Ombudsman** detects the silence: heartbeats stop, the task is marked stale, and the Chief is woken. The Chief sees the stale task among whatever else is happening and reassigns it as part of its normal pass. Recovery is not a special case.

The `review_required` lifecycle — the states a task moves through, the revision back-edge, and the Ombudsman recovery path:

<p align="center">
  <img src="assets/diagram_lifecycle.svg" alt="Lifecycle diagram: review_required profile" width="680">
</p>

Durable Board, reactive Chief, governed lifecycle, deterministic Hydration: that combination is the **Reactive Governed Blackboard**, an adaptation of the classical Blackboard pattern for stateless LLM agents. It is *governed* (transitions enforced mechanically, illegal moves rejected rather than logged), *hydrated* (context injected at invocation from current Board state, never accumulated or queried on demand), and *reactive* (the coordinator wakes on change, surveys everything, and acts — no polling, no callbacks carrying partial state).

The practical consequence: adding more agents, task types, or pipeline stages does not degrade the coordinator's decision quality. The Chief always sees a clean board. Workers always see a fully hydrated, deterministic task.

How the components relate at runtime — the Board at centre, the Chief reacting to it, Workers reading and writing tasks, the Ombudsman monitoring heartbeats, and the Sponsor governing runtime lifetime from outside the pattern (dashed, to keep coordination and lifetime visually separate):

<p align="center">
  <img src="assets/diagram_runtime.svg" alt="Runtime diagram: the Sponsor (external authority) above, issuing Continue/Drain/Stop into a cluster of Board, Chief, Workers, and Ombudsman" width="680">
</p>

---

## Sagas — declarative work inside a stage

The Reactive Governed Blackboard decides *which* stage runs next and *who* runs it. The work *inside* a stage — the sequence of LLM calls, deterministic validations, audit captures, retries, and rollback semantics — is declared with a **saga**.

A saga is a frozen, declarative description of one stage's work. Where a hand-rolled stage function mixes LLM calls with database writes, retry loops, and ad-hoc telemetry, a saga separates the declaration of what runs from the execution bookkeeping the runtime owns. The runtime persists each step's output to the Board, resumes from the saved program counter on worker restart, emits structured telemetry per step, and walks compensations in reverse order if a later step fails. A worker that crashes mid-saga picks up exactly where it left off; saga state is just another Board record.

Every shipping example pipeline is saga-driven. The newsroom uses sagas for all four stages. The ordering example uses sagas with `.compensate(...)` directives for the rollback path.

### When to use a saga

A Quadro stage supports four authoring shapes:

- `stage(execute_fn=...)` for stages that are one Python call.
- `stage(workflow=...)` to drive an existing MAF workflow.
- `stage(supervisor=...)` / `stage(graph=...)` to drive an existing LangGraph supervisor or graph.
- `stage(saga=...)` when the stage has structure the framework-native paths don't capture.

Use a saga when at least two of these are true: the stage has more than one side effect, you need to resume after a worker crash without repeating completed work, you need rollback if a later step fails, you need an audit trail of what happened inside the stage, or you need typed retry/deadline behavior on individual operations. For a single-call stage, `execute_fn` is right and a saga is overkill.

### Step kinds

Eight step kinds cover the common shapes of work inside a stage:

- `deterministic` — pure Python, sync or async. Reads task input, writes outputs to the saga state.
- `reason` — one LLM reasoning episode. The runtime hands prompt and user message to a registered `Reasoner` and stores the validated output.
- `gate` — predicate-driven branching. The chosen branch is recorded for audit.
- `guard` — pre-condition check. Halts the saga with `guard_failed:<step>` if the predicate fails.
- `expect` — post-condition with the same halt machinery as `guard`, but a distinct telemetry event so audit queries can separate pre-conditions from post-conditions.
- `evidence` — best-effort audit capture. Failures are logged, never fail the saga.
- `stamp` — ordered, timestamped audit marker. Useful for version numbers, release tags, revision counters.
- `parallel` — concurrent mini-sagas with three join modes: `all` waits for every branch, `any` continues on first success and cancels the rest, `n_of_m` waits for a quorum.

Three step modifiers attach to the most recently added step:

- `.retry(attempts=N, on=(...,), backoff=...)` — typed retry loop with fixed or exponential backoff.
- `.deadline(within=timedelta(...))` — per-attempt wall-clock timeout.
- `.idempotent(by="<task_field>")` — saga-wide idempotency key for resume-after-crash semantics.

### Compensation

Any step can declare a compensation:

```python
saga = (
    Saga("order")
    .deterministic("accept_order", _accept)
    .compensate("accept_order", undo=_release_order_slot)
    .deterministic("reserve_inventory", _reserve)
    .compensate("reserve_inventory", undo=_release_inventory)
    .deterministic("charge_card", _charge)
    .compensate("charge_card", undo=_refund)
    .deterministic("ship", _ship)
    .build()
)
```

If a later step fails, the runtime walks completed steps in reverse insertion order and invokes each registered compensation. The default failure mode is `continue`: a failed compensation is logged but the walker proceeds. Set `on_failure="halt"` to stop on the first compensation failure, which is the right discipline when later compensations depend on earlier ones succeeding. Rollback works the same way through `parallel` steps: completed branches' compensations walk in reverse; cancelled branches don't fire compensations because they never finished their side effects.

### Reasoners — bring your own LLM stack

The `Reasoner` protocol is the substrate's deep-agent escape hatch. A reason step looks like a single LLM call from the saga's perspective, but the reasoner behind it can be anything: one API call, a multi-turn ReAct loop, a hierarchical agent, a tool-using supervisor, an entire LangGraph wrapped behind one async method. The saga sees one input and one output; the reasoner owns everything in between.

Three adapter packages ship reasoners for common stacks: `quadro_maf` (Microsoft Agent Framework), `quadro_langchain` (LangChain / LangGraph), and `quadro_anthropic` (Anthropic SDK). The proof of the protocol's reach is [`examples/minimal/`](examples/minimal/README.md): a 30-line reasoner adapter wrapping the OpenAI SDK directly, paired with a tiny saga, runs end-to-end with no adapter package installed. The same shape works for Google, LiteLLM, in-house frameworks, or any SDK that can fulfill prompt-in / response-out.

For sagas that need polyglot reasoning — one step through MAF, another through LangChain — register multiple reasoners and add `via="langchain"` to the relevant `.reason()` calls. The runtime dispatches each step to the named reasoner. The neutrality is easy to verify: the substrate package (`src/quadro/`) imports no LLM frameworks.

### Authoring reference

[`docs/guides/saga-authoring.md`](docs/guides/saga-authoring.md) is the full authoring guide: writing a saga from blank file to tested pipeline stage, one section per step kind, a compensation walkthrough, the custom reasoner pattern, and testing patterns. Read the guide when you want to write a saga; read this section when you want to know what sagas are.

---

## Lifetime — the Sponsor

The Reactive Governed Blackboard defines how work is coordinated, not how long the runtime should keep running. *Should we still be working on this?* is a different question from *has the work completed?*, and it's usually answered by something outside the runtime: a mission goal, a scheduled window, a CRM ticket being open, a budget still positive.

Quadro exposes that seam as a **Sponsor**. A Sponsor is consulted by the `RunLoop` at startup and on lease expiry, and returns `Continue`, `Drain`, or `Stop`. The built-in `GoalSponsor(predicate)` covers the common "run until my goal is met" case. `DeadlineSponsor`, `TickBudgetSponsor`, `LlmTokenBudgetSponsor`, and `HttpSponsor` add wall-clock, cost, or external-authority caps, composable with `AllOf` / `AnyOf` / `Priority`.

This is a lifetime model, not a pattern primitive. The Board, Chief, and governed lifecycle are unchanged whether your Sponsor is a one-line predicate or an HTTP call to a ticketing system. See [`docs/design/sponsor.md`](docs/design/sponsor.md) for the design and [`examples/crm_sponsor/`](examples/crm_sponsor/README.md) for a worked CRM-gated run.

---

## Cost projection — the Estimator

The sleep pattern measures coordination waste: token spend not backed by a governed task. `LlmTokenBudgetSponsor` measures runtime waste: token spend past an authorised cap. Both observe what *has* happened. Neither answers the question that arrives before the run starts: *if I commit to this queue, what will it actually cost?*

That's the **Estimator**. Given a `Pipeline` with at least one saga stage and a queue of tasks, `Estimator.from_dry_run(...)` performs a two-pass run: pass 1 walks the queue without LLM calls to characterise input shapes; pass 2 samples a small number of representative tasks through the real reasoner, sorted across the input distribution so the sample spans the variation the queue contains. The result is a projection with mean cost, a 95% confidence interval, a per-stage breakdown, and a coefficient-of-variation warning when inputs are heterogeneous enough that the estimate is genuinely uncertain.

Pricing is configured on the runtime, not the Estimator. The same pricing model attributes actual costs in the Board UI's Costs tab, so before-the-run estimates and after-the-run attributions are denominated against the same source.

```python
from quadro import Estimator, Pipeline, QuadroRuntime
from quadro.board.backends.sqlite import SqliteBoardBackend

runtime = (
    QuadroRuntime(SqliteBoardBackend("translation.db"))
    .with_profiles(
        profile_resolver={"translation": "translation"},
        custom_profiles={"translation": TRANSLATION_PROFILE},
    )
    .with_pricing({
        "claude-sonnet-4-6": {"input": 3.0, "output": 15.0, "io_ratio": 0.30},
    })
)

pipeline = (
    Pipeline(runtime.board)
    .reasoner(my_reasoner)
    .stage("translate", saga=translation_saga, active_status="translating")
)

estimator = Estimator.from_dry_run(
    pipeline=pipeline,
    queue=my_queue,
    max_samples=8,
    max_sample_cost_dollars=1.0,
)
print(estimator.format())
```

A real run from `examples/estimator/main.py` (50 translation tasks, 8 samples, Claude Sonnet pricing) prints something like:

```text
=== Estimator dry run ===
Pass 1 (input collection): 50 tasks scanned in 0.4s
Pass 2 (sampling): 6 tasks executed (cost: $0.04)

Sample distribution chosen by input-size span:
  Smallest input:  76 chars
  Largest input:   359 chars
  Middle samples:  4 across the distribution

=== Projection for 50 tasks ===
Total tokens:  ~26K
  Range (95% CI):  19K - 33K
  Per-stage breakdown (mean):
    translating       26K  (100.0%)
  Stdev/task: 211 (CoV 0.40)

Total dollars: ~$0.30
  Range (95% CI):  $0.21 - $0.38

Variance warning: HIGH
   Coefficient of variation: 0.40 (>0.30 threshold)
   Recommendation: run additional samples for a tighter estimate.

Pricing source: configured at runtime startup
Verify current rates at https://anthropic.com/pricing
Sample run cost: $0.04 (already spent; included in your billing)
```

The sample run is bounded by `max_sample_cost_dollars`; the Estimator never spends more than the cap before reporting back. Two more constructor paths cover related needs:

- `Estimator.from_history(client, pricing=...)` — project from existing Board token records when you've already executed a slice of work and want to project the rest from real per-task costs.
- `python -m quadro.estimate <board.db>` — CLI projection from a board's persisted history, for ad-hoc estimates without writing a script. Accepts `--project-tasks N`, `--pricing-file path.json`, and `--confidence 0.95`.

That closes the cost loop: the Estimator projects spend before a run, the token-budget Sponsor caps it during, and the Costs tab attributes it afterwards. Two examples cover it end to end:

- [`examples/estimator/`](examples/estimator/) — the minimal demonstration: a 50-task translation queue, projected and optionally executed, with the Costs tab showing the result.
- [`examples/synthetic_data/`](examples/synthetic_data/) — the industry-shaped one: real Wikipedia passages driving two different sagas (SQuAD-style QA and Alpaca-style multi-hop reasoning), with the Estimator surfacing per-saga cost asymmetry on heterogeneous inputs and projecting against full-scale workloads.

## Realtime usage — Board UI

<p align="center">
  <img src="assets/newsroom_cost.gif" alt="Newsroom cost attribution in the Board UI" width="49%" style="display:inline-block; vertical-align:top; margin-right:8px;">
  <img src="assets/orders_cost.gif" alt="Ordering cost attribution in the Board UI" width="49%" style="display:inline-block; vertical-align:top;">
</p>

---

## How this relates to existing work

The ingredients are not novel. Blackboard architecture dates to Newell and Simon in the 1960s. Event sourcing is well-established in distributed systems. State machine governance appears in every workflow engine. LangGraph, Temporal, and Durable Functions each touch parts of this space.

In 2025, the blackboard model resurfaced independently across the industry. Google researchers demonstrated a blackboard MAS outperforming RAG and master-slave baselines by 13–57% on data science benchmarks (arXiv:2510.01285). A separate paper proposed the first formal blackboard framework for general LLM-based multi-agent systems (arXiv:2507.01701). Confluent named blackboard as one of four key patterns for event-driven multi-agent systems. AWS published the Arbiter Pattern, which uses a shared semantic blackboard as its coordination substrate — though the Arbiter goes well beyond that, adding LLM-driven task decomposition, dynamic agent generation via a Fabricator, and contextual memory across cycles. The Chief does none of those things; its correctness comes from governance structure rather than reasoning. They are different answers to different questions about multi-agent coordination.

Quadro is a ground-up implementation of the Blackboard pattern for stateless LLM agents, extended with three constraints: governed lifecycle transitions (the Board rejects illegal moves mechanically), deterministic hydration (same state → same context, verifiable by hash), and reactive coordination (the Chief surveys the full board on wake, never a partial event stream). I call the result the Reactive Governed Blackboard: not a new pattern, but a specific discipline applied to a classical one.

The substrate is framework-neutral. Adapters for Microsoft Agent Framework, LangChain, and the Anthropic SDK ship as sibling packages; custom adapters for any other LLM SDK plug in through the `Reasoner` protocol. Quadro runs alongside the LLM stack you already have rather than wrapping it.

| If you have...            | Quadro adds...                                                                                  |
|---------------------------|-------------------------------------------------------------------------------------------------|
| Agent Framework / AutoGen | A governed Board, lifecycle enforcement, a saga DSL for stage internals, and pre-run cost projection — without wrapping the framework |
| LangGraph                 | An explicit task lifecycle with validated transitions, audit trail, saga compensation, and an Estimator for pre-run cost projection; runs alongside LangGraph rather than replacing it |
| Temporal                  | The agent-specific hydration contract, a saga DSL tuned for LLM workloads (reason steps, polyglot reasoners, compensation rollback), and pre-run token-cost projection |
| A raw message bus         | Named vocabulary, lifecycle semantics, saga authoring surface for agent work, and cost visibility from estimate through attribution |

---

## Reference implementation

Python 3.11+. The substrate package (`quadro`) has zero LLM-framework dependencies and ships with the Board, governed lifecycle, A2A dispatch, chief and worker coordination, the Sponsor lifetime model, the Estimator, the zero-dependency Board UI, and the saga DSL. LLM-framework adapters live in sibling packages installed as optional extras. This is a reference implementation of the pattern; production hardening was in progress at time of archiving.

**Substrate (`src/quadro/`)**

- `QuadroBoard` — board, SQLite backend, validated lifecycle, immutable event log
- `BoardClient` — typed wrapper around the board's A2A interface (`board.client()`)
- `ChiefAgent` — reactive coordinator, pending-wake serialisation, telemetry
- `WorkerAgent` — stateless worker, automatic `HUMAN_REVIEW` transition on crash, heartbeat
- `WorkerPool` — fluent builder for N-worker-per-capability pools with Ombudsman
- `Pipeline` — substrate builder. Compose adapters via `.reasoner(...)` and `.with_framework_runtime(...)`
- `Saga` / `SagaBuilder` — declarative DSL for stage internals (eight step kinds, three modifiers, compensation rollback)
- `QuadroSagaRuntime` — saga dispatch, persistence, deterministic chief mode for substrate-only pipelines
- `Reasoner` — framework-neutral protocol for LLM reasoning. Anything with `reasoner_id` and async `reason()` qualifies
- `FrameworkRuntime` — protocol for stage-level adapter integrations (chief tooling, native stage paths)
- `RunLoop` — sponsor-governed poll loop, per-cycle callback, Ombudsman integration
- `Ombudsman` — stale heartbeat detection for standard and custom profiles
- `LifecycleBuilder` — fluent builder for custom task lifecycle profiles
- `lifecycle()` — function-form lifecycle declaration from a list of transitions
- `Estimator` — pre-run cost projection via two-pass dry-run sampling or historical replay
- `Pricing` / `ModelPricing` / `Projection` — pricing model and projection result types
- `serve_board()` — zero-dependency live Kanban server with Costs tab, stdlib only

**Adapter packages**

<img src="assets/quadro_runtime_peers.svg" alt="Adapter packages as runtime peers" width="700">

Adapters live as siblings to the core, installed via optional extras:

- `quadro_maf` — Microsoft Agent Framework. Provides `MafReasoner` (reason-step adapter) and `MafChiefRuntime` (chief-loop adapter for `stage(workflow=...)` paths). `pip install -e ".[maf]"`.
- `quadro_langchain` — LangChain / LangGraph. Provides `LangChainReasoner` and `LangChainChiefRuntime` with the same shape. `pip install -e ".[langchain]"`.
- `quadro_anthropic` — Anthropic SDK. Provides `AnthropicReasoner` for reason-step integration with Claude models. Reasoner-only: Anthropic ships an SDK rather than a full agent framework, so there is no `AnthropicChiefRuntime`; use `stage(execute_fn=...)` for Claude-driven chief logic. `pip install -e ".[anthropic]"`.

Adapter packages import `quadro`; `quadro` does not import them. User code constructs LLM clients with the underlying SDK directly and registers them on a `Pipeline`:

```python
from agent_framework.openai import OpenAIChatClient
from quadro import Pipeline
from quadro_maf import MafReasoner, MafChiefRuntime

def client_factory():
    return OpenAIChatClient(model="gpt-4o", api_key="...")

pipeline = (
    Pipeline(board)
    .reasoner(MafReasoner(client_factory=client_factory))
    .with_framework_runtime(MafChiefRuntime(client_factory=client_factory))
    .stage(...)
    .build()
)
```

Custom adapters for any other LLM SDK plug in through the same `.reasoner(...)` seam — see [`examples/minimal/`](examples/minimal/README.md).

**Built-in lifecycle profiles**

Two profiles ship out of the box:

- `review_required` — `UNASSIGNED → IN_PROGRESS → PENDING_REVIEW → APPROVED → COMPLETE`
- `fast` — `UNASSIGNED → IN_PROGRESS → COMPLETE`

Both automatically include `HUMAN_REVIEW` and `ON_HOLD` as global exits from any state, and `STALE → UNASSIGNED` for Ombudsman recovery.

**Custom lifecycle profiles**

For multi-stage pipelines, use `LifecycleBuilder` to declare the exact transitions your domain requires. The Board enforces them; nothing else needs to know the rules.

```python
from quadro import LifecycleBuilder

ARTICLE_LIFECYCLE = (
    LifecycleBuilder()
    .phase("UNASSIGNED",     "ideating")
    .phase("ideating",       "idea_ready")
    .phase("idea_ready",     "researching")
    .phase("researching",    "research_ready")
    .phase("research_ready", "writing")
    .phase("writing",        "draft_ready")
    .phase("draft_ready",    "reviewing")
    .phase("reviewing",      "published")
    .revision("reviewing",  "idea_ready")   # reviewer can send back for rework
    .build()
)
```

Four builder methods cover the transition types:

- `.phase(from, to)` — main pipeline progression. Both states appear in Board UI column order in declaration sequence.
- `.revision(from, to)` — back-edge for revision loops. The transition is enforced; column order is unchanged because the destination state is already declared.
- `.loop(from, to)` — self-healing cycle back to an earlier stage (e.g. a procurement step that loops back to stock-checking).
- `.branch(from, to)` — alternative exit from a state (e.g. a validation step that can also produce `validation_failed`).

Register the lifecycle with the board at construction time:

```python
board = QuadroBoard(
    SqliteBoardBackend(),
    profile_resolver={"article": "article"},   # task_type → profile name
    custom_profiles={"article": ARTICLE_LIFECYCLE},
    network=network,
    url="a2a://board",
)
```

For simpler cases, `lifecycle()` accepts a plain list of `(from, to)` tuples and derives column order from declaration sequence:

```python
from quadro import lifecycle

ORDER_LIFECYCLE = lifecycle([
    ("UNASSIGNED", "validating"),
    ("validating",  "validated"),
    ("validated",   "delivering"),
    ("delivering",  "delivered"),
])
```

**TOML lifecycle files**

Lifecycle profiles can also be declared in `.lifecycle.toml` files, which is useful for versioning lifecycle definitions separately from code or sharing them across services. Uses stdlib `tomllib` (Python 3.11+), no extra dependencies.

```toml
name = "article"

phases = [
    ["UNASSIGNED", "ideating"],
    ["ideating", "idea_ready"],
    ["idea_ready", "researching"],
    ["researching", "research_ready"],
    ["research_ready", "writing"],
    ["writing", "draft_ready"],
    ["draft_ready", "reviewing"],
    ["reviewing", "published"],
]

revisions = [
    ["reviewing", "idea_ready"],
]
```

Load and register it:

```python
from quadro import load_lifecycle

name, lifecycle = load_lifecycle("article.lifecycle.toml")

board = QuadroBoard(
    SqliteBoardBackend(),
    profile_resolver={"article": name},
    custom_profiles={name: lifecycle},
    network=network,
)
```

All examples support a `--lifecycle` flag to load from a TOML file instead of the built-in Python declaration.

**WorkerPool**

For pipelines with multiple capabilities, `WorkerPool` handles agent creation, registration, and Ombudsman configuration in one fluent call:

```python
from quadro import WorkerPool

pool = (
    WorkerPool(bc)
    .workers(3)                  # 3 agents per capability
    .wakes("a2a://chief")
    .add("ideation", run_ideation, active_status="ideating",    max_working_time=5.0)
    .add("research", run_research, active_status="researching", max_working_time=5.0)
    .add("writing",  run_writing,  active_status="writing",     max_working_time=5.0)
    .add("review",   run_review,   active_status="reviewing",   max_working_time=5.0)
    .build()
)

ombudsman = pool.ombudsman()   # pre-configured with per-capability timeouts
```

`max_working_time` is in minutes. Workers that exceed it without posting a heartbeat are marked stale and reassigned by the Chief automatically.

---

### Examples

Examples are organized by what they teach, not by which LLM framework they use. Each folder is self-contained; copy any of them as a starting point for a new pipeline.

LLM-backed (require `OPENAI_API_KEY` or `ANTHROPIC_API_KEY`):

### `examples/ordering/` — LLM order fulfilment with inventory management

Continuous dispatch under pressure. Orders arrive, get validated against customer records, checked against warehouse inventory, and dispatched for delivery. The Chief barely sleeps here — in the demo it's almost permanently in "Acting" state, assigning workers, routing orders through stock checks, and pushing fulfilled orders to shipping. Sagas with `.compensate(...)` directives drive the rollback path; pass `--inject-failure` to trigger a synthetic failure and watch compensation walk live. This is Quadro under sustained load, with every transition governed and every assignment auditable even at speed.

![Ordering System Demo](assets/ordering_system.gif)

### `examples/newsroom/` — 9-stage newsroom pipeline with PubMed research and revision loop

Long-running generative work with a sleeping coordinator. Each task is a full article: topic ideation, PubMed research, draft writing, editorial review, publication. Each stage involves genuine LLM generation — a research agent queries PubMed and synthesizes findings, a writing agent produces a full draft, a reviewer sends it back for revision or approves it. These stages take time, and in that time the Chief sleeps. You can watch it in the Board UI: the coordinator wakes, dispatches a writer, and returns to sleep. Minutes pass. The writer finishes, writes to the board, the signal fires, the Chief wakes, reads the full board state, dispatches the next stage, and sleeps again. Published articles, complete with PubMed citations, are in `examples/newsroom/output/`.

![Newsroom Demo](assets/newsroom_example.gif)

### `examples/synthetic_data/` — heterogeneous LLM training-data generation with cost projection

The Estimator against a real workload. Loads Wikipedia passages from HuggingFace and runs them through two distinct sagas: SQuAD-style extractive QA pairs and Alpaca-style multi-hop reasoning chains with chain-of-thought traces. The Estimator runs a cost-bounded dry run before any real generation, surfaces per-saga cost asymmetry (the reasoning saga is roughly 2x more expensive per task than the QA saga), and projects against full-scale workloads with confidence intervals that widen honestly when extrapolating beyond the sample size. Outputs JSONL files directly loadable by the HuggingFace `datasets` library.

### `examples/anthropic_minimal/` — smallest example using Claude as the reasoner

Posts one task, runs a saga that asks Claude to summarise an article with a Pydantic-enforced output schema, and exits when the task reaches `summarized`. Uses the `quadro_anthropic` adapter. Also demonstrates that the standard token records and Costs UI work through any reasoner with no special integration.

### `examples/estimator/` — minimal Estimator demonstration on a translation saga

Scans a 50-task translation queue, samples representative tasks under a `$1.00` cap, and prints a token-and-dollar projection with variance reporting. Shorter and faster than the synthetic-data example; use this one to learn the Estimator API.

### `examples/token_budget/` — sponsor system enforcing an LLM token cap

`LlmTokenBudgetSponsor` enforces a hard cap on total LLM tokens consumed across a run, with soft warnings before the cap is reached. When the budget is exhausted, the runtime drains in-flight tasks and stops cleanly. Driven through the `quadro_langchain` adapter.

### `examples/minimal/` — bare-OpenAI-SDK reasoner with no framework adapter

A 30-line reasoner adapter wrapping the OpenAI SDK directly, paired with a tiny saga. Shows that any LLM SDK plugs into Quadro through the `Reasoner` protocol — the shipped adapter packages are convenient defaults, not architectural requirements. Mirror the same shape for Google, LiteLLM, or any other library. The saga authoring guide ([`docs/guides/saga-authoring.md`](docs/guides/saga-authoring.md)) uses this example as its reference walkthrough.

Deterministic (no API key required):

- [`examples/cooperation/main.py`](examples/cooperation/README.md) — research / write / review pipeline using built-in lifecycle profiles. Smallest possible cooperative-worker setup.
- [`examples/ordering_minimal/main.py`](examples/ordering_minimal/README.md) — same compensation rollback pattern as `ordering/` but substrate-only, driven by the deterministic chief, no LLM. Run with `--inject-failure reserve_inventory` to see compensation walking through a real failure path.
- [`examples/crm_sponsor/`](examples/crm_sponsor/README.md) — `HttpSponsor` wired to a CRM ticket status. Demonstrates the Sponsor protocol's external-authority shape with no LLM involvement.
- [`examples/workflow_stage_minimal/main.py`](examples/workflow_stage_minimal/) — native MAF workflow as a stage entrypoint (`stage(workflow=...)` instead of `stage(saga=...)`), for users who prefer the pure MAF workflow runtime.
- [`examples/supervisor_stage_minimal/main.py`](examples/supervisor_stage_minimal/) — symmetric for LangGraph (`stage(supervisor=...)` or `stage(graph=...)`).

**Known limitations**

- `LocalA2ANetwork` only — no HTTP transport for multi-process deployments (the `A2ATransport` Protocol is in place; `HttpA2ANetwork` was the next step)
- SQLite backend only — PostgreSQL, MySQL, Redis were planned

See [`TODO.md`](TODO.md) for the full open item list and [`IMPLEMENTATION_ROADMAP.md`](IMPLEMENTATION_ROADMAP.md) for milestone status at time of archiving.

---

## Pattern reference

| Concept | Definition |
|---|---|
| **The Board** | Single durable surface. Every agent reads from it before acting, writes to it when done. |
| **Hydration** | Reconstructing an agent's full context from the Board at invocation time. Deterministic. |
| **Stateless invocation** | One invocation: read Board, act, write Board, exit. Next invocation starts fresh. |
| **The Chief** | Coordinator that reacts to Board changes, dispatches workers, never executes tasks. |
| **Substrate** | The `quadro` core package. Zero LLM-framework dependencies. Owns the Board, lifecycle, chief, workers, Sponsor, Estimator, Board UI, and saga DSL. |
| **Adapter package** | Sibling package that imports `quadro` and provides framework-specific reasoners or runtimes (`quadro_maf`, `quadro_langchain`, `quadro_anthropic`). |
| **Lifecycle profile** | Declared valid state transitions for a task type. Board enforces; illegal moves rejected. |
| **LifecycleBuilder** | Fluent API for declaring custom lifecycle profiles with phases, revisions, and branches. |
| **Saga** | Frozen, declarative description of one stage's work. Eight step kinds, three modifiers, compensation rollback, concurrent branches via `parallel`. |
| **Reasoner** | Framework-neutral protocol for LLM reasoning. Any class with `reasoner_id` and async `reason()` qualifies. |
| **Compensation** | Undo function declared on a saga step via `.compensate(...)`. Invoked in reverse insertion order if a later step fails. |
| **Frozen taxonomy** | Fixed set of event types emitted by the Board. Every transition is auditable. |
| **The Ombudsman** | Detects stale heartbeats, marks tasks for reassignment. Recovery by design, not exception. |
| **Reactive Wakeup** | Chief wakes on a signal (no payload), reads full board, acts on all visible concerns, sleeps. |
| **Sponsor** | External authority that decides whether the runtime should continue. Returns `Continue` / `Drain` / `Stop` from the `RunLoop`. Lifetime concern, separate from coordination. |
| **Estimator** | Pre-run cost projection. Walks a queue, samples representative tasks, returns mean cost with confidence interval and per-stage breakdown. |
| **Pricing model** | Per-model input/output rates and IO ratio. Drives both Estimator projections and Costs tab attribution. |

---

## Contributing

The project is archived, so issues and pull requests are closed. [`CONTRIBUTING.md`](CONTRIBUTING.md) is preserved as documentation of the setup, test conventions, and architecture invariants. Fork freely — it's MIT.

---

## License

MIT. See [LICENSE](LICENSE).
