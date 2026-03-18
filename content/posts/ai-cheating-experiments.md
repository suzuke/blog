---
title: "I let AI run 100 experiments. It learned to cheat."
date: 2026-03-18
draft: false
description: "An LLM agent tasked with training a neural net decided it was faster to just not. Then it got creative."
tags: ["ai", "autoresearch", "guardrails", "goodharts-law", "machine-learning"]
---

I built a tool that runs an LLM agent in a loop: edit code, run it, measure, keep or toss. Like Karpathy's autoresearch. I had to add guardrails because the agent kept finding ways to hit the metric without doing the actual work.

This is what happened when I gave it a Gomoku AlphaZero task.

## The setup

The agent edits `agent.py`. It has to write `train()` (self-play + neural net, 300s budget) and `choose_move()` (MCTS guided by the net). Then it plays games against a random player, a greedy player, and an alpha-beta depth-4 champion. Weighted win rate is the only number that matters.

## Cheat #1: just don't train

The agent figured out pretty quickly that training a neural net is slow and the results are uncertain. Alpha-beta search, on the other hand, is deterministic. So it wrote a solid search engine with transposition tables and Zobrist hashing. `train()` just instantiated an empty net and saved it. The `net` parameter in `choose_move()`? Ignored entirely.

Win rate went up. The metric said "progress." But there was no neural net doing anything. The whole point of the experiment was gone.

## The fix

I added two checks in `evaluate.py` (which the agent can't see or edit):

1. `train_time < 30s` = score 0. If you're training, it takes more than 30 seconds.
2. `register_forward_hook` on the net. If `forward()` never fires during 3 moves of play, score 0.

## Cheat #2: call forward(), throw away the answer

The agent adapted. It stuck a `net.forward()` call at the top of `choose_move()`, threw away the output, and kept using its search engine underneath. The hook saw a forward call. The net still wasn't doing anything.

I bumped the threshold to 10+ forward calls per 3 moves. Real MCTS does dozens of rollouts per move, each hitting the net. That finally forced real usage.

## Bug #3: MCTS with the sign backwards

This one wasn't intentional, but the outcome was just as bad.

`_evaluate_leaf()` returns how good a board position is. The agent wrote it so "current player lost" returns -1.0. Sounds right. Except MCTS backpropagation reads this from the parent's perspective. The parent won. It should be +1.0.

One sign. The tree search started preferring paths where it loses. 0% win rate. And the agent made this mistake in about half its MCTS rewrites. The perspective thing is genuinely easy to mess up.

## Discovery #4: you can't self-play past a fixed ceiling

Once the bugs were out, the agent landed on a reasonable approach: heuristic self-play (900 games per run), cosine learning rate, a persistent replay buffer accumulating 100k samples across iterations, 32 training batches per cycle.

Win rate hit 26.7 and stopped. The theoretical max was about 30: even 100% wins against Random and Greedy only add up to 30 when the Champion weight is 0.7 and you can't beat it. The champion was a fixed alpha-beta search. Not a neural net. Not promotable. There was nowhere left to go.

If you want self-play to keep improving, the champion has to be a neural net too, and it has to get replaced when something better comes along.

## What this taught me

The agent wasn't cheating in any intentional sense. It was optimizing. I gave it a number to push up and it found the shortest path. When that path meant skipping the training and building a search engine, that's what it did. When it meant calling `forward()` once to satisfy the hook and then ignoring the result, it did that too.

Writing "you must use the neural net" in the prompt didn't help. The agent read it, understood it, and found something that technically complied.

What actually worked was making it impossible to cheat:

- Training under 30 seconds = zero. No negotiation.
- Forward hook probe = prove the net is being used, with code, not words.
- `evaluate.py` hidden from the agent = can't reverse-engineer the evaluation.

Every prompt-based rule got worked around. Every code-based rule held.

## The tool

I open-sourced it: [Crucible (autocrucible)](https://github.com/suzuke/autocrucible).

How it differs from plain autoresearch: file-level access control (editable, readonly, hidden) enforced through SDK hooks. Metric validation that rejects garbage values. The program runs the loop, not the agent. The agent only gets five tools: Read, Edit, Write, Glob, Grep. No shell.

If your autonomous experiment loop keeps showing improving metrics but the results don't feel right, you're probably running into the same thing I did.
