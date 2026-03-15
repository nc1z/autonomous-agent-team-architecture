# Plan: GitLab + self-hosted runner

| Layer | Choice |
|-------|--------|
| Event system | GitLab (issues, MRs) |
| Runner | GitLab CI + self-hosted runner |
| LLM worker | Any CLI/API |

Same idea as GitHub + Actions, with GitLab semantics (MRs instead of PRs, GitLab API).

---

## Prerequisites

- GitLab repo (or GitLab.com); project with Issues and Merge Requests enabled
- GitLab CI/CD; a self-hosted runner registered to the project (or group)
- LLM worker (Codex CLI or API) available from the runner (e.g. installed on runner host or callable via API)

## Steps

1. **Pipeline config `.gitlab-ci.yml`** — jobs triggered by:
   - **Schedule:** pipeline schedule (e.g. every 15 min) runs "orchestrator" stages: PM → Dev → QA.
   - **Rules:** optional `on: merge_request` for QA job; `on: issue` (if supported) or schedule-only for PM/Dev.
2. **Orchestrator logic** in repo (script or small app): use GitLab API for issues (list by label, create, add label), merge requests (create MR, get diff, post comments). Labels: e.g. `proposed`, `approved`, `qa-done`.
3. **Runner:** Self-hosted so you can install Codex CLI or run Docker with Codex; no job time limit from GitLab.com's shared runners (you control the runner).
4. **Secrets:** `GITLAB_TOKEN` (or CI variable), `CODEX_API_KEY` (or CLI on runner); stored as CI/CD variables (masked).
5. **State:** GitLab issues and MRs are the only state; each job is stateless; "in progress" can be a label or issue assignee.

## Differences from GitHub

- **API:** GitLab REST API for issues, MRs, notes (comments), labels.
- **MR = PR:** Create MR from branch; get diff; post "merge request notes" or use the review API if available.
- **Pipeline triggers:** Schedules and MR/issue events; no "issue labeled" webhook in the same way — use schedule + "list issues with label approved" or GitLab triggers.

## Done when

- Scheduled pipelines (and optionally MR-triggered QA) run PM/Dev/QA against GitLab issues and MRs; you approve issues and merge MRs; runner has access to Codex CLI or API.
