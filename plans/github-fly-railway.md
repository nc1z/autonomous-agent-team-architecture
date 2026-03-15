# Plan: GitHub + Fly.io / Railway

| Layer | Choice |
|-------|--------|
| Event system | GitHub (issues, PRs) |
| Runner | Small app on Fly.io or Railway |
| LLM worker | Any HTTP/CLI coding API |

Managed "daemon" without managing a raw VPS.

---

## Prerequisites

- GitHub repo; PAT with `repo` scope
- Fly.io or Railway account
- LLM worker reachable via HTTP (Codex API or another coding API)

## Steps

1. **Build a small app** (e.g. Python/Node) that implements the orchestrator loop: poll GitHub (issues/PRs), dispatch to PM/Dev/QA logic, call LLM via HTTP, push results back to GitHub (issues, PRs, comments).
2. **No Codex CLI on the runner** — use **Codex API** (or other HTTP API) from the app. The app sends prompts and repo context (e.g. file contents, diffs) in requests and applies suggested edits via Git in the app (clone in temp dir, apply patches, push). So the "dev" agent is: fetch issue → call API with "implement this" + context → receive diff or instructions → apply in clone → commit and push → open PR.
3. **Deploy to Fly.io or Railway** — long-running process or cron-style worker. Fly: `fly launch`, then `fly scale count 1` and run the orchestrator as the main process. Railway: deploy a worker that runs the same loop.
4. **Secrets:** Set `GITHUB_TOKEN`, `CODEX_API_KEY` (or equivalent) in Fly/Railway secrets. Clone the repo in the worker (over HTTPS with token) or use GitHub API only and no local clone (more API-heavy but possible).
5. **Clone vs API-only:** If the app clones the repo, it needs enough disk and Git; then it can run "dev" by applying API-suggested changes. If API-only, "dev" must drive everything via GitHub API (create branch, create/update files via API, create PR) — more complex but doable.

## Tradeoffs

- **Pros:** No VPS SSH; managed runtime; scales with the platform.
- **Cons:** Need to implement "apply API response to repo" (patch or file edits); cold starts and resource limits per plan.

## Done when

- The app runs on Fly or Railway, polls GitHub, runs PM/Dev/QA using an HTTP coding API, and you approve issues and merge PRs from GitHub.
