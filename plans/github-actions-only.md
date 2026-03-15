# Plan: GitHub Actions only

| Layer | Choice |
|-------|--------|
| Event system | GitHub |
| Runner | Actions on `schedule` + `issues` / `pull_request` |
| LLM worker | Codex API or another API |

No VPS or laptop daemon; constrained by Actions limits and timeouts.

---

## Prerequisites

- GitHub repo; PAT or GitHub App for API access (Actions has `GITHUB_TOKEN` with repo scope in workflow)
- Codex API (or other HTTP coding API) callable from Actions; API key in repo secrets

## Steps

1. **Workflows in `.github/workflows/`** — e.g.:
   - `schedule.yml`: cron (e.g. `*/15 * * * *`) runs "orchestrator" job: call PM logic, then Dev, then QA (or split into separate scheduled jobs).
   - Optional: `on: issues.labeled` to run Dev when `approved` is added; `on: pull_request` to run QA when PR is opened/updated.
2. **Orchestrator as a script** in the repo (e.g. `orchestrator/` or inline in workflow); each job checks out repo, installs deps, runs one "phase" (PM / Dev / QA) using GitHub API and Codex API.
3. **Secrets:** `CODEX_API_KEY` (or equivalent), optionally `GITHUB_TOKEN` (provided by Actions) or a PAT with broader scope.
4. **Persistence:** Actions are stateless; use GitHub (issues, PRs, labels) as the only state. No long-running process; each run is a fresh job.

## Limits to be aware of

- **Time:** Job timeout (e.g. 6 hours on private, 6 hours on public); Codex may need multiple calls — keep each job focused.
- **Concurrency:** Avoid multiple jobs fighting over the same issue (e.g. use "claim" via label `in-progress` and only one scheduled job per cycle).
- **Secrets:** No persistent disk; only GitHub and API calls.

## Done when

- Scheduled workflow(s) propose issues (PM), open PRs for approved issues (Dev), and review PRs (QA); you approve issues and merge PRs; no VPS or local daemon.
