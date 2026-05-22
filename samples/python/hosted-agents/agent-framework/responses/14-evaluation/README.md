# Evaluating a hosted agent

This sample is the **evaluation learning path** for the Python hosted-agent
samples. New to evaluation? Read the next two sections first — they explain
the *what* and *why* before any code.

## What is evaluation?

Once your agent is deployed, **evaluation** is how you answer
*"is my agent actually good?"* You run the agent against a set of test
inputs and let one or more **evaluators** — automated scorers that
produce a **score and rationale** for each response — grade each turn.
Common reasons to run one:

* **Catch regressions** before your users do — re-run after every prompt or
  model-deployment change.
* **Compare agent versions** numerically (v1 averaged 3.2 on task
  completion; v2 averages 4.1).
* **Decide if the agent is ready to ship** — block a release until a
  baseline of evaluators passes.
* **Probe for unsafe behavior** under adversarial input (red-teaming).

The output of every evaluation is a row of scores per input + a portal page
where you can drill into per-row scores and rationales. *Nothing in this
sample needs you to write your own evaluator from scratch* — the
**Custom Rubric Evaluator** ⭐ generates one tailored to your agent from a
short prompt (the recommended path), and Foundry also ships built-in
quality and safety evaluators you can mix in.

### Heads-up: scores don't all use the same scale

Different evaluator families use different scoring shapes. Always open
the report URL and read the rationale first — the words tell you more
than the digit. The numbers themselves mean different things:

| Evaluator family | Scale | Direction |
|---|---|---|
| **Custom Rubric Evaluator** ⭐ (your generated rubric) | **1-5 per dimension**, weighted | Higher is better; the rubric weights each dimension. **Primary recommended evaluator** — tailored to *your* agent. |
| **Safety / content** (`builtin.violence`, `builtin.self_harm`, `builtin.hate_unfairness`, `builtin.sexual`) | **0-7 severity** | **Higher is worse.** 0 = safe; 4+ = concerning; 6-7 = severe. Default pass threshold is severity ≤ 3. **Primary for any user-facing agent.** |
| **Attack detection** (`builtin.indirect_attack`) | **Detected / Not detected** | A "detected" result means the agent appears to have been manipulated by a prompt-injection-style attack (bad). |
| Quality (`builtin.fluency`, `builtin.relevance`, `builtin.coherence`, `builtin.groundedness`) | 1-5 | Higher is better. Generic signal — useful as a sanity check; prefer the Custom Rubric Evaluator for anything you care about. |
| Agent task (`builtin.task_adherence`, `builtin.task_completion`, `builtin.customer_satisfaction`) | Pass / Fail + numeric where present | Trust `passed` + the rationale first. Generic signal — the Custom Rubric Evaluator usually tells you more. |

"Passed" rows mean *score ≥ pass-threshold* (quality) or
*severity ≤ pass-threshold* (safety). The `result_counts` the scripts
print already does that math — you just need to remember the direction.

## Concepts at a glance

| Term | What it means |
|---|---|
| **Evaluator** | The judge that scores one row. Three flavors: **custom rubric** (auto-generated from your prompt — primary recommended), *built-in* (`builtin.violence`, `builtin.fluency`, …), or *code-based* (yours). |
| **Dataset** | The rows you evaluate against. Either inline `{query: ...}` items, a registered Foundry dataset, or generated from traces. |
| **Trace** | A recording of one real agent invocation (request, tool calls, response, latencies) sent to Application Insights by the agent runtime. |
| **Eval** | A reusable "test suite" definition — schema + evaluators. Created once, run many times. |
| **Eval run** | One execution of an eval against a specific dataset / agent / time window. Has a status, a result-counts summary, and a `report_url`. |
| **Single-turn vs. multi-turn** | Single-turn evaluators score one `{query, response}` pair. Multi-turn evaluators score a whole `messages: [...]` conversation. |
| **Score shape** | See the table above — quality is 1-5 (higher better), safety is 0-7 severity (higher worse), some are boolean. |

