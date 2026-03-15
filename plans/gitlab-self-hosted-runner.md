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

## What You Build

**Orchestrator** (pipeline stages or script), **PM / Dev / QA** using GitLab API (issues, MRs, notes). Repo: script or app in repo; `prompts/`; Codex CLI or API on runner. GitLab: labels `proposed`, `approved`, `qa-done`; protect default branch; `GITLAB_TOKEN` and `CODEX_API_KEY` in CI variables.

## Implementation Phases

| Phase | What to do |
|-------|------------|
| 1. GitLab client | List issues by label, create issue, create branch, create MR, get diff, post MR notes (review). |
| 2. Config | Repo path from CI; token and API key from variables; schedule interval. |
| 3. PM agent | Job: read repo context; call Codex; create issues with `proposed`. |
| 4. Dev agent | Job: list approved issues; pick one; branch; Codex; push; open MR. |
| 5. QA agent | Job: list open MRs; get diff; Codex review; post notes; add `qa-done`. |
| 6. Orchestrator | Schedule runs PM → Dev → QA; or separate jobs by trigger. |
| 7. Safety | Branch protection; CI gate; Codex in Docker on runner if desired. |

## Checklist Before Going "Live"

- [ ] CI variables set; labels and protected branch on GitLab.
- [ ] Runner has Codex CLI or can call API.

## Done when

- Scheduled pipelines (and optionally MR-triggered QA) run PM/Dev/QA against GitLab issues and MRs; you approve issues and merge MRs; runner has access to Codex CLI or API.
