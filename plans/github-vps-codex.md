# Plan: GitHub + VPS + Codex CLI

| Layer | Choice |
|-------|--------|
| Event system | GitHub (issues, PRs) |
| Runner | Python/Node daemon on VPS |
| LLM worker | OpenAI Codex CLI |

Full control; Codex runs in containers or on host. 24/7 operation.

---

## Prerequisites

- GitHub repo; PAT with `repo` scope
- VPS (e.g. DigitalOcean, Linode, EC2) with SSH access
- Codex CLI installable on the VPS (or run Codex in Docker on the VPS)

## Steps

1. **Provision VPS** — Linux (Ubuntu/Debian); enough RAM/CPU for Codex and git (e.g. 2GB+ RAM).
2. **Install on VPS:** Git, Python 3 (or Node), Codex CLI (or Docker + Codex image).
3. **Clone repo** on VPS; store `GITHUB_TOKEN` in env (e.g. systemd service or `.env` not in repo).
4. **Copy orchestrator** (`orchestrator/` + `prompts/`) to repo; install deps (`pip install -r orchestrator/requirements.txt`).
5. **Run as a service** — systemd unit or supervisor so `orchestrator.py` restarts on failure and on reboot.
6. **Optional:** Run Codex inside Docker on the VPS for isolation; orchestrator on host calls `docker run ... codex ...`.

## Security

- Restrict SSH (key-only); firewall (only needed ports).
- Token only in env or secret manager; never in repo.
- Prefer running Codex in a container so agent code runs in a sandbox.

## What You Build

| Component | Responsibility |
|-----------|-----------------|
| **Orchestrator** | Loop: poll GitHub every N minutes → run PM → run Dev → run QA → sleep. Tracks "in progress" to avoid duplicate work. |
| **PM agent** | Propose features; create GitHub issues with label `proposed`. |
| **Dev agent** | Implement approved issues → branch, Codex, push, open PR. |
| **QA agent** | Review open PRs; post comments; add label when done. |

Repo layout: `orchestrator/` (orchestrator.py, pm_agent.py, dev_agent.py, qa_agent.py, github_client.py, config.py), `prompts/` (pm.md, dev.md, qa.md). GitHub setup: labels `proposed`, `approved`, `in-progress`, `qa-done`; branch protection on main; token in env.

## Implementation Phases

| Phase | What to do |
|-------|------------|
| 1. GitHub client | List issues by label, create issue/branch/PR, post review comments, add/remove labels. |
| 2. Config | Repo path, `GITHUB_TOKEN`, poll interval, owner/name. |
| 3. PM agent | Read repo context; call Codex; create issues with label `proposed`. |
| 4. Dev agent | Pick approved issue; branch; Codex; commit and push; open PR. |
| 5. QA agent | Get PR diff; Codex review; post comments; add `qa-done`. |
| 6. Orchestrator loop | PM → Dev → QA → sleep; prevent same issue picked twice. |
| 7. Safety | Codex in Docker; branch protection; CI gate. |

## Checklist Before Going "Live"

- [ ] Token in env on VPS; never in repo.
- [ ] Labels and branch protection configured.
- [ ] Orchestrator runs as systemd/supervisor; restarts on failure.

## Done when

- Orchestrator runs 24/7 on the VPS; PM/Dev/QA agents use GitHub and Codex; you approve issues and merge PRs from anywhere.
