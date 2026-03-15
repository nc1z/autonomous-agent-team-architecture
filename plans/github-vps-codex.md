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

## Done when

- Orchestrator runs 24/7 on the VPS; PM/Dev/QA agents use GitHub and Codex; you approve issues and merge PRs from anywhere.