> The tags you may see in the Python scripts (`<imports_and_includes>`,
> `<run_eval>`, …) are documentation extraction markers used by the docs
> pipeline. They are inert in Python — feel free to ignore them when
> reading or copying code.

## Your first run

This folder contains *both* a tiny demo agent (`main.py`, `agent.yaml`) **and**
the eval scripts. The agent is a minimal `gpt-4.1-mini` chat agent with
tracing turned on — just enough surface for the eval scripts to have
something to grade. The flow is:

```
                      ┌───────────────────────────┐
                      │  this sample's tiny agent │
                      │ (deploy once via Foundry) │
                      └─────────────┬─────────────┘
                                    │
                                    ▼
   evaluate_*.py ──── eval run ──── scores ──── report_url in Foundry portal
```

```bash
# 1. Deploy the tiny demo agent (one time).
#    These commands follow the same pattern as every other Python sample
#    in samples/python/hosted-agents/ — see the parent README for details.
mkdir hosted-agent-evaluation && cd hosted-agent-evaluation
azd ai agent init -m ../path/to/foundry-samples/samples/python/hosted-agents/agent-framework/responses/14-evaluation/agent.manifest.yaml
azd up

# After `azd up` succeeds, copy the project endpoint it prints (or grab it
# from your Foundry project's Overview page) into the env var below.

# 2. Set env + install eval deps locally.
az login
export FOUNDRY_PROJECT_ENDPOINT="https://<account>.services.ai.azure.com/api/projects/<project>"
export AZURE_AI_MODEL_DEPLOYMENT_NAME="gpt-4.1-mini"
pip install -r requirements.txt

# 3. Run the primary recommended evaluator. Edit the agent description at
#    the top of submit_generation_job() in the script first so the
#    generated rubric matches what your agent is supposed to do.
python evaluate_custom_rubric.py
```

> **Windows / PowerShell?** Replace `export FOO=bar` with `$env:FOO = "bar"`.

> Want a 60-second sanity check before kicking off the rubric
> generation? Run [`evaluate_basic.py`](./evaluate_basic.py) first — it
> uses generic built-in evaluators to confirm the end-to-end plumbing
> works, then come back to `evaluate_custom_rubric.py` for the real signal.

What you'll see (trimmed):

```
Using API version: 2025-11-15-preview
Project: https://<account>.services.ai.azure.com/api/projects/<project>
Target agent: {'type': 'azure_ai_agent', 'name': 'agent-framework-agent-evaluation-responses', 'version': '1'}

Submitting evaluator generation job…
  status: queued
  status: in_progress
  status: completed
Generated rubric "custom-rubric-…" v1 with 6 dimensions:
  - factuality (weight 0.25)
  - tone (weight 0.15)
  - completeness (weight 0.20)
  - policy_citation (weight 0.15)
  - safety_disclaimers (weight 0.15)
  - hallucination_resistance (weight 0.10)

Eval created: eval_abc123…
Eval run created: evalrun_def456…
  status: queued
  status: in_progress
  status: completed

✓ Eval run completed.
Result counts: {'passed': 3, 'failed': 1, 'errored': 0, 'total': 4}
Report URL: https://ai.azure.com/.../evaluations/evalrun_def456…

Showing 3 of 4 output items:
(set EVAL_DEBUG=1 to also see the raw payload.)

  [1] Question: What's your return policy on hiking boots?
      Answer:   You can return unused boots within 60 days for a full refund.
    custom_rubric                 score=4.6    PASS
        rationale: Accurate policy, friendly tone, cites the 60-day window. Lacks an explicit policy link.
```

Open the **Report URL** in the Foundry portal to see every row, every
dimension's per-row score and rationale, and an aggregate chart.

> **Already shipping to users?** Also run
> [`evaluate_redteam.py`](./evaluate_redteam.py) — the Custom Rubric Evaluator grades
> *quality on what you asked for*; safety evaluators grade *whether the
> agent ever produces harmful content under adversarial input*. They're
> complementary, not redundant.

