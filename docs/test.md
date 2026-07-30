# DS Experimentation Setup — Design Review Summary

**Date:** 30 July 2026
**Scope:** Branching, environment isolation, and configuration strategy for data science experimentation on the existing GitHub Actions + Databricks + Airflow platform.

---

## 1. Background

### Current platform

| Area | Current state |
|---|---|
| CI/CD | GitHub Actions |
| Environments | LAB (lower env, dev/test), PROD |
| Branching | `feature/*` → `develop` (LAB tests) → `main` → PROD |
| Compute | Databricks (LAB and PROD), Unity Catalog enabled |
| Orchestration | Airflow — one LAB instance, one PROD instance, both shared across multiple DS teams |
| DAGs | Stored in a `DAGs` folder in the project repo |
| Config | `config.yaml` in-repo (repo-specific) + Azure App Configuration (cross-repo/common) |
| Packaging | One Python package per project, consumed from Databricks notebooks; UV as package manager |
| Team size | 6 data scientists on this project |

### Proposal under review

1. `experiment/*` branches for LAB experiment development.
2. CI/CD deploys notebooks, DAGs, and configuration to LAB for `experiment/*`.
3. CI/CD deploys the same to PROD for `experiment/main`.
4. Driver: reduce experiment code in the `main` branch.

---

## 2. Key finding — two different things share one name

The discussion surfaced that "experiment" is currently used for two workloads with very different risk profiles:

| | **Exploration** | **Production A/B testing** |
|---|---|---|
| What it is | DS trying ideas | Challenger variant scoring real customers |
| Output consumed by | Nobody | Downstream system, via a consolidated result |
| Blast radius | None | Real customers |
| Correct home | LAB, ephemeral | `main`, full production gating |

Adopting distinct vocabulary for these two ("exploration" vs "experiment") is recommended, as most of the design decisions below depend on which one is being discussed.

> Note: calling an A/B test an "experiment" is standard industry terminology. The ambiguity comes from reusing the same word for exploratory work.

---

## 3. Recommendations

### 3.1 Branching

- **Do not implement `experiment/main` → PROD.** It creates a second path to production with weaker gates. The A/B workload has real customer blast radius and warrants *more* gating than the standard path, not less.
- **`experiment/*` → LAB, for exploration only.** These branches are deleted, not merged. Anything worth keeping is rewritten into the package via a normal `feature/*` PR.
- **A/B code merges to `main`** through the existing `feature/*` → `develop` → `main` path. The challenger variant, assignment logic, `variant` column, and traceability fields are production code and belong there.
- **Optional:** a separate `<project>-experiments` repo, pinned to a released version of the core package, if reducing exploration code in the main repo remains a priority.

### 3.2 LAB hygiene for exploration branches

- **Namespace everything per branch** — DAG IDs (`exp_<branch>_<dag>`), Databricks job names, App Configuration labels. Without this, DS on different branches will silently overwrite each other's DAGs on the shared LAB instance.
- **TTL and auto-cleanup** — a scheduled job removing DAGs, jobs, and config for branches deleted or untouched for N days.
- **Per-branch package installs** via UV (`git+…@<branch>` or a pre-release wheel) so notebook and package versions cannot drift.
- **Airflow should be optional for exploration.** DS iterating on a model should be able to run Databricks jobs/notebooks directly, without authoring a DAG.

### 3.3 Isolation — enforce via Unity Catalog grants, not policy

Experiment Service Principal permissions:

| Permission | Grant? |
|---|---|
| Read prod source tables | Yes |
| Write to `exp_` scoped catalog/schema and volume | Yes |
| Create model versions, apply tags | Yes |
| Set model aliases (`champion` / prod) | **No** — promotion job SP only |
| Write to prod serving tables | **No** |

**On shared tables and volumes:** an initial objection to experiment output landing in the production table was withdrawn once it became clear both arms are production output written by a single production pipeline. One writer producing two arms, differentiated by a `variant` column, is a correct design. The concern was a second writer with a weaker identity holding write access on a production table — which is not the case here.

**On model registry:** registering experiment runs as new versions on the existing model is the right approach. Versions are immutable and aliases are the pointer, so a version that never receives `champion` cannot affect serving.

