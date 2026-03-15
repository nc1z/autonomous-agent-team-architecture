# Plan: Custom event bus

| Layer | Choice |
|-------|--------|
| Event system | Your own queue (e.g. SQS, Redis, RabbitMQ) |
| Runner | Workers consuming queue |
| LLM worker | Any |

Maximum flexibility; you define events and triggers.

---

## Prerequisites

- A queue or message bus (e.g. AWS SQS, Redis Streams, RabbitMQ, Google Cloud Tasks)
- Workers (e.g. processes on a VM, Lambda, or Kubernetes) that consume messages
- Some way to produce events (e.g. GitHub webhooks → your API → enqueue; or cron → enqueue "tick")

## Steps

1. **Define events** — e.g. `issue.approved`, `pr.opened`, `tick` (schedule). Events carry minimal payload: repo id, issue/PR id, etc.
2. **Produce events:**
   - **Webhooks:** GitHub (or GitLab) webhook hits your endpoint; you push `issue.labeled` → `issue.approved` to the queue when label is `approved`. Similarly `pull_request.opened` → `pr.opened`.
   - **Schedule:** Cron or Cloud Scheduler sends `tick` to queue every N minutes; worker runs "orchestrator" logic (PM, then Dev, then QA) or pulls one task per tick.
3. **Workers** subscribe to queue; for each message, run the right agent (PM / Dev / QA) using repo + GitHub API + LLM. Push results back to GitHub (create issue, open PR, post review). Optionally enqueue follow-up events (e.g. `pr.opened` → QA worker).
4. **No dependency on GitHub's UI for "state"** — your source of truth can be the queue + GitHub, or you add a small DB for "claimed" issues to avoid duplicate work.
5. **Scale:** Multiple workers for different queues (e.g. `dev-tasks` vs `qa-tasks`); use message visibility timeout or locking so one worker handles one issue/PR at a time.

## Tradeoffs

- **Pros:** Decouple from GitHub's event model; run anywhere (AWS, GCP, on-prem); mix webhooks and schedule; scale workers independently.
- **Cons:** More moving parts; you own webhook endpoint, queue, and worker deployment.

## What You Build

**Producers:** webhook handler and/or cron that enqueue events (`issue.approved`, `pr.opened`, `tick`). **Workers:** one or more processes that consume events and run **PM / Dev / QA** logic using GitHub API and LLM. Repo: orchestrator logic, `prompts/`, GitHub client; deploy workers and (optionally) webhook endpoint. GitHub: labels `proposed`, `approved`, `qa-done`; branch protection; token for API.

## Implementation Phases

| Phase | What to do |
|-------|------------|
| 1. GitHub (or Git) client | Same as other plans; workers use it to read/write issues, PRs, comments. |
| 2. Config | Queue connection; token; repo owner/name; worker concurrency. |
| 3. PM agent | Triggered by `tick` or manual event; call LLM; create issues with `proposed`. |
| 4. Dev agent | Triggered by `issue.approved` or `tick`; claim issue; call LLM; push and open PR. |
| 5. QA agent | Triggered by `pr.opened` or `tick`; get diff; call LLM; post review; add `qa-done`. |
| 6. Orchestrator | Producers enqueue events; workers process; optional "claimed" state (DB or labels) to avoid duplicate work. |
| 7. Safety | Branch protection; workers in sandbox; no merge by workers. |

## Checklist Before Going "Live"

- [ ] Webhook (if used) and queue and workers deployed; token and secrets set.
- [ ] At most one worker handles a given issue/PR at a time (visibility timeout or locking).

## Done when

- Events (from webhooks and/or schedule) drive PM/Dev/QA workers; workers use GitHub API and LLM; you still approve work and merge via GitHub (or your own UI if you build it).
