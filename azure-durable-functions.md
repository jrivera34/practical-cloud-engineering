# Azure Durable Functions

---

## What Problem Do They Solve?

Regular Azure Functions are stateless. They run once and die. If you need a multi-step pipeline (query data → export PDFs → email members → log results), a regular function can't coordinate that. You'd have to manage queues, track state, handle retries, and aggregate results yourself.

Durable Functions give you orchestration, state persistence, and parallel execution on top of stateless infrastructure. You write sequential Python code. The framework handles the distributed systems work underneath.

---

## The Three Components

Every Durable Function has three parts. All three are required.

**Client** — Starts the orchestration. Doesn't do any work itself. Just says "go."

```python
# Timer trigger that starts the orchestrator
@app.timer_trigger(arg_name="timer", schedule="0 0 14 25-31 * *")
@app.durable_client_input(client_name="client")
async def monthly_trigger(timer: func.TimerRequest, client):
    instance_id = await client.start_new("report_orchestrator")
```

**Orchestrator** — Coordinates the workflow. Calls activities, handles branching, tracks progress. This is the brain. It never does I/O directly.

```python
@app.orchestration_trigger(context_name="context")
def report_orchestrator(context: df.DurableOrchestrationContext):
    return orchestrator_function(context)
```

**Activities** — Do the actual work. Database queries, API calls, file uploads, emails. These are the hands.

```python
@app.activity_trigger(input_name="job")
def export_and_upload(job: dict):
    return _export_and_upload(job)
```

---

## How Registration Works

When the Function App starts, the runtime scans all decorators and builds a registry:

- `@app.orchestration_trigger` → registers an orchestrator
- `@app.activity_trigger` → registers an activity
- `@app.timer_trigger` → registers a timer function
- `@app.route` → registers an HTTP function
- `@app.durable_client_input` → injects a client to start orchestrations

The decorated function's **Python name** becomes its identifier. `context.call_activity("export_and_upload", job)` looks up the function named `export_and_upload` in the registry. The name in `call_activity` must match the decorated function name exactly. File names don't matter — they're convention, not a requirement.

---

## Replay: The Core Mechanism

The orchestrator doesn't stay alive in memory while activities run. That would waste resources and wouldn't survive crashes.

Instead, the framework uses **replay**:

1. Orchestrator runs, hits `yield context.call_activity(...)`.
2. Framework saves the orchestrator's state to Azure Storage, schedules the activity, and **kills the function**.
3. Activity runs on its own, finishes, result is saved to Azure Storage.
4. Framework **restarts the orchestrator from line 1**.
5. On replay, every `yield` that already completed returns its saved result instantly — no actual work.
6. Orchestrator fast-forwards to the next unfinished `yield`.

If there are 10 `yield` calls and you're on the 7th, the orchestrator has already run 7 times. Each time it replayed from the top, skipped past completed yields, and stopped at the next one.

**Why replay?** Python doesn't have a "resume from line 15" feature. The only way to get back to where you were is to re-run from the top and skip past completed work. The `yield` points are checkpoints. The saved results are how the framework knows what to skip.

---

## Determinism: The One Rule

Because the orchestrator replays from the top every time, it **must produce the same sequence of decisions on every run**. This is called determinism.

**Banned inside orchestrators:**

- `datetime.now()` — returns different values on each replay
- API calls — responses can change
- Database queries — data can change
- `random.random()` — different every time
- Any I/O whatsoever

**Safe inside orchestrators:**

- `context.current_utc_datetime` — frozen to when the orchestration first started, same across all replays
- `context.call_activity(...)` — delegates work to activities, results are saved
- Config values loaded at module level — constants for the life of the worker
- Pure logic (math, string manipulation, conditionals based on saved results)

**Why it matters:** If the orchestrator takes a different code path on replay, the sequence of `yield` calls won't match the saved history. The framework throws an exception and the orchestration fails.

---

## yield and context.call_activity

`context.call_activity("name", input)` doesn't run the activity immediately. It creates a task object. `yield` hands that task to the framework.

The framework looks at the task and decides:

- **"I don't have a result for this yet"** → schedule the activity, save state, shut down the orchestrator
- **"I already ran this on a previous replay"** → return the saved result instantly, keep going

```python
# This is a checkpoint. On first run, it schedules build_snapshot.
# On replay, it returns the saved result immediately.
snapshot = yield context.call_activity("build_snapshot", None)
```

---

