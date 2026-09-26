---
layout: single
title: "Tuning the Jump Arc: Platformer Physics, Recovered and Designed"
date: 2026-09-26
permalink: /posts/2026/09/tuning-jump-arc-platformer-physics-ai/
categories:
  - blog
tags:
  - ai-tools
  - claude-code
  - android
  - java
  - game-development
  - reverse-engineering
  - side-project
excerpt: "A jump arc is just a handful of numbers, but they're the numbers that decide whether a platformer feels right. One game got its numbers from a 20-year-old design decision I never wrote down; the other got them by decompiling a Spine skeleton and an obfuscated state machine."
---

![A recorded jump arc, mid-flight](../images/supermorse_jump_arc.png)

Post #38 in the [Super Morse](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/)
series. If a platformer has one system where "close enough" isn't good enough, it's movement —
players feel a wrong jump arc within seconds, even if they can't name what's off about it. This
post is about two very different ways I ended up with trustworthy movement numbers: one game
where I already knew the constants and just had to carry them over faithfully, and one where the
constants had to be excavated from a decompiled binary, one guarded state transition at a time.

## TL;DR

- Mario's movement constants (`MAX_SPEED=60`, `GRAVITY_STEP=0.42`, `JUMP_BASE=-11`, ...) are
  carried over **verbatim** from the original desktop Java game, expressed in a "per 60fps tick"
  unit rather than px/sec — deliberately, so the port's feel can never silently drift from what I
  already knew was right.
- The reverse-engineered game's movement constants had no such source of truth — they had to be
  **recovered** from a decompiled actor-rule VM, one guarded state transition at a time, and
  graded by confidence the whole way.
- The recovered result includes some genuinely counter-intuitive facts a naive port would have
  missed entirely: asymmetric left/right deceleration, a hold-to-sustain jump with an exact
  16-update window, and a gravity divisor that only applies while entering water *while falling*.
- In both games, the design principle holds: **movement feel is the one thing a shared engine
  layer should never try to derive automatically.**

---

## Mario: Keep the Numbers, Change Everything Around Them

`Player` is the largest single class in the Mario port (~1,150 lines) and the one I was most
careful not to "improve" during the port. Its physics constants were tuned years ago in the
original desktop game, implicitly assuming a fixed ~60fps loop:

| Constant | Value | Meaning |
|---|---|---|
| `ACCEL` | 2 | Horizontal acceleration per tick |
| `MAX_SPEED` / `MAX_SPEED_TURBO` | 60 / 100 | Run cap, normal vs. run-button-held |
| `GRAVITY_STEP` / `GRAVITY_CAP` | 0.42 / 10 | Per-tick fall acceleration and terminal velocity |
| `JUMP_BASE` | -11 | Base jump impulse, plus a speed-scaled bonus/penalty |
| `WATER_GRAVITY_STEP` / `WATER_GRAVITY_CAP` | 0.1 / 2 | Sea-level gravity — much gentler |
| `WATER_JUMP_GRAVITY` | -3.5 | A tap-to-paddle impulse, no on-ground gate |

Rather than re-derive each of these into a px/sec figure for a modern delta-time loop, the port
keeps every constant exactly as written and scales it by `frames = delta * 60` each frame — "how
many original 60fps ticks did this real frame cover." At exactly 60fps that's the identical
formula the original game used; the one rule for anyone tuning feel later is to keep working in
that per-tick unit, never px/sec, because that's what every existing constant is expressed in.

That single design decision — preserve the *unit*, not just the *value* — is a small thing that
would have been easy to get subtly wrong under time pressure, and it's exactly the kind of
detail an AI assistant is good at holding onto consistently across a thousand small edits, once
it's stated once and written down.

