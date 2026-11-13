# App Factory

Write a Linear ticket describing a web app, and AI agents spec it, build it, test it, and deploy it.

## How it works

1. Moving a Linear ticket to **In Spec** sends a webhook to a FastAPI server (`orchestrator/api.py`).
2. The server calls `run_pipeline` (`orchestrator/pipeline.py`), which creates a GitHub repo, a Vercel project, and a Supabase database for the app.
3. It then runs a LangGraph graph. Each node starts a Claude Code agent through the Claude Agent SDK (`run_agent` in `orchestrator/agent_runner.py`).
4. Each agent loads one skill file from `.claude/skills/` that tells it what its job is.
5. All agents share one markdown memory file per ticket (`memory/LIN-xxx.md`). Each agent reads the whole file, then appends its own section.

## The pipeline

```
pm_agent -> gate_1 -> architect_agent -> gate_2 -> decompose -> dev_parallel
         -> review_agent + test_agent (in parallel) -> gate_3 -> deploy_agent -> done_handler
```

| Node | What it does |
|------|--------------|
| `pm_agent` | Turns the ticket into a detailed spec |
| `gate_1` | Pauses for a human to approve the spec |
| `architect_agent` | Picks the tech stack and splits the work into 3-6 subtasks |
| `gate_2` | Pauses for a human to approve the plan |
| `decompose` | Parses the subtasks out of the architect's plan |
| `dev_parallel` | Runs one dev agent per subtask at the same time on one branch, then opens a single PR |
| `review_agent` | Reviews the PR |
| `test_agent` | Writes and runs tests on the PR |
| `gate_3` | Pauses for a human to approve the QA results |
| `deploy_agent` | Deploys the frontend to Vercel and the database to Supabase |
| `done_handler` | Posts a final summary with the repo and deploy links |

If any step fails or a human rejects it, the graph goes to `blocked_handler` and the ticket moves to **Blocked**.

## Human gates

- Each gate calls LangGraph's `interrupt()`, which pauses the graph.
- The graph's state is saved to SQLite, so the pipeline can wait as long as needed.
- Moving the ticket to the next Linear state (**In Arch**, **In Dev**, **In Deploy**) sends another webhook, which resumes the graph with `Command(resume=...)`.
- Moving the ticket to **Blocked** rejects it.

## Tracking

- Linear: each ticket gets one sub-issue per stage, and every step posts a comment on the ticket.
- Slack: posts a message at each gate and on errors.
- LangSmith: traces every agent run and API call.
- `audit/`: an append-only log of every event.

## Built with

Python, FastAPI, LangGraph, Claude Agent SDK, Linear, GitHub, Vercel, Supabase, Slack, LangSmith, Docker

## Run it

```bash
cp .env.example .env   # add your API keys
docker compose up
```

Then point a Linear webhook at `/webhook/linear` and move a ticket to **In Spec**.

## Files

```
orchestrator/
  api.py             # FastAPI webhook endpoint
  pipeline.py        # Starts or resumes the pipeline, creates the app's repo
  graph.py           # LangGraph graph
  agent_runner.py    # Runs a Claude Code agent with a skill file
  memory.py          # Memory file read and append
  linear.py          # Linear API calls
  slack.py           # Slack messages
  nodes/
    agents.py        # PM, architect, review, test, deploy nodes
    dev.py           # Subtask parsing and parallel dev agents
    gates.py         # Human approval gates
    terminal.py      # Done and blocked handlers
.claude/skills/      # One skill file per agent
memory/              # One shared memory file per ticket
```
