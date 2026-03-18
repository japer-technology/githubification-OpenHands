# Githubification Analysis — OpenHands

### How `japer-technology/githubification-OpenHands` could become a GitHub Action based mechanism

---

## Executive Summary

OpenHands is a full-stack AI software engineering platform — a Python backend, React frontend, Docker-based runtimes, and an enterprise layer — designed to autonomously write code, resolve issues, and review pull requests. Unlike most repositories examined for Githubification, **OpenHands already contains its own Githubification mechanism**: the `openhands-resolver.yml` workflow, which installs the `openhands-ai` package on a GitHub Actions runner and uses the agent's own `openhands.resolver` module to fix issues and create pull requests. The `pr-review-by-openhands.yml` workflow adds a complementary AI code review loop.

This analysis examines how to deepen and extend that existing mechanism — transforming this repository from one where the agent occasionally runs on GitHub Actions into one where **GitHub is the primary runtime, GitHub Issues are the primary user interface, and Git is the primary memory layer**.

---

## Current State

### What Already Works

| GitHub Primitive | Current Mapping |
|---|---|
| **GitHub Actions** | `openhands-resolver.yml` runs the agent on a runner when an issue is labeled `fix-me` or a collaborator mentions `@openhands-agent` |
| **Git** | The agent creates branches and draft PRs; code changes are committed. But conversation history and agent memory are **not** persisted to git |
| **GitHub Issues** | Issues serve as task inputs. Comments report success or failure. But Issues are **not** used as a persistent conversational UI |
| **GitHub Secrets** | `LLM_API_KEY`, `PAT_TOKEN`, `LLM_BASE_URL` — credential management is fully operational |

### What Does Not Yet Exist

| Capability | Status |
|---|---|
| **Persistent conversation memory** | The resolver treats each issue as a one-shot task. There is no session file committed to git, no ability to continue a conversation across comments |
| **Issue-as-conversation** | A user can trigger the agent, but cannot have a back-and-forth dialogue within a single issue |
| **Git-committed session state** | No `state/` directory, no session-to-issue mapping, no JSONL transcripts |
| **Agent personality / identity** | No hatching system, no configurable persona |
| **Modular skill system** | The agent's capabilities are defined in code (`agenthub/`), not in composable, user-editable Markdown skill files |
| **Self-installing distribution** | The resolver workflow must be manually configured; there is no one-workflow-file install path |

### The Core Gap

