# Blitzy Client Volume Test

Plans, scenarios, and tooling for **isolated** load tests of the
Blitzy client-side stack (mediator, relay, Redis, worker, GKE).
Driven by the Apr-2026 incident where a 40M-LOC repo broke the
system with cascading failures across all components.

## Folder layout

| File | Purpose |
|---|---|
| `PLAN.md` | The master plan — goals, environments, per-component test design, tooling, metrics, safety rules |
| `redis-tests/scenarios.md` | Redis-only workload definitions (memtier + RQ-shaped traffic) |
| `mediator-tests/scenarios.md` | Mediator workload definitions with mixed-payload ratios (commands × runner creates) |
| `relay-tests/scenarios.md` | Relay workload definitions with multi-mediator simulators |
| `local-setup.md` | How to bring up mediator + relay + Redis on a laptop for local tests |
| `cluster-setup.md` | How to point the same load drivers at the isolated dev cluster |
| `metrics-and-dashboards.md` | What to capture, where, and how to share results |

## Reading order

1. **`PLAN.md`** — start here. Covers everything; the per-component
   files are deep-dives.
2. **`local-setup.md`** — get a working stack on your laptop.
3. **`redis-tests/scenarios.md`** — easiest first run, no service code involved.
4. **`mediator-tests/scenarios.md`** — adds mixed-payload load.
5. **`relay-tests/scenarios.md`** — multi-mediator (the most complex driver).
6. **`cluster-setup.md`** — only after local results are stable.

## Status

Plan only. No tooling installed, no scripts written. The plan is
designed to be executable iteratively — pick one component, run its
scenarios locally, capture results, decide whether to run on cluster,
move to the next component.
