# Plan: GitHub + local laptop + Codex CLI

| Layer | Choice |
|-------|--------|
| Event system | GitHub (issues, PRs) |
| Runner | Same daemon on your machine |
| LLM worker | OpenAI Codex CLI |

Same as VPS; runs whenever the laptop is on (e.g. overnight).

---

## Prerequisites

- GitHub repo; PAT with `repo` scope
- Laptop with Git, Python 3, Codex CLI installed
- Clone of the repo locally

## Steps

1. **Clone repo** and install orchestrator deps (`pip install -r orchestrator/requirements.txt`).
2. **Set `GITHUB_TOKEN`** in your shell or env; never commit it.
3. **Run orchestrator** from repo root: `python orchestrator/orchestrator.py`.
4. **Keep laptop on** when you want the fleet active (e.g. overnight); optionally disable sleep or use `caffeinate` (macOS).
5. **Approve issues** (label `approved`) and **merge PRs** yourself; agents never merge.

## Tradeoffs

- **Pros:** No VPS cost; same control as VPS; simple.
- **Cons:** Only runs when the laptop is on and awake.

## Full implementation plan

See [../PLAN.md](../PLAN.md) for the full step-by-step (GitHub client, config, PM/Dev/QA agents, orchestrator loop, safety).

## Done when

- Running the orchestrator locally creates proposed issues, implements approved ones via PRs, and runs QA; you gate by approving issues and merging PRs.
