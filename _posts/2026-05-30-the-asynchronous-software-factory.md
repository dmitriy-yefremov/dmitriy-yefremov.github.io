---
layout: post
title: The Asynchronous Software Factory
---

## The Asynchronous Software Factory

For the last six months, I’ve been experimenting with different ways to build software using AI.

The space of options is wide. At one end, you sit in front of the screen and handhold an agent through every step - accept this diff, reject that one, nudge it back on track. At the other end, you let a swarm of agents loose to tackle a problem fully independently, OpenClaw-style.

Most of us live near the handholding end. That's where current tooling pushes us: prompting, re-prompting, and watching tokens stream by in real time. I've come to think of that as *babysitting*, and it caps both our leverage and our patience.

I don't think the handholding end is where the real value lies. But I want to be clear about what I'm optimizing for: **institutional software development, not prototyping.** YOLO mode - turning an agent loose and hoping for the best - is the wrong tool when you need something predictable and repeatable that a team will maintain and live with for years. For this kind of work, boring is good. The goal is a process you can trust to behave the exact same way on the hundredth feature as it did on the first.

This post explores the autonomous end of the spectrum and introduces [`software-factory`](https://github.com/dmitriy-yefremov/software-factory) - a prototype I am working on to get there. It’s heavily inspired by Jamon Holmgren’s [Night Shift](https://jamon.dev/night-shift).

## Three bets about where things are going

Everything below rests on three opinions I hold about the near future of software development.

**Software development is going asynchronous.** We don't ask a coworker to do a task and then stand behind them watching their screen. Instead, we hand them the work and expect them to return when it's done-or when they're genuinely stuck and have a question. The same should be true of AI agents. I want to delegate and walk away, not monitor execution in real time.

**The future is spec-driven.** Describe the end state and let the system determine how to build it. The human artifact of value is a precise description of *what* and *why*, not a running dialog on *how*. GitHub's [spec-driven development toolkit](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/) and OpenAI's [harness engineering](https://openai.com/index/harness-engineering/) writeups confirm this is where the industry is heading. The introduction of `/goal` in tools like Claude Code and Codex is another clear validation of this shift.

**A "software factory" is the right metaphor.** Picture a conveyor belt: a fully automated assembly line that takes a specification in one end and emits reviewed, tested, and integrated code out the other. Humans design the line and feed it work; the line does the building.

There is a direct parallel here with the shift from imperative to declarative programming. Prompting an agent step-by-step is *imperative*: you specify *how*, instruction by instruction. Building a factory is *declarative*: you specify *what* you want (the spec) and let the system determine how to produce it. Functional programmers gave up manual control over execution order and received composability and reliability in return. This is the exact same trade, applied to how we direct the machines that write our code.

## The core idea: two non-overlapping phases

The factory works by splitting the engineering process into two distinct phases that never overlap. Human deep work happens synchronously during working hours, while machine execution happens asynchronously in the background, entirely unattended.

### Phase 1: Human definition & review (synchronous)

During working hours, engineers spend their attention exclusively on high-context, high-value tasks:

* **Architecture & specs:** You work *with* a synchronous agent to produce highly detailed specifications. The spec is where you organize technical thinking, lay out system architecture, and pin down edge cases. This is the real engineering work, and it is worth doing exceptionally well.
* **Review & validate:** You review what the factory produced overnight and validate results that an automated test suite can’t fully verify - such as eyeballing a data diff after a database migration or checking visual UI layouts.
* **Factory maintenance:** This is the critical mindset shift. When the AI stumbles, fix the factory floor, not the code. Patch the agent prompts, test suite, docs, or lint rules that let the mistake through - so that entire *class* of error can never happen again.

### Phase 2: The AI assembly line (asynchronous)

Once the specs are set, you queue the work, trigger the agents, and step away. In the background, a multi-agent assembly line takes over with no human on the happy path:

1. **Plan:** An agent reads the spec, studies the codebase, and writes a concrete implementation plan.
2. **Pre-implementation review:** *Before any application code is written*, a separate reviewer agent critiques that plan-grounded in the project's own constitution-and surfaces the strongest objections it can find. Catching a bad architectural design here is far cheaper than catching it in a massive git diff.
3. **Implement & iterate:** An implementer agent turns the approved plan into code, runs strict linters and the test suite, and iterates in a closed feedback loop until everything is green.
4. **Post-implementation review:** A code-review agent reviews the diff against the spec, the plan, and the acceptance criteria. Its findings go back to the implementer to be addressed.

## How `software-factory` actually implements this

The vision above is compelling, but vision alone doesn't ship code. `software-factory` is a **spec-first, agent-built, autonomously running** template that you fork into your project and customize. It turns a written spec into reviewed, CI-green, merged code with no human in the loop.

A few design choices are worth calling out, because they are what make unattended execution actually safe:

* **Specifications and design choices are durable:** All past specifications and Architecture Decision Records (ADRs) live in the repository and remain fully accessible to agents, maintaining context and architectural consistency over time.
* **PRs are the shared context:** Pull requests offer a natural, GitHub-native abstraction to keep shared context between agents and maintain a human-readable log for verification. Once merged, however, they are no longer referenced for future work, preventing historical clutter from slowing down current runs.
* **Deterministic order of operations:** I chose *not* to use agentic orchestration to dynamically define the sequence of operations on the conveyor belt. Instead, I implemented a strict state machine using GitHub Actions. Each operation on the assembly line is a discrete workflow, and the state transitions are fixed. This trades the flexible reasoning of an agent-led orchestrator for simplicity, predictability, and strict guardrails.
* **Labels are the state machine:** Every transition in the pipeline is a GitHub label swap, driven by GitHub Actions. A spec PR moves systematically through:

  `state:queued` → `state:plan` → `state:plan-review` → `state:implementation` → `state:implementation-ci` → `state:code-review` → `state:merge`.

  There is no hidden orchestrator state to lose; the PR's labels *are* the state, fully visible to anyone visiting GitHub.

Here's how it works in practice: a human writes a specification in `specs/NNNN-name.md`, opens a pull request with that file, and labels it `state:queued`. From that moment forward, the assembly line takes over - no human intervention needed until the PR lands on main or hits a blocker that requires attention.

```
human writes specs/NNNN-name.md, opens PR, labels it state:queued
        │
        ▼  scheduler picks the next queued PR, rebases onto main
   state:plan ──▶ planner writes the implementation plan
        │
   state:plan-review ──▶ plan-reviewer posts APPROVE / REQUEST_CHANGES
        │ approve
   state:implementation ──▶ implementer pushes code commits
        │
   state:implementation-ci ──▶ CI runs; ci-gate routes on the result
        │ green
   state:code-review ──▶ code-reviewer posts APPROVE / REQUEST_CHANGES
        │ approve
   state:merge ──▶ auto-merger rebases, checks guardrails, squash-merges
        │
      main ◀── scheduler advances to the next queued PR
```

* **Every agent runs in a fresh session with no shared context:** Independent critique is crucial because a reviewer who inherits the planner's context easily becomes a "yes-man," agreeing with earlier reasoning rather than questioning it. Fresh sessions force each agent to ground its work strictly in the spec and codebase. As a bonus, this also lets you run different LLM models for different agents.
* **Failure is bounded, and parking is a first-class outcome:** A `REQUEST_CHANGES` from a reviewer or a failed CI run triggers automatic retries, but only up to two times per stage. Anything that cannot make progress lands at `state:needs-attention`, where a human takes over. The factory is designed to give up loudly rather than thrash silently and run up your API bill.
* **Auto-merge is gated by guardrails:** When a PR reaches `state:merge`, the auto-merger rebases onto `main`, runs an eligibility check, and only then squash-merges the changes. If a guardrail trips, it parks the PR instead of merging. The scheduler then safely advances to the next queued spec.

It ships as a **fork-and-own template**: it carries the orchestration machinery and a generic TypeScript/pnpm skeleton, but *none* of your product's actual code. You fork it, run a single script to stamp it with your project’s name, write your "constitution" (`CLAUDE.md` and `docs/principles.md` - the files the agents read first), file your first spec, and let the cascade run.

## Building the factory floor

An unattended factory manufactures technical debt at machine speed, unless the floor is specifically built to prevent it. Three investments are non-negotiable:

* **Bulletproof automated testing:** The asynchronous loop simply does not work without a robust test harness. The agents rely entirely on test *failures* to self-correct in the background; the red bar is their steering wheel. Weak tests mean a confidently wrong factory.
* **Systematic documentation:** Centralized routing documents like an `AGENTS.md` or `CLAUDE.md` point agents to the right workflow rules, domain knowledge, and architectural guidelines. This allows individual specs to stay short. The shared context lives in one place in the docs, rather than being copied and pasted into every spec.
* **Strict static analysis:** Crank typing and linting to the strictest settings. What feels restrictive to a human developer is an essential guardrail for an autonomous agent. It is the difference between an agent that wanders and one that is fenced into a correct solution space.

## What I've learned so far

I've been running this on a project for about a month, and it has written a substantial amount of real, production code in that time. A few honest takeaways:

* **The upfront cost is real:** Standing up the factory and tweaking it as agents find new ways to fail is heavy work. In the short term, it's much easier to just prompt an agent and accept the change. The factory only pays off once you stop hand-fixing code and start fixing the floor.
* **When it works, it's genuinely amazing:** Filing a spec, walking away, and coming back to a reviewed, CI-green, merged PR is a completely different category of developer experience from babysitting a chat window.
* **The jury is still out on scale:** I've run this with dozens of specs and ADRs. I don't yet know how it behaves with *hundreds* - whether the accumulated constitution keeps the agents on track or starts to buckle under its own weight. As the constitution grows, will agents struggle to maintain internal consistency across conflicting principles? Will context windows become a bottleneck, forcing me to truncate or summarize older ADRs? And will contradictions naturally emerge in the guidance itself, leaving agents confused about which principle to follow? That is the part I'm most curious about.

Overall the approach looks promising for long-lived, institutional work and worth the investment. But it's still an experiment, not a finished solution.

***

If you want to dig in, the template is open source on GitHub: [dmitriy-yefremov/software-factory](https://github.com/dmitriy-yefremov/software-factory). It is early and opinionated, and I would love to hear your feedback!