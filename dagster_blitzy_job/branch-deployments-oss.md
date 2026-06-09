# Branch Deployments on self-hosted Dagster

> Decision context: BlackBox / customer-hosted forces self-hosted OSS Dagster
> (Dagster+ is a SaaS offering — not available for our deployment model).
> This doc covers what Dagster+ Branch Deployments are, and how to recreate
> equivalent functionality on OSS Dagster.

## 1. What Dagster+ Branch Deployments are (the SaaS feature)

Dagster+ ships a feature where every Git branch / PR gets its own
**ephemeral Dagster deployment**:

- Separate isolated Dagster instance (Postgres + run storage + webserver + daemon).
- Spun up automatically by a Dagster+ GitHub Action when you open a PR.
- Unique URL like `https://<org>.dagster.cloud/branch/<pr-number>/`.
- Code loaded from the PR branch.
- Can run jobs end-to-end with that branch's code without touching prod.
- Auto-torn down when the PR is merged or closed.
- Resources (DB, GCS, etc.) configured per environment so PR runs hit
  dev/test data, not prod.

**The point:** safely test pipeline changes against realistic data before
merging, without breaking prod runs.

## 2. Why it matters for Blitzy

A PR touching e.g. `archie-job-reverse-thinking-generator`'s LangGraph
logic today has one validation path: deploy to dev, hope nothing else is
running. With Branch Deployments:

- Trigger a real run on the PR branch's code, end-to-end through the chain.
- View the run in a per-PR Dagster UI.
- Compare metadata (token counts, files touched, percent_complete) against
  a baseline run on `main`.
- PR reviewer links to the actual orchestrated run, not just code diffs.

The value is real. The question is **how much infrastructure is worth
spending** to get it on self-hosted OSS Dagster.

## 3. Four implementation approaches on OSS Dagster

### Approach A — Code Locations per branch (lightest)

OSS Dagster supports loading multiple **code locations** in one instance,
each an isolated Python module that publishes its own jobs/assets. Code
locations can be added/removed dynamically.

**How:**
- One shared Dagster instance (call it "review env").
- CI on PR open: build container image tagged `archie-jobs:pr-123`,
  register as code location named `pr-123`.
- CI on PR close: remove the code location.
- Reviewers visit `review.dagster.dev.blitzy.io` and see PR branches as
  separate code locations alongside `main`.

**Pros:** Cheap. One Dagster instance — no extra Postgres or webserver.

**Cons:** Shared run storage — PR runs and main runs share history.
Shared resources (DB, GCS, Neo4j) — runs from PR-123 will mutate the same
dev DB as PR-124 and as main. **No real isolation.**

**When to use:** If your dev environment is already a free-for-all and
you just want to demo a branch.

### Approach B — Ephemeral Helm release per PR (closest to Dagster+)

Full per-PR Dagster instance.

**How:**
- Dagster OSS Helm chart (`dagster/dagster`) parameterized with a release name.
- GitHub Action on PR open:
  ```bash
  helm install dagster-pr-${PR_NUMBER} dagster/dagster \
    --namespace dagster-pr-${PR_NUMBER} --create-namespace \
    --set webserver.image.tag=pr-${PR_NUMBER} \
    --set postgresql.enabled=true \
    --set ingress.hosts[0].host=pr-${PR_NUMBER}.dagster.dev.blitzy.io
  ```
- Wildcard cert for `*.dagster.dev.blitzy.io`.
- GitHub Action on PR close:
  ```bash
  helm uninstall dagster-pr-${PR_NUMBER} -n dagster-pr-${PR_NUMBER} \
    && kubectl delete ns dagster-pr-${PR_NUMBER}
  ```

**Pros:** Real isolation. PR runs can't pollute main's history. Closest
to Dagster+ semantics.

**Cons:** Heaviest. Each PR spins up Postgres + webserver + daemon pods.
K8s cluster cost scales with open PRs. CI cycle time goes up
(~2-3 min just for Postgres readiness).

