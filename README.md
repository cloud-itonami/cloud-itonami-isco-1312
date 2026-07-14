# cloud-itonami-isco-1312

Open Occupation Blueprint for **ISCO-08 1312**: Aquaculture and Fisheries Production Managers.

This repository designs a forkable OSS aquaculture/fisheries management support system: a data-collection robot performs yield logging, scheduling, and resource planning under a governor-gated actor, so an aquaculture operation keeps its own records of production and inputs instead of renting a closed aquaculture management SaaS.

## Robotics premise

All cloud-itonami verticals are designed on the premise that a **robot performs
the physical domain work**. Here a data-collection robot performs harvest scheduling, yield logging, supply ordering, and stock/water-quality anomaly detection under an actor that proposes
actions and an independent **Aquaculture Management Governor** that gates them. The governor never
dispatches hardware itself; `:high`/`:safety-critical` actions (such as
large capital expenditures, land-use changes, or anomaly escalations) require human sign-off.

A live sample of the operator console (robotics safety console, shared template) is rendered in [docs/samples/operator-console.html](docs/samples/operator-console.html) — pure-data HTML output of `kotoba.robotics.ui`.

## Core Contract

```text
facility registration + production records + seasonal calendar
        |
        v
Aquaculture Advisor -> Aquaculture Governor -> schedule, log, order supplies, or human sign-off
        |
        v
robot actions (gated) + operating records + audit ledger
```

No automated advice can dispatch a robot action the governor refuses, suppress
an operating record, or disclose sensitive data without governor approval and
audit evidence.

## Capability layer

Resolves via [`kotoba-lang/occupation`](https://github.com/kotoba-lang/occupation)
(ISCO-08 `1312`). Required capabilities:

- :robotics
- :identity
- :forms
- :dmn
- :bpmn
- :audit-ledger

See [`docs/business-model.md`](docs/business-model.md) and
[`docs/operator-guide.md`](docs/operator-guide.md).

## Reference implementation (`:maturity :implemented`)

Full itonami Actor pattern (per ADR-2607011000 / CLAUDE.md's Actors
section, alongside `cloud-itonami-isco-2411`, `-6130`, `-8160`, `-2166`, `-2641`,
`-2651`, `-2652`, `-2654`, `-1219`, `-1223`, `-1330`, `-1341`, `-1349`,
`-1412`, `-1439`, `-2144` and `-2320`): a real
[`kotoba-lang/langgraph`](https://github.com/kotoba-lang/langgraph)
`StateGraph`, with the Advisor and Governor as distinct graph nodes and
human-in-the-loop interrupt/resume via checkpointing.

```text
:intake -> :advise -> :govern -> :decide -+-> :commit            (:ok? true)
                                           +-> :request-approval   (:escalate? true, interrupt-before)
                                           +-> :hold               (:hard? true)
```

- `src/aquaculture_management/store.cljc` — `Store` protocol + `MemStore`:
  registered aquaculture facilities, production records, supply orders, an append-only audit ledger.
- `src/aquaculture_management/advisor.cljc` — `Advisor` protocol; `mock-advisor`
  (deterministic, default) proposes an aquaculture operation from a
  request; `llm-advisor` wraps a `langchain.model/ChatModel` — either
  way the advisor only ever produces a `:propose`-effect proposal,
  never a committed record, and LLM parse failures always yield
  `:confidence 0.0` (forces escalation, never fabricated confidence).
- `src/aquaculture_management/governor.cljc` — `AquacultureGovernor/check`: a pure
  function, wired as its own `:govern` node. Hard invariants
  (unregistered facility, a proposal whose `:effect` isn't `:propose`)
  always route to `:hold`. Escalation invariants (`:flag-stock-anomaly`,
  `:order-supplies` above cost threshold, or low advisor confidence) always route to
  `:request-approval` — an `interrupt-before` node that the graph
  checkpoints and only resumes on explicit human approval
  (`actor/approve!`), matching the README's robotics-premise statement
  that stock/water-quality anomalies and significant supply decisions always require
  human sign-off.
- `src/aquaculture_management/actor.cljc` — `build-graph`, `run-request!`,
  `approve!`: the `langgraph.graph/state-graph` wiring itself.

```bash
clojure -M:test
```

This is what backs this repo's `:maturity :implemented` entry in
[`kotoba-lang/occupation`](https://github.com/kotoba-lang/occupation).

## License

AGPL-3.0-or-later.