## If a score is low, what next?

A low score is information, not a verdict. Walk this checklist:

1. **Open the `report_url`** the script prints. The portal shows the
   evaluator's *rationale* per row — the words tell you more than the
   number.
2. **Read 3-5 failing rows in full.** Patterns emerge fast:
   * Same evaluator failing across many rows → the agent has a systemic
     weakness (e.g. always loses context after turn 2).
   * One row failing across many evaluators → that single input is hard
     (or the dataset row is malformed).
3. **Edit one thing at a time.** Change the agent's instructions in
   `agent.yaml`, re-deploy (`azd up`), re-run the eval, and
   compare `result_counts` to the previous run. If you change three
   things at once, you can't tell which one helped.
4. **Promote real failure cases into the dataset.** If a row failed and
   you can hand-correct the prompt or expected behavior, add it to the
   eval dataset so the next regression on that case is caught
   automatically.

If `result_counts.errored > 0`, the eval *itself* failed on those rows
(not the agent) — check the portal for the per-row error message
(rate limits, auth, missing fields, etc.).

## Cost and data usage

Most scripts in this folder cost only a small amount of model usage (a
handful of inference calls + an LLM judge per row). A few flows are
heavier — be deliberate before running them:

| Script | What it consumes | Heads-up |
|---|---|---|
| `evaluate_custom_rubric.py` ⭐ | One generation LRO + the same eval-run cost | **Primary path.** Generation is a multi-stage LLM job; budget a few minutes the first time. |
| `evaluate_redteam.py` | One agent call per adversarial prompt + judge | **Primary for user-facing agents.** ⚠ See the privacy callout below — adversarial prompts + agent responses are *logged*. |
| `evaluate_basic.py` | A few agent calls + a few judge calls | Cheapest. Useful as a sanity check; lower signal than a custom rubric for real projects. |
| `evaluate_multiturn_simulation.py` | Up to *N seeds × turns-per-conversation* agent calls + judge | Costs scale with how many seeds you load — start small. |
| `evaluate_multiturn_traces.py` | Judge calls **over existing traced conversations** — no live agent calls | Trim the trace time window or `agent_filter` to control judge cost and result volume. |
| `generate_dataset_*.py` | Generation LRO (service requires `max_samples ≥ 15`) + eval cost | Each run **registers a new dataset** in your project — clean up old ones in the portal if you iterate a lot. |
| `evaluate_scheduled.py` | One eval row **per new agent response** (event-triggered) | ⚠ **Continues running after the script exits.** Use the portal (or the delete snippet at the bottom of the script) to pause or remove the schedule when you're done. |

If you're on a sandbox project with cost alerts, set them up before
running the multi-turn / scheduled / red-team flows.

## Pick the right flow

**Primary recommended:** [`evaluate_custom_rubric.py`](./evaluate_custom_rubric.py) ⭐ for quality tailored to *your* agent, plus [`evaluate_redteam.py`](./evaluate_redteam.py) for safety. Pick additional flows from the table when you need them.

| You want to … | Use this script |
|---|---|
| **Get a tailored evaluator for your agent (primary)** ⭐ | [`evaluate_custom_rubric.py`](./evaluate_custom_rubric.py) |
| **Probe the agent against adversarial / red-team prompts (primary for user-facing agents)** | [`evaluate_redteam.py`](./evaluate_redteam.py) |
| Sanity-check end-to-end plumbing with generic built-in evaluators | [`evaluate_basic.py`](./evaluate_basic.py) |
| Evaluate multi-turn behavior **without** any existing dataset (the service generates conversations for you) | [`evaluate_multiturn_simulation.py`](./evaluate_multiturn_simulation.py) |
| Evaluate multi-turn behavior over **your own live traces** | [`evaluate_multiturn_traces.py`](./evaluate_multiturn_traces.py) |
| Turn recent agent **traces** into a reusable evaluation dataset | [`generate_dataset_from_traces.py`](./generate_dataset_from_traces.py) |
| Bootstrap an evaluation dataset from a few **topic seeds** | [`generate_dataset_synthetic.py`](./generate_dataset_synthetic.py) |
| Score every new agent response **continuously** (or on a schedule) | [`evaluate_scheduled.py`](./evaluate_scheduled.py) |

