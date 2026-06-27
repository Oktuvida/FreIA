# FreIA — Product Requirement Document

> **Status:** Skeleton v0.3 — iterative module-by-module construction.  
> **Target Agent:** OpenCode (initial adapter).  
> **Inspiration:** Gentle-AI (ecosystem configuration), OpenSpec (SDD artifact workflow).

---

## 1. Vision & Problem Statement

**What FreIA is:**  
A project-level CLI tool that bootstraps an AI-native development ecosystem inside a repository. It does not install global AI agents — it installs *into* a project the workflows, agent instructions, tooling harness, and SDD scaffolding so that OpenCode (and later others) can operate with spec-driven, test-first, integration-heavy discipline.

**What FreIA is NOT:**  
A global multi-agent installer (that is Gentle-AI's scope). FreIA is local and opinionated. It is also **not a loose scaffolder**: it does not hand you a pile of optional configs and wish you luck. FreIA is a *framework* — the toolchain and workflow are prescribed and enforced, the way a web framework prescribes its structure.

**Two halves, cleanly split:**  
- The **`freia` CLI** is a deterministic Go binary: no LLM, no network. It bootstraps the skeleton so FreIA can *exist* in the repo.
- The **provider init** (inside OpenCode) is where intelligence lives: it defines the stack and authors the project-specific scripts. Until it runs, the scaffolded scripts fail loudly by design.

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
| **Sad-path first-class** | Tests must cover error validation and failure modes, not just the happy path. Invalid input, boundary conditions, partial failures, and contract violations are required scenarios — not optional extras. |
| **Integration over Unit** | Prioritize integration tests with real dependencies (testcontainers, local services) over mocked unit tests. |
| **Mutation Testing** | Verify test suite strength by deliberately introducing mutants. Surviving mutants reveal weak or missing tests. |
| **Profiling as escape hatch** | When behavior is not understood — a test fails for unclear reasons, performance is off, or output is inexplicable — agents stop guessing and run deterministic profiling/diagnostics (CPU/mem profiles, traces, logs) before changing code. Profiling is the escape gate out of "I don't know what's going on". |
| **Understand before you test (brownfield)** | On existing codebases, the project is mapped and understood — via deterministic scripts, not AI guessing — before any test or change is written. |
| **Local Reproducibility** | Every test and service runs locally via devenv / testcontainers / docker-compose. No cloud-only infra. |
| **Brownfield-first** | Works on existing codebases, not just greenfield. Incremental adoption. |
| **Strict framework, not a buffet** | The toolchain and workflow are opinionated and mandatory. There is one FreIA way; tools are pillars, not à la carte options. |
| **Deterministic CLI, LLM at the edges** | The `freia` binary never calls an LLM or the network. Anything requiring intelligence happens inside the agent provider (Phase B). The CLI is reproducible; the intelligence is replaceable. |
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

**Test flags:** The quality-gates script and the relevant FreIA/test commands accept flags so gates and tests can be run selectively. Two levels:

1. **Gate selection** — run a subset of the gate pipeline: `--unit`, `--integration`, `--e2e`, `--mutation`, `--format`, `--lint`, `--skip-e2e`, `--only <gate>`, etc. Fast inner loops skip the slow gates; CI runs them all.
2. **Native runner passthrough** — forward flags to the language-native runner for focused execution during RED-GREEN (e.g. `-run <pattern>`, focus/skip, Gherkin tag filters like `--tags @auth`).

Selective runs are a developer convenience only — DONE still requires a full `freia-quality-gates.sh` (exit 0, all gates).

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
│   ├── scout              → Deterministic code-intel scripts (dedup, dead code, deps)
│   └── project            → Deterministic repo inspection (greenfield vs brownfield; no stack interpretation)
├── agent-manifests/       → Templates for OpenCode agents
│   ├── freia-init.md      → Bootstrap agent: Phase B — define stack, author scripts
│   ├── orchestrator.md    → Orchestrator prompt & decision logic
│   ├── legacy-scout.md    → Legacy Scout: map & understand existing code (brownfield)
│   ├── diagrammer.md      → Diagrammer: architecture diagrams (Mermaid/C4)
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
│            Legacy Scout  (brownfield)       │
│  - Runs BEFORE any test or change           │
│  - Uses deterministic scripts, NOT AI, to   │
│    map the code: dedup, dead code, deps,    │
│    entrypoints, test coverage gaps          │
│  - Produces a factual project map           │
│  - Feeds the Diagrammer and the Spec Agent  │
└──────────┬──────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────┐
│            Diagrammer                       │
│  - Turns the scout's map (brownfield) or    │
│    the proposed design (greenfield) into    │
│    Mermaid / C4 diagrams                    │
│  - Visual contract for humans + agents      │
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
| **freia-init (bootstrap)** | Phase B: defines the stack, authors the project-specific quality-gates/mutation/scout scripts, seeds `design.md` for frontends, replacing the CLI's erroring placeholders. | Once, on first run after `freia init` | `subagent` |
| **Orchestrator** | Decides *how* to execute given task complexity. Loads spec context. Enforces quality gates. | Every task | `primary` |
| **Legacy Scout** | Maps and understands an existing codebase via deterministic scripts (dedup, dead code, dependency graph, entrypoints, coverage gaps) — never AI guessing. Produces a factual project map. | Brownfield: before testing/changing unfamiliar code | `subagent` |
| **Diagrammer** | Renders architecture diagrams (Mermaid/C4) from the scout's map (brownfield) or the proposed design (greenfield). | After Scout (brownfield) or after Spec/Design (greenfield) | `subagent` |
| **Spec Agent** | Conversational partner. Proposes architecture, asks human questions, debates decisions, writes OpenSpec artifacts. | Task is medium+ or touches architecture | `subagent` |
| **Gherkin Author** | Distills contracts from specs into Gherkin scenarios. Business-focused, production mindset. | After Spec Agent produces specs | `subagent` |
| **TDD Craftsman** | Writes code following RED-GREEN-REFACTOR. Three Laws of TDD. | Always, either solo (small) or post-spec (medium+) | `subagent` |
| **Reviewer** | Validates diff against spec + Gherkin scenarios. Checks all scenarios have at least one concrete test. Loops back to Implementer if gaps found. | After implementation on medium+ tasks | `subagent` |
| **Mutation Tester** | Generates deterministic mutants, runs tests, records survivors. Hands back to Implementer. | After Reviewer approves | `subagent` |

**Complexity Heuristics (v1):**

"Small" is **not** defined by file count. A small task often touches more than one file — production code plus its tests is the normal case, and a one-line behavior change can still require new test files. The real signals are:

- **Small:** No new contract or interface, no architecture decision, no new dependency, the change is fully described by its tests, and the blast radius is local and obvious. May span several files (code + tests + a fixture). → Orchestrator → TDD Craftsman solo.
- **Medium:** Introduces or changes an interface/contract, needs new behavior scenarios, or its blast radius is non-obvious. → Orchestrator → (Legacy Scout if brownfield) → Spec → Gherkin Author → TDD Craftsman → Reviewer → Mutation Tester.
- **Large:** New service, DB schema, infra change, or cross-cutting architecture. → Full SDD flow: (Scout) → Spec → Design → Diagrammer → Gherkin Author → Tasks → TDD Craftsman → Reviewer → Mutation Tester → Verify → Archive.

**When in doubt, ask.** If the Orchestrator cannot confidently classify a task as Small, it does **not** silently pick a path — it asks the human: *use the full SDD flow, or proceed as a direct change?* The human decides. Defaulting to "small" on an ambiguous task is the failure mode this rule exists to prevent.

---

## 4. Module Breakdown

### 4.1 M1 — CLI Skeleton & Init (deterministic, offline)

**Goal:** `freia init` lays down everything FreIA needs to *exist* in the repo — **with zero LLM and zero network**. Anything that requires intelligence (stack definition, script bodies, design.md content) is deliberately left undefined and is produced later by the **provider-side init** (§4.1.1).

This split is a hard rule: the CLI is deterministic and reproducible; the LLM never runs inside the binary.

#### 4.1.1 Two-phase init

| Phase | Runs where | LLM? | Produces |
|-------|-----------|------|----------|
| **A — `freia init`** | FreIA CLI (Go binary) | **No** | `.freia/` skeleton, agent manifests, devenv backbone scaffold, **erroring placeholder scripts**, empty SDD scaffold |
| **B — provider init** | Inside OpenCode (FreIA installs a `freia-init` agent/command) | **Yes** | Defines the stack, authors the project-specific scripts (quality gates, mutation, scout), seeds `design.md` when a frontend exists, fills `config.yaml` |

**Phase A — `freia init` (CLI) does:**
1. Decides **greenfield vs brownfield** by inspecting whether the repo already has source (presence of files, not what they mean) and records `mode` in config. No stack *interpretation* happens here.
2. Generates the `.freia/` directory:
   - `.freia/config.yaml` — skeleton config: `mode`, **per-agent model overrides** (§4.1.6), and an empty `stack:` block marked `undefined`.
   - `.freia/agents/` — OpenCode agent manifests (orchestrator, legacy-scout, diagrammer, spec, gherkin-author, implementer, reviewer, mutation-tester) — including the **`freia-init` bootstrap agent** that runs Phase B.
   - `.freia/skills/` — reusable skill prompts (TDD, integration-test pattern, etc.).
   - `.freia/openspec/` — SDD artifact scaffold (changes/, archive/).
   - `.freia/scout/` — placeholder code-intel scripts (§4.1.4).
3. Scaffolds the **devenv backbone** (`devenv.nix` + `.envrc`) as the single, mandatory entrypoint for the toolchain (§4.2). Not optional, not à la carte.
4. Writes **erroring placeholder scripts** (`freia-quality-gates.sh`, the mutation runner, the scout scripts). Each exits non-zero with a clear message: *"not yet defined — run `freia-init` (provider) to generate this for your stack."* Nothing silently no-ops.

**Phase A explicitly does NOT:** detect or interpret the stack, choose tools, or author any script body. There is no LLM and no guessing in the CLI.

#### 4.1.2 Provider init (Phase B, LLM)

FreIA installs a **`freia-init` bootstrap agent** into the provider. On first run it:
1. Defines the stack (language(s), test layout, frontend presence) and writes it into `config.yaml`.
2. Authors the **project-specific** quality-gates script (§2.2) and **mutation runner** (§4.1.3) for the detected stack/layout — replacing the erroring placeholders.
3. Authors the deterministic scout scripts for the stack (§4.1.4).
4. Seeds `design.md` if a frontend exists (§4.1.5).
5. In brownfield, hands off to the **Legacy Scout** to map the existing code before any change.

Until Phase B completes, the placeholders fail loudly — that is the signal the ecosystem isn't defined yet, not a bug.

#### 4.1.3 Mutation script (defined in Phase B)

Mutation tooling is language- and layout-specific, so the CLI cannot write it. The provider init generates the mutation runner configured for *this* project: the right tool for the stack (`go-mutesting`, `Stryker`, `mutmut`, …), correct source/test paths, sensible operators, and a deterministic survivor-report format the Mutation Tester consumes. It plugs into `freia-quality-gates.sh` as the mutation gate.

#### 4.1.4 Code intelligence: scripts, not AI

To find existing code — duplication, dead code, dependency edges, entrypoints, coverage gaps — FreIA uses **deterministic tooling, never the LLM** (AI hallucinates references, misses call sites, is non-reproducible). The Legacy Scout *runs and interprets* these scripts; it does not eyeball the codebase. The CLI lays down placeholders in `.freia/scout/`; Phase B fills them with stack-appropriate tools (e.g. `jscpd`/`dupl` for dedup, `deadcode`/`ts-prune`/`vulture` for dead code, language-native dependency graphers).

#### 4.1.5 design.md for frontends

When the stack defined in Phase B includes a frontend (React, Vue, Svelte, etc.), the provider init seeds `design.md` in the SDD scaffold so visual/UX decisions, component architecture, and interaction contracts are specified before code. Backend-only projects don't get it by default.

#### 4.1.6 Customizable agent models

`.freia/config.yaml` carries a `models` map so each agent's model is overridable, with a sensible default per agent. Cheap/mechanical agents (e.g. archive, diagrammer) can use a small model; architectural agents (Spec, Design) a stronger one. `freia agent sync` writes the resolved `model:` into each `.opencode/agents/*.md` frontmatter. This is part of the deterministic Phase A config skeleton.

```yaml
# .freia/config.yaml (excerpt)
models:
  default: anthropic/claude-sonnet-4-20250514
  spec: anthropic/claude-opus-4-20250514     # architecture → stronger model
  diagrammer: anthropic/claude-haiku-4-20250514
  legacy-scout: anthropic/claude-haiku-4-20250514
```

#### 4.1.7 Greenfield vs Brownfield flow

| Aspect | Greenfield (new project) | Brownfield (joining existing code) |
|------|--------------------------|-------------------------------------|
| Code understanding | Nothing to scan — go straight to design | **Legacy Scout runs first**: deterministic scripts map the code before anything is tested or changed |
| Diagrams | Diagrammer renders the *proposed* architecture from the spec/design | Diagrammer renders the *existing* architecture from the scout's map |
| First artifact | `proposal.md` / `design.md` seeded from the user's intent | Project map + diagrams + a baseline coverage/dead-code report |
| Active agents | Full pipeline minus Scout (no legacy to scout) | Full pipeline including Scout, which gates the rest |

The Orchestrator reads `mode` from `.freia/config.yaml` to know which flow it is in.

**Out of scope v1:**  
- Global install. Everything is project-local.
- Multi-agent support beyond OpenCode.

### 4.2 M2 — Harness Manager (opinionated, devenv-backed)

**Goal:** manage the mandatory toolchain that the FreIA framework prescribes.

**Framework principle:** FreIA is a **strict framework, not a buffet.** The toolchain — devenv, testcontainers, a Gherkin runner, local services — is a set of **mandatory pillars**, not à la carte options. There is one FreIA way and it is opinionated by design.

**Two complementary layers — devenv and a container runtime.** These are not the same thing and one does not provision the other:

- **devenv is the *environment/runner* backbone.** Via nix it provides language toolchains, CLIs, libraries (including the testcontainers bindings and the Gherkin runner), env vars, and project scripts. It makes the developer environment reproducible. FreIA owns *what* the environment must contain and pins it; devenv/nix owns *how* it is materialized.
- **A container runtime (Docker/Podman) is the *infra* layer.** testcontainers and `docker-compose.dev` need a live container backend on the host. **devenv does NOT provision the Docker/Podman daemon** — it is a host-level prerequisite, outside nix's reproducibility. devenv can ship the *client* CLI, but the daemon/socket is the host's responsibility.

So the user installs **two** things, not one: **devenv** (env backbone) and a **container runtime** (Docker or Podman). Everything language-level flows from devenv; the container daemon is host infra that testcontainers builds on.

**`freia doctor` fails hard.** It checks both layers: devenv present, **and** a usable container runtime (daemon reachable, socket available). If either is missing — or the environment doesn't provide a required pillar — doctor exits non-zero. There is no "warning, continue anyway." Strict means strict.

**Prescribed pillars (v1):**
| Pillar | Layer | Role | Provisioned by |
|------|------|------|----------------|
| `devenv` | env/runner | Reproducible environment backbone | user (manual install) |
| container runtime (Docker/Podman) | infra | Host daemon that containers run on | user (manual install) — **not** devenv |
| `testcontainers` | infra (on runtime) | Integration tests against real dependencies | bindings via devenv; **requires the container runtime** |
| `gherkin` runner | env/runner | BDD acceptance tests | devenv |
| local services | infra (on runtime) | postgres, redis, etc. for integration/e2e | `docker-compose.dev` on the container runtime |

`freia harness add/remove` tunes *configuration within* this prescribed set (versions, which services); it does not make the pillars optional.

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
- `.opencode/agents/freia-init.md` — bootstrap subagent, Phase B: stack definition + script authoring.
- `.opencode/agents/orchestrator.md` — primary agent, decision logic, complexity heuristics, task permissions for all subagents.
- `.opencode/agents/legacy-scout.md` — subagent, brownfield code mapping via deterministic scripts.
- `.opencode/agents/diagrammer.md` — subagent, Mermaid/C4 diagrams from scout map or design.
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

**TDD-aware tasks (requirement):** `tasks.md` is not a flat checklist of files. Each task is authored as a RED → GREEN → REFACTOR slice: the failing test(s) to write first, the minimal production code to make them pass, then the refactor — including the **sad-path** scenarios (§2). A task is the smallest behavior that can be driven by a failing test, which is why a "small" task routinely spans code + tests (§3.3). 

> **Open item:** align this task format with how OpenSpec actually generates `tasks.md`, so FreIA's TDD slicing layers cleanly on top of the OpenSpec convention rather than diverging from it. Tracked, not yet resolved.

### 4.5 M5 — Quality Gates Script Generator

**Goal:** define the quality-gates contract and produce the project's script.

**Split across the two phases:**
- **Phase A (CLI, deterministic):** lays down `scripts/freia-quality-gates.sh` as an **erroring placeholder** that encodes the *fixed contract* — gate order, exit semantics (0 = all pass), and the flag interface (§2.2). It contains no stack commands and fails loudly until Phase B.
- **Phase B (provider, LLM):** fills in the **stack-specific commands** for each gate based on the defined stack, turning the placeholder into a working script.

This keeps the *contract* deterministic and version-controlled while the *commands* — which need stack knowledge — are authored by the agent.

**Gate execution order (fixed, set by Phase A):**
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
- If the task is ambiguous to classify, asks the human: full SDD flow or direct change (see §3.3).
- If brownfield and the code is unfamiliar: runs Legacy Scout → Diagrammer first to build understanding.
- If small: delegates to TDD Craftsman via OpenCode's Task tool.
- If medium+: runs (Scout → Diagrammer if brownfield) → Spec Agent → Gherkin Author → TDD Craftsman (per task) → Reviewer → Mutation Tester.
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
- [ ] **Cover the sad path** — error validation and failure modes, not just the happy path.
- [ ] **No test without a Gherkin scenario** for integration/acceptance.
- [ ] **Profile before guessing** — when behavior isn't understood, run diagnostics, don't speculate.
- [ ] **Find code with scripts, not the LLM** — dedup/dead-code/deps come from deterministic tools.
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
| **M1** | CLI skeleton + `freia init` (deterministic, no LLM) + `.freia/` scaffold + erroring placeholders | `freia init` runs offline, produces a valid skeleton, and placeholder scripts fail loudly until Phase B |
| **M1.5** | Provider init (`freia-init` agent, Phase B): stack definition + script authoring | First run inside OpenCode defines the stack and replaces placeholders with working scripts |
| **M2** | Harness manager: devenv backbone + prescribed pillars | `freia init` produces a working devenv.nix; `devenv shell` provides the toolchain; `freia doctor` fails hard if devenv is missing |
| **M3** | Agent manifest generator: OpenCode agent prompts | `.opencode/agents/` contains all agents (incl. freia-init bootstrap, Legacy Scout, Diagrammer) with proper frontmatter |
| **M4** | SDD scaffold: `freia sdd new` + OpenSpec artifact layout | Running `freia sdd new auth` creates proposal/spec/design/tasks files |
| **M5** | Quality gates: `freia quality init` + deterministic script generation | `freia-quality-gates.sh` runs format/typecheck/lint/test/mutation |
| **M6** | End-to-end demo: Orchestrator → Spec → Gherkin Author → TDD Craftsman → Reviewer → Mutation Tester | A real feature implemented via the full SDD flow |
| **M7** | Multi-harness ecosystem: docker-compose.dev, observability, local tracing | Local infra reproducible with one `devenv up` |
| **M8** | Multi-agent adapter: beyond OpenCode (Cursor, Claude Code, etc.) | Gentle-AI-style per-agent manifest generation |

---

## 8. Decisions Log

| # | Decision | Rationale | Date |
|---|----------|-----------|------|
| 1 | **No bundled binaries** *(amended by #26)* | Tools like devenv/testcontainers are complex external deps; FreIA does not ship or compile them. Originally: user installs each binary. **Amended:** the user installs devenv (env backbone) + a container runtime (Docker/Podman); devenv provides the language-level toolchain from there. | 2026-06-03 |
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
| 15 | **Sad-path testing is first-class** | Tests must cover error validation and failure modes, not just the happy path. Invalid input, boundaries, partial failures, and contract violations are required scenarios. | 2026-06-27 |
| 16 | **Profiling is the escape hatch** | When behavior isn't understood, agents run deterministic profiling/diagnostics before changing code instead of guessing. | 2026-06-27 |
| 17 | **Legacy Scout agent (brownfield)** | A scouting agent maps and understands existing code before any test or change. Gates the brownfield pipeline. | 2026-06-27 |
| 18 | **Code intelligence via scripts, not AI** | Dedup, dead code, dependency graphs, and coverage gaps are found with deterministic tools, never the LLM (which hallucinates and is non-reproducible). The Scout runs and interprets them. | 2026-06-27 |
| 19 | **Diagrammer agent** | A dedicated agent renders Mermaid/C4 diagrams — from the scout's map in brownfield, from the proposed design in greenfield. | 2026-06-27 |
| 20 | **Greenfield vs brownfield are distinct init flows** | `freia init` branches on whether the repo is new or existing; the flows differ in scanning, active agents, and seeded artifacts. Mode is stored in config. | 2026-06-27 |
| 21 | **init generates a project-specific mutation script** | Mutation tooling is stack/layout-specific; init wires the correct tool and paths, not a generic placeholder. | 2026-06-27 |
| 22 | **Per-agent customizable models** | `.freia/config.yaml` carries a `models` map; each agent's model is overridable with a per-agent default. `agent sync` resolves it into frontmatter. | 2026-06-27 |
| 23 | **design.md auto-seeded for frontends** | When a frontend is detected, `freia init` seeds `design.md` so UI/UX and component architecture are specified before code. | 2026-06-27 |
| 24 | **Tasks are TDD slices; "small" ≠ one file** | Tasks are authored as RED-GREEN-REFACTOR slices (incl. sad-path); a small task is defined by blast radius and contract impact, not file count. When classification is ambiguous, the Orchestrator asks the human: SDD flow or direct change. | 2026-06-27 |
| 25 | **Selective test flags** | Quality-gates script and test commands accept flags for gate selection (`--unit`, `--mutation`, `--skip-e2e`…) and native-runner passthrough; DONE still requires a full gate run. | 2026-06-27 |
| 26 | **Strict framework, devenv + runtime** | FreIA is opinionated: the toolchain is mandatory pillars, not à la carte. **Two complementary layers:** devenv (env/runner backbone, via nix) and a container runtime (Docker/Podman, host infra). devenv does NOT provision the Docker daemon. User installs both; `freia doctor` checks both and fails hard. Amends #1. | 2026-06-27 |
| 27 | **Deterministic CLI, LLM-free** | The `freia` binary makes no LLM/network calls. It only lays down the skeleton + erroring placeholders. Reproducibility and offline operation over convenience. | 2026-06-27 |
| 28 | **Two-phase init** | Phase A (`freia init`, CLI, deterministic) bootstraps; Phase B (`freia-init` agent inside the provider, LLM) defines stack and authors project-specific scripts. Placeholders fail loudly until Phase B runs. | 2026-06-27 |

---

## 9. Appendix: Terminology

| Term | Definition |
|------|------------|
| **Harness** | A mandatory pillar of the FreIA toolchain (devenv, testcontainers, Gherkin runner, local services), prescribed by the framework — not optional. |
| **devenv backbone** | The env/runner layer (via nix): language toolchains, CLIs, libraries, scripts — reproducible via its lockfile. Complementary to, and distinct from, the container runtime. |
| **Container runtime** | The host-level infra layer (Docker/Podman daemon) that testcontainers and local services run on. A manual host prerequisite — devenv does not provision it. |
| **Phase A / `freia init`** | The deterministic, LLM-free CLI step that bootstraps the skeleton and erroring placeholders. |
| **Phase B / provider init** | The LLM-driven step (the `freia-init` agent inside OpenCode) that defines the stack and authors project-specific scripts. |
| **Erroring placeholder** | A scaffolded script that exits non-zero with a clear message until Phase B defines it — never a silent no-op. |
| **Agent Manifest** | A markdown file with YAML frontmatter that defines an OpenCode agent's description, mode, model, permissions, and system prompt. |
| **Orchestrator** | The top-level agent that decides which subagents to invoke based on task complexity. |
| **Legacy Scout** | Brownfield agent that maps and understands existing code via deterministic scripts before any testing or change. |
| **Diagrammer** | Agent that renders Mermaid/C4 architecture diagrams from the scout's map (brownfield) or proposed design (greenfield). |
| **Greenfield / Brownfield** | New project vs joining an existing codebase. `freia init` runs distinct flows for each. |
| **Sad-path** | Error/failure scenarios (invalid input, boundaries, partial failures, contract violations) that tests must cover alongside the happy path. |
| **Profiling escape hatch** | The rule that agents run deterministic diagnostics/profiling — instead of guessing — when behavior isn't understood. |
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
