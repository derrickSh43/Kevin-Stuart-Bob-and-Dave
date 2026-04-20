# Autonomous Code Maintenance System (ACMS)

An open architecture exploration of how to build an unattended, agentic code maintenance system on AWS — inspired by Stripe's Minions engineering writeups, with original additions to the git coordination layer and tool security model.

This is a build-in-public project. The goal is to document the architecture, thinking, and tradeoffs as they develop — not to ship a finished product. Expect the design to evolve.

---

## What this is

A system for automating bounded, well-defined code maintenance tasks — things like fixing flaky tests, dependency upgrades, minor refactoring, and config changes. An engineer describes a task, the system executes it autonomously, and produces a pull request ready for human review.

This is not a general purpose AI coding assistant. It is code maintenance automation with an LLM in the middle of a tightly controlled workflow.

## What this is not

- A replacement for engineers or engineering judgment
- A system for building new features autonomously
- A general purpose coding agent
- Production ready software

---

## Architecture overview

The system is composed of several layers, each documented separately.

### Intake & task qualification

A FastAPI entry point receives tasks from Slack, CLI, or ticketing systems. An intake agent classifies and scopes each task before it enters the system. Tasks that are too large, too ambiguous, or architectural in nature are rejected back to a human. Only well scoped, bounded tasks enter the queue.

### Scaling controller

A Lambda running on a 60 second EventBridge schedule manages concurrency. It cleans up completed or stuck executions, then calculates how many new agents to start based on EC2 pool availability, inference API headroom, and budget limits. Controls the SQS pull rate. Step Functions executions are never modified mid-flight.

### Blueprint orchestration (Step Functions)

Each task gets its own ephemeral Step Functions execution. The state machine sequences deterministic steps (linting, pushing, CI) with agentic steps (implement, eval, fix). All inference steps are hard bounded — maximum 2 attempts before escalation. Every execution path ends with DestroyDevbox and SignalComplete regardless of outcome.

### Isolated execution environment (EC2 devboxes)

Each agent run executes inside a pre-warmed EC2 instance. The repo is cloned from CodeCommit over a VPC-internal connection at run start. The devbox is destroyed after the run. Nothing persists.

### Rule file system

Plain text rule files committed to the repo alongside code, scoped per directory. The agent collects rules by walking up the directory tree from the files it is working on. Minimal global rules, specific rules close to the code they describe.

### Governed toolshed (MCP)

The internal MCP server that exposes tools to the agent inside the devbox. Unlike an open MCP setup where any tool can register and be called, every tool in this system carries a declared identity and must be explicitly registered before it can be added to a workflow.

When a tool is added to a workflow, that event is recorded — what tool, what identity, what workflow, at what time. This means the audit trail starts at registration, not just at call time. There are no phantom tools. The agent cannot call something that has not been declared and approved.

Each devbox execution starts with a locked tool manifest derived from the Step Functions input for that run. The manifest is resolved against the tool registry at execution start and does not change mid-flight. The agent operates within that boundary for the lifetime of the run.

This builds on a pattern developed separately: the model narrates and reasons, but tools own state and execution. The policy layer sits between the agent and the tools — the agent does not call tools directly, it requests tool execution, and the governed MCP server decides whether to allow it.

### Ghost branch merge queue

A custom merge queue manages concurrent agent PRs safely. This is an original addition to the architecture — Stripe's published design treats git operations as a manual step. This system automates that coordination without removing human judgment from the promotion decision.

PRs are speculatively merged against an ephemeral ghost branch. CI runs against the ghost. On success the ghost is squashed to dev/testing and destroyed. Main is never touched directly — promotion to main is always a manual human action.

Multiple agents can work concurrently. The ghost branch serializes their PRs safely, catching integration conflicts before they reach dev/testing.

---

## Branch hierarchy

```
main (production)
    ↑ manual human promotion only

dev/testing
    ↑ human verification gate

ghost (ephemeral, created on demand)
    ↑ automated promotion on passing CI threshold
    ↑ destroyed after each promotion attempt

agent branches
    ↑ merge queue entry point, CI runs against ghost
```

---

## AWS services used

- **CodeCommit** — private git, VPC-internal, IAM auth
- **EC2** — pre-warmed devbox pool
- **Step Functions** — blueprint orchestration
- **Lambda** — all deterministic steps and the scaling controller
- **SQS** — task queue with DLQ as failsafe
- **EventBridge** — scaling controller schedule and CI result routing
- **Secrets Manager** — runtime credentials for LLM API and tooling
- **CloudWatch** — metrics and logging

---

## What is documented so far

- [x] Rule file system — structure, loading logic, CodeCommit storage
- [x] Task intake and qualification layer — classification, file ownership, conflict prevention
- [x] Scaling controller Lambda — concurrency management, cleanup, cost model
- [x] Step Functions state machine — full blueprint with failure states and inference bounds
- [x] Ghost branch merge queue — lifecycle, conflict detection, CodeCommit operations
- [x] Branch hierarchy — agent branches through to production
- [x] Governed toolshed — tool identity model, registration audit, locked execution manifests

## What still needs to be defined

- [ ] Blueprint internals — how deterministic and agentic nodes are structured within each Lambda step
- [ ] Each Lambda step — detailed implementation of every state machine node
- [ ] Devbox pool management — pre-warming strategy, pool sizing, EC2 configuration
- [ ] Eval step — how agent output quality is assessed before linting
- [ ] Observability — structured run telemetry, cost tracking per run, failure analysis
- [ ] Intake agent — full classification logic and task scoping rules
- [ ] Security model — IAM role scoping, VPC design, network isolation, MCP policy rules

---

## Status

Early architecture phase. Everything here is design and documentation — no running code yet. The goal right now is to get the architecture right before writing any implementation.

Feedback, questions, and challenges to the design are welcome.

---

## Inspiration

Stripe's public engineering writeups on their Minions system:

- [Minions: Stripe's one-shot, end-to-end coding agents — Part 1](https://stripe.com/blog/minions-real-time-software-maintenance)
- [Minions: Stripe's one-shot, end-to-end coding agents — Part 2](https://stripe.com/blog/minions-how-we-built-it)

This project is an independent exploration of similar concepts on AWS, with original additions to the git coordination and tool security layers. It is not affiliated with Stripe.
