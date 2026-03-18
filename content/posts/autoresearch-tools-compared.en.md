---
title: "Autoresearch tools compared: 5 ways to run autonomous experiments"
date: 2026-03-18
draft: true
summary: "A practical comparison of karpathy/autoresearch, pi-autoresearch, autoexp, Claude Autoresearch, and Crucible. What each does well, where each breaks down."
description: "A practical comparison of karpathy/autoresearch, pi-autoresearch, autoexp, Claude Autoresearch, and Crucible. What each does well, where each breaks down."
tags: ["autoresearch", "ai", "autonomous-experiments", "comparison"]
ShowToc: true
TocOpen: true
---

Karpathy released autoresearch on March 6, 2026. Within two weeks, dozens of forks and reimplementations appeared. They all share the same basic idea: let an AI agent edit code, run it, check the metric, keep or discard, repeat.

The implementations differ in ways that actually matter, though. I've been building one of these tools ([Crucible](https://github.com/suzuke/autocrucible)) and spent time reading through the others. Here's what I found.

## The contenders

| Tool | Stars | Type | Agent | Language |
|------|-------|------|-------|----------|
| [karpathy/autoresearch](https://github.com/karpathy/autoresearch) | 40k | Python script | Claude Code CLI | Python |
| [pi-autoresearch](https://github.com/davebcn87/pi-autoresearch) | 2k | IDE extension | Pi | TypeScript |
| [autoexp](https://github.com/wizwand/autoexp) | 55 | Prompt templates | Any coding agent | Markdown only |
| [Claude Autoresearch](https://github.com/uditgoenka/autoresearch) | 1.2k | Claude Code skill | Claude Code | Shell + Markdown |
| [Crucible](https://github.com/suzuke/autocrucible) | — | CLI tool | Claude Agent SDK | Python |

## How they're built

These five tools represent three different architectures for the same idea.

In the agent-driven approach (autoresearch, pi-autoresearch, autoexp, Claude Autoresearch), the agent is in charge. It decides when to edit, when to run, when to commit, when to revert. The tool provides infrastructure (dashboard, result logging, git helpers) but the loop lives inside the agent's context window.

In the orchestrator-driven approach (Crucible), a Python program controls the loop. The agent is called once per iteration to produce code edits. Everything else (running the experiment, parsing the metric, committing, reverting) is handled by the orchestrator. The agent never touches the loop control flow.

This is the most important architectural difference, and most of the other tradeoffs flow from it.

## karpathy/autoresearch

The original. 630 lines of Python. Designed for one task: training nanochat on a single GPU.

It reads a `program.md` file, calls Claude Code CLI in a subprocess, runs the training script, checks val_bpb, keeps or discards, writes to `results.tsv`. Clean and minimal.

What it doesn't do: file access control, metric validation, multi-task support, session recovery. It's a script for one specific workflow, and it does that workflow well.

The 40k stars come from the idea, not the implementation. Most forks exist because people wanted to run it on different hardware (MLX, Windows, CPU) or generalize it beyond nanochat.

## pi-autoresearch

Tobi Lütke (Shopify CEO) used this to optimize the Liquid template engine: 53% faster parse+render, 61% fewer allocations, across ~120 automated experiments. That got attention.

It's a Pi IDE extension written in 1,888 lines of TypeScript. Three tools: `init_experiment`, `run_experiment`, `log_experiment`. The agent calls these tools to drive the loop.

What works well:

- Dashboard UI with live status widget and fullscreen overlay. You can watch experiments in real time.
- Backpressure checks via `autoresearch.checks.sh`. After each benchmark, you can optionally run tests, lint, and type checks. If anything fails, the result can't be kept. This prevents fast-but-broken optimizations.
- Keep/discard is handled in code, not by the prompt. When `log_experiment` gets status=discard, it auto-reverts via git. Reliable.
- Session recovery through `autoresearch.md`, designed so a fresh agent with no history can pick up where the last one left off.
- Ideas backlog in `autoresearch.ideas.md` stores promising ideas the agent didn't have time to try. Survives context window limits.

Where it falls short:

- Tied to the Pi IDE. No CLI, no other agents.
- File protection is prompt-only. The skill says "Off Limits: don't touch these files." Nothing actually stops the agent from touching them.
- No metric validation. The agent reports whatever number it wants.

## autoexp

The simplest possible approach: three markdown files, zero code. `make_autoexp.md` tells the agent to scan your repo and generate a customized `autoexp_program.md`. Then you tell the agent to follow it.

It's appealing because there's genuinely zero setup. One prompt and you're running. No installation, no config files, no dependencies. It works with any coding agent that reads markdown. And the template itself is solid: `program_template.md` covers baseline runs, keep/discard logic, failure handling, branching.

The downside is that everything is prompt. No timeout enforcement, no file protection, no metric validation, no git automation. The agent is on the honor system for all of it. Results tracking depends on the agent actually writing to the TSV log correctly. There's no resume mechanism beyond "read the markdown and figure out where we were."

Autoexp answers the question "what's the minimum viable autoresearch?" The answer is a good prompt. Whether that's enough depends on how much you trust the agent.

## Claude Autoresearch (uditgoenka)

A Claude Code skill that generalizes autoresearch beyond ML. Supports `/autoresearch` (main loop), `/autoresearch:plan` (setup wizard), `/autoresearch:security` (security audit), `/autoresearch:debug`, `/autoresearch:fix`, `/autoresearch:ship`.

Where it's interesting:

- Domain-agnostic by design. The core principles doc explicitly says this works for code, marketing, sales, DevOps, anything with a measurable metric. Not just ML.
- Has a guard command, separate from the metric. The guard (e.g., `npm test`) must always pass, preventing regressions while the metric is being optimized. This is something Crucible doesn't have.
- Good principles documentation: 7 core principles extracted from Karpathy's autoresearch, well articulated. "Constraints = enablers", "metrics must be mechanical", "verification must be fast."
- Bounded iterations. `Iterations: N` to auto-stop after N rounds. Useful for CI/CD or overnight runs.

Where it struggles:

- Same agent-driven limitations as the others. The loop is in the agent's context window. File protection is prompt-based. The agent manages its own git.
- Scope creep. 6 subcommands, a security audit with STRIDE/OWASP, a ship workflow. That's a lot of surface area for a tool whose core job is "edit, run, check, keep/discard."
- Claude Code only.

## Crucible (autocrucible)

Full disclosure: I built this one. I'll try to be honest about the tradeoffs.

What I think works:

- File access control is enforced through Claude Agent SDK `PreToolUse` hooks, not prompts. The agent literally cannot read hidden files. This isn't "please don't look at evaluate.py"; the SDK blocks the Read tool before it executes.
- The orchestrator controls the loop. The agent can't skip iterations, forget to commit, or revert the wrong thing.
- Metric validation rejects NaN, Inf, and garbage values programmatically.
- The agent only gets Read, Edit, Write, Glob, Grep. No shell, no subprocess, no web access.
- Git management: branch per run, failed attempts tagged, results logged automatically.

What doesn't work as well:

- No dashboard. CLI only. `crucible status` and `crucible history` exist but they're text output, not a live UI.
- Higher setup cost. You write `config.yaml` + `program.md` + `evaluate.py`. Compare that to autoexp (one prompt) or pi-autoresearch (`/autoresearch optimize X`).
- Single metric. No secondary metrics tracking. You bake constraints into evaluate.py, which works but is less visible than a dashboard showing multiple numbers.
- No backpressure checks. pi-autoresearch's `checks.sh` concept is useful and I should probably steal it. For now you'd put test assertions inside your evaluation script.
- Claude Agent SDK only. Locked to one provider.
- Slower iterations due to orchestrator overhead and an SDK call per iteration. Agent-driven tools can sometimes batch edits faster.

## The real comparison

| | autoresearch | pi-autoresearch | autoexp | Claude Autoresearch | Crucible |
|---|---|---|---|---|---|
| **Who controls the loop** | Agent | Agent | Agent | Agent | Orchestrator |
| **File protection** | None | Prompt | Prompt | Prompt | Code (SDK hooks) |
| **Metric validation** | None | None | None | None | Code (rejects NaN/Inf) |
| **Keep/discard** | Script | Code | Prompt | Prompt | Code |
| **Git management** | Script | Code (auto-revert) | Prompt | Prompt | Code (branch/tag/revert) |
| **Dashboard** | None | Yes (TUI) | None | None | None |
| **Setup effort** | Edit program.md | /autoresearch X | One prompt | /autoresearch:plan | Write config + evaluate.py |
| **Agent platform** | Claude CLI | Pi | Any | Claude Code | Claude Agent SDK |
| **Backpressure checks** | No | Yes | No | Yes (guard) | No |
| **Session recovery** | Git | autoresearch.md | Manual | Git + results log | Git + results log |
| **Scope** | ML only | Any metric | ML (template) | Any metric | Any metric |

## Which one should you use

If you just want to try the idea, use autoexp or Claude Autoresearch. You'll be running in under a minute.

If you want a live dashboard, pi-autoresearch is the best option (assuming you use Pi). Claude Autoresearch gives you the most features if you're in Claude Code.

If you're running overnight and need to trust the results, I'd point you to Crucible. The orchestrator won't forget to commit, the agent can't modify the evaluation, and the metric gets validated. But I'm biased, so weigh that accordingly.

If you're working on the original nanochat task, just use karpathy/autoresearch. It was designed for exactly that and it does the job.

Realistically, the choice comes down to what you care about. Speed-to-start favors prompt-based tools. Reliability-of-results favors code-enforced ones. Most people start with the first and move to the second after the agent does something unexpected.

## Real test: same task, four tools

I ran the same sorting optimization task with four different tool approaches. Same starting `sort.py` (Python built-in `arr.sort()`), same `benchmark.py`, same metric (ops/sec, higher is better). Each got ~5 iterations.

All four tests ran under Claude Code Max with Claude Opus 4.6. Same model, same subscription, no difference in reasoning ability. The only variables are the tool architecture and what the agent can access. Full results, code, and prompts are in the [benchmark repo](https://github.com/suzuke/autoresearch-benchmark).

In Crucible, benchmark.py is hidden from the agent. In the other three, the agent can read it.

### Results summary

| Tool style | Baseline | Best ops/sec | Improvement | Iters | Can see benchmark? |
|------------|----------|-------------|-------------|-------|--------------------|
| autoresearch | 92.80 | 276.79 | +198% | 4 | yes |
| autoexp | 97.55 | **322.94** | +231% | 6 | yes |
| Claude Autoresearch | 98.55 | 300.11 | +205% | 5 | yes |
| Crucible | ~92 | 292.40 | +218% | 5 | **no** |

### What each agent did

**autoresearch style** (4 iterations, best 276.79):

| Iter | ops/sec | Status | Description |
|------|---------|--------|-------------|
| 0 | 92.80 | baseline | Python built-in sort |
| 1 | 214.37 | keep | numpy sort |
| 2 | 161.06 | revert | numpy stable sort (slower) |
| 3 | 276.79 | keep | numpy int32 dtype |

**autoexp style** (6 iterations, best 322.94):

| Iter | ops/sec | Status | Description |
|------|---------|--------|-------------|
| 0 | 97.55 | baseline | Python built-in sort |
| 1 | 213.77 | keep | numpy sort |
| 2 | 244.89 | keep | numpy fromiter + quicksort |
| 3 | 167.99 | revert | numpy stable sort (radix) |
| 4 | **322.94** | keep | numpy int32 quicksort |
| 5 | 300.64 | revert | np.array int32 quicksort (slower variant) |

**Claude Autoresearch style** (5 iterations, best 300.11):

| Iter | ops/sec | Status | Description |
|------|---------|--------|-------------|
| 0 | 98.55 | baseline | Python built-in sort |
| 1 | 222.84 | keep | numpy sort |
| 2 | 244.94 | keep | numpy fromiter + quicksort |
| 3 | **300.11** | keep | Wrote a C radix sort, compiled to .dylib, called via ctypes |
| 4 | 87.32 | revert | ctypes array instead of numpy, slower |

**Crucible** (5 iterations, best 292.40):

| Iter | ops/sec | Status | Description |
|------|---------|--------|-------------|
| 1 | 206.22 | keep | Switched to numpy introsort (C/SIMD) |
| 2 | 271.72 | keep | int32 instead of int64, halved memory bandwidth |
| 3 | 277.01 | keep | Pre-allocated reusable numpy buffer |
| 4 | 264.70 | discard | Tried struct.unpack, slower |
| 5 | **292.40** | keep | array.array as zero-copy bridge to numpy |

### The iteration count problem

One thing I didn't expect: I told all four agents "run 5 iterations." They each interpreted that differently.

- **autoresearch**: ran 4 (counted baseline as iteration 1, so "5" meant baseline + 3 experiments)
- **autoexp**: ran 6 (counted baseline as iteration 0, then ran 5 experiments on top)
- **Claude Autoresearch**: ran 5 (got it right)
- **Crucible**: ran 5 (`--max-iterations 5` in code, no ambiguity)

Three agents, three interpretations of the same instruction. Crucible's orchestrator doesn't have this problem because a Python `for` loop counts to 5 the same way every time. It's a small thing, but over a 100-iteration overnight run, "off by one" adds up.

### What this tells you

The autoexp agent got the highest raw number (322.94). The Claude Autoresearch agent did the most creative thing (writing and compiling a C library). Crucible didn't win on the metric.

But look at what the Claude Autoresearch agent did: it wrote a C radix sort, compiled it into a shared library, and loaded it via ctypes. It could do this because it had shell access and could read the benchmark to understand exactly what input sizes and data types to optimize for. In Crucible, the agent has no shell and can't see the benchmark. It had to work from first principles.

Whether that matters depends on your use case. If the goal is "make this code as fast as possible by any means," let the agent see everything and give it shell access. If the goal is "find improvements that generalize and aren't fitted to the specific benchmark," hide the evaluation.

The 4-tool test also confirmed something I expected: all the prompt-based tools work fine for well-behaved tasks. The sorting benchmark is simple, the metric is honest, and there's no reason for the agent to cheat. The differences between tools only show up when the task gets adversarial (see my [previous post on agent cheating](/blog/posts/ai-cheating-experiments/)).
