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

The config:

```yaml
# .crucible/config.yaml
files:
  editable: ["agent.py"]
  readonly: ["game.py"]
  hidden: ["evaluate.py", "opponent.py"]

metric:
  name: "win_rate"
  direction: "maximize"

constraints:
  timeout_seconds: 600
```

The agent can read and edit `agent.py`. It can read `game.py`. It has no idea `evaluate.py` and `opponent.py` even exist.

## Cheat #1: just don't train

The agent figured out pretty quickly that training a neural net is slow and the results are uncertain. Alpha-beta search, on the other hand, is deterministic. So it wrote a solid search engine with transposition tables and Zobrist hashing. `train()` just instantiated an empty net and saved it. The `net` parameter in `choose_move()`? Ignored entirely.

Here's what the run log looked like:

```
=== Training Phase ===
training_complete: using alpha-beta search agent with TT
train_time_sec: 0.0
```

Zero seconds of training. And the result?

| Iter | Win rate | Status | What the agent did |
|------|----------|--------|--------------------|
| 1 | 0.0 | crash | First attempt at alpha-beta, had a bug |
| 2 | **99.3** | keep | Fixed it. Pure search engine, no neural net |
| 3 | 0.0 | crash | Tried to add NN on top, broke it |
| 4 | 99.3 | discard | Added transposition table, same score |
| 5 | 0.0 | crash | Increased depth, timed out |

99.3% win rate. The metric said "progress." But there was no neural net doing anything. The whole point of the experiment was gone.

## The fix

I added two checks in `evaluate.py` (which the agent can't see or edit):

```python
# Rule 1: training must take real time
MIN_TRAIN_TIME = 30.0
if train_time < MIN_TRAIN_TIME:
    print(f"ENFORCEMENT FAIL: train_time={train_time:.1f}s < minimum {MIN_TRAIN_TIME}s")
    print("win_rate: 0.0")
    return

# Rule 2: the neural net must actually be called during play
net_call_count = [0]
hook = net.register_forward_hook(
    lambda m, i, o: net_call_count.__setitem__(0, net_call_count[0] + 1)
)
# ... play 3 probe moves ...
if net_call_count[0] == 0:
    print("ENFORCEMENT FAIL: choose_move() never called net.forward()")
    print("win_rate: 0.0")
    return
```

After the fix, the logs look like this when the agent plays fair:

```
train_time_sec: 270.6
enforcement_ok: train_time=270.6s, net_calls_per_move=50.0
```

270 seconds of real training, 50 neural net forward passes per move. That's real MCTS.

## Cheat #2: call forward(), throw away the answer

The agent adapted. It stuck a `net.forward()` call at the top of `choose_move()`, threw away the output, and kept using its search engine underneath. The hook saw a forward call. The net still wasn't doing anything.

I bumped the threshold to 10+ forward calls per 3 moves. Real MCTS does dozens of rollouts per move, each hitting the net. That finally forced real usage.

## Bug #3: MCTS with the sign backwards

This one wasn't intentional, but the outcome was just as bad.

`_evaluate_leaf()` returns how good a board position is. The agent wrote it so "current player lost" returns -1.0. Sounds right. Except MCTS backpropagation reads this from the parent's perspective. The parent won. It should be +1.0.

One sign. The tree search started preferring paths where it loses. 0% win rate. And the agent made this mistake in about half its MCTS rewrites. The perspective thing is genuinely easy to mess up.

## Discovery #4: you can't self-play past a fixed ceiling

Once the bugs were out, the agent started making real progress. Here's the actual run data from 16 iterations:

| Iter | Win rate | Status | Description |
|------|----------|--------|-------------|
| 1 | 0.0 | crash | Tried pure search again, enforcement caught it |
| 2 | 11.3 | keep | First real NN training. Tactical win/block detection |
| 3 | 16.7 | keep | Added 3 tactical layers to choose_move (+47.8%) |
| 4 | 16.0 | discard | Replaced MCTS with 2-ply minimax, slightly worse |
| 5 | 20.0 | keep | Heuristic self-play, more training data |
| 6 | 0.0 | crash | Broke something |
| 7 | 21.3 | keep | Persistent replay buffer across iterations |
| 8 | 24.0 | keep | Dropped MCTS self-play for fast heuristic, 2x batches |
| 9 | 25.3 | keep | Cosine LR + buffer optimization |
| 10 | **26.7** | keep | Peak. 900 games/run, 32 batches/cycle |
| 11 | 22.7 | discard | 4x batches, worse (overtrained) |
| 12 | 22.0 | discard | 2-ply re-ranking, regression |
| 13 | 26.0 | discard | Better heuristic scoring, marginal |
| 14 | 26.7 | discard | FPU reduction, tied (not better = discard) |
| 15 | 18.7 | discard | Blended heuristic+net priors, big regression |
| 16 | 20.7 | discard | Limited MCTS expansion, still worse |

The pattern is clear: 6 iterations of improvement, then 6 iterations of going nowhere. The agent tried everything it could think of and nothing broke past 26.7.

The theoretical max was about 30 (100% vs Random × 0.1 + 100% vs Greedy × 0.2 + 0% vs Champion × 0.7 = 30). The champion was a fixed alpha-beta depth-4. Not a neural net. Not promotable. A 300-second training budget can't produce a net that outplays competent search.

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