## The scripts

The list below is in *recommended exploration order*. The two primary
scripts (**custom rubric** + **red-team**) come first; everything else
is either a sanity-check, a multi-turn variant, or a supporting flow.

1. [`evaluate_custom_rubric.py`](./evaluate_custom_rubric.py) ⭐ — **primary
   recommended evaluator.** Generates a 5-7 dimension rubric tailored
   to *your* agent's job (tone, completeness, "did it cite a source?")
   from a short prompt, then evaluates against it. **Edit the prompt at
   the top of `submit_generation_job()` first** — the default is a
   generic placeholder. **Use this for any project you care about.**
2. [`evaluate_redteam.py`](./evaluate_redteam.py) — **primary safety
   evaluator.** Sends adversarial prompts (violence, self-harm, hate,
   sexual) and scores responses on the **0-7 severity** scale (higher
   is worse). Run this for any user-facing agent in addition to the
   custom rubric. ⚠ **Writes adversarial prompts + agent responses to
   your traces** — use a non-production project.
3. [`evaluate_basic.py`](./evaluate_basic.py) — four inline questions,
   built-in evaluators (`task_adherence`, `fluency`, `relevance`).
   Finishes in under a minute. Useful as an end-to-end **sanity check**
   that your project endpoint + agent + creds are wired up correctly;
   for real signal use `evaluate_custom_rubric.py`.
4. [`evaluate_multiturn_simulation.py`](./evaluate_multiturn_simulation.py) —
   Foundry simulates full multi-turn conversations from seed scenarios
   and scores each. **Run this before you have real traffic.**
5. [`evaluate_multiturn_traces.py`](./evaluate_multiturn_traces.py) —
   same four multi-turn evaluators, scored against **real traced
   conversations**. **Run this once you have traffic.**
6. [`generate_dataset_from_traces.py`](./generate_dataset_from_traces.py)
   — materializes recent traces into a registered, reusable dataset and
   evaluates the rows. Scores past production behavior; to re-run the
   same questions through the *current* agent, wrap the data source in
   `azure_ai_target_completions` (see `evaluate_basic.py`).
7. [`generate_dataset_synthetic.py`](./generate_dataset_synthetic.py) —
   bootstraps a domain-relevant dataset from short topic seeds when you
   have no traffic yet. **Default**: runs the generated questions through
   your deployed agent and scores the answers. Set
   `EVAL_AGAINST_DATASET_ONLY=true` to grade only the synthetic rows.
8. [`evaluate_scheduled.py`](./evaluate_scheduled.py) — scores every new
   agent response automatically (or every hour over recent traces with
   `EVAL_SCHEDULE_INTERVAL=1h`). ⚠ **The schedule keeps running after
   the script exits** — see "Cost and data usage" for cleanup.

## Prerequisites

1. **A deployed hosted agent.** This folder ships its own tiny demo agent
   (`main.py`, `agent.yaml`). Deploy it once with the `azd ai agent init`
   + `azd up` flow in "Your first run" above (the same pattern as every
   other Python sample in `samples/python/hosted-agents/`). The eval
   scripts target the deployed agent identified by `EVAL_AGENT_NAME`
   (default `agent-framework-agent-evaluation-responses`) and
   `EVAL_AGENT_VERSION` (default `1`) — change those env vars to evaluate
   any other deployed agent.
2. **A Foundry project endpoint** in `FOUNDRY_PROJECT_ENDPOINT`. After
   `azd up` finishes, copy the value from the deploy output or from the
   Foundry portal **Overview** page (form
   `https://<account>.services.ai.azure.com/api/projects/<project>`).
3. **AAD credentials** — `az login`, or any other source the
   `DefaultAzureCredential` chain understands.
4. **Python deps** — `pip install -r requirements.txt`.

