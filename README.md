# Autonomous Agent Fleet: Architecture & Design

A design document for planning a **multi-agent autonomous development loop** with **gated human approvals**. The system runs continuously around a repository; you stay in control via issue approval and PR merge.

---

## Goals

| Goal | Description |
|------|-------------|
| **Autonomous execution** | Agents propose, implement, and review work without constant human intervention. |
| **Gated approvals** | Humans approve which work is done (issues) and what ships (PRs). |
| **Repository as source of truth** | Issues, PRs, and code live in the repo; all automation is event- and state-driven. |
| **Continuous operation** | A persistent layer runs agents on a schedule or in response to events. |

---

## What You Need (Agnostic Overview)

Three layers are required:

1. **Event system** — Issues, PRs, comments (e.g. GitHub, GitLab, or similar).
2. **Persistent runners** — A service or job that keeps polling/triggering and dispatching work.
3. **LLM workers** — One or more coding engines that execute tasks (any CLI/API that can edit code from prompts).

The rest of this document describes how these pieces fit together in an agent fleet with gated approvals.

---

## High-Level Architecture

```mermaid
flowchart TB
    subgraph Events["Event system (e.g. GitHub)"]
        Issues
        PRs
        Comments
    end

    subgraph Orchestrator["Orchestrator / Scheduler"]
        Poll
        Dispatch
    end

    subgraph Agents["Agent fleet"]
        PM["PM Agent"]
        Dev["Dev Agents"]
        QA["QA Agent"]
    end

    subgraph Gate["Human gates"]
        ApproveIssue["Approve issue"]
        MergePR["Merge PR"]
    end

    Issues --> Poll
    PRs --> Poll
    Poll --> Dispatch
    Dispatch --> PM
    Dispatch --> Dev
    Dispatch --> QA
    PM --> Issues
    Dev --> PRs
    QA --> Comments
    ApproveIssue --> Dev
    MergePR --> Gate
```

---

## Repository as Source of Truth

All work is driven by repository state: issues (with labels/status) and pull requests.

```mermaid
flowchart LR
    subgraph Repo["Repository"]
        Issues["Issues\n(proposed → approved)"]
        PRs["Pull requests\n(open → reviewed)"]
        Code["Codebase"]
    end

    subgraph Fleet["Agent fleet"]
        PM["PM Agent"]
        Dev["Dev Agents"]
        QA["QA Agent"]
    end

    PM -->|Creates / labels| Issues
    Dev -->|Reads approved, opens| PRs
    QA -->|Reviews| PRs
    Issues -->|Approved list| Dev
    PRs -->|Ready to merge| Human
```

---

## Agent Loop with Gated Approvals

The loop is **autonomous** between gates; **you** control what gets built and what gets merged.

```mermaid
sequenceDiagram
    participant PM as PM Agent
    participant Human as You
    participant Dev as Dev Agent(s)
    participant QA as QA Agent
    participant Repo as Repo (Issues/PRs)

    PM->>Repo: Create / propose issues
    Human->>Repo: Approve issue (label/comment)
    Dev->>Repo: Pick approved issue
    Dev->>Repo: Branch + implement + open PR
    QA->>Repo: Review PR (comments)
    loop Until QA satisfied
        Dev->>Repo: Fix review feedback, push
        QA->>Repo: Re-review
    end
    Human->>Repo: Merge PR
```

**Gates:**

- **Gate 1 — Issue approval:** Only issues you approve (e.g. label `approved`) are picked up by dev agents.
- **Gate 2 — PR merge:** Only you (or your rules) merge; agents never merge.

---

## Agent Roles (Agnostic)

| Role | Responsibility | Trigger | Output |
|------|----------------|---------|--------|
| **PM Agent** | Propose scope and prioritization | Schedule or manual | New/updated issues (e.g. label `proposed`) |
| **Dev Agent(s)** | Implement approved work | Approved issues (label + open) | Branch, commits, PR |
| **QA Agent** | Review PRs for quality and policy | New/updated PR | Review comments, status (pass/fail) |

```mermaid
flowchart LR
    subgraph Inputs
        Readme["README / docs"]
        Roadmap["Roadmap"]
        Approved["Approved issues"]
    end

    subgraph Roles
        PM["PM Agent\n→ proposals"]
        Dev["Dev Agent\n→ implementation"]
        QA["QA Agent\n→ review"]
    end

    Readme --> PM
    Roadmap --> PM
    Approved --> Dev
    Dev --> QA
```

