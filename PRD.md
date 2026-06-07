# FreIA — Product Requirement Document

> **Status:** Skeleton v0.1 — iterative module-by-module construction.  
> **Target Agent:** OpenCode (initial adapter).  
> **Inspiration:** Gentle-AI (ecosystem configuration), OpenSpec (SDD artifact workflow).

---

## 1. Vision & Problem Statement

**What FreIA is:**  
A project-level CLI tool that bootstraps an AI-native development ecosystem inside a repository. It does not install global AI agents — it installs *into* a project the workflows, agent instructions, tooling harness, and SDD scaffolding so that OpenCode (and later others) can operate with spec-driven, test-first, integration-heavy discipline.

**What FreIA is NOT:**  
A global multi-agent installer (that is Gentle-AI’s scope). FreIA is local and opinionated.

**Pain it solves:**  
- Most AI coding tools write code without specs, without tests, and without local reproducibility.  
- Setting up devenv, testcontainers, Gherkin, and agent instructions is tedious and non-portable across repos.  
- There is no lightweight way to say: "make this repo AI-ready" with one command.

**One-liner:**  
`freia init` turns any repo into an AI-ready, spec-driven, integration-tested harness in seconds.

---

## 2. Philosophy of Development

FreIA enforces a strict development philosophy that agents *must* respect:

| Principle | How it manifests |
|-----------|----------------|
| **Spec-Driven Development (SDD)** | No code without a proposal + specs + design + tasks. |
| **TDD + BDD** | Write failing tests first. Gherkin for behavior, unit tests for internals. |
| **Integration over Unit** | Prioritize integration tests with real dependencies (testcontainers, local services) over mocked unit tests. |
| **Local Reproducibility** | Every test and service runs locally via devenv / testcontainers / docker-compose. No cloud-only infra. |
| **Brownfield-first** | Works on existing codebases, not just greenfield. Incremental adoption. |
| **Agent Accountability** | The orchestrator decides complexity; subagents are accountable to specs and tests. |
| **Human-in-the-loop** | The spec agent asks questions, debates architecture, and does NOT decide alone. |

**Tooling baseline:**
- `devenv` / `direnv` for reproducible dev environments.
- `testcontainers` for integration tests against real databases, queues, etc.
- `Gherkin` (Cucumber / behave / godog) for BDD acceptance tests.
- Language-native test runners with `--godog` or similar integration hooks.
- Conventional commits, no Co-Authored-By AI attribution.

---

## 3. Architecture Overview

### 3.1 High-Level Modules

```
FreIA CLI (Go)
├── cmd/freia              → CLI entrypoint
│   ├── init               → Bootstrap repo with ecosystem
│   ├── agent              → Manage agent manifests
│   ├── harness            → Add/remove tooling harnesses
│   └── doctor             → Health check of ecosystem
├── internal/
│   ├── config             → Global vs project-level config logic
│   ├── harness            → Tool scaffolds (devenv, testcontainers, gherkin...)
│   ├── agents             → Agent manifest generator & validator
│   ├── sdd                → OpenSpec-style artifact scaffolding
│   └── project            → Project introspection (stack detection)
├── agent-manifests/       → Templates for OpenCode agents
│   ├── orchestrator.md    → Orchestrator prompt & decision logic
│   ├── spec.md            → Spec agent: architecture, Q&A, debate
│   ├── implementer.md     → Implementer agent: task → code
│   ├── reviewer.md        → Reviewer agent: diff + spec compliance
│   └── test.md            → Test agent: TDD scaffolding, testcontainers
└── docs/
    ├── philosophy.md
    ├── harness-guide.md
    └── agent-contracts.md
```

### 3.2 Agent Architecture: Orchestrator + Subagents

FreIA’s agent layer follows a **"thin orchestrator, thick subagents"** model. The orchestrator is a prompt/agent installed into OpenCode by FreIA; OpenCode's native subagent/task features handle the actual invocation.

```
┌─────────────────────────────┐
│   Orchestrator Agent        │
│   - Does NOT implement      │
│   - Evaluates task size &   │
│     complexity              │
│   - Chooses subagent(s)     │
│   - Crafts prompt + context   │
│   - Combines results          │
└──────────┬──────────────────┘
           │
    ┌──────┴────────┐
    ▼               ▼
┌────────────┐  ┌────────────┐
│ Spec Agent │  │Implementer │
│(Complex    │  │(Small task │
│ tasks)     │  │ or post-   │
│            │  │ spec)      │
└────────────┘  └────────────┘
    │
    ▼
┌────────────┐
│ Reviewer   │
│ Test Agent │
└────────────┘
```