### Tracing — and a privacy callout

This sample's agent sets `ENABLE_INSTRUMENTATION=true` and
`ENABLE_SENSITIVE_DATA=true` in [`agent.yaml`](./agent.yaml),
[`agent.manifest.yaml`](./agent.manifest.yaml), and
[`.env.example`](./.env.example). **Tracing** means the Foundry runtime
records every agent request, tool call, and response to Application
Insights (see [`08-observability/`](../08-observability/) for the full
story). The trace-based and continuous eval scripts
([`evaluate_multiturn_traces.py`](./evaluate_multiturn_traces.py),
[`generate_dataset_from_traces.py`](./generate_dataset_from_traces.py), and
[`evaluate_scheduled.py`](./evaluate_scheduled.py)) read those recordings
— turn tracing on once and every script in this folder works.

⚠ **`ENABLE_SENSITIVE_DATA=true` means user inputs, model prompts, and
model outputs (including any PII the user pasted) are written verbatim to
your Application Insights workspace.** That's necessary for trace-based
evaluation to score the *content*, but it also means your trace storage
is now a copy of every conversation. Keep this **off** in production
unless you have an explicit data-handling policy that allows it, and
treat the App Insights workspace as customer data. For non-production
demos this is usually fine; for anything customer-facing, decide
deliberately.

> The trace-based and continuous scripts also need actual **traffic** in
> the trace window (i.e. someone has to have called the agent recently)
> before they have anything to score.

If you're adapting the scripts for the
[`01-basic/`](../01-basic/) sample (which does **not** enable tracing by
default), copy the env-var pattern from
[`08-observability/`](../08-observability/) onto your `01-basic`
deployment first.

## Where to view results

Every script prints:

* the **eval ID** and **run ID** (use them to look up the run via
  the SDK), and
* a **report URL** that opens the run in the Foundry portal's
  [Evaluations](https://ai.azure.com/) page.

Trace-based and continuous flows additionally surface results on the
**Traces** page next to the original agent invocation — same UX as
[`08-observability/`](../08-observability/).

The per-row summary the scripts print is trimmed for readability. Set
`EVAL_DEBUG=1` before running any script to also see the raw payload.

## Related samples

* [`01-basic/`](../01-basic/) — also ships **multi-turn evaluation scripts**
  (simulation + traces) co-located with the basic agent for the multi-turn
  learning path. Same patterns as scripts 3-4 above, narrowed to the
  `01-basic` agent.
* [`08-observability/`](../08-observability/) — the canonical tracing
  sample. Trace-driven and continuous evaluation depend on the same
  `ENABLE_INSTRUMENTATION` / `ENABLE_SENSITIVE_DATA` pattern this sample
  turns on.

## Learn more

* [Azure AI Foundry — Evaluation overview](https://learn.microsoft.com/azure/ai-foundry/concepts/evaluation-approach-gen-ai)
* [Built-in evaluators reference](https://learn.microsoft.com/azure/ai-foundry/how-to/develop/evaluate-sdk)
* [Continuous evaluation in Foundry](https://learn.microsoft.com/azure/ai-foundry/how-to/develop/agent-evaluate-sdk)
* [Content-safety severity scale (0-7)](https://learn.microsoft.com/azure/ai-services/content-safety/concepts/harm-categories)

## For maintainers

* Scripts pin **API version `2025-11-15-preview`** in
  [`eval_common.py`](./eval_common.py). Bump in one place when GA lands.
* The evaluator-generation LRO, data-generation LRO, and continuous-eval
  configuration are preview surfaces; some are still exposed as raw REST
  in these scripts (via `requests` + a `DefaultAzureCredential` bearer
  token). When a typed Python surface ships, the calls will collapse to
  the typed client.
* For each script, the prerequisites block in the docstring spells out
  which Foundry resources must already exist (deployed agent, dataset,
  traces).
* Add new scripts by following the `evaluate_*` / `generate_dataset_*`
  naming pattern and wiring them into "Pick the right flow" + "The
  scripts" above.
