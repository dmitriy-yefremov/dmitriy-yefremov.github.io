---
layout: post
title: The asynchronous software factory
---

For the last 6 month I've been experimenting with different ways to build software using AI.
The space of options is wide. At one end you sit in front of the screen and handhold an agent
through every step - accept this diff, reject that one, nudge it back
on track. At the other end you let a swarm of agents loose to go at a problem fully
independently, OpenClaw-style. Most of us live near the handholding end, because that's where
the tooling pushes us - prompting, re-prompting, and watching tokens stream by in real time.
I've come to think of that as *babysitting*, and it caps both our leverage and our patience.

I don't think the handholding end is where the value is. But I also want to be clear about
what I'm optimizing for: **institutional software development, not prototyping.** YOLO mode 
turn an agent loose and hope for the best -
is the wrong tool when you need something predictable and repeatable that a team will live
with for years. For this kind of work, boring is good. The goal is a process you can trust to
behave the same way on the hundredth feature as it did on the first.

This post is about the other end of the spectrum, and about
[`software-factory`](https://github.com/dmitriy-yefremov/software-factory) - a working
template I built to get there. It's heavily inspired by Jamon Holmgren's
[Night Shift](https://jamon.dev/night-shift).

## Three bets about where this is going

Everything below rests on three opinions I hold about the near future of software
development.

**Software development is going asynchronous.** We don't ask a coworker to do a task and
then stand behind them watching their screen. We hand them the work and expect them to come
back when it's done - or when they're genuinely stuck and have a question. The same should be
true of AI agents. I don't want to watch one churn through tokens. I want to delegate and
walk away.

**The future is spec-driven.** I describe the end state - the spec - and the system makes it
happen. The human artifact of value is a precise description of *what* and *why*, not a
running commentary on *how*, keystroke by keystroke. GitHub's
[spec-driven development toolkit](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/)
and OpenAI's [harness engineering](https://openai.com/index/harness-engineering/) writeups
are good signals that this is where the industry is heading. The introduction of `/goal` in
both OpenAI's Codex and Anthropic's Claude Code is another validation.

**A "software factory" is the right metaphor.** Picture a conveyor belt: a fully automated
assembly line that takes a specification in one end and emits reviewed, tested, merged code
out the other. Humans design the line and feed it work. The line does the building.

There's a parallel here with the shift from imperative to declarative programming. Prompting
an agent step by step is *imperative*: you specify *how*, instruction by instruction. Building
a factory is *declarative*: you specify *what* you want - the spec - and let the system work
out how to produce it. Functional programmers gave up manual control over execution order and
got composability and reliability in return. This is the same trade, applied to how we direct
the machines that write our code.

## The core idea: two non-overlapping phases

The factory works by splitting the engineering process into two distinct phases that never
overlap. Human deep-work happens synchronously, during your working hours. Machine execution
happens asynchronously, in the background, unattended.

### Phase 1 - Human definition & review (synchronous)

During working hours, engineers spend their attention only on high-context, high-value work:

- **Architecture & specs.** You work *with* a synchronous agent to produce highly detailed
  specifications. The spec is where you organize technical thinking, lay out system
  architecture, and pin down edge cases. This is the real work, and it's worth doing well.
- **Review & validate.** You review what the factory produced overnight, and validate
  results that a test suite can't fully judge - eyeballing a data diff after a pipeline run,
  for instance.
- **Factory maintenance.** This is the mindset shift. When the AI makes a mistake, you do
  *not* hand-fix the code. You fix the **factory floor** - the system docs, the test harness,
  the lint rules that let the mistake through - so that *class* of mistake can never happen
  again.

### Phase 2 - The AI assembly line (asynchronous)

Once the specs are set, you queue the work, trigger the agents, and step away. In the
background, a multi-agent line takes over with no human on the happy path:

1. **Plan.** An agent reads the spec, studies the codebase, and writes a concrete
   implementation plan.
2. **Pre-implementation review.** *Before any application code is written*, a separate
   reviewer critiques that plan - grounded in the project's own constitution - and surfaces
   the strongest objections it can find. Catching a bad design here is far cheaper than
   catching it in a diff.
3. **Implement & iterate.** An implementer turns the approved plan into code, runs the strict
   linters and the test suite, and iterates in a closed feedback loop until everything is
   green.
4. **Post-implementation review.** A code-review agent reviews the diff against the spec, the
   plan, and the acceptance criteria. Its findings go back to the implementer to be
   addressed.

## How `software-factory` actually implements this

The vision above is nice, but I wanted something that runs. `software-factory` is a
**spec-first, agent-built, autonomously-running** template you fork into a new project and
own. It turns a written spec into reviewed, CI-green, merged code with no human in the loop on
the happy path.

A few design choices are worth calling out, because they're what make unattended execution
actually safe.

**Specifications and design choices are durable** All past specifications and architecture
decision records (ADRs) are saved in the repository and accessible by agents. This is important
to maintain consistency.

**PRs are the shared context** Native to GitHub development PRs offer a good abstraction to
keep shared context between agents and maintain human-readable log for verifications. But after
they are merged they are not referred to for any future work.  

**Deterministic order of operations.** I chose to *not* use agentic capabilities to define the
sequence of operation in the conveyor. Instead I implemented a state machine use GitHub Actions.
Each operation on the assembly line is a workflow, the state transitions are fixed. It trades
flexibility of agent-led orchstration for simlpicity and predictability.

**Labels *are* the state machine.** Every transition in the pipeline is a GitHub label swap,
driven by GitHub Actions. A spec PR moves `state:queued → state:plan → state:plan-review →
state:implementation → state:implementation-ci → state:code-review → state:merge`. There's no
hidden orchestrator state to lose; the PR's labels *are* the state, fully visible on GitHub.

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

**Every agent runs in a fresh session with no shared context.** This sounds wasteful until
you see why it matters: a reviewer who shares context with the author becomes a yes-man. We
want independent critique grounded in the spec, not an agent agreeing with its own earlier
reasoning. The planner and reviewers can also run on different models.

**Failure is bounded, and parking is a first-class outcome.** A `REQUEST_CHANGES` from a
reviewer and a red CI run both trigger automatic retries - but only up to twice per stage.
Anything that can't make progress lands at `state:needs-attention`, where a human takes over.
The factory is allowed to give up loudly rather than thrash silently.

**Auto-merge is gated by guardrails.** When a PR reaches `state:merge`, the auto-merger
rebases onto `main`, runs an eligibility check, and only then squash-merges. If a guardrail
trips, it parks instead of merging. Then the scheduler advances to the next queued spec.

It ships as a **fork-and-own template**: it carries the orchestration machinery and a generic
TypeScript/pnpm skeleton but *none* of any product's code. You fork it, run one script to
stamp it with your project's name, write your constitution (`CLAUDE.md` and
`docs/principles.md` - the files the agents read first), file your first spec, and the cascade
runs.

## Building the factory floor

A factory that runs unattended will manufacture technical debt at machine speed unless the
floor is built to prevent it. Three investments are non-negotiable.

**Bulletproof automated testing.** The asynchronous loop simply does not work without a
robust test harness. The agents rely on test *failures* to self-correct in the background; the
red bar is the steering wheel. Weak tests mean a potentially confidently-wrong factory.

**Systematic documentation.** Centralized routing documents - an `AGENTS.md` / `CLAUDE.md`
constitution that points agents at the right workflow rules, domain knowledge, and
architectural guidelines - let individual specs stay short. The shared context lives once, in
the docs, not copy-pasted into every spec.

**Strict static analysis.** Crank typing and linting to the strictest setting you can stand.
What feels restrictive to a human developer is essential guardrailing for an autonomous agent
- it's the difference between an agent that wanders and one that's fenced into a correct
solution space.

## What I've learned so far

I've been running this on a project for about a month, and it has written a good amount
of real code in that time. A few honest takeaways:

- **The upfront cost is real.** Standing up the factory - and then tweaking it as the agents
  keep finding new ways to go wrong - is a lot of work. In the short term it is *much* easier
  to just prompt an agent and get your change. The factory only starts paying off once you've
  stopped hand-fixing code and started fixing the floor.
- **When it works, it's genuinely amazing.** Filing a spec, walking away, and coming back to a
  reviewed, CI-green, merged PR is a different category of experience from babysitting a chat
  window.
- **The jury is still out on scale.** I've run this with dozens of specs and ADRs. I don't yet
  know how it behaves with *hundreds* - whether the accumulated constitution keeps the agents
  on track or starts to buckle under its own weight. That's the part I'm most curious about.

Overall: promising, and worth the investment for the kind of long-lived, institutional work I
care about. But it's an experiment, not a finished answer.

---

If you want to dig in, the template is on GitHub:
[dmitriy-yefremov/software-factory](https://github.com/dmitriy-yefremov/software-factory).
It's early and opinionated, and I'd love feedback.

### References

- Jamon Holmgren - [Night Shift](https://jamon.dev/night-shift)
- OpenAI - [Harness engineering](https://openai.com/index/harness-engineering/)
- GitHub - [Spec-driven development with AI](https://github.blog/ai-and-ml/generative-ai/spec-driven-development-with-ai-get-started-with-a-new-open-source-toolkit/)