**When to use:** Steady-state ≤10 open PRs and the cluster can absorb that.

### Approach C — Local `dagster dev` (the lightweight default)

Dagster's killer feature: `dagster dev` runs the whole stack (webserver,
daemon, ephemeral run storage) in one process with zero setup. Each
engineer runs it locally on their branch checkout.

**Pros:** Zero infra. Already supported.

**Cons:** No shared URL for reviewers — engineer must screen-share or
push results. Won't run k8s-based ops (`k8s_job_op`) the same way without
an extra K8s context.

**When to use:** Day-to-day dev. Doesn't replace shared review envs.

### Approach D — Shared preview environment with isolated resources (recommended)

One Dagster instance dedicated to PR review, with **resource configuration
that isolates data per branch**.

**How:**
- One Dagster instance named `dagster-review`, separate from `dagster-prod`.
- CI on PR open: register branch's code as code location `pr-${PR_NUMBER}`
  (Approach A's mechanism).
- Each code location's `Definitions` uses a **resource configuration keyed
  off the code location name** — different Postgres schema, different GCS
  prefix, different Neo4j graph.
- Example: PR-123 writes to `gs://blitzy-dev/pr-123/...` and Neo4j graph
  `pr_123`.
- Reviewers visit `dagster-review.blitzy.io`, pick the PR-123 code
  location, trigger a run, see materializations and metadata.
- CI on PR close: deregister code location, delete data prefix.

**Pros:** One Dagster control plane, but real per-PR data isolation.
Cheaper than Approach B. More useful than Approach A.

**Cons:** You write the resource-isolation logic yourself. Run history is
shared but tagged by code location, which Dagster's UI handles well.

**When to use:** **Recommended middle ground for OSS Dagster.**

## 4. The harder problem: data isolation (true for Dagster+ too)

Dagster+ Branch Deployments give you an **isolated control plane** out of
the box, but they don't solve **isolated data** — that's still on you to
configure via resources. Even on Dagster+, you need:

- Branch-specific Postgres schema or test DB (workers do real DB writes
  via `common_models`)
- Branch-specific Neo4j namespace/graph (workers write code graphs)
- Branch-specific GCS prefix (workers upload artifacts)
- Branch-specific Pub/Sub-or-MQ topic if external subscribers still listen

The bulk of the engineering effort for Branch Deployments isn't the
Dagster control plane — it's the **resource isolation**. **Self-hosted
Dagster pays the same cost as Dagster+ here.** The "lose Branch
Deployments by going OSS" argument is overstated: you lose the
control-plane convenience but the data-isolation work is identical
either way.

## 5. Recommendation for Blitzy

1. **Default dev:** Approach C (`dagster dev`) for engineer-local iteration.
2. **Shared review:** Approach D — one `dagster-review` instance with
   code-location-keyed resource isolation. Standard CI registers /
   deregisters code locations on PR open / close.
3. **Avoid Approach B** unless you have a hard isolation requirement that
   D can't satisfy. Helm-per-PR works but the operational tax compounds.
4. **Don't treat Branch Deployments as a deal-breaker** for the Dagster
   vs Argo decision. The bulk of the value (real isolated runs against
   branch code) is achievable in OSS with ~1-2 weeks of CI + resource-
   config work, not the months of orchestrator rebuild it would take to
   move to Dagster+.

## 6. Open questions

- **Branch deployment strategy** — confirm Approach D, or pick another?
- **Resource isolation naming convention** — schema/prefix/graph naming
  convention (`pr_${PR_NUMBER}` vs `branch_${slug}` etc.)
- **CI integration** — GitHub Action workflow to register/deregister code
  locations, including image build + push to artifact registry.
- **Cleanup policy** — do we delete data on PR close, or retain for N days
  for post-merge debugging?
- **Cost ceiling** — what's the max open-PR count the review env should
  support before back-pressure?