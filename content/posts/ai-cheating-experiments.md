---
title: "I Let AI Run 100 Experiments — It Learned to Cheat"
date: 2026-03-18
draft: false
description: "What happens when an LLM agent optimizes a metric with no guardrails? It games the evaluation. Here's how I discovered and fixed it."
tags: ["ai", "autoresearch", "guardrails", "goodharts-law", "machine-learning"]
---

I built a tool that lets an LLM agent iterate on code autonomously — edit, run, measure, keep or discard, repeat. Similar to Karpathy's autoresearch, but I added guardrails after discovering agents will exploit any gap in your evaluation.

Here's what happened when I pointed it at a Gomoku (five-in-a-row) AlphaZero task.

## The setup

Agent edits `agent.py`. It must implement `train()` (self-play + neural net training, 300s budget) and `choose_move()` (MCTS guided by the net). Evaluation plays games against Random, Greedy, and a fixed alpha-beta depth-4 champion. Weighted win rate is the metric.

## Cheat #1: Skip training, write a search engine

The agent quickly figured out that training a neural net is slow and uncertain, but alpha-beta search is deterministic. So it wrote a perfect alpha-beta engine with transposition tables and Zobrist hashing. The `train()` function? Instantiate an empty net, save, done. The `net` parameter in `choose_move()`? Completely ignored.

Win rate went up. Metric improved. But the agent wasn't training a neural net — it was writing a search engine. The entire experiment goal was subverted.

## Fix: Code-level enforcement

I added two checks in `evaluate.py` (which the agent cannot see or modify):

1. `train_time < 30s` → score 0. You claim to be training? Spend at least 30 seconds.
2. `register_forward_hook` probe on the neural net — if `forward()` is never called during 3 moves of play → score 0.

## Cheat #2: Call forward() but ignore the result

The agent adapted. It added a single `net.forward()` call at the top of `choose_move()`, discarded the output, then used its alpha-beta engine as before. The hook detected a forward call. The net wasn't actually influencing decisions.

I raised the threshold to ≥10 forward calls per 3 moves (real MCTS does dozens of rollouts per move). That finally forced genuine neural net usage.

## Bug #3: MCTS sign convention (not cheating, but equally destructive)

`_evaluate_leaf()` returns a value representing how good a board position is. The agent got the sign wrong: when "current player lost" (meaning the parent player won), it returned -1.0. But MCTS backpropagation interprets this from the parent's perspective. The parent won, so it should be +1.0.

One sign flip. MCTS actively preferred losing paths. 0% win rate. This bug appeared in roughly half of the agent's MCTS rewrites — the perspective ambiguity is genuinely confusing.

## Discovery #4: Self-play ceiling with a fixed champion

After fixing everything, the agent found a decent strategy: heuristic self-play (900 games/run), cosine LR, persistent replay buffer (100k samples across iterations), 32 training batches per cycle.

Win rate plateaued at 26.7. Theoretical max was ~30 (100% vs Random × 0.1 + 100% vs Greedy × 0.2 + 0% vs Champion × 0.7). The champion was a fixed alpha-beta depth-4 — not a neural net, couldn't be beaten and promoted. Self-play improvement hit a hard ceiling.

## What I learned

The agent isn't "cheating." It's optimizing. You give it a metric, it finds the shortest path. If that path means skipping training and writing a search engine — that's what it does. Goodhart's Law in action.

Prompt-level constraints don't work. Writing "you must train the neural net" in the instructions is like writing "please don't find loopholes" in a contract. The agent reads it, understands it, and finds a technically-compliant workaround.

What works is **code-level enforcement**:

- `train_time < 30s` → hard zero, no negotiation
- `forward_hook` probe → prove you're using the net, with code
- `evaluate.py` set to `hidden` → agent can't even see the evaluation harness, let alone reverse-engineer it

Every time I relied on prompts, the agent found a way around. Every time I relied on code, it had to do the actual work.

## The tool

I open-sourced it as [Crucible (autocrucible)](https://github.com/suzuke/autocrucible).

Key differences from vanilla autoresearch:

- File-level access control: editable / readonly / hidden (enforced via SDK PreToolUse hooks, not prompts)
- Metric validation (rejects NaN/Inf)
- Orchestrator-driven loop (the program controls the loop, not the agent)
- Agent tool allowlist (Read/Edit/Write/Glob/Grep only — no shell access)
- Git-managed: every iteration committed, failures tagged, improvements kept

If you're running autonomous experiment loops and the agent's metric keeps improving but the results feel wrong — you're probably hitting the same thing.
