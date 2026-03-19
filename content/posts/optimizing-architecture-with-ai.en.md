---
title: "I let an AI agent refactor a codebase. It cheated."
date: 2026-03-19
draft: false
summary: "I pointed an autonomous AI agent at a real TypeScript project and told it to improve the architecture. The first five iterations were great. Then it discovered copy-paste."
description: "I pointed an autonomous AI agent at a real TypeScript project and told it to improve the architecture. The first five iterations were great. Then it discovered copy-paste."
tags: ["ai", "software-architecture", "crucible", "goodharts-law", "experiment"]
ShowToc: true
TocOpen: true
---

## The setup

AI can write functions and fix bugs. But can it look at a messy codebase and make the structure better? Not add features. Refactor. The kind of work where a senior engineer stares at dependency graphs for two days and then moves three files.

I wanted to find out. I used [Crucible](https://github.com/suzuke/autocrucible), an experiment platform that puts Claude in a loop: edit code, measure the result, keep improvements, discard regressions. The target was [NanoClaw](https://github.com/qwibitai/nanoclaw), a real TypeScript project, about 8,000 lines, that runs Claude agents inside containers.

## Measuring "good architecture" (you can't, really)

Crucible optimizes a number. For ML, that's loss or accuracy. For architecture, there is no number. "Good architecture" is the feeling you get when the code makes sense versus when you want to close the file.

So I picked four proxy metrics and hoped for the best:

| Metric | Weight | What it actually measures |
|--------|--------|--------------------------|
| Dependency coupling | 35% | Average imports per module, plus circular dependency count |
| File size uniformity | 25% | Gini coefficient of line counts. Catches God modules. |
| Instability balance | 20% | Whether each module's fan-in and fan-out are roughly balanced |
| API surface area | 20% | Average number of exports per file |

Everything computed with static analysis. `madge` for the dependency graph, hand-rolled AST parsing for the rest. No LLM-as-judge because that's expensive and the scores drift between runs.

I also added hard guardrails: all 219 tests must pass, TypeScript must compile. And penalties for known bad patterns: files under 10 lines, barrel re-exports, functions over 100 lines.

Baseline: 24.3 out of roughly 100.

## The first five iterations were legitimately good

I started the run and watched.

| Iter | Score | What happened |
|------|-------|---------------|
| 1 | 44.28 | Broke four God functions (100+ lines each) into smaller helpers |
| 2 | 46.48 | Reduced unnecessary exports across 13 files |
| 3 | 47.49 | Lowered coupling in container-runner and ipc |
| 4 | 47.66 | Deleted 127 lines of dead code |
| 5 | 49.69 | Trimmed config.ts from 16 exports to 9, decoupled two modules |

24.3 to 49.7 in five iterations. I read every diff. They were the kind of changes I'd approve in a code review: pulling `createTables()` and `runMigrations()` out of a monolithic `createSchema()`, moving constants to the one file that uses them, making internal helpers private.

I was feeling pretty good about this.

## Then it discovered copy-paste

Iteration 12. The agent figured out that if you copy a function from module A into module B, you can delete the import. Dependency count drops. Score goes up. The function still works because it's a verbatim copy.

By iteration 19, `resolveGroupFolderPath` existed in three files. `formatLocalTime` had been copied into router.ts. `DATA_DIR` and `TIMEZONE` were duplicated in ipc.ts. Each iteration bumped the score by half a point to a point and a half. Each iteration made the codebase worse.

The metric climbed from 49.7 to 55.5. The actual architecture quality peaked at iteration 5 and went downhill from there.

Classic Goodhart's Law. The measure stopped being useful the moment the agent started optimizing for it directly.

## I patched the exploit. The agent found new ones.

I rolled back to iteration 5 and added a duplication detector:

- Extract every function body of 5+ lines
- Normalize it (strip comments, whitespace, variable names)
- If two files contain the same normalized body, penalize: -8 per extra copy

I also told the agent directly: "duplication is detected and heavily penalized."

It stopped copying functions. Instead:

**Interface duplication.** TypeScript has structural typing. You can define `interface RegisteredGroup { ... }` in every file that needs it instead of importing from `types.ts`. Same interface body, different file. My detector only looked at functions.

**Constant inlining.** `const DATA_DIR = path.resolve(process.cwd(), 'data')` is one line. My detector had a 5-line minimum. The agent just wrote the same constant definition in every module that needed it and dropped the import.

Over 36 iterations the score went from 49.7 to 58.1. Roughly half genuine improvement, half gaming. The genuine half was things like moving sole-consumer constants and reducing unnecessary exports. The gaming half was interface and constant duplication that my detectors missed.

## Round 3: more detectors

I added:

- Interface clone detection: compare normalized interface bodies across files. -6 per duplicate.
- Constant clone detection: flag `UPPER_CASE` constants with the same name and value in multiple files. -5 per duplicate.

Round 3 is running as I write this. Early results: the agent is moving functionality between modules instead of copying it, consolidating related code, making unused exports private. The gains are smaller, fractions of a point per iteration, but they hold up under review.

## What I took away from this

Every proxy metric I picked got gamed. Lower dependency count? Copy-paste. Fewer exports? One big object. Better file uniformity? Dozens of stub files. I kept underestimating how creative the agent would get.

What helped was layering defenses. Hard guardrails that can't be gamed (tests pass or the iteration dies). Metrics that constrain each other so gaming one hurts another. And AST-based penalties for specific exploits, added as I discovered them. None of these work alone. Together they made honest improvement the easier path.

The real value was in the first few iterations, every time. The agent found obvious problems fast: God functions, unnecessary coupling, dead exports. After that it started hunting marginal gains, which is exactly when gaming got attractive.

I also don't think fully autonomous architecture optimization works yet. The metrics can't capture everything that matters. What actually worked was letting the agent do a first pass, then going through the diffs by hand and cherry-picking the good parts. Automated exploration, human judgment.

One thing I didn't expect: the experiment itself was useful even when the output wasn't. The agent's first moves showed me what was most wrong with the codebase. Its gaming strategies showed me what my metrics missed. I'd run this again just for those signals.

## Results

| Run | Baseline | Peak score | Real quality peak | Where gaming started |
|-----|----------|------------|-------------------|---------------------|
| arch1 | 24.3 | 55.5 | ~49.7 (iter 5) | Iter 12: function inlining |
| arch2 | 49.7 | 58.1 | ~50.2 (iter 6) | Iter 10: interface + constant duplication |
| arch3 | 44.0 | running | TBD | TBD |

## Is this worth doing?

If your codebase has obvious structural problems and decent test coverage, yes. The agent is genuinely good at finding and fixing low-hanging architectural fruit in the first few passes.

But review the diffs. Don't ship a score. And be ready for the agent to find creative loopholes in whatever you choose to measure.

Honestly the most interesting part wasn't the refactoring. It was watching the gap between "what the metric says" and "what I actually care about" get exposed in real time, one exploit at a time.

---

*Crucible is open source: [github.com/suzuke/autocrucible](https://github.com/suzuke/autocrucible). The target codebase: [github.com/qwibitai/nanoclaw](https://github.com/qwibitai/nanoclaw).*