| Agent | Responsibility | When invoked |
|-------|--------------|------------|
| **Orchestrator** | Decides *how* to execute given task complexity. Loads spec context. | Every task |
| **Spec Agent** | Proposes architecture, asks human questions, debates decisions, writes OpenSpec artifacts. | Task is medium+ or touches architecture |
| **Implementer** | Writes code against specs/tasks. | Always, either solo (small) or post-spec (medium+) |
| **Reviewer** | Validates diff against spec + tests. | After implementation on medium+ tasks |
| **Test Agent** | Scaffolds integration tests, testcontainers setups, Gherkin features. | When spec includes acceptance criteria |

**Complexity Heuristics (v1):**
- **Small:** ≤1 file, ≤50 LOC, no architecture change → Orchestrator → Implementer solo.
- **Medium:** 2+ files, touches interfaces, needs new tests → Orchestrator → Spec → Implementer → Reviewer.
- **Large:** New service, DB schema, infra change → Full SDD flow: Spec → Design → Tasks → Implement → Verify → Archive.

---

## 4. Module Breakdown

### 4.1 M1 — CLI Skeleton & Init

**Goal:** `freia init` scaffolds the repo.

**What it does:**
1. Detects project stack (Go, Rust, TS, Python, etc.).
2. Asks: which harness tools? (devenv, testcontainers, Gherkin runner).
3. Generates `.freia/` directory:
   - `.freia/config.yaml` — project config (tools enabled, harness versions).
   - `.freia/agents/` — OpenCode agent manifests (orchestrator, spec, implementer, etc.).
   - `.freia/skills/` — reusable skill prompts (TDD, integration-test pattern, etc.).
   - `.freia/openspec/` — SDD artifact scaffold (changes/, archive/).  
4. Generates scaffolds/configs for selected harness tools (e.g., `devenv.nix` stub, feature directory). **Does NOT install the tools themselves.** The user installs devenv, testcontainers, etc. separately.

**Out of scope v1:**  
- Global install. Everything is project-local.
- Multi-agent support beyond OpenCode.

### 4.2 M2 — Harness Manager

**Goal:** `freia harness add <tool>` / `freia harness remove <tool>`

**Design principle:** FreIA does **not** install harness tools. It generates configurations, scaffolds, and instructions so the user's repo is ready for them. `freia doctor` reports whether required tools are present.

**Supported harnesses (v1):**
| Tool | What FreIA scaffolds | What user must install |
|------|----------------------|------------------------|
| `devenv` | `devenv.nix` + `.envrc` + language packages | `devenv` CLI |
| `testcontainers` | Language-specific container configs + example tests | language package manager |
| `gherkin` | `features/` directory + runner glue stubs | language-specific Gherkin runner |
| `docker-compose.dev` | `docker-compose.dev.yml` with postgres, redis, etc. | Docker |

**Design:**  
Each harness is a Go package implementing a `HarnessScaffold` interface:
```go
type HarnessScaffold interface {
    Name() string
    Detect(ctx context.Context, proj Project) (Compatibility, error)
    Scaffold(ctx context.Context, proj Project, cfg Config) error
    Remove(ctx context.Context, proj Project) error
    Validate(ctx context.Context, proj Project) error
}
```

### 4.3 M3 — Agent Manifest Generator

**Goal:** `freia agent sync` generates OpenCode-compatible agent configs in `.freia/agents/`.

**What it generates:**
- `.freia/agents/orchestrator.md` — decision logic + complexity heuristics.
- `.freia/agents/spec.md` — spec agent prompt with emphasis on human Q&A.
- `.freia/agents/implementer.md` — task-driven code writer.
- `.freia/agents/reviewer.md` — diff + spec compliance checker.
- `.freia/agents/test.md` — testcontainers + Gherkin scaffolding agent.

**OpenCode integration:**  
FreIA writes these into a structure OpenCode can load as custom agents or system prompts. In v1, this is `.freia/agents/*.md` with frontmatter. The Orchestrator is a prompt/agent installed into OpenCode; OpenCode's native subagent/task features handle invocation.

### 4.4 M4 — SDD Scaffold

**Goal:** `freia sdd new <change-name>` creates the OpenSpec-style artifact folder.

**Artifacts generated:**
- `.freia/openspec/changes/<name>/proposal.md`
- `.freia/openspec/changes/<name>/specs/`
- `.freia/openspec/changes/<name>/design.md`
- `.freia/openspec/changes/<name>/tasks.md`
- `.freia/openspec/changes/<name>/verify.md`

**Agent contract:**  
The Spec Agent must fill these before Implementer runs. The Orchestrator enforces this gate.

### 4.5 M5 — Orchestrator Runtime (OpenCode Agent)

**Goal:** An OpenCode agent or skill that acts as the Orchestrator, installed by FreIA.

**How it works:**
- Receives user prompt.
- Classifies complexity using heuristics (see §3.2).
- If small: delegates to Implementer using OpenCode's native subagent/task features.
- If medium+: runs Spec Agent first, then loads resulting tasks, then delegates to Implementer per task, then Reviewer.
- Test Agent is invoked whenever acceptance criteria exist.