OpenHands on GitHub Actions today is a **task executor**: label an issue → agent runs → PR appears (or doesn't). Githubification envisions a **conversational partner**: open an issue → agent replies → comment again → agent continues → all history persisted in git. The infrastructure for the first exists. The second requires a persistence and conversation layer that does not yet exist.

---

## Githubification Strategy Assessment

Using the five-strategy framework from [lesson-consolidation.md](https://github.com/japer-technology/githubification/blob/main/.githubification/lesson-consolidation.md):

```
Does the agent exist yet?
└── Yes
    └── Can it run on GitHub Actions?
        └── Partially — the resolver module runs, but the full platform
            (FastAPI server, WebSocket streaming, Docker sandboxes)
            does not fit the ephemeral runner model
            └── Two viable paths:
                Path A: Extend the existing resolver (Strategy 2 — Wrapping)
                Path B: Deploy a GitHub-native agent alongside (Strategy 3 — Substitution)
```

### Path A — Extend the Existing Resolver (Wrapping / Self-Referential)

**The agent wraps itself.** The `openhands.resolver` module is already a Githubification engine. Extending it means adding:

1. **Session persistence** — After each agent run, commit a session transcript (JSONL or JSON) to a `state/` directory. Map issue numbers to session files so the agent can resume context.
2. **Conversation continuity** — Modify the workflow to trigger on `issue_comment` (not just `labeled`) and load prior session state before running the agent.
3. **Reply-as-comment** — Instead of only creating PRs, have the agent post its response as an issue comment for conversational interactions, reserving PR creation for code changes.
4. **Memory layer** — Use the existing `memory/` module (conversation condensers) to summarize long conversations and stay within token limits across sessions.

**Advantages:**
- Uses OpenHands' own codebase — no external agent needed
- The resolver already handles issue parsing, agent configuration, branch creation, and PR submission
- Self-referential: the agent maintains the repo that defines the agent

**Challenges:**
- The resolver currently installs the full `openhands-ai` pip package on each run — heavy for conversational back-and-forth
- The agent's runtime model assumes Docker sandboxes for command execution, which adds complexity on ephemeral runners
- Session management and git-committed state would need to be added to the resolver module

### Path B — Deploy a GitHub-Native Agent Alongside (Substitution)

**A lightweight, GitHub-native agent runs beside the OpenHands codebase.** Following the pattern proven by [GitHub Minimum Intelligence](https://github.com/japer-technology/github-minimum-intelligence):

1. **Add a `.github-minimum-intelligence/` folder** — containing the GMI agent, lifecycle scripts, session state, and skills
2. **Single workflow file** — `github-minimum-intelligence-agent.yml` handles issue-driven conversation, git-committed memory, and authorization
3. **OpenHands codebase as context** — The substituted agent can read, explain, analyze, and modify the OpenHands source — it just doesn't run the OpenHands agent loop itself
4. **Coexistence** — Both the GMI conversational agent and the existing `openhands-resolver.yml` can operate in the same repository, serving different purposes

**Advantages:**
- Proven pattern (GMI is fully Githubified, single dependency, two-file lifecycle)
- Dramatically simpler: one dependency (`pi-coding-agent`), two TypeScript files, git-committed sessions
- Persistent memory, personality hatching, modular skills — all available out of the box
- The existing OpenHands resolver continues to handle `fix-me` labeled issues independently

**Challenges:**
- The conversational agent is not OpenHands — it's a separate agent (pi) that operates on the codebase
- Users expecting OpenHands-level capabilities (Docker sandboxing, browser automation, 30+ tools) would get a lighter agent for conversations
- Two agent systems in one repo increases conceptual complexity

### Recommended Path: Hybrid (A + B)

The strongest Githubification leverages both:

| Concern | Mechanism |
|---|---|
| **Conversational AI via Issues** | GMI agent (Path B) — lightweight, persistent, immediate |
| **Code fixing and PR creation** | OpenHands resolver (Path A, existing) — heavyweight, powerful, task-oriented |
| **PR review** | `pr-review-by-openhands.yml` (existing) — automated code review |
| **Agent memory** | Git-committed sessions via GMI's `state/` directory |
| **Agent identity** | GMI's hatching system for personality; `AGENTS.md` already exists in this repo |

The GMI agent handles the conversational surface — users ask questions, request analysis, discuss architecture, get explanations. When a conversation identifies a concrete code change, the user labels the issue `fix-me` and the OpenHands resolver takes over with full agent capabilities. Both agents share the same repository as context. Both commit their work to git.

---

## Architecture for Full Githubification

### The Four Primitives

| GitHub Primitive | Role | Implementation |
|---|---|---|
| **GitHub Actions** | Compute | Two workflows: GMI for conversation, OpenHands resolver for code changes |
| **Git** | Storage & Memory | Session transcripts in `state/`, code changes in branches/PRs, full audit trail |
| **GitHub Issues** | User Interface | Each issue is a conversation (GMI) or a task (resolver). Labels route to the right agent |
| **GitHub Secrets** | Credentials | `LLM_API_KEY` for both agents; `PAT_TOKEN` for resolver PR creation |

### Proposed File Structure

```
.github/
  workflows/
    github-minimum-intelligence-agent.yml   # NEW — Conversational agent
    openhands-resolver.yml                  # EXISTING — Code fixing agent
    pr-review-by-openhands.yml              # EXISTING — PR review agent
  ISSUE_TEMPLATE/
    github-minimum-intelligence-chat.md     # NEW — Chat issue template
    github-minimum-intelligence-hatch.md    # NEW — Personality hatching template

.github-minimum-intelligence/              # NEW — GMI agent folder
  .pi/
    settings.json                          # LLM provider configuration
    APPEND_SYSTEM.md                       # System prompt (OpenHands domain knowledge)
    skills/                                # Composable skill packages
  lifecycle/
    indicator.ts                           # 🚀 reaction indicator
    agent.ts                               # Core orchestrator
  state/
    issues/                                # Issue-to-session mappings
    sessions/                              # Conversation transcripts (JSONL)
  AGENTS.md                                # Agent identity (post-hatching)
  package.json                             # Single dependency (pi-coding-agent)

.openhands/                                # EXISTING — Microagents
  microagents/
    documentation.md
    glossary.md

openhands/                                 # EXISTING — Agent core
  resolver/                                # EXISTING — GitHub issue resolver
```

### Workflow Routing

```
User opens an issue
├── No label / general question
│   └── GMI agent responds conversationally
│       └── Conversation persisted to git
│       └── User can continue by commenting
├── Label: `fix-me`
│   └── OpenHands resolver activates
│       └── Agent reads issue, modifies code
│       └── Creates draft PR or pushes branch
├── Label: `hatch`
│   └── GMI agent begins personality discovery
│       └── Collaborative identity creation
└── Mention: `@openhands-agent`
    └── OpenHands resolver activates
        └── Same as fix-me path
```

---

## Implementation Steps

### Phase 1 — Add the Conversational Layer

1. **Copy the GMI workflow file** (`github-minimum-intelligence-agent.yml`) into `.github/workflows/`
2. **Run the GMI installer** (manual or `workflow_dispatch`) to create the `.github-minimum-intelligence/` folder
3. **Configure LLM settings** — set the provider and model in `.pi/settings.json`
4. **Add the LLM API key** as a repository secret
5. **Create issue templates** for chat and hatching
6. **Test** — open an issue, verify the agent responds and persists the conversation to git

### Phase 2 — Customize for OpenHands Domain

1. **Write an OpenHands-specific system prompt** in `.pi/APPEND_SYSTEM.md` — teach the GMI agent about OpenHands architecture, the resolver, the frontend, the enterprise layer
2. **Create skills** relevant to OpenHands: "explain the event system," "trace a resolver run," "describe the agent lifecycle," "analyze a microagent"
3. **Hatch the agent** — give it an identity appropriate to the OpenHands project
4. **Link to the resolver** — document in the system prompt that when users need code changes, they should label the issue `fix-me` to engage the OpenHands resolver

### Phase 3 — Deepen the Resolver (Optional, Path A)

1. **Add session persistence** to the resolver — after each run, commit a summary of what the agent did and why to a state file
2. **Enable multi-turn resolution** — allow the resolver to read prior resolution attempts from git before starting a new one
3. **Connect memory systems** — feed the GMI conversation history as context to the resolver when it activates on the same issue

### Phase 4 — Governance and Documentation

1. **Document the dual-agent architecture** — which agent handles what, how they coexist
2. **Define DEFCON levels** (following GMI's pattern) for operational readiness
3. **Write security guidelines** — both agents have write access to the repo; document the threat model
4. **Add authorization guards** — both workflows should verify the commenter is a collaborator before acting

---

## Why This Works

### The Lesson from the Githubification Corpus

The [githubification winners ranking](https://github.com/japer-technology/githubification/blob/main/.githubification/winners.md) places OpenHands at position 20 out of 20, noting:

> *"Persistent components and WebSocket streaming are the hardest patterns to reconcile with ephemeral execution."*

This is true for the **full OpenHands platform** (FastAPI server, Docker runtimes, React frontend). But it is **not true for the resolver**, which already runs ephemerally on GitHub Actions. And it is **not true for a substituted conversational agent** (GMI), which was born ephemeral.

The hybrid approach sidesteps the fundamental incompatibility by **not trying to run the full platform on Actions**. Instead:

- The resolver module runs the agent core in a constrained mode — no persistent server, no WebSocket, no frontend — just issue reading, code editing, and PR creation
- The GMI agent provides the persistent conversational surface that the resolver cannot
- Together, they deliver the Githubification promise: an AI agent that lives in the repo, runs on GitHub, converses through Issues, and remembers through Git

### The Self-Referential Property

OpenHands is unique in the Githubification corpus because **the agent being Githubified is itself a Githubification tool**. The `openhands.resolver` module is, by definition, a mechanism for running an AI agent on GitHub Actions to resolve issues. When this module resolves issues on its own repository, Githubification becomes self-referential:

- The agent maintains the codebase that defines the agent
- The resolver fixes bugs in the resolver
- PRs are reviewed by the same AI that created them

No other repository in the Githubification series demonstrates this property. It is both the strongest form of Githubification and the most philosophically interesting: **the repository is simultaneously the subject and the tool of Githubification**.

---

## The Four GitHub Primitives — Mapped

| Primitive | Current State | After Githubification |
|---|---|---|
| **GitHub Actions** | Resolver runs on `fix-me` label; PR review on PR open | + GMI agent runs on every issue/comment; persistent conversational compute |
| **Git** | Code changes committed via PRs | + Session transcripts, conversation history, agent state committed to `state/` |
| **GitHub Issues** | One-shot task triggers | + Persistent conversational threads; chat templates; hatching |
| **GitHub Secrets** | `LLM_API_KEY`, `PAT_TOKEN` | Same secrets, shared by both agents |

---

## Risk Assessment

| Risk | Mitigation |
|---|---|
| **Actions minutes consumption** | GMI's single-dependency install is fast (~30s). Resolver is heavier. Use labels to gate the resolver; let GMI handle lightweight queries |
| **Git repo size growth** | Session files are small (JSONL text). Prune old sessions periodically. The node_modules concern is addressed by `.gitignore` |
| **Two-agent confusion** | Clear documentation, distinct issue templates, label-based routing. GMI handles conversation; resolver handles code |
| **Security — agent has write access** | Both workflows check collaborator permissions. GMI uses workflow-level authorization. Resolver checks `author_association`. Public repos expose conversation history — use private repos for sensitive work |
| **Stale sessions** | GMI's session-per-issue model means closing the issue effectively ends the conversation. Git history preserves everything |

---

## Summary

This repository can become a GitHub Action based mechanism through a **hybrid approach**:

1. **The conversational layer** — GitHub Minimum Intelligence provides issue-driven, persistent, git-committed AI conversations with a single-dependency, two-file lifecycle
2. **The code execution layer** — The existing OpenHands resolver provides heavyweight, autonomous code fixing and PR creation
3. **The review layer** — The existing PR review workflow provides automated code review

Together, these three mechanisms map the four GitHub primitives (Actions as compute, Git as memory, Issues as UI, Secrets as credentials) into a complete Githubification of the OpenHands repository — not by forcing the full platform onto ephemeral runners, but by combining a lightweight conversational agent with the resolver module that was purpose-built for GitHub Actions execution.

> **The repo doesn't need to run its entire platform on GitHub. It needs to expose the right capabilities through the right mechanisms. The resolver is already the right mechanism for code changes. GMI is the right mechanism for conversation. Together, they make GitHub the infrastructure.**
