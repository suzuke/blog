---
title: "Multi-Agent Is Already Running: Field Notes From an AI Dev Team"
date: 2026-03-28
draft: false
summary: "Papers say multi-agent boosts parallel tasks by 80% and tanks sequential ones by 70%. We ran real development with CCD for a day. The numbers tracked — but the details matter more."
description: "Papers say multi-agent boosts parallel tasks by 80% and tanks sequential ones by 70%. We ran real development with CCD for a day. The numbers tracked — but the details matter more."
tags: ["multi-agent", "ai-agent", "claude-code", "ccd", "developer-tools"]
ShowToc: true
TocOpen: true
---

Today I ran an experiment.

Had a mid-size project that needed code review, bug fixes, and new features. I decided to hand all of it to AI agents — and not just one. Three agents running simultaneously, each handling different tasks, coordinating through [claude-channel-daemon](https://github.com/suzuke/claude-channel-daemon) (CCD)'s P2P messaging.

After a full day, some results matched expectations. Others didn't.

## What the research says

Start with theory. Google Research published [Towards a Science of Scaling Agent Systems](https://arxiv.org/abs/2512.08296) earlier this year, testing 180 agent configurations in a large-scale experiment. The conclusion was clear:

Multi-agent improves parallelizable tasks by 80.8%, but degrades sequential tasks by 39% to 70%.

The paper also identified three effects: the more tools a task requires, the more coordination overhead eats into performance; when a single agent already performs well, adding more agents yields diminishing returns; and in decentralized architectures, one agent's errors can propagate to others.

Professor Hung-yi Lee at National Taiwan University compared topologies in his AI Agent course — Chain, Star, Tree, Mesh, Random. Mesh (all agents interconnected) generally performed best, and more agents generally helped, but only if the task is decomposable.

SWE-bench data backs this up: multi-agent (manager + researcher + engineer + reviewer) scored 7.2% higher than single-agent. Customer routing was even more dramatic — multi-agent hit 97-99% accuracy versus 79-86% for single-agent.

Sounds great on paper. But how does it play out in practice?

## Three agents doing code review

First experiment: code review. Sent the same PR to three agents simultaneously — one focused on architecture, one on code quality, one on UX and user flow.

Each agent completed their analysis, then sent results to the General instance via CCD's `send_to_instance`. General played orchestrator — deduplicating, prioritizing, filtering out false positives.

Result: three agents raised 26 issues total. General cross-referenced and filtered down to 11 real issues. 8 were false positives (one agent flagged something the other two considered fine). The remaining 11 were high quality — every one worth fixing.

This mirrors the logic behind Grok 4.20's debate architecture. Grok 4.20 runs four personas (captain, researcher, logician, contrarian) in parallel, then has them debate, cutting hallucination rates by 65%. Our approach was less sophisticated — no explicit debate mechanism, just cross-verification — but the direction was the same: multiple perspectives on the same thing catch more errors.

## Parallel feature development

Second experiment: developing two independent features simultaneously. Two agents each working in isolated git worktrees, completely independent. General instance handled review and merge afterward.

This scenario worked best. The two features had no dependencies, so running in parallel was roughly 2x faster. Google's paper says +80.8% for parallel tasks. Our experience tracked.

One catch: both agents modified related files, causing merge conflicts. General had to judge the resolution — reading both sides, understanding intent, picking the more reasonable approach. Usually got it right. Occasionally didn't, requiring human intervention.

## Multi-agent research

Third experiment: a research task. Investigating "real cases of AI agents making money," three agents each searched different directions — one on freelance arbitrage, one on content automation, one on crypto and e-commerce.

The combined results covered a lot of ground with almost zero overlap between agents. This is multi-agent's most intuitive benefit: parallel search is just faster than one agent searching sequentially.

## When not to use multi-agent

The biggest takeaway from the day wasn't "multi-agent is powerful." It was knowing when not to use it.

Sequential tasks get slower with multi-agent. One bug required reading logs, tracing the call stack, examining related code, then devising a fix — each step depending on the previous one. A single agent running start to finish was fastest. Splitting it across agents only added communication overhead.

Google's data nails it: sequential tasks degrade 39-70%. My experience says: if a task has clear sequential steps, don't split it.

Another trap is agent conflicts. Two agents editing the same file means someone has to resolve it later. If you can't cleanly separate task boundaries, coordination costs eat the parallelism gains.

## No special framework needed

One interesting discovery: all the multi-agent collaboration we did today used zero multi-agent frameworks. No LangGraph, no CrewAI, no AutoGen.

CCD provides three primitives: fleet (multiple Claude Code instances), `send_to_instance` (inter-agent messaging), and worktree (isolated workspaces). That's it. That was enough.

General instance figured out how to be an orchestrator on its own — decomposing tasks, assigning them to other instances, collecting results, reviewing, merging, reporting back. These behaviors emerged from Claude Code itself. Nobody wrote orchestration logic.

Grok 4.20 uses carefully designed personas and debate workflows. We designed nothing. We just gave agents the tools to communicate with each other, and collaboration patterns emerged naturally. Less polished than Grok, but good enough for day-to-day development.

## Numbers summary

| Scenario | Method | Result |
|---|---|---|
| Code Review | 3 agents parallel + cross-verification | 26 issues → 11 real, 58% filter rate |
| Feature Dev | 2 agents parallel worktree | ~2x speed, but merge conflicts |
| Research | 3 agents, different directions | Broad coverage, near-zero overlap |
| Bug Fix | Single agent | Fastest, multi-agent was slower |

Cross-referencing with academic data: Google says +80.8% for parallel tasks — our feature development matched. Google says -39~70% for sequential tasks — our bug fix confirmed it. SWE-bench says +7.2% for multi-agent — our code review cross-verification did improve quality.

## Back to basics

Multi-agent's value boils down to one thing: multiple perspectives on the same problem at the same time.

Academia calls it diversity of thought, ensemble methods, cross-validation. In practice it means: one agent will confidently make mistakes, but three agents looking at the same code catch errors more reliably.

The key is knowing when to use it. Decomposable tasks — the more agents the better. Sequential tasks — one is enough. Splitting the wrong task is slower than not splitting at all.

On tooling: you don't need a complex framework. As long as agents can message each other and work in isolated spaces, collaboration patterns emerge on their own. CCD does exactly this — provides the infrastructure and lets agents develop teamwork organically.

---

Related links:
- [Google: Towards a Science of Scaling Agent Systems](https://arxiv.org/abs/2512.08296)
- [Grok 4.20 Multi-Agent Architecture](https://www.buildfastwithai.com/blogs/grok-4-20-beta-explained-2026)
- [SWE-bench Multi-Agent Results](https://dev.to/nikita_benkovich_eb86e54d/coding-agent-teams-outperform-solo-agents-722-on-swe-bench-verified-4of5)
- [claude-channel-daemon (CCD)](https://github.com/suzuke/claude-channel-daemon)