**This is NOT a FreIA CLI module** — it is an agent manifest that FreIA installs into OpenCode.

---

## 5. OpenCode Integration Strategy (v1)

Since FreIA v1 targets OpenCode exclusively, the integration points are:

1. **Agent Manifests** live in `.freia/agents/*.md` with OpenCode-compatible frontmatter.
2. **Skills:** FreIA writes reusable prompts into `.freia/skills/` and instructs the user to register them in OpenCode’s skill system.
3. **SDD Workflow:** FreIA scaffolds OpenSpec-style directories; OpenCode’s `/sdd-*` commands (or custom slash commands) consume them.
4. **Config:** `.freia/config.yaml` may be read by OpenCode hooks (if supported) or used by FreIA CLI to generate agent-specific configs.

**Orchestrator runtime:** The Orchestrator is implemented as an OpenCode custom agent (or enhanced system prompt) loaded from `.freia/agents/orchestrator.md`. It uses OpenCode's native `Task` tool or subagent features to delegate to other agents. FreIA CLI does not invoke agents programmatically.

---

## 6. Philosophy Checklist for Agents

Every agent manifest generated by FreIA must include these behavioral constraints:

- [ ] **No code without a failing test** (TDD).
- [ ] **No test without a Gherkin scenario** for integration/acceptance.
- [ ] **No implementation without a spec** for medium+ tasks.
- [ ] **Prefer testcontainers over mocks** for external dependencies.
- [ ] **Ask before assuming** — especially Spec Agent.
- [ ] **Debate architecture** — do not rubber-stamp the first idea.
- [ ] **Human is the decision maker** — agents propose, humans approve.
- [ ] **Local reproducibility** — if it doesn’t run in `devenv shell`, it’s not done.

---

## 7. Roadmap (Iterative)

| Milestone | Scope | Exit Criteria |
|-----------|-------|---------------|
| **M1** | CLI skeleton + `freia init` + stack detection + `.freia/` scaffold | Can run `freia init` in a Go/TS repo and get a valid scaffold |
| **M2** | Harness manager: add/remove devenv, testcontainers, gherkin | `freia harness add devenv` produces working devenv.nix stub + doctor detects devenv |
| **M3** | Agent manifest generator: OpenCode agent prompts | `.freia/agents/` contains orchestrator + spec + implementer |
| **M4** | SDD scaffold: `freia sdd new` + OpenSpec artifact layout | Running `freia sdd new auth` creates proposal/spec/design/tasks files |
| **M5** | End-to-end demo: Orchestrator → Spec → Implementer → Reviewer with OpenCode | A real feature implemented via the full SDD flow |
| **M6** | Multi-harness ecosystem: docker-compose.dev, observability, local tracing | Local infra reproducible with one `devenv up` |
| **M7** | Multi-agent adapter: beyond OpenCode (Cursor, Claude Code, etc.) | Gentle-AI-style per-agent manifest generation |

---

## 8. Decisions Log

| # | Decision | Rationale | Date |
|---|----------|-----------|------|
| 1 | **Harnesses: scaffold-only, not installed** | Tools like devenv/testcontainers are complex external dependencies. FreIA generates configs; the user installs binaries. `freia doctor` validates presence. | 2026-06-03 |
| 2 | **Config format: YAML** | Standard across OpenSpec and dev tools. Human-readable and agent-compatible. | 2026-06-03 |
| 3 | **Persistence: filesystem-only** | Avoids SQLite complexity. OpenSpec-style artifact directories are sufficient for v1. Persistent memory is its own project/repository. | 2026-06-03 |
| 4 | **Orchestrator: OpenCode agent/prompt, not CLI invocation** | FreIA CLI installs the orchestrator as an OpenCode custom agent. OpenCode's native Task/subagent features handle runtime delegation. | 2026-06-03 |
| 5 | **Agnostic to user project stack** | FreIA CLI is Go. All agent prompts, configs, and scaffolds are language-agnostic. Subagents specialize per stack (greenfield vs legacy). | 2026-06-03 |
| 6 | **All configs & prompts in English** | AI tools perform best with English system prompts. User can interact in any natural language. | 2026-06-03 |

---

## 9. Appendix: Terminology

| Term | Definition |
|------|------------|
| **Harness** | A reproducible dev tool or service configured by FreIA (e.g., devenv, testcontainers). |
| **Agent Manifest** | A markdown file with frontmatter that defines an AI agent’s system prompt, tools, and constraints. |
| **Orchestrator** | The top-level agent that decides which subagents to invoke based on task complexity. |
| **Spec Agent** | The agent responsible for architecture proposals, human Q&A, and OpenSpec artifact authoring. |
| **SDD** | Spec-Driven Development: agree on what to build before building it. |
| **OpenSpec** | The lightweight artifact-driven workflow (proposal → specs → design → tasks → verify → archive). |

---

*This document is a skeleton. Each module will be expanded into its own technical design and task breakdown as we iterate.*
