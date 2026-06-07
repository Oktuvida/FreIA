# FreIA — Product Requirement Document

> **Status:** Skeleton v0.2 — iterative module-by-module construction.  
> **Target Agent:** OpenCode (initial adapter).  
> **Inspiration:** Gentle-AI (ecosystem configuration), OpenSpec (SDD artifact workflow).

---

## 1. Vision & Problem Statement

**What FreIA is:**  
A project-level CLI tool that bootstraps an AI-native development ecosystem inside a repository. It does not install global AI agents — it installs *into* a project the workflows, agent instructions, tooling harness, and SDD scaffolding so that OpenCode (and later others) can operate with spec-driven, test-first, integration-heavy discipline.

**What FreIA is NOT:**  
A global multi-agent installer (that is Gentle-AI's scope). FreIA is local and opinionated.

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
| **Mutation Testing** | Verify test suite strength by deliberately introducing mutants. Surviving mutants reveal weak or missing tests. |
| **Local Reproducibility** | Every test and service runs locally via devenv / testcontainers / docker-compose. No cloud-only infra. |
| **Brownfield-first** | Works on existing codebases, not just greenfield. Incremental adoption. |
| **Agent Accountability** | The orchestrator decides complexity; subagents are accountable to specs and tests. |
| **Human-in-the-loop** | The spec agent asks questions, debates architecture, and does NOT decide alone. |

### 2.1 TDD Craftsman: The Three Laws (Non-Negotiable)

Every code generation by the Implementer agent follows the Three Laws of TDD:

1. **You may not write production code** unless it is to make a failing test pass.
2. **You may not write more of a test** than is sufficient to fail (compilation/import failures count).
3. **You may not write more production code** than is sufficient to pass the currently failing test.

The cycle is always: **RED → GREEN → REFACTOR**.

### 2.2 Code Quality Gates

Every implementation is considered incomplete until all quality gates pass deterministically. FreIA generates a `freia-quality-gates.sh` (or equivalent) script per project that runs:

| Gate | Command | Purpose |
|------|---------|---------|
| **Format** | Language-specific formatter (e.g., `gofmt`, `prettier --check`, `black --check`) | Consistent code style |
| **Type Check** | Language-specific type checker (e.g., `go vet`, `tsc --noEmit`, `mypy`) | Static type safety |
| **Lint** | Language-specific linter (e.g., `golangci-lint`, `eslint`, `ruff`) | Code quality & patterns |
| **Unit Tests** | Language-native test runner | Fast feedback on isolated logic |
| **Integration Tests** | Testcontainers-based tests | Real dependency validation |
| **E2E Tests** | Full-stack acceptance tests | End-to-end behavior verification |
| **Mutation Tests** | Mutation testing tool (e.g., `go-mutesting`, `Stryker`, `mutmut`) | Test suite strength verification |

**Rule:** A task is DONE only when `freia-quality-gates.sh` exits 0. The Orchestrator enforces this gate.

### 2.3 Tooling Baseline

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
│   ├── quality            → Quality gates script generator
│   └── project            → Project introspection (stack detection)
├── agent-manifests/       → Templates for OpenCode agents
│   ├── orchestrator.md    → Orchestrator prompt & decision logic
│   ├── spec.md            → Spec agent: architecture, Q&A, debate
│   ├── gherkin-author.md  → Gherkin author: business-focused BDD scenarios
│   ├── implementer.md     → TDD Craftsman: RED-GREEN-REFACTOR
│   ├── reviewer.md        → Reviewer: spec compliance + coverage check
│   └── mutation-tester.md → Mutation tester: deterministic mutant generation
└── docs/
    ├── philosophy.md
    ├── harness-guide.md
    └── agent-contracts.md
```

### 3.2 Agent Architecture: Orchestrator + Subagents

FreIA's agent layer follows a **"thin orchestrator, thick subagents"** model. The orchestrator is a prompt/agent installed into OpenCode by FreIA; OpenCode's native Task/subagent features handle the actual invocation.

```
┌─────────────────────────────────────────────┐
│            Orchestrator Agent               │
│  - Does NOT implement                       │
│  - Evaluates task size & complexity         │
│  - Chooses subagent(s)                      │
│  - Crafts prompt + context                  │
│  - Combines results                         │
│  - Enforces quality gates                   │
└──────────┬──────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│            Spec Agent                       │
│  - Conversational partner                   │
│  - Asks many questions                      │
│  - Debates architecture decisions           │
│  - Writes OpenSpec artifacts                │
└──────────┬──────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│          Gherkin Author                     │
│  - Distills contracts from specs            │
│  - Business-focused, production mindset     │
│  - Thinks in business terms, not code       │
│  - Writes Gherkin scenarios                 │
└──────────┬──────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│       TDD Craftsman (Implementer)           │
│  - RED → GREEN → REFACTOR cycle             │
│  - Three Laws of TDD (non-negotiable)       │
│  - Writes code only to pass failing tests   │
└──────────┬──────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│            Reviewer                         │
│  - Validates diff against spec              │
│  - Checks all Gherkin scenarios covered     │
│  - Loops back to Implementer if issues      │
│  - Proceeds to Mutation Tester if clean     │
└──────────┬──────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│         Mutation Tester                     │
│  - Runs deterministic bash scripts          │
│  - Generates mutants (new files, not        │
│    replacing originals)                     │
│  - Runs tests against each mutant           │
│  - Records survivors: file, line, mutation, │
│    test result                              │
│  - Hands survivors back to Implementer      │
└─────────────────────────────────────────────┘
```

### 3.3 Agent Contracts

| Agent | Responsibility | When invoked | OpenCode Mode |
|-------|--------------|------------|---------------|
| **Orchestrator** | Decides *how* to execute given task complexity. Loads spec context. Enforces quality gates. | Every task | `primary` |
| **Spec Agent** | Conversational partner. Proposes architecture, asks human questions, debates decisions, writes OpenSpec artifacts. | Task is medium+ or touches architecture | `subagent` |
| **Gherkin Author** | Distills contracts from specs into Gherkin scenarios. Business-focused, production mindset. | After Spec Agent produces specs | `subagent` |
| **TDD Craftsman** | Writes code following RED-GREEN-REFACTOR. Three Laws of TDD. | Always, either solo (small) or post-spec (medium+) | `subagent` |
| **Reviewer** | Validates diff against spec + Gherkin scenarios. Checks all scenarios have at least one concrete test. Loops back to Implementer if gaps found. | After implementation on medium+ tasks | `subagent` |
| **Mutation Tester** | Generates deterministic mutants, runs tests, records survivors. Hands back to Implementer. | After Reviewer approves | `subagent` |

**Complexity Heuristics (v1):**
- **Small:** ≤1 file, ≤50 LOC, no architecture change → Orchestrator → TDD Craftsman solo.
- **Medium:** 2+ files, touches interfaces, needs new tests → Orchestrator → Spec → Gherkin Author → TDD Craftsman → Reviewer → Mutation Tester.
- **Large:** New service, DB schema, infra change → Full SDD flow: Spec → Design → Gherkin Author → Tasks → TDD Craftsman → Reviewer → Mutation Tester → Verify → Archive.

---

## 4. Module Breakdown

### 4.1 M1 — CLI Skeleton & Init

**Goal:** `freia init` scaffolds the repo.

**What it does:**
1. Detects project stack (Go, Rust, TS, Python, etc.).
2. Asks: which harness tools? (devenv, testcontainers, Gherkin runner).
3. Generates `.freia/` directory:
   - `.freia/config.yaml` — project config (tools enabled, harness versions, language).
   - `.freia/agents/` — OpenCode agent manifests (orchestrator, spec, gherkin-author, implementer, reviewer, mutation-tester).
   - `.freia/skills/` — reusable skill prompts (TDD, integration-test pattern, etc.).
   - `.freia/openspec/` — SDD artifact scaffold (changes/, archive/).
4. Generates scaffolds/configs for selected harness tools (e.g., `devenv.nix` stub, feature directory). **Does NOT install the tools themselves.** The user installs devenv, testcontainers, etc. separately.
5. Generates `freia-quality-gates.sh` with format, typecheck, lint, test, and mutation test commands for the detected stack.

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

**Goal:** `freia agent sync` generates OpenCode-compatible agent configs in `.opencode/agents/`.

**What it generates (OpenCode markdown format with YAML frontmatter):**
- `.opencode/agents/orchestrator.md` — primary agent, decision logic, complexity heuristics, task permissions for all subagents.
- `.opencode/agents/spec.md` — subagent, conversational partner, architecture Q&A.
- `.opencode/agents/gherkin-author.md` — subagent, business-focused BDD scenario writer.
- `.opencode/agents/implementer.md` — subagent, TDD Craftsman, RED-GREEN-REFACTOR.
- `.opencode/agents/reviewer.md` — subagent, spec compliance + Gherkin coverage check.
- `.opencode/agents/mutation-tester.md` — subagent, deterministic mutant generation + test execution.

**OpenCode format example (`.opencode/agents/implementer.md`):**
```markdown
---
description: TDD Craftsman implementing code following RED-GREEN-REFACTOR cycle
mode: subagent
model: anthropic/claude-sonnet-4-20250514
temperature: 0.2
permission:
  edit: allow
  bash:
    "*": ask
    "go test*": allow
    "npm test*": allow
    "pytest*": allow
---
You are a TDD Craftsman. You follow the Three Laws of TDD:
1. You may not write production code unless it is to make a failing test pass.
2. You may not write more of a test than is sufficient to fail.
3. You may not write more production code than is sufficient to pass the failing test.

Cycle: RED → GREEN → REFACTOR.
[... full prompt ...]
```

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

### 4.5 M5 — Quality Gates Script Generator

**Goal:** `freia quality init` generates a deterministic quality gates script.

**What it generates:**
- `scripts/freia-quality-gates.sh` — runs all quality gates in order.
- Language-specific commands based on detected stack.
- Exit code 0 = all gates pass. Non-zero = failure with details.

**Gate execution order (fixed):**
1. Format check
2. Type check
3. Lint
4. Unit tests
5. Integration tests
6. E2E tests
7. Mutation tests

### 4.6 M6 — Orchestrator Runtime (OpenCode Agent)

**Goal:** An OpenCode agent that acts as the Orchestrator, installed by FreIA.

**How it works:**
- Receives user prompt.
- Classifies complexity using heuristics (see §3.2).
- If small: delegates to TDD Craftsman via OpenCode's Task tool.
- If medium+: runs Spec Agent → Gherkin Author → TDD Craftsman (per task) → Reviewer → Mutation Tester.
- After Mutation Tester, if survivors exist, loops back to TDD Craftsman.
- Quality gates must pass before declaring DONE.

**This is NOT a FreIA CLI module** — it is an agent manifest that FreIA installs into OpenCode.

---

## 5. OpenCode Integration Strategy (v1)

Since FreIA v1 targets OpenCode exclusively, the integration points are:

1. **Agent Manifests** live in `.opencode/agents/*.md` with YAML frontmatter following OpenCode's format.
2. **Skills:** FreIA writes reusable prompts into `.opencode/skills/` and instructs the user to register them in OpenCode's skill system.
3. **SDD Workflow:** FreIA scaffolds OpenSpec-style directories; OpenCode's `/sdd-*` commands (or custom slash commands) consume them.
4. **Config:** `.freia/config.yaml` is used by FreIA CLI to generate agent-specific configs. OpenCode reads `.opencode/agents/*.md` directly.
5. **Instructions:** FreIA may add `freia-philosophy.md` to OpenCode's `instructions` array in `opencode.json` so the philosophy is always loaded.

**Orchestrator runtime:** The Orchestrator is implemented as an OpenCode primary agent (`.opencode/agents/orchestrator.md`) with `task` permissions pointing to all subagents. It uses OpenCode's native `Task` tool to delegate. FreIA CLI does not invoke agents programmatically.

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
- [ ] **Local reproducibility** — if it doesn't run in `devenv shell`, it's not done.
- [ ] **Mutation testing** — test suite must be validated by mutation testing before declaring complete.
- [ ] **Quality gates** — `freia-quality-gates.sh` must exit 0 before a task is considered done.

---

## 7. Roadmap (Iterative)

| Milestone | Scope | Exit Criteria |
|-----------|-------|---------------|
| **M1** | CLI skeleton + `freia init` + stack detection + `.freia/` scaffold | Can run `freia init` in a Go/TS repo and get a valid scaffold |
| **M2** | Harness manager: add/remove devenv, testcontainers, gherkin | `freia harness add devenv` produces working devenv.nix stub + doctor detects devenv |
| **M3** | Agent manifest generator: OpenCode agent prompts | `.opencode/agents/` contains all 6 agents with proper frontmatter |
| **M4** | SDD scaffold: `freia sdd new` + OpenSpec artifact layout | Running `freia sdd new auth` creates proposal/spec/design/tasks files |
| **M5** | Quality gates: `freia quality init` + deterministic script generation | `freia-quality-gates.sh` runs format/typecheck/lint/test/mutation |
| **M6** | End-to-end demo: Orchestrator → Spec → Gherkin Author → TDD Craftsman → Reviewer → Mutation Tester | A real feature implemented via the full SDD flow |
| **M7** | Multi-harness ecosystem: docker-compose.dev, observability, local tracing | Local infra reproducible with one `devenv up` |
| **M8** | Multi-agent adapter: beyond OpenCode (Cursor, Claude Code, etc.) | Gentle-AI-style per-agent manifest generation |

---

## 8. Decisions Log

| # | Decision | Rationale | Date |
|---|----------|-----------|------|
| 1 | **Harnesses: scaffold-only, not installed** | Tools like devenv/testcontainers are complex external dependencies. FreIA generates configs; the user installs binaries. `freia doctor` validates presence. | 2026-06-03 |
| 2 | **Config format: YAML** | Standard across OpenSpec and dev tools. Human-readable and agent-compatible. | 2026-06-03 |
| 3 | **Persistence: filesystem-only** | Avoids SQLite complexity. OpenSpec-style artifact directories are sufficient for v1. Persistent memory is its own project/repository. | 2026-06-03 |
| 4 | **Orchestrator: OpenCode agent/prompt, not CLI invocation** | FreIA CLI installs the orchestrator as an OpenCode custom agent. OpenCode's native Task/subagent features handle runtime delegation. | 2026-06-03 |
| 5 | **Agnostic to user project stack** | FreIA CLI is Go, but everything else (prompts, scaffolds) is language-agnostic. Subagents specialize per stack (greenfield vs legacy). | 2026-06-03 |
| 6 | **All configs & prompts in English** | AI tools perform best with English system prompts. User can interact in any natural language. | 2026-06-03 |
| 7 | **Mutation Testing added to philosophy** | Test suites must be validated by deliberate mutation. Surviving mutants reveal weak or missing tests. | 2026-06-03 |
| 8 | **Deterministic quality gates** | `freia-quality-gates.sh` runs format, typecheck, lint, unit, integration, e2e, mutation tests. A task is DONE only when gates pass. | 2026-06-03 |
| 9 | **Spec Agent is conversational partner** | Must ask many questions, debate architecture, be a sparring partner — not a rubber stamp. | 2026-06-03 |
| 10 | **Gherkin Author: business-focused, not code-focused** | Distills contracts from specs. Thinks in business terms, not implementation details. Production mindset. | 2026-06-03 |
| 11 | **Implementer is TDD Craftsman** | RED-GREEN-REFACTOR cycle. Three Laws of TDD are non-negotiable. | 2026-06-03 |
| 12 | **Reviewer loops back to Implementer** | If Gherkin scenarios are not fully covered or spec compliance fails, Reviewer sends back to Implementer before proceeding. | 2026-06-03 |
| 13 | **Mutation Tester: deterministic bash scripts, new files** | Mutants are written to new files (not replacing originals). Each mutant is tested. Survivors are recorded with file, line, mutation, test result. Hands back to Implementer. | 2026-06-03 |
| 14 | **Agent manifests use `.opencode/agents/` format** | OpenCode reads agents from `.opencode/agents/*.md` with YAML frontmatter. FreIA generates there, not in `.freia/agents/`. | 2026-06-03 |

---

## 9. Appendix: Terminology

| Term | Definition |
|------|------------|
| **Harness** | A reproducible dev tool or service configured by FreIA (e.g., devenv, testcontainers). |
| **Agent Manifest** | A markdown file with YAML frontmatter that defines an OpenCode agent's description, mode, model, permissions, and system prompt. |
| **Orchestrator** | The top-level agent that decides which subagents to invoke based on task complexity. |
| **Spec Agent** | Conversational partner for architecture proposals, human Q&A, and OpenSpec artifact authoring. |
| **Gherkin Author** | Business-focused agent that distills specs into Gherkin scenarios with a production mindset. |
| **TDD Craftsman** | Implementer agent following RED-GREEN-REFACTOR and the Three Laws of TDD. |
| **Reviewer** | Agent that validates implementation against spec and Gherkin coverage. |
| **Mutation Tester** | Agent that generates deterministic mutants and records surviving ones. |
| **Code Quality Gates** | Deterministic script that verifies format, types, lint, tests, and mutation testing pass. |
| **SDD** | Spec-Driven Development: agree on what to build before building it. |
| **OpenSpec** | The lightweight artifact-driven workflow (proposal → specs → design → tasks → verify → archive). |

---

*This document is a skeleton. Each module will be expanded into its own technical design and task breakdown as we iterate.*
