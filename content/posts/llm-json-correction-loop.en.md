---
title: "Where your LLM stops being able to fix a 100-entity JSON"
date: 2026-05-08
draft: false
tags: ["llm", "json", "agent", "rfc6902", "open-source"]
categories: ["Engineering"]
summary: "When you ask an LLM to repair a large JSON object on critic feedback, the prevailing 'full-regen' pattern breaks down much earlier than you'd expect. We measured how and why gpt-4o-mini falls over on a 100-entity / 183-edge KG — and why sub-agent decomposition turns out to be a correctness component, not a cost optimization."
author: "Rick Choi"
---

When an LLM produces a large JSON object — a multi-act story plan,
an agent's memory, a generated knowledge graph, a config tree — and a
critic flags a defect, the dominant pattern is to ask the LLM to
**re-emit the entire object**. We measured *when* this pattern
breaks. The result was surprising: it breaks much earlier, and much
more dramatically, than we expected.

Library + measurements + experiment code are all open source at
[github.com/warpspaceinc/json-correction](https://github.com/warpspaceinc/json-correction).

## What we measured

The target task is *knowledge-graph (KG) correction*. We deliberately
inject defects into a known-good KG (so we can measure fix rate /
drift objectively), have a critic detect them, and ask an LLM to
repair. Six defect operators:

- `relation_swap`: swap an edge's predicate to a wrong one
- `entity_drop`: delete an entity (breaks referential integrity)
- `entity_paraphrase`: change an entity's label
- `type_violation`: flip an entity's type (Person ↔ Film)
- `dangling_ref`: add an edge referring to a non-existent entity
- `duplicate`: duplicate an entity under a different ID

Six conditions:

| Condition | Mechanism |
|---|---|
| `oracle` | Receives the ground-truthed defect log and applies inverses deterministically. Upper bound. No LLM. |
| `B0` | Ask the LLM to "re-emit the entire KG" — the common production pattern |
| `B1` | Ask the LLM to "emit RFC 6902 patch ops in one shot" — JSON Whisperer style |
| `O1` | B1 inside a critic loop (no sub-agents) |
| `O2` | O1 + path_finder sub-agent (full context) |
| `O2N` | O2 + context narrowing |

All `gpt-4o-mini` (OpenRouter), `temperature=0`. Metrics: structural
fix rate, *excess drift* (count of fields the critic didn't ask to
change but were modified anyway), tokens, wall clock.

## Experiment progression — Phase A through ablation

We split the work into phases to keep the diagnosis tight:

| Phase | What it validates |
|---|---|
| **A** — oracle | Whether a deterministic patcher with the ground-truth log fixes 100%. Pipeline sanity check. |
| **B-0** — B0 | Quantify the drift of the standard full-regen pattern. |
| **B-1** — B1 | Limits of single-shot RFC 6902 patching (JSON Whisperer style). |
| **B-2** — O1 / O2 | Isolate the contribution of the loop and of path_finder. |
| **B-3** — O2N | Add context narrowing on top. |
| **B-4** — size sweep | Where does it break as the KG grows? |
| **B-5** — ablation | Separate each component's contribution at scale. |

Small fixture (16 entities / 14 edges, 6 defects, mean of 6 seeds):

![per-condition comparison](/blog/images/posts/llm-json-correction-loop/fig1_condition_compare.png)

| Cond | Fix% | Drift | Tokens | Wall |
|---|---|---|---|---|
| oracle (UB) | 100 | 0.2 | 0 | 0s |
| B0 full-regen | **100** | **4.0** | 2,506 | 14.5s |
| B1 single-shot patch | 55 | 5.5 | **1,774** | 3.1s |
| O1 loop+patch | 64 | 4.8 | 6,218 | 9.6s |
| O2 +path_finder | 84 | 5.5 | 13,244 | 19.7s |
| **O2N +narrowing** | **97** | **4.5** | 8,482 | 17.2s |

### Finding 1: full-regen "drifts"

B0 (full-regen) achieves 100% fix rate, but **mutates 4 fields per
run on average that the critic never asked to change** — label
paraphrasing, edge reordering, slight ID rewrites. That's "drift" —
small corruption hiding behind a 100% fix rate.

It may not show on a small KG, but if **downstream systems depend on
IDs** (e.g. a chapter-4 sequence referencing a character ID in a
narrative-synthesis system), drift silently breaks integrity. And the
LLM always drifts a little — even at temperature 0.

### Per-seed variance, honestly

Means look safe; per-seed scatter is the truer picture:

![per-seed scatter](/blog/images/posts/llm-json-correction-loop/fig3_per_seed_scatter.png)

Oracle is always at (0 drift, 100% fix). B0 has stable fix but drift
ranges 0–7. B1/O1's fix rate itself bounces between 0 and 100%
(single-shot's downside). **Only O2N consistently lands in the right
corner** — that's the *reliability* sub-agents add.

### Finding 2: a naive surgical patcher fixes *less*

Intuitively, "send the diff instead of the whole document" should be
both cheaper and drift-free. JSON Whisperer (arXiv 2510.04717)
reports 31% token savings at <5% quality loss, and that's the
expected story.

We measured something different:

- **B1 (single-shot patch)**: fix rate **55%**. Half of what the
  critic flagged.
- **O1 (loop + patch, no sub-agents)**: **64%**. The loop helps but
  doesn't close the gap.

Why? We debugged a specific case (seed=1):

```
8 critic-flagged issues:
  /edges/2:  dangling_ref (Q2 not in entities)
  /edges/3:  dangling_ref (Q2 not in entities)
  /edges/4:  type_violation ('spouse_of' wants Person→Person, got Person→Film)
  /edges/5:  dangling_ref (Q15 not in entities)
  /edges/8:  type_violation ('starred_in' wants Person→Film, got Film→Film)
  ...

8 patch ops the LLM emitted:
  {'op': 'remove', 'path': '/edges/2'}    ← should *restore* Q2, not *delete* the edge
  {'op': 'remove', 'path': '/edges/3'}
  {'op': 'remove', 'path': '/edges/4'}
  ...
  {'op': 'remove', 'path': '/edges/9'}    ← invalid: indices shifted by previous removes
  {'op': 'remove', 'path': '/edges/10'}   ← also invalid
```

**Two failure modes simultaneously:**

1. **"Lazy remove."** Instead of *fixing* the defect, the LLM
   *deletes* the offending item. The right fix for a dangling_ref on
   missing Q2 is to restore Q2; the LLM removes every edge that
   references Q2. Data loss.

2. **"Index drift."** Removing `/edges/2` shifts later indices down
   by 1. Eight removes emitted in ascending order — the fifth one
   onwards points at the wrong element. This is exactly what JSON
   Whisperer's EASE encoding was designed to fix.

Prompt hardening can mitigate both:
- "Restoration > deletion. Do not lose data."
- "Emit `remove` ops in descending index order."

But there's a deeper third failure mode that prompting alone cannot
fix.

### Finding 3: symptom and root cause are different

The hardest defect is `type_violation`. When entity Q12's type is
flipped from Film to Person, **Q12's type field itself is just
"Person"** — the critic can't catch it directly on Q12. Instead the
critic catches it on every edge that *touches* Q12. For example:
"Song Kang-ho `starred_in` Q12" violates `starred_in`'s requirement
of (Person, Film) because the actual types are (Person, Person).

The critic points at the *symptom* (`/edges/N`). Not the *root
cause* (`/entities/Q12/type`).

Given `/edges/N`, the LLM "fixes" it by *changing the edge's
predicate*. The critic is satisfied — but Q12's type is still wrong,
the next iteration flags it on a different edge, and the LLM mutates
that edge's predicate too. Eventually every edge touching Q12 has a
silently-corrupted predicate while the critic happily passes.

This is the **motivation for the path_finder sub-agent**.

### Finding 4: path_finder + narrowing closes the gap

`path_finder` is an LLM call that analyzes critic-flagged symptom
pointers and redirects each to its root cause: "this edge has a
type_violation; per the predicate-type table, Q12's type is wrong;
redirect: `/edges/N` → `/entities/Q12/type`."

`Context narrowing` shows both the sub-agent and the patcher only the
*affected slice* of the KG — entities/edges referenced by flagged
pointers, plus 1-hop neighbors.

Result:
- O1 (loop only): 64%
- O2 (+ path_finder, full context): 84%
- **O2N (+ narrowing): 97%**

On the small fixture O2N comes within 3pp of the oracle's 100%.

## At 100 entities — the breaking point

This is the genuinely interesting part.

If you stop at the small fixture, O2N's tokens are **3.4× higher**
than B0. Sub-agents add LLM calls. "Surgical patching is expensive,
right?" That's a fair question.

Scale the KG to 100 entities / 183 edges, inject the same 8 defects,
B0 vs O2N:

![size sweep](/blog/images/posts/llm-json-correction-loop/fig2_size_sweep.png)

**At size=100, B0 *catastrophically fails*:**

```
size=100 (100 ent, 183 edges, 8 defects):
  B0:   fix=  0%  tokens=73,740   wall=435s
  O2N:  fix=100%  tokens=17,117   wall=26s
```

8.5 minutes, 73K tokens burned, zero defects fixed. Five hardcap
iterations of the loop trying.

Cause: `gpt-4o-mini`'s `max_tokens=8192` ceiling. Re-emitting 100
entities + 183 edges of JSON exceeds the completion budget; the array
truncates mid-stream. The critic sees the truncated state and finds
*more* dangling_refs (truncation = dangling refs). The next iteration
re-tries the regen → truncates again → repeat. Without the loop's
hardcap policy, this would burn tokens unbounded.

O2N? **17K tokens, 26 seconds, 100% fix.** Tokens barely changed
(small fixture 18K → larger KG 17K).

Reason: a surgical patcher's token cost is bounded by the *edit
footprint*, not the *KG size*. 8 defects → ~10–15 affected entities
→ constant cost. The KG could be 100× larger and the cost wouldn't
move.

This is the central empirical claim:

> **Surgical patching's cost scales with edit footprint. Full-regen's
> scales with document size. For any non-trivial JSON state, surgical
> strictly dominates — both on tokens, and on whether the method
> works at all.**

The breaking point is model-specific (Claude Opus's 200K context
buys you a much bigger KG before regen fails). But *that there is a
breaking point* is structural to autoregressive decoding — the
longer the output, the more wall time and the more per-token error.

## Ablation — narrowing is correctness, not cost

We initially thought of narrowing as a "token-saving trick." Then we
ran the size-100 ablation:

![ablation](/blog/images/posts/llm-json-correction-loop/fig4_ablation_size100.png)

| Cond | path_finder | narrowing | Fix% | Drift | Tokens |
|---|:---:|:---:|---|---|---|
| B0 | n/a | n/a | 0% | 8 | 73,740 |
| O1 | ✗ | ✗ | 35% | 23 | 42,621 |
| O1N | ✗ | ✓ | 57% | 13 | 14,392 |
| **O2** | ✓ | ✗ | **FAILED** | - | - |
| O2N | ✓ | ✓ | 100% | 6 | 17,117 |

**O2 (path_finder + full context) doesn't work.** path_finder's JSON
output was *unparseable*. The full-context prompt at size=100 is
~15K input tokens; the model's structured-output reliability degrades
even though it nominally supports much longer contexts.

So:

> **Narrowing is not "premature optimization." It is a prerequisite
> for sub-agent reliability.**

If you skip narrowing in production thinking you'll add it later as
an optimization, your sub-agents will silently start emitting broken
JSON at some point. We measured this.

## Summary

When repairing a large JSON state through an LLM-driven critic loop:

1. **Full-regen breaks earlier than you think.** 100 entities is a
   production-realistic size, and `gpt-4o-mini` cannot do it. Bigger
   models push the breaking point later, but the *fact that there is
   a breaking point* is structural to autoregressive decoding.

2. **A simple surgical patcher (JSON Whisperer + critic loop) is not
   enough.** Symptom-vs-root-cause keeps fix rate stuck at ~64%.

3. **Sub-agents (path_finder) close the gap** — +33pp fix rate.

4. **Context narrowing is a correctness component, not a cost
   optimization** — drop it and the sub-agent itself stops working.

Each component is load-bearing — drop any one and the whole thing
falls over.

### Limitations (honestly)

Scope of these measurements:
- **Single model** (`gpt-4o-mini`). Larger Claude / GPT-4o variants
  push the breaking point later, but surgical's relative advantage
  shows up at any scale where the KG is too big for reliable
  single-shot regen.
- **Single domain** (KG). Generality is an architectural argument;
  empirical confirmation on a second domain (config refactor,
  narrative state) is the natural next step.
- **Synthetic perturbations.** Real LLM-emitted defects have
  different distributions.
- **Ablation is one seed at size=100**. Robustness check with 5+
  seeds is on the list.

### Try it

- Library: [github.com/warpspaceinc/json-correction](https://github.com/warpspaceinc/json-correction).
  Apache-2.0, `pip install` (PyPI registration coming), quickstart
  example through full KG benchmark.
- To adapt to your domain: plug in your critic and the
  domain-specific knowledge for path_finder (schemas, constraints).
  The loop driver, planner, and convergence policies are
  domain-agnostic.

If you run an LLM system that produces large JSON state, this is
worth measuring on your own pipeline. The lesson we paid the most
attention to was: **the breaking point is closer than the
intuitions of the people writing the code suggest.** Better to find
out under controlled conditions than at 3am in production.