> **Rule:** nothing loads a model by version number in production — alias only. Enforce this in the package's model-loading helper.

### 3.4 Shared Airflow — under-appreciated risk

Both Airflow instances are shared across multiple DS teams. Before pointing experiment DAGs at PROD:

- Dedicated **pool and queue** with hard slot caps for `exp_` DAGs, so a runaway backfill cannot starve other teams' production pipelines.
- Enforce `catchup=False` and `max_active_runs=1`.
- Mandatory `expires_on` tag on every experiment DAG, plus a weekly job that pauses expired DAGs.
- Cost tags on Databricks jobs/clusters (`purpose=experiment`, `owner`, `project`) and budget alerts.

### 3.5 MLflow — differentiating regular vs experiment runs

Implement both:

1. **Separate experiment paths** — `/Prod/<project>/production` vs `/Prod/<project>/experiments/<name>`.
2. **Mandatory run tags** — `run_type`, `git_sha`, `branch`, `dag_id`, `triggered_by`, `service_principal`.

Enforce tagging with a `start_run()` wrapper in the core package that raises if tags are missing. Since every notebook already imports the package, this gives enforcement without relying on individual discipline.

### 3.6 Configuration

**Do not use separate App Configuration labels for the experiment dimension.** A key resolves as (key, label), giving one dimension only — already used for environment. Adding experiment as a second use collides the first time an experiment config must differ between LAB and PROD.

Instead:

- **Key prefixes** for the experiment dimension: `myproject:experiments:<experiment_id>:allocation`, label `PROD`.
- **Feature flags** (App Configuration's native construct, with targeting filters) for enable/disable and percentage allocation. Purpose-built for this, and provides a real kill switch.

**Split the existing `experiment` section of `config.yaml` by change cadence, not by repo scope:**

| Stays in `config.yaml` (git, reviewed, deployed) | Moves to App Configuration |
|---|---|
| Model variant, feature set | Live / not live |
| Output schema | Allocation percentage |
| Metric definitions, guardrail thresholds | Start and end dates |
| | Exclusion list reference |

Supporting requirements:

- **Reproducibility** — resolve full config at job start, hash it, and log the hash plus the App Configuration snapshot reference as an MLflow run tag and on output rows.
- **Disjoint ownership** — no key exists in both places. The config loader should fail loudly on duplicates rather than applying a precedence rule.
- **Access control** — whoever can change allocation can change what real customers receive. If store RBAC cannot scope to a key prefix, consider a separate store for experiment control keys.

---

## 4. Highest-priority actions

### 4.1 Automate the human consolidation step

Currently a person reads job output, builds a consolidated result flagging each customer as regular or experiment, and shares it downstream. This is an unversioned, unaudited, and unreproducible step in a customer-facing path, with no rollback if the flag is set incorrectly.

Replace with:

- **Deterministic, versioned assignment** — hash of customer ID + experiment ID, or a stored assignment table with effective dates. Must be reproducible months later.
- **Per-row traceability** — `variant`, `experiment_id`, `model_version`, `mlflow_run_id`, `scored_at`. Near-free at write time and essential for attribution analysis.

### 4.2 Confirm governance ownership

Customer-level experimentation requires a named owner for the assignment decision and an exclusion list (opted-out or vulnerable customers) honoured before assignment.

---

## 5. Bottom line

Once assignment and traceability are handled in the pipeline, the branching question largely resolves itself:

**One repo, one `main`, one production path, experiments toggled by configuration.**

---

## 6. Open items

- [ ] Agree vocabulary: "exploration" vs "experiment"
- [ ] Confirm owner of customer-assignment decision and exclusion list
- [ ] Design assignment table / deterministic hashing approach
- [ ] Define `exp_` catalog, schema, and volume layout
- [ ] Set up experiment Service Principal and grant matrix
- [ ] Configure Airflow pool, queue, and slot caps for `exp_` DAGs
- [ ] Implement `start_run()` tagging wrapper in the core package
- [ ] Split `config.yaml` experiment section and define App Configuration keys
- [ ] Build TTL / cleanup job for expired experiment branches and DAGs