---

## Safety Mechanisms

Autonomous coding must be constrained. Recommended controls:

```mermaid
flowchart TB
    subgraph Safety["Safety layer"]
        BP["Branch protection\n(require review / status)"]
        CI["CI gate\n(tests must pass)"]
        Size["PR size / scope limits"]
        Sandbox["Sandboxed execution\n(e.g. containers)"]
    end

    PR["Every PR"] --> BP
    PR --> CI
    PR --> Size
    Agents["Agent runs"] --> Sandbox
```

| Mechanism | Purpose |
|-----------|---------|
| **Branch protection** | No direct push to main; PR required; optional required reviews. |
| **Test gate** | CI must pass before merge (and before human approval). |
| **PR size / scope limits** | Reject or flag PRs over a certain size to keep reviews manageable. |
| **Sandbox** | Agents run in isolated environments (e.g. Docker) so they cannot affect host or secrets. |

---

## Deployment Options (Overview)

| Option | Pros | Cons |
|--------|------|------|
| **Persistent daemon (VPS / server)** | Full control, 24/7, parallel agents, simple mental model. | You operate the server. |
| **Scheduled CI (e.g. cron-style jobs)** | No server to maintain, event-driven. | Less flexible than a daemon; job limits and timeouts. |
| **Hybrid** | Critical loops on daemon, heavy/rare jobs on CI. | Two systems to configure. |

```mermaid
flowchart LR
    subgraph OptionA["Daemon (VPS)"]
        D1["Orchestrator"]
        D2["Poll repo"]
        D3["Run agents"]
        D1 --> D2 --> D3 --> D1
    end

    subgraph OptionB["Scheduled CI"]
        C1["Trigger on schedule\nor issue/PR events"]
        C2["Run agent job"]
        C1 --> C2
    end
```

---

## Summary: What You Need

1. **Event system** — Issues and PRs (and optionally comments) as the interface.
2. **Orchestrator** — Polls or reacts to events and dispatches to the right agent.
3. **PM agent** — Proposes work; you approve by labeling or commenting.
4. **Dev agent(s)** — Implement only approved issues; open PRs, fix QA feedback.
5. **QA agent** — Reviews PRs and posts comments; no merge.
6. **Gates** — You approve issues and merge PRs; agents never merge.
7. **Safety** — Branch protection, CI, optional size limits, sandboxed runs.

The document above is **tool-agnostic**: it applies to any event backend (e.g. GitHub, GitLab), any runner (VPS, Kubernetes, serverless), and any LLM-based coding engine.

---

## Possible Solutions (Concrete Options)

This section lists **example** ways to implement the architecture; the rest of the document does not depend on these.

| Approach | Event system | Runner | LLM worker | Notes |
|----------|--------------|--------|------------|--------|
| **GitHub + VPS + Codex CLI** | GitHub (issues, PRs) | Python/Node daemon on VPS | OpenAI Codex CLI | Full control; Codex runs in containers or on host. |
| **GitHub Actions only** | GitHub | Actions on `schedule` + `issues` / `pull_request` | Codex API or another API | No VPS; constrained by Actions limits and timeouts. |
| **GitHub + Fly.io / Railway** | GitHub | Small app on Fly/Railway | Any HTTP/CLI coding API | Managed “daemon” without managing a raw VPS. |
| **GitLab + self-hosted runner** | GitLab (issues, MRs) | GitLab CI + runner | Any CLI/API | Same idea as GitHub + Actions, with GitLab semantics. |
| **Custom event bus** | Your own queue (e.g. SQS, Redis) | Workers consuming queue | Any | Maximum flexibility; you define events and triggers. |

**Codex-specific note:** The OpenAI Codex desktop app is an interactive assistant and does not run as a persistent background agent. For an autonomous fleet, use the **Codex CLI or API** from your orchestrator/runner so that agents can execute coding tasks in a headless, scripted way.

Choosing among these is a matter of where your repo lives, how much you want to operate (VPS vs managed), and which LLM coding engine you use; the architecture and gated-approval flow stay the same.