![Mario's power-state and morph-transition sprite sheets](../images/mario_player_states.png)

The three power states (Small/Big/Fire) layer on top of that same physics core without changing
it: growing/shrinking swaps the whole frame strip and hitbox size, gated behind a short
transformation animation, and taking a hit demotes you one step (Fire → Small directly, not
Fire → Big → Small) — a genre convention, not a physics change.

## The Reverse-Engineered Game: When the Constants Are Buried in Bytecode

The other game had no such source of truth — "Bobby's" (the player character) movement had to be
recovered from a decompiled Spine-skeleton actor and its accompanying rule set, and the first
pass at this got embarrassingly far from correct before the evidence caught up with it. An
earlier round of research had assumed *no acceleration, fixed jump height* — a completely
plausible read of a 20-year-old mobile platformer. The actual decompiled rules said otherwise,
and the correction is worth walking through because of *how* it got found, not just what it
found.

**Horizontal movement has separate left/right accumulators, not one velocity.** Holding a
direction ramps an accumulator toward a cap (10 for the default hero, 13 for some others) at
+1 every 3 updates; releasing coasts it back toward zero — but not symmetrically:

```text
Golden fixture, from a standing start:
  Holding either direction reaches magnitude 10 at update 30.
  Right release from cap reaches zero after 21 updates.
  Left release from cap reaches zero after 30 updates.
```

That's a *recorded, reproducible* left/right asymmetry in the original's deceleration — not a
bug I'm choosing to preserve out of nostalgia, but a fact I'd have simply never invented if I'd
designed the movement from scratch, and would have quietly erased if I'd "cleaned up" the numbers
into something that looked more symmetric and reasonable.

**Vertical movement is a seven-state machine, not a boolean "grounded."**

| State | Meaning |
|---:|---|
| 0 | Inactive/no-gravity context |
| 1 | Ascending |
| 2 | Ascent-ended / head-hit transition |
| 3 | Descent beginning |
| 4 | Descending |
| 5 | Floor-contact transition |
| 6 | Grounded |

Jump itself has a hold-to-sustain contract with exact numbers: a grounded jump impulse of 28, a
16-update sustain window while the button stays held (writing vertical speed 21, or 24 above
horizontal speed 9 — a speed-scaled jump, same instinct as Mario's own `JUMP_BASE` bonus), and
release simply stops the sustain writes rather than applying some invented "jump-cut" multiplier
that would've been a reasonable guess but isn't what the source does.

**Water entry has a one-time discontinuity, not a smooth transition.** Entering water while
already descending divides the current vertical speed by 4 on that single tick, then gravity
drops to 1.8 (from a 2.0 baseline) for as long as you're submerged. A naive "swimming = low
gravity" implementation would have missed the entry-tick speed division entirely and produced a
visibly different splash-down feel.

Every one of these facts carries the same evidence discipline from the [overview
post](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/): **recovered**
(read straight from the decompiled class), **inferred** (fits every observed rule, not named by
the code), or **still undecoded**. The jump-hold numbers above are recovered; exactly which
in-game event maps a "fast run" animation variant to a real gameplay state is still marked
unresolved rather than papered over with a plausible guess.

## Why Keep Reading Someone Else's Numbers At All?

The honest reason isn't "faithful ports are morally superior." It's narrower: **movement feel is
almost impossible to get right by intuition alone**, and this particular studio had presumably
already spent real playtesting hours tuning theirs. Recovering their numbers exactly — including
the asymmetric release cadence I would never have chosen to *design* — means the new game
inherits that tuning for free, and I get to spend my own design effort on the parts that are
actually new (level layouts, a different actor roster, an original story) rather than
re-discovering "how floaty should a jump feel" from scratch through trial and error.

That's also exactly why the [architecture post](/posts/2026/09/platformer-engine-architecture-guidebee-game-engine/)
in this series draws such a hard line around never letting a shared engine layer auto-derive
physics constants from tile size or genre convention. A `PowerStateActor` base class can share
timers, checkpoints, and morph-transition machinery across two different games for free — but
the moment it tries to share `applyMovement` itself, or auto-scale gravity to match a chosen
tile size, it's making a feel decision on behalf of a game it's never been played. Two different
platformers with the same tile-size-relative jump height can still feel completely different
relative to their own gap widths and enemy heights — there's no formula that reconciles that
automatically, only a human (or, in this project's case, a decompiled binary that already did
the reconciling) actually playing it.

---

## Related Posts

- [Building a Mario-Like Platformer — and Resurrecting Three Lost Android Games](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/) — the series overview
- [From One Game to a Toolkit: Extracting Reusable Platformer Architecture](/posts/2026/09/platformer-engine-architecture-guidebee-game-engine/) — why movement feel stays outside the shared layer
- [Giving NPCs a Brain: Enemy Design and Reverse-Engineered Actor Rules](/posts/2026/09/actors-behavior-rules-reverse-engineered-enemies/) — the same recovered-rules discipline, applied to enemies instead of the player

## Connect

- **GitHub**: [guidebee](https://github.com/guidebee)
- **LinkedIn**: [James Shen](https://www.linkedin.com/in/james-shen-5190926/)

---

*This is post #38 in the Super Morse series — five posts on building an AI-assisted platformer
and reverse-engineering three abandoned Android games alongside it.*
