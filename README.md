# cloud-itonami-isco-2114

Open Occupation Blueprint for **ISCO-08 2114**: Geologists and Geophysicists.

This repository designs a forkable OSS survey support operation for geology and geophysics: an autonomous advisor proposes survey operations (data analysis, report drafting, anomaly flagging, equipment requests) under a governor-gated actor, ensuring geological safety, data verification, and human-in-the-loop escalation for hazard findings and anomalies.

## Robotics premise

All cloud-itonami verticals are designed on the premise that a **robot performs
the physical domain work**. Here an autonomous survey advisor proposes analysis pipelines, report preparation, and equipment allocation under an actor that gates all proposals and an independent **Geology Governor** that enforces geological safety. The governor never
dispatches hardware itself; `:high`/`:safety-critical` actions (such as
flagging anomalous formations, or proposing hazard findings in reports) require human sign-off.

## Core Contract

```text
survey-site + survey-data + equipment + geological timeline
        |
        v
Survey Advisor -> Geology Governor -> analyze/draft/flag, or human sign-off
        |
        v
robot actions (gated) + operating records + audit ledger
```

No automated advice can dispatch a robot action the governor refuses, suppress
an operating record, or finalize a result without governor approval and
audit evidence.

## Capability layer

Resolves via [`kotoba-lang/occupation`](https://github.com/kotoba-lang/occupation)
(ISCO-08 `2114`). Required capabilities:

- :robotics
- :identity
- :forms
- :dmn
- :bpmn
- :audit-ledger

## Reference implementation (`:maturity :implemented`)

Full itonami Actor pattern (per ADR-2607011000 / CLAUDE.md's Actors
section, alongside `cloud-itonami-isco-2111`, and other occupation actors): a real
[`kotoba-lang/langgraph`](https://github.com/kotoba-lang/langgraph)
`StateGraph`, with the Advisor and Governor as distinct graph nodes and
human-in-the-loop interrupt/resume via checkpointing.

```text
:intake -> :advise -> :govern -> :decide -+-> :commit            (:ok? true)
                                           +-> :request-approval   (:escalate? true, interrupt-before)
                                           +-> :hold               (:hard? true)
```

- `src/geology/store.kotoba` — `Store` protocol + `MemStore`:
  registered survey sites, survey data, equipment, committed records, an append-only audit ledger.
- `src/geology/advisor.kotoba` — `Advisor` protocol; `mock-advisor`
  (deterministic, default) proposes a survey operation from a
  request; `llm-advisor` wraps a `langchain.model/ChatModel` — either
  way the advisor only ever produces a `:propose`-effect proposal,
  never a committed record, and LLM parse failures always yield
  `confidence 0.0` (forces escalation, never fabricated confidence).
- `src/geology/governor.kotoba` — `GeologyGovernor/check`: a pure
  function, wired as its own `:govern` node. Hard invariants
  (unregistered site, missing data for analysis, a proposal whose
  `:effect` isn't `:propose`, finalized claims in draft proposals)
  always route to `:hold`. Escalation invariants (`:flag-anomalous-formation`,
  `:draft-report` with hazard findings, or low advisor confidence) always route to
  `:request-approval` — an `interrupt-before` node that the graph
  checkpoints and only resumes on explicit human approval
  (`actor/approve!`), matching the README's robotics-premise statement
  that anomaly flags and hazard findings always require
  human sign-off.
- `src/geology/actor.kotoba` — `build-graph`, `run-request!`,
  `approve!`: the `langgraph.graph/state-graph` wiring itself.

Proposal operations (advisor-only, all `:effect :propose`):
- `:analyze-survey-data` — run/propose an analysis pipeline over recorded survey data.
- `:draft-report` — prepare a geological/geophysical report draft (never finalized).
- `:flag-anomalous-formation` — surface a geological anomaly deviating from expectation (escalates).
- `:request-field-equipment` — propose field survey equipment allocation.

```bash
kbb -M:test
```

This is what backs this repo's `:maturity :implemented` entry in
[`kotoba-lang/occupation`](https://github.com/kotoba-lang/occupation).

## License

AGPL-3.0-or-later.
