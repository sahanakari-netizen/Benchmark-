# AI Agent Benchmarking Platform

A framework-independent benchmarking platform built for the **"Build a
Benchmarking Platform for AI Agents, Not Just Models"** hackathon track.

Any agent — regardless of what framework, language, or model powers it
— can be benchmarked here as long as it exposes one HTTP endpoint
matching a small, documented contract. Two reference agents are built
in (so the platform demos with zero setup), and a standalone example
third-party agent shows the HTTP path working for real.

## What it measures

For every run: **task success** (keyword-based grading against the
agent's own final answer), **execution steps**, **tool calls**,
**token usage**, **cost**, and **wall-clock completion time** — timed
by the platform itself, never trusted from the agent. Repeating a run
`N` times and calling `/compare` aggregates all of this per agent.

## Fault injection

Each run can toggle:
- `tool_failure` — the first tool call an agent makes is forced to
  raise; a robust agent should retry and recover, a naive one won't.
- `model_unavailable` — simulates the agent's own model backend being
  down; a robust agent should degrade gracefully (e.g. fall back to
  raw tool output), a naive one has no fallback.
- `misleading_instructions` — the platform appends a misleading hint
  to the prompt sent to the agent; a robust agent should keep trusting
  its own verified tool output over the hint.

The two built-in reference agents (`robust_agent`, `naive_agent`) are
written to actually differ under these faults — see
`app/adapters/builtin_agents.py` — so a benchmark run produces a real,
interesting comparison, not just identical numbers.

## The agent contract (how to plug in your own agent)

`POST <your endpoint>` receives:
```json
{
  "task_id": "...", "prompt": "...", "available_tools": ["search"],
  "inject_tool_failure": false, "inject_model_unavailable": false,
  "inject_misleading_instructions": false
}
```
and must return:
```json
{
  "final_answer": "Paris",
  "steps": [{"type": "tool_call", "detail": "search(...) -> Paris"}],
  "tool_calls": 1, "tokens_used": 35, "cost_usd": 0.00014
}
```
See `examples/external_agent_example.py` for a complete, dependency-free
reference implementation you can run as its own process.

## Architecture

```
frontend/index.html             dashboard: register agents, add tasks, trigger runs, compare
app/main.py                      FastAPI routes (agents, tasks, runs, compare)
app/runner.py                    executes a batch of runs: resolves adapter, times it, grades it, stores the trace
app/adapters/base.py             the framework-independent contract (dataclasses)
app/adapters/http_adapter.py     calls an arbitrary external agent over HTTP
app/adapters/builtin_agents.py   robust_agent / naive_agent reference implementations
app/grading.py                   keyword-based success grading
app/seed.py                      seeds the two built-in agents + 5 demo tasks on startup
examples/external_agent_example.py  standalone 3rd-party agent demonstrating the HTTP contract
```

## Run it

```bash
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8002
```

Open `http://127.0.0.1:8002/` for the dashboard (interactive API docs
at `/docs`). `robust_agent`, `naive_agent`, and 5 demo tasks are seeded
automatically.

To see the HTTP contract work against a real external process:
```bash
uvicorn examples.external_agent_example:app --port 9001
# then, from the dashboard, register an agent with endpoint_url
# http://127.0.0.1:9001/invoke and run it against any task
```

## Demo script

1. From the dashboard, run `robust_agent` on "Capital of Japan" with
   **Inject tool failure** checked → passes (it retried).
2. Run `naive_agent` on the same task with the same fault → fails (no
   retry logic).
3. Run both on "Capital of Italy (robustness probe)" with **Inject
   misleading instructions** checked → `robust_agent` still passes,
   `naive_agent` gets talked out of its correct answer.
4. Click **Compare** to see the aggregated success rate / avg tokens /
   avg cost / avg duration side by side.
5. Start `examples/external_agent_example.py` on port 9001, register it
   as a new agent, and run it against a task to show the platform
   working against a genuinely separate, independently-written process.

## Tests

```bash
pytest tests/ -q
```

Covers: seeding, a clean successful run, the tool-failure fault
differentiating robust vs naive, the misleading-instructions fault
doing the same, the `/compare` aggregation, and registering an
external HTTP agent.
