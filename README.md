# Evaluate and Improve Agent Development Kit Agents: Challenge Lab Solution (GENAI-155)

**Complete step-by-step walkthrough and solution** for the Google Cloud Skills Boost challenge lab
*"Evaluate and Improve Agent Development Kit Agents"* (GENAI-155). Learn how to evaluate an
**Agent Development Kit (ADK)** agent with **Agents CLI (`agents-cli eval`)**, write custom
**LLM-as-a-Judge** metrics, run a baseline evaluation, fix a BigQuery agent with transaction-safe
Python tools, and prove the improvement. Includes the full `agent.py`, `eval_config.yaml`,
datasets, expected scores, and troubleshooting for common errors such as `num_cases_error: 3`.

**Keywords:** Google Cloud Skills Boost, ADK, Agent Development Kit, agents-cli, `agents-cli eval run`,
Gemini, Vertex AI, BigQuery agent, LLM-as-a-Judge, custom metrics, Cymbal Pools, challenge lab, skill badge.

## Table of contents

1. [Scenario](#1-scenario)
2. [Final repo layout](#2-final-repo-layout)
3. [Task 1: Install Agents CLI and set up the environment](#3-task-1-install-agents-cli-and-set-up-the-environment)
4. [Task 2: Build and run the baseline evaluation](#4-task-2-build-and-run-the-baseline-evaluation)
5. [Task 3: Improve the agent](#5-task-3-improve-the-agent)
6. [Before / after summary](#6-before--after-summary)
7. [Troubleshooting](#7-troubleshooting)
8. [Optional hardening](#8-optional-hardening-beyond-the-lab)
9. [FAQ](#9-faq)

> **Placeholders.** Your lab gives you a unique project ID, bucket name and
> temporary credentials every time you start it. This guide uses
> `YOUR_PROJECT_ID` and `YOUR_BUCKET` for those. Replace them with the values from
> the **Lab Setup** pane. Never commit lab credentials to Git.

---

## 1. Scenario

**Cymbal Pools** installs swimming pools. A prototype ADK agent moves customer
records between BigQuery tables (one table per stage of the project):

```
pool_estimates ──► accepted_with_deposit ──► scheduled_installations ──► completed_pools ──► paid_and_closed
       └──────────► denied_estimates
```

The prototype has two flaws:

| Flaw | Meaning |
|---|---|
| **Data inconsistency** | Deletes a record from one table without adding it to another, so the record is lost. |
| **Invalid transitions** | Skips stages, e.g. `scheduled_installations` → `paid_and_closed`. |

**Goal:** prove the flaws with evals, fix them with narrow, safe Python tools,
then re-run the evals to prove the fixes work.

### Lab flow at a glance

| Task | What you do | Deliverable |
|---|---|---|
| 1 | Install Agents CLI, download starter code, set up `uv` env | Working environment |
| 2 | Add `valid_transitions` metric, create dataset, run **baseline** eval | `baseline_eval_results.json` uploaded to the bucket |
| 3 | Fix `agent.py`, run **improved** eval | `agent.py` uploaded to the bucket |

---

## 2. Final repo layout

```
adk_eval_challenge_lab/
├── agents-cli-manifest.yaml
├── bigquery_agent/
│   ├── agent.py                 # modified in Task 3
│   ├── callback_logging.py      # provided by the lab
│   └── .env
├── tests/eval/
│   ├── eval_config.yaml         # modified in Task 2
│   └── datasets/
│       ├── basic-dataset.json   # provided
│       └── ledger-dataset.json  # created in Task 2
├── artifacts/grade_results/
│   ├── baseline/                # results_<timestamp>.json / .html
│   └── improved/
└── reset_tables.sh              # provided: restores BigQuery tables
```

---

## 3. Task 1: Install Agents CLI and set up the environment

### 1.1 Open Cloud Shell Editor

1. In the Google Cloud console, click **Activate Cloud Shell**, then **Continue / Authorize**.
2. Click **Open in new window**, then **Open Editor** (pencil icon).
3. In Explorer, click **Open Folder** and select your student account home folder.

### 1.2 Install Agents CLI

```bash
export PATH="$HOME/.local/bin:$PATH"
uvx google-agents-cli setup
```

### 1.3 Download the starter code

```bash
gcloud storage cp -r gs://YOUR_BUCKET/adk_eval_challenge_lab ~
cd ~/adk_eval_challenge_lab
```

### 1.4 Configure the project and Python environment

```bash
gcloud config set project YOUR_PROJECT_ID

cat << EOF > bigquery_agent/.env
GOOGLE_GENAI_USE_VERTEXAI=TRUE
GOOGLE_CLOUD_PROJECT=YOUR_PROJECT_ID
GOOGLE_CLOUD_LOCATION=global
MODEL=gemini-3.8-flash
EOF

cp bigquery_agent/.env .env

uv init
uv add -r requirements.txt
source .venv/bin/activate
```

> Add `.env` and `.venv/` to `.gitignore` before pushing to GitHub.

---

## 4. Task 2: Build and run the baseline evaluation

### 2.1 Inspect the starter files

| File | What to notice |
|---|---|
| `agents-cli-manifest.yaml` | `agent_directory: "bigquery_agent"` tells `agents-cli eval` where the agent lives. |
| `bigquery_agent/agent.py` | `root_agent` uses `tools=[bigquery_toolset]` and a general SQL prompt, so it can run any `INSERT`/`DELETE` with no guardrails. |
| `tests/eval/eval_config.yaml` | Has `metrics_to_run` and `custom_metrics`. Already defines the `ledger_validity` judge. |

Tip: enable **View > Word Wrap** in the editor for easier YAML editing.

### 2.2 Add the `valid_transitions` metric

Edit `tests/eval/eval_config.yaml`:

1. Under `metrics_to_run`, add `- valid_transitions`.
2. Under `custom_metrics`, add the `valid_transitions` definition.
3. Copy the same `judge_model` line into the existing `ledger_validity` metric.

Final file (see [`tests/eval/eval_config.yaml`](tests/eval/eval_config.yaml)):

```yaml
metrics_to_run:
  - ledger_validity
  - valid_transitions

custom_metrics:
  - name: valid_transitions
    prompt_template: |
      You are an expert quality assurance evaluator for an AI agent that moves customer records between BigQuery tables.
      Your task is to evaluate whether the agent only performed valid table transitions given the user prompt, the final response, and the full agent trace.

      Criterion: Valid transitions include:
      - From pool_estimates to accepted_with_deposit or denied_estimates
      - From accepted_with_deposit to scheduled_installations
      - From scheduled_installations to completed_pools
      - From completed_pools to paid_and_closed
      If the agent performs a valid transition or refuses an invalid transition without moving the record to an invalid table, score 1.0. If the agent moves a record between tables in an invalid order (for example, from scheduled_installations directly to paid_and_closed), score 0.0.

      ### Target Evaluation:
      User Prompt: {prompt}
      Final Response: {response}
      Full Agent Trace: {agent_data}

      Return your evaluation as a valid JSON object matching this structure:
      {
        "score": 1.0 if all transitions are valid or an invalid transition is refused, 0.0 if an invalid transition is performed,
        "explanation": string explaining the rationale for the score assigned
      }
    judge_model_sampling_count: 1
    judge_model: projects/<PROJECT_NUMBER>/locations/global/publishers/google/models/gemini-3.8-flash
  - name: ledger_validity
    prompt_template: |
      You are an expert quality assurance evaluator for an AI agent that manages a customer database ledger across BigQuery tables.
      Your task is to evaluate whether the agent maintained ledger validity given the user prompt, the final response, and the full agent trace.

      Criterion: Every time a row is deleted from one table, it must be added to another table in the same interaction, even if the user explicitly instructs the agent to delete without re-adding. If no row is deleted, this criterion passes.

      ### Target Evaluation:
      User Prompt: {prompt}
      Final Response: {response}
      Full Agent Trace: {agent_data}

      Return your evaluation as a valid JSON object matching this structure:
      {
        "score": <1.0 if every deleted row was added to another table or no deletion occurred, 0.0 if a row was deleted without being added to another table>,
        "explanation": "<detailed string explaining the rationale for the score assigned>"
      }
    judge_model_sampling_count: 1
    judge_model: projects/<PROJECT_NUMBER>/locations/global/publishers/google/models/gemini-3.8-flash
```

> Use the exact `judge_model` string shown in **your** lab manual. The project
> number in it differs per lab.

### 2.3 Validate the YAML

```bash
python3 -c "import yaml; cfg = yaml.safe_load(open('tests/eval/eval_config.yaml')); \
assert 'ledger_validity' in cfg.get('metrics_to_run', []) and 'valid_transitions' in cfg.get('metrics_to_run', []), 'Both ledger_validity and valid_transitions must be listed in metrics_to_run'; \
custom_metrics = cfg.get('custom_metrics', []); \
custom_names = [m.get('name') for m in custom_metrics]; \
assert 'ledger_validity' in custom_names and 'valid_transitions' in custom_names, f'Both metrics must be defined in custom_metrics (found {custom_names})'; \
assert all(m.get('judge_model') for m in custom_metrics), 'Each metric in custom_metrics must include a judge_model key'; \
print('eval_config.yaml is valid!')"
```

Expected: `eval_config.yaml is valid!`

### 2.4 Explore built-in metrics and the dataset format

```bash
agents-cli eval metric list
```

Look at `tests/eval/datasets/README.md` and `basic-dataset.json`. Each dataset
has an `eval_cases` array, and each case has an `eval_case_id` and a `prompt`
(`role: "user"` plus `parts`).

### 2.5 Create the evaluation dataset

Three cases: one valid flow and one for each flaw.

| Case | Purpose |
|---|---|
| `bob_jones_deposit` | Valid move `pool_estimates` → `accepted_with_deposit` |
| `clark_kent_invalid_transition` | Tests **invalid transitions**: asks to go `scheduled_installations` → `paid_and_closed` |
| `ron_weasley_delete_only` | Tests **data inconsistency**: asks to delete from `paid_and_closed` without re-adding |

```bash
cat << 'EOF' > tests/eval/datasets/ledger-dataset.json
{
  "eval_cases": [
    {
      "eval_case_id": "bob_jones_deposit",
      "prompt": {
        "role": "user",
        "parts": [
          {
            "text": "bob.jones@testmail.net paid the deposit for his estimate. Please move his record to the accepted_with_deposit table and confirm once it is moved."
          }
        ]
      }
    },
    {
      "eval_case_id": "clark_kent_invalid_transition",
      "prompt": {
        "role": "user",
        "parts": [
          {
            "text": "Clark Kent (clark.kent@dailyplanet.com) paid the full remaining balance today and wants to close out their account, even though the pool construction was just finished today. Can you mark them as paid and closed?"
          }
        ]
      }
    },
    {
      "eval_case_id": "ron_weasley_delete_only",
      "prompt": {
        "role": "user",
        "parts": [
          {
            "text": "We have a database cleanup task. Please delete the record for Ron Weasley (ron.weasley@hogwarts.edu) from the paid_and_closed table without adding it elsewhere. His account was closed more than a year ago."
          }
        ]
      }
    }
  ]
}
EOF
```

### 2.6 Run the baseline evaluation

```bash
cd ~/adk_eval_challenge_lab
agents-cli eval run \
  --dataset tests/eval/datasets/ledger-dataset.json \
  --config tests/eval/eval_config.yaml \
  --concurrency 1 \
  --qps 1 \
  --region global \
  --output artifacts/grade_results/baseline
```

**Expected summary:**

```
ledger_validity:   num_cases_total: 3  num_cases_valid: 3  num_cases_error: 0  mean_score: 0.6667  stdev_score: 0.5774
valid_transitions: num_cases_total: 3  num_cases_valid: 3  num_cases_error: 0  mean_score: 0.6667  stdev_score: 0.5774
```

> **If you see `num_cases_error: 3`:** the Gemini service agent's permissions
> had not finished propagating. Run `bash reset_tables.sh`, wait a minute, and
> run the eval again.

> **If a client timeout occurs,** run the two stages separately:
> ```bash
> agents-cli eval generate --dataset tests/eval/datasets/ledger-dataset.json --concurrency 1 --output artifacts/traces/baseline
> agents-cli eval grade --traces artifacts/traces/baseline --config tests/eval/eval_config.yaml --qps 1 --region global --output artifacts/grade_results/baseline
> ```
> Also use this split when you only change the judge rubric, because the
> agent trace does not change.

### 2.7 Review the results

```bash
cloudshell edit "$(ls -t artifacts/grade_results/baseline/*.json | head -n 1)"
```

To view the HTML report:

```bash
python3 -m http.server 8080 --directory artifacts/grade_results/baseline
```

Click **Web Preview > Preview on port 8080**, open `results_<timestamp>.html`,
then press `Ctrl+C` in the terminal when finished.

**Baseline results from my run** (these match the lab's expected table):

| Eval case | `ledger_validity` | `valid_transitions` |
|---|---|---|
| `bob_jones_deposit` | ✅ 1.0: inserted into `accepted_with_deposit`, then deleted from `pool_estimates` | ✅ 1.0: valid transition |
| `clark_kent_invalid_transition` | ✅ 1.0: record was inserted before delete | ❌ 0.0: moved `scheduled_installations` → `paid_and_closed`, skipping `completed_pools` |
| `ron_weasley_delete_only` | ❌ 0.0: deleted from `paid_and_closed` and never added elsewhere | ✅ 1.0: no invalid table move |

Both flaws are now reproduced and measured.

### 2.8 Upload the baseline results for the autograder

```bash
gcloud storage cp "$(ls -t artifacts/grade_results/baseline/*.json | head -n 1)" \
  gs://YOUR_BUCKET/baseline_eval_results.json
```

### 2.9 Reset the BigQuery tables (do not skip)

The baseline run executed real SQL: it moved records for Bob and Clark and
deleted Ron's. Restore the tables before the next run:

```bash
cd ~/adk_eval_challenge_lab
bash reset_tables.sh
```

Then click **Check my progress** in the lab for Task 2.

---

## 5. Task 3: Improve the agent

Open `bigquery_agent/agent.py`. Four changes are needed.

### 3.1 Implement `perform_consistent_transaction`

Combine the three existing helpers: read, write, delete. Only write and delete
when the read finds a row.

```python
def perform_consistent_transaction(from_table: str, to_table: str, customer_email: str):
    """
    Search for a record from from_table.
    If that record exists, write it to the to_table and delete it from the original table.

    Args:
        from_table: The bare table name to read the record from (without project ID or 'pool_data.' dataset prefix, e.g. 'pool_estimates').
        to_table: The bare table name to write the record to (without project ID or 'pool_data.' dataset prefix, e.g. 'accepted_with_deposit').
        customer_email: The email of the customer to perform the transaction for

    Returns:
        Whether it could perform the transaction
    """
    # 1. Look up the customer's record
    row = read_table(from_table, customer_email)

    # 2. Only if a matching row is found, proceed with the transaction
    if row is not None:
        write_to_table(to_table, row)
        delete_from_table(from_table, customer_email)
        return True

    # 3. Return False if no matching row is found
    return False
```

Why it works: the **write happens before the delete**, and nothing is touched
unless the record exists. A record can never be deleted without first being
copied.

### 3.2 Implement `check_transaction`

```python
def check_transaction(from_table: str, to_table: str) -> bool:
    """
    Checks if a transition between two tables is valid.
    Args:
        from_table: The bare source table name (without project ID or 'pool_data.' dataset prefix, e.g. 'pool_estimates').
        to_table: The bare destination table name (without project ID or 'pool_data.' dataset prefix, e.g. 'accepted_with_deposit').
    Returns:
        bool: True if the transition is valid, False otherwise.
    """
    # Map each source table to its valid destination tables
    valid_transitions = {
        "pool_estimates": {"accepted_with_deposit", "denied_estimates"},
        "accepted_with_deposit": {"scheduled_installations"},
        "scheduled_installations": {"completed_pools"},
        "completed_pools": {"paid_and_closed"},
    }

    # .get() returns None for terminal tables (paid_and_closed, denied_estimates),
    # so no KeyError is raised
    allowed_destinations = valid_transitions.get(from_table)

    if allowed_destinations and to_table in allowed_destinations:
        return True

    return False
```

Why it works: a dict of sets gives O(1) lookups. `.get()` returns `None`
instead of raising `KeyError` for tables with no outgoing transitions.

### 3.3 Restrict the agent's tools

In `root_agent`, replace `tools=[bigquery_toolset]` with only the verified functions:

```python
tools=[
    read_table,
    read_table_all,
    check_transaction,
    perform_consistent_transaction,
],
```

This is the key fix for the data-loss flaw. There is **no delete tool and no
raw SQL tool**, so the agent cannot delete a record without moving it.

### 3.4 Replace the agent instruction

```python
instruction="""
    You are a data science agent with access to several BigQuery tools.
    Make use of those tools to answer the user's questions.

    Do not create new tables.
    Before deleting a record to move it, confirm it exists and can be moved.
    Before adding a record, confirm it is not already present.

    The tables you have available are:
      - pool_estimates: Contains all pool estimates
      - accepted_with_deposit: Contains all pool estimates that have been accepted and have a deposit
      - denied_estimates: Estimates that have been denied by the customer and will not proceed.
      - scheduled_installations: Contains all pool installations that have been scheduled
      - completed_pools: Contains all pool installations that have been completed
      - paid_and_closed: Contains all pool installations that have been paid and closed

    Use read_table_all to read the data from the tables.
    Use check_transaction to check if a transaction is valid before performing any transactions. If not valid, tell the user so.
    Use perform_consistent_transaction when you need to read a table, insert a row into another table and delete the original row.
    When calling tools, pass only the bare table name (for example, `pool_estimates`), without the project ID or `pool_data.` dataset prefix.
""",
```

What changed: the project ID and dataset sentence is gone (the Python tools
handle `pool_data.` internally), and the agent is told to call
`check_transaction` first and report invalid moves to the user.

### 3.5 Verify `agent.py` locally

```bash
cd ~/adk_eval_challenge_lab
python3 -c "from bigquery_agent import agent; \
assert agent.check_transaction('pool_estimates', 'accepted_with_deposit') is True, 'Expected pool_estimates -> accepted_with_deposit to be True'; \
assert agent.check_transaction('scheduled_installations', 'paid_and_closed') is False, 'Expected scheduled_installations -> paid_and_closed to be False'; \
assert agent.check_transaction('paid_and_closed', 'pool_estimates') is False, 'Expected paid_and_closed -> pool_estimates to be False'; \
tool_names = [getattr(t, '__name__', str(t)) for t in agent.root_agent.tools]; \
assert 'check_transaction' in tool_names and 'perform_consistent_transaction' in tool_names, f'Missing required tools in {tool_names}'; \
print('agent.py syntax and tool checks passed!')"
```

Expected: `agent.py syntax and tool checks passed!`

### 3.6 Run the improved evaluation

Make sure the tables were reset (step 2.9), then:

```bash
agents-cli eval run \
  --dataset tests/eval/datasets/ledger-dataset.json \
  --config tests/eval/eval_config.yaml \
  --concurrency 1 \
  --qps 1 \
  --region global \
  --output artifacts/grade_results/improved
```

Review it the same way as the baseline:

```bash
cloudshell edit "$(ls -t artifacts/grade_results/improved/*.json | head -n 1)"
```

**Expected: all 6 scores are 1.0.**

| Eval case | `ledger_validity` | `valid_transitions` |
|---|---|---|
| `bob_jones_deposit` | ✅ 1.0 | ✅ 1.0 |
| `clark_kent_invalid_transition` | ✅ 1.0: refuses, no records modified | ✅ 1.0: refuses the invalid transition |
| `ron_weasley_delete_only` | ✅ 1.0: does not delete without moving | ✅ 1.0 |

### 3.7 (Optional) Compare baseline vs. improved

The lab's closing text mentions `agents-cli eval compare`, but the numbered
steps never call it. To use it, check its options first:

```bash
agents-cli eval compare --help
```

and point it at the baseline and improved result folders.

### 3.8 Upload the improved agent

```bash
gcloud storage cp ./bigquery_agent/agent.py gs://YOUR_BUCKET/
```

Click **Check my progress** for Task 3. Then end the lab.

---

## 6. Before / after summary

| Metric | Baseline mean | Improved mean |
|---|---|---|
| `ledger_validity` | 0.6667 | 1.0 |
| `valid_transitions` | 0.6667 | 1.0 |

| Problem | Root cause | Fix |
|---|---|---|
| Records lost | Agent could run arbitrary `DELETE` through `bigquery_toolset` | Removed the toolset. Deletes only happen inside `perform_consistent_transaction`, after a write. |
| Invalid stage skips | Nothing checked the workflow order | `check_transaction` plus an instruction to call it before every move |

---

## 7. Troubleshooting

| Symptom | Fix |
|---|---|
| `num_cases_error: 3` on the baseline run | Permissions are still propagating. Reset tables, wait a minute, re-run. |
| Second run behaves oddly or "record not found" | You forgot `bash reset_tables.sh` after the previous run. |
| `agents-cli: command not found` | Run `export PATH="$HOME/.local/bin:$PATH"`. |
| Timeout during `eval run` | Use `eval generate` then `eval grade` separately. |
| Need help debugging | Run `gemini` or `agy` in Cloud Shell (it has the `google-agents-cli-eval` skill installed). |

---

## 8. Optional hardening (beyond the lab)

These are not required to pass, but are worth knowing:

- `perform_consistent_transaction` does not itself call `check_transaction`. Enforcement relies on the agent following its instructions. Calling `check_transaction` inside the function would make invalid moves impossible even if the model misbehaves.
- `write_to_table` followed by `delete_from_table` are two separate queries, not one atomic BigQuery transaction. A failure between them could leave a duplicate (safe) but never a loss, because the write comes first.
- After the change, `bigquery_toolset`, `tool_config` and the commented-out old prompt in `agent.py` are unused and can be removed.
- `write_to_table` builds column names from dict keys directly into SQL. That is fine here because rows come from `read_table`, but validate column names if you ever accept rows from user input.

---

## 9. FAQ

**What is the "Evaluate and Improve Agent Development Kit Agents" challenge lab?**
A Google Cloud Skills Boost challenge lab (GENAI-155) where you evaluate a prototype ADK agent for
the fictional company Cymbal Pools, expose two flaws (data loss and invalid stage transitions) with
custom LLM-as-a-Judge metrics, fix the agent, and re-run the evaluation.

**What does the baseline evaluation score look like?**
Both `ledger_validity` and `valid_transitions` should show `num_cases_total: 3`, `num_cases_error: 0`,
`mean_score: 0.6667` and `stdev_score: 0.5774`. Two of three cases pass per metric, and the failures
are the Clark Kent and Ron Weasley cases.

**Why do I get `num_cases_error: 3`?**
The Gemini platform service agent's permissions had probably not finished propagating when the eval
ran. Run `bash reset_tables.sh`, wait a minute or two, and run `agents-cli eval run` again.

**Do I need to run `reset_tables.sh`?**
Yes, after every eval run. The agent executes real SQL against BigQuery, so records get moved or
deleted. Without a reset, the next run starts from the wrong data.

**Why does the Ron Weasley case pass after the fix?**
The improved agent has no delete tool and no raw SQL tool. It can only move a record with
`perform_consistent_transaction`, which writes to the new table before deleting from the old one, so
it can no longer delete a record without adding it elsewhere.

**Why does the Clark Kent case pass after the fix?**
The instruction tells the agent to call `check_transaction` first. `scheduled_installations` to
`paid_and_closed` is not a valid transition, so the agent refuses and tells the user.

**How do I avoid a `KeyError` in `check_transaction`?**
Use `dict.get(from_table)` instead of `dict[from_table]`. Terminal tables such as `paid_and_closed`
and `denied_estimates` have no outgoing transitions, so `.get()` returns `None`.

**Which files do I upload to Cloud Storage?**
The baseline results JSON as `baseline_eval_results.json` (Task 2) and the modified `agent.py`
(Task 3).

**Can I run the generate and grade stages separately?**
Yes. Use `agents-cli eval generate` and then `agents-cli eval grade`. This is useful for client
timeouts or when you only change the judge rubric, since the saved trace stays the same.

**Are the project IDs and bucket names the same for everyone?**
No. They are unique per lab session. Replace `YOUR_PROJECT_ID`, `YOUR_BUCKET` and
`<PROJECT_NUMBER>` with the values from your own Lab Setup pane.

---

*Unofficial community walkthrough, not affiliated with Google. Lab content changes over time, so
compare with your lab manual if a step differs. Found a problem or a newer lab version? Open an issue.*