## Fan-Out / Fan-In

Running multiple activities in parallel through a single `yield`:

```python
export_tasks = [
    context.call_activity("export_and_upload", job) for job in batch
]
export_results = yield context.task_all(export_tasks)
```

`task_all` schedules all activities simultaneously. The framework waits until **every** activity finishes, then replays the orchestrator **once**. The `yield` returns a list of all results.

5 exports that take 30 seconds each finish in ~30 seconds total instead of ~150 seconds sequentially.

**Critical:** If any activity in `task_all` throws an uncaught exception, the entire `task_all` fails. That's why `export_and_upload` catches errors internally and returns them as data in the dict, so `task_all` always succeeds and the orchestrator can handle failures per-job.

---

## Batching

Fan-out with rate limiting:

```python
batch_size = 5

for i in range(0, len(ready), batch_size):
    batch = ready[i : i + batch_size]
    export_tasks = [
        context.call_activity("export_and_upload", job) for job in batch
    ]
    export_results = yield context.task_all(export_tasks)
```

Why not `task_all` all 50 at once? API rate limits. Power BI's export API throttles concurrent requests. Batching to 5 keeps you under the ceiling. Also limits blast radius — if a batch fails, you lose 5 jobs, not 50.

---

## Activity Input: Always One Value

Activities receive a single input. Not two parameters, not keyword arguments. One value.

For multiple pieces of data, pass a dict:

```python
# Orchestrator
yield context.call_activity("get_sent", {"year": 2026, "month": 4})

# Activity registration in function_app.py
@app.activity_trigger(input_name="payload")
def get_sent(payload: dict):
    return _get_sent(payload["year"], payload["month"])
```

---

## Immutable Dict Pattern

Activities should not mutate their input dict. Use `{**dict}` to create a new dict with additional fields:

```python
# GOOD — creates a new dict, original untouched
return {**job, "sas_url": sas_url}

# BAD — mutates the original, can cause replay bugs
job["sas_url"] = sas_url
return job
```

Why: The orchestrator passes `job` into the activity. If the activity mutates it, the orchestrator's data changes from inside an activity. During replay, this can cause the orchestrator to see different data than the first run.

---

## Separation: Wiring vs Logic

Decorators live in `function_app.py`. Business logic lives in the activity modules. Thin wrappers connect them:

```python
# function_app.py — wiring
@app.activity_trigger(input_name="job")
def export_and_upload(job: dict):
    return _export_and_upload(job)

# export_and_upload.py — logic
def export_and_upload(job: dict) -> dict:
    # actual Power BI export, blob upload, SAS generation
    ...
```

Why: Activity code is plain Python. You can call it from a test script, a notebook, or another function without the Azure Functions framework. The decorators are the glue — they connect Azure's trigger system to your code. Swap the wiring without touching the logic. Test the logic without standing up the framework.

---

## Async vs Sync

Timer triggers and HTTP triggers are `async` because they `await` durable client calls. Activities are sync because they use synchronous libraries (httpx, pyodbc) and there's no benefit to async — each activity processes one job on its own thread.

```python
async def monthly_trigger(timer, client):    # async — awaits client.start_new()
def build_snapshot(payload):                  # sync — blocking I/O, one job
```

---

## Calling Functions Outside the Framework

Not everything needs to go through `context.call_activity`. A plain timer function can import and call activity code directly:

```python
from exodus.activities.log_results import get_monthly_run_count

@app.timer_trigger(arg_name="timer", schedule="0 0 14 2 * *")
def pipeline_health_check(timer: func.TimerRequest):
    count = get_monthly_run_count(year, month)
```

The activity trigger decorator registers a function with the Durable Functions framework. Without the decorator, it's just a Python function. You only need `context.call_activity` inside orchestrators where replay and state persistence matter.

---

## Quick Reference

| Concept | Rule |
|---|---|
| Orchestrator I/O | Never. All I/O goes in activities. |
| `datetime.now()` in orchestrator | Never. Use `context.current_utc_datetime`. |
| Mutating input dicts in activities | Don't. Use `{**dict, "key": val}`. |
| Activity input | Always one value. Use a dict for multiple fields. |
| `task_all` failure | One uncaught exception kills the whole batch. Catch inside activities. |
| Function names | `call_activity("name")` must match the decorated function name exactly. |
| Replay count | One replay per completed `yield`. 10 yields = 10 replays. |
| `task_all` replays | One replay for the entire batch, not per activity. |
