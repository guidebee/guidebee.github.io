---
layout: single
title: "Engine 2: Designing a Clean-Room Rule VM for a Decompiled Platformer"
date: 2026-09-26
permalink: /posts/2026/09/engine2-rule-vm-design-guidebee-game-engine/
categories:
  - blog
tags:
  - ai-tools
  - claude-code
  - android
  - java
  - game-development
  - software-architecture
  - reverse-engineering
  - lua
  - side-project
excerpt: "A spec-first look at Engine 2: the data-driven rule VM I'm designing to run three decompiled Android platformers' original scene and actor logic, how it slots into the Guidebee Game Engine, and why the implementation hasn't started yet — on purpose."
---

Post #41 in the [Super Morse](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/)
series. The [architecture post](/posts/2026/09/platformer-engine-architecture-guidebee-game-engine/)
in this series described a proposed compiler/VM pair for executing behavior recovered from a
decompiled binary, in one paragraph. This post is the deep dive on that system — codenamed
**Engine 2** — written the way I'd write it for a team about to build it: purpose, spec shape,
runtime contract, how it sits under the Guidebee Game Engine, and where a future embedded Lua
frontend fits. Everything below is pulled directly from `docs/superbobgo/rules/` in the project
repo, which is documentation only — **no Engine 2 code exists yet**, and that's a deliberate
architectural stance I want to defend, not an admission of stalled work.

## TL;DR

- **Engine 2** is a clean-room, purpose-built rule VM that reproduces the *orchestration* layer
  of three decompiled Android platformers (Super Bob Go, Madino, DreamWorld) — scene programs
  and per-actor behavior rules — without copying their obfuscated bytecode or reinventing their
  actual game mechanics.
- It has **two small VMs** (an actor machine and a scene machine) sharing values, arithmetic, and
  provenance, but deliberately separate opcode namespaces and control-flow rules, because the
  original engine treats them as genuinely different machines.
- The reusable core is designed to live in `gameengine` (the shared library three of my Android
  games already build on); each game's specific opcode registry, codecs, and world bindings stay
  in that game's own app module — the same layering discipline from the
  [architecture post](/posts/2026/09/platformer-engine-architecture-guidebee-game-engine/).
- As of today, **11 of 14 reverse-engineering evidence gates are closed**, cross-checked across
  all three decompiled games — but the runtime, compiler, and every VM are still **M0: evidence
  and inventory**, the first of nine planned implementation milestones. Zero production code
  exists. That's intentional: this project doesn't write a VM until it knows, with cited
  evidence, what that VM is supposed to do.
- A future **embedded Lua frontend** is explicitly scoped as optional, sandboxed, and last —
  after the Java compatibility boundary is proven, not instead of it.

---

## 1. Purpose: Why a Custom Rule VM, and Why Not Just Java

Three of the decompiled games I'm reverse-engineering (see the
[overview post](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/)) aren't
just collections of hand-written actor classes. Their original engine is **data-driven**: binary
**scene programs** (`.W123` files) coordinate stage-wide flow — cutscenes, camera moves, victory
sequences — and binary **actor-resource rules** (`.mo123` files) give each of 316 shared actor
definitions its own reusable state machine. A Java runtime turns numeric condition/action type IDs
into engine operations. Recreating that separation — content as data, mechanics as verified code —
without copying the original's obfuscation or its bugs, is the whole reason Engine 2 exists.

The alternative — hand-writing a Java kernel for all 316 actor resources and 356 scenes — is
exactly what the project's own docs warn against as a non-goal: *"blind line-by-line translation
of obfuscated Java"* and *"one Java class per actor resource."* That doesn't scale, and it also
throws away the one thing the decompiled binary actually gives me for free: a precise, if
obfuscated, record of *exactly* how the original game's mushrooms, buttons, and bosses behave.
Engine 2's job is to make that record executable, safely, as data — while every actual mechanic
(physics, collision, damage, animation, camera) stays exactly where it belongs: in typed,
verified Java services that both a legacy-recovered rule and a brand-new authored rule can call
into identically.

## 2. Brief Spec: What the Recovered Format Actually Looks Like

The recovered binary grammar is small and strict — big-endian, no generic length prefixes, and it
refuses to guess past an unknown opcode:

```text
SceneFile = i16(programCount), SceneProgram[programCount]
SceneProgram = i16(ruleCount), SceneRule[ruleCount]
SceneRule = i8(conditionJoin), i8(branchMode), i8(activationSelector),
            i16(repeat), i16(conditionCount), SceneCondition[conditionCount],
            i16(actionCount), SceneAction[actionCount]

ActorFile = i16(ruleCount), ActorRule[ruleCount]
ActorRule = i8(branchMode),
            [i16(serializedJumpTarget) when branchMode in {4,5}],
            i8(conditionJoin),
            i16(conditionCount), ActorCondition[conditionCount],
            i16(actionCount), ActorAction[actionCount]
```

Four **independent** opcode families live in that grammar — scene conditions (0–14), scene
actions (0–55), actor conditions (0–32), actor actions (0–60), 165 registered slots total — and
the spec is explicit that a scene action and an actor action sharing a numeric ID can mean
completely unrelated things. That single rule ("do not share a switch on a bare type ID between
scene and actor execution") is exactly the kind of detail that's invisible until it silently
corrupts a decoder, and it's stated as a hard constraint rather than a footnote.

Every claim in the spec carries one of four evidence tags, and the tagging is load-bearing, not
cosmetic:

| Tag | Meaning |
|---|---|
| **Recovered** | Backed by serialized data and/or an identified original method |
| **Derived** | A consequence of those records/control flow, reasoning stated |
| **Design** | A decision for the *new* implementation — not an original-engine fact |
| **Open** | Ambiguous, truncated, obfuscated, or not yet followed to its dependencies |

A good example of why "Derived" needs its own tag, distinct from "Recovered": actor condition 16
answers "is this actor a spawned clone or the original scene placement" — a question every
spawner-style actor in the game asks constantly. The recovered Java doesn't store two named
booleans anywhere. It compares one raw instance-ID slot against the scene's serialized placement
count:

```java
self.f38044r[1] >= scene.QJO358.length
```

The spec derives `isSpawnedClone` / `isScenePlacement` from that comparison — a clean, typed API
worth exposing to authors — while explicitly flagging that *"two explicitly stored booleans...
and universal support across every Engine 2 game are not established by this evidence."` That's
the discipline this whole project runs on: build the clean abstraction, but don't let the
abstraction quietly overclaim what the evidence actually proves.

### The value/arithmetic trap that would have shipped silently wrong

The spec calls out one gotcha explicitly because it's the kind of thing that passes code review
and still produces a wrong game: there are **three separate arithmetic namespaces** — ordinary
mutation, typed transfer, and general arithmetic — and the same numeric mode means different
things in each:

| Ordinary mutation mode 2 | Transfer write mode 2 |
|---|---|
| Legacy random-range operation | `min(current, value)` |

With `current=10, value=3`: ordinary mutation mode 2 draws a random number in a range; transfer
mode 2 deterministically yields `min(10, 3) = 3`. Reusing one lookup table for both would produce
a program that's silently non-deterministic where the original was fixed, or vice versa — exactly
the kind of defect that only shows up as "this enemy behaves weirdly sometimes" three weeks later.
Naming this collision explicitly, with a worked fixture, is cheaper than debugging it after the
fact.

## 3. From Numeric Opcodes to a Human Language

Nobody should hand-author `{"typeId": 16, "fields": {"f118a": true}}`. Engine 2 defines a small,
strict authoring language (`.e2`) that compiles down to the same typed IR both source and
decoded-legacy-binary converge on:

```engine2
engine2 1;
profile "superbobgo";

actor Effect resource 61 {
    animation idle = 0;

    rule removeFinishedClone
    when self.isSpawnedClone
         and self.animation.id == idle
         and self.animation.completionSignaled {
        self.disable();
        self.remove();
    }
}
```

That one rule is the entire recovered behavior of actor resource 61 (a one-shot visual effect):
wait for a spawned clone whose idle animation has genuinely completed, then disable and remove
it. The compiler pipeline that gets it there is deliberately linear and inspectable:

```text
Human .e2 source ---\
                      +--> typed IR --> target validation --> legacy binary writer (.mo123/.W123)
Decoded legacy JSON -/                                    \-> .e2b module (payload + metadata + source map)
```

Two binary outputs, two different jobs: an exact-recovered-bytes **legacy export** for decoder
round-trips, and a versioned **`.e2b` module** carrying the same payload plus symbols,
capabilities, and source maps for the shipping game. Neither accepts an unresolved operation
silently — an operation without verified semantics blocks production compilation rather than
degrading into a guess.

## 4. Runtime Summary: Two Machines, One Discipline

Engine 2 has an **actor machine** and a **scene machine**, and the spec is emphatic that they are
not the same state machine wearing two hats — they have different branch-dispatch rules, different
action-result meanings, and different jump semantics:

```text
Actor branch dispatch (modes 0-5):
  pc = 0
  while pc < ruleCount:
      r = rules[pc]
      if r.mode in {1,3,5} and rules[pc-1].status in {1,2}:   # else-if eligibility
          status[pc] = 2; pc += 1; continue
      matched = evaluateConditionGroup(r)                     # ordered short-circuit
      if not matched: pc += 1; continue
      executeActorActionsUntilEndOrResult1(r.actions)
      if r.mode in {2,3}: return
      if r.mode in {4,5}: pc = r.storedShortTarget             # jump — actors only
      pc += 1
```

Scene rules, by contrast, have **no jump operand at all** — only branch modes 0–3 — and their
action list resumes across ticks through an explicit cursor rather than a single-frame loop:

```text
Scene action continuation:
  while actionCursor < actionCount:
      action = actions[actionCursor]; actionCursor += 1       # cursor advances BEFORE dispatch
      result = executeSceneAction(action)
      if result == 1: return BLOCKED                          # e.g. a `wait ticks(n)`
      if result == 2: break
  owner.executing = false; return COMPLETED
```

That cursor-advances-before-dispatch detail is exactly the kind of thing that's trivial to get
backwards and only surfaces as "the wait action ran twice" days later. It's called out explicitly
in the runtime design specifically so nobody has to rediscover it by debugging a duplicated sound
effect.

**One invariant governs both machines and the whole authority model:** exactly one system may
mutate a given actor or scene program at a time — `JAVA_KERNEL`, `LEGACY_RULES` (recovered,
running), or `GUIDEBEE_RULES` (newly authored) — with a non-mutating `SHADOW_COMPARE` mode that
runs a candidate rule against a cloned world purely to diff its intended effects against the live
authority, before that candidate is ever trusted. This is precisely how a resource gets
*certified*: run it in shadow for a while, compare traces, only then flip authority — never two
systems racing to write the same actor in the same tick.

Everything the runtime touches is checkpoint-able for the same reason a fighting game needs
deterministic replay: content hash, per-rule branch/cursor state, waits, PRNG state, and pending
commands all serialize as stable IDs, never object pointers — "restore refuses incompatible
content" is a stated requirement, not an afterthought.

## 5. How Engine 2 Sits Under the Guidebee Game Engine

This is where Engine 2 connects directly to the [architecture
post](/posts/2026/09/platformer-engine-architecture-guidebee-game-engine/)'s bigger picture. The
Guidebee Game Engine (GGE) is my ~2013-era libGDX fork — `Stage`/`Actor`, the `microedition`
`LayerManager`/`Sprite`/`TiledLayer` API, Box2D — already shared by three games in this app.
Engine 2's ownership boundary is designed the same way the platformer toolkit's is: reusable
mechanism in the shared library, game-specific everything else in the app module.

```text
Guidebee Game Engine (gameengine/)      — generic 2D framework: Stage/Actor,
                                           LayerManager/Sprite/TiledLayer, Box2D. Untouched.
        │
        ▼
Engine 2 core (gameengine/.../engine2/) — parser, typed IR, compiler pipeline, binary
                                           module loader/writer, actor/scene VMs, arithmetic,
                                           continuations, tracing, capability interfaces.
        │
        ├─────────────────────┬─────────────────────
        ▼                     ▼
Super Bob Go profile      Madino / DreamWorld profile (future)
+ adapter (app/)          + adapter (app/)             — game-specific opcode registry,
                                                          codecs, symbols, and the typed
                                                          host bindings into that game's
                                                          own MorseWorld/ActorStore.
```

The rule stated for this boundary is direct: *"Do not put Super Bob Go class names, resource IDs,
decoded assets, or `MorseWorld` dependencies in `gameengine`. Do not put generic VM control flow
or compiler logic in the Super Bob Go adapter."* Concretely, that means the core `ActorMachine`
knows how to walk a branch table and dispatch a condition group — it has no idea what "resource
61" or "a mushroom" is. The Super Bob Go profile knows that actor condition 16 means "is this a
spawned clone," and hands the core a typed operation to execute; the adapter underneath translates
that into an actual read against `MorseWorld`. A future Madino or DreamWorld profile — a real
possibility, since all three games share this exact 165-operation registry shape almost
byte-for-byte — plugs into the identical core interfaces without touching a line of Super Bob Go
code.

Just as important is what Engine 2 explicitly refuses to own: physics, collision, damage,
animation playback, and rendering all stay Java services the rule layer merely *calls*. A rule can
say "start this actor ascending" or "stamp a one-way platform material onto this cell" — it can
never redefine what ascending or one-way material *means*. That's the same "never genericize
movement feel" boundary from the [physics post](/posts/2026/09/tuning-jump-arc-platformer-physics-ai/)
in this series, generalized from the player character to every actor in the game: orchestration is
data, mechanics are code, and the line between them doesn't move just because the orchestration
layer got fancier.

## 6. Where the Evidence Actually Stands Today

This is the part I want to be precise about, because "not finished" undersells how much
groundwork is actually done. The junior implementation plan lists 14 open reverse-engineering
questions (R01–R14) — things like "does actor condition evaluation short-circuit, and in what
order" or "what exactly does a scene's `wait` resume boundary look like across frames." Cross-
referencing all **three** decompiled games (Super Bob Go, Madino, DreamWorld) against each other —
since they share the same underlying engine with different obfuscation — has closed **11 of 14**:

| Status | Gates |
|---|---|
| **Closed** | R01 (actor condition order), R02 (branch/jump semantics), R03 (scene condition order), R04 (scene scheduler), R05 (delay/wait boundary), R06 (repeat/retry scope), R08 (animation signal contract), R10 (flag/base-info contract), R11 (arithmetic/numeric edge cases), R12 (nested actor rules), R14 (scene activation selectors) |
| **Open** | R07 (actor reference-resolution fallback matrix), R09 (spawn/remove/pooling lifecycle), R13 (long-tail opcode semantics) |

Every closed gate has a named evidence file with source citations, DEX comparisons across all
three games, and — for the numeric ones — dozens of extracted-body assertions (R11 alone verifies
its arithmetic tables with 150 such assertions). That's the standard this whole reverse-engineering
effort holds itself to: a gate isn't "closed" because a decoder stopped crashing, it's closed
because independent evidence from multiple builds agrees on the exact behavior, with citations.

And yet: the implementation plan's own milestone tracker still reads **M0 — freeze evidence and
support inventory**, the first of nine (M0 through M8). No `ActorMachine`, no `SceneMachine`, no
compiler, no adapter exists in code yet. That gap — eleven closed research gates, zero lines of
VM — is the architectural decision I most want to defend here: **a spec is allowed to outrun its
implementation, on purpose, when the alternative is code built on unverified assumptions.** Writing
`ActorMachine` against an *inferred* branch-dispatch order, only to discover from R02's evidence
three weeks later that leading `else-if` reads `rules[-1]` and fails on purpose in the original —
a real, cited detail in this spec — would mean rewriting the VM's core loop after already building
game content on top of it. Better to let the documentation be embarrassingly far ahead of the code
for a while.

## 7. Two Higher-Level Frontends, Deliberately Sequenced Last

Two more pieces sit *above* Engine 2's compiler, and both are scoped as explicitly optional,
explicitly later, and explicitly gated behind the Java compatibility boundary being proven first —
not because they're unimportant, but because sequencing them any earlier would mean designing
against an unstable foundation.

**Guidebee custom authoring** (`guidebee-scene-rules-v1` / `guidebee-actor-rules-v1`) is a
friendlier, versioned JSON schema for *new* content — not recovered legacy behavior — with named
operations like `cameraFollow`, `delayUpdates`, `showDialog` instead of raw opcode numbers. It
shares Engine 2's compiler, IR, and one-authority invariant; the difference is that its operations
only need to be authoring-safe for new scenes I write myself, not compatible with a decompiled
binary's exact historical behavior.

**An embedded Lua frontend** is the furthest-out piece, and its scoping is worth quoting directly
because it's a good model for introducing a scripting layer into any engine without it becoming a
security or determinism problem: *"Lua does not supply missing recovered semantics or live
services."* A Lua adapter would query the same immutable fixed-tick snapshot and emit the same
typed command intents the Java VM does — no direct access to `GameScreen`, the renderer, Android
objects, files, threads, or unrestricted reflection. `io`, `os`, `package`, `debug`, `dofile`, and
Lua-to-Java reflection are explicitly excluded from the gameplay sandbox. The pilot sequence is
four gated steps: finish the Java semantic/authority boundary first; run a restricted technical
spike proving sandboxing, budgets, and device cost; author *one* new scene in shadow mode and
compare traces before granting real authority; only then consider a fully certified recovered
scene. Complex hero physics stays in Java "unless separately justified and certified" — Lua is for
orchestration convenience, never a backdoor around the mechanics boundary Section 5 draws.

## 8. A Solution Architect's Read on the Design

Stepping back from the spec itself: the single decision I'd defend hardest to a skeptical
reviewer is the **evidence-tagging discipline** (Recovered/Derived/Design/Open) applied uniformly
across a 165-operation, three-game corpus. It's more upfront overhead than just writing a VM and
patching bugs as players find them — but the failure mode it prevents is worse than a bug: a
*confidently wrong* rule engine that looks complete because every opcode has a handler, while a
third of those handlers quietly do the wrong thing for a variant nobody fixture-tested. The
project's own status doc says this plainly: *"a coverage report with zero unsupported type IDs
does not certify a scene."* That sentence is the whole design philosophy in one line.

The risk I'd flag for anyone picking this up next is the same one the architecture post raised
about the platformer toolkit: **don't let the reusable core's shape get frozen around Super Bob
Go's specific needs before a second profile (Madino or DreamWorld) actually exercises it.** The
three-game cross-referencing that closed 11 evidence gates is genuinely strong validation that the
*opcode registry* generalizes — but Section 5's core/profile boundary hasn't yet been proven by
building a second real adapter against it, the same "wait for the second consumer" discipline this
whole project keeps returning to.

## Conclusion

Engine 2 is, right now, a very well-evidenced plan and zero production code — and I think that's
the correct state for it to be in. A decompiled binary only tells you the truth once; if the VM
built to execute its recovered logic gets the branch-dispatch order or the arithmetic-namespace
collision wrong, that mistake ships silently, because nothing about "the code compiles and the
game runs" would catch it. Closing 11 of 14 evidence gates with three-game cross-referencing before
writing `ActorMachine`'s first line is slower than shipping a plausible VM and fixing it in the
field — and it's the only way I'd trust the result enough to eventually let it run a real scene.

---

## Related Posts

- [Building a Mario-Like Platformer — and Resurrecting Three Lost Android Games](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/) — the series overview
- [From One Game to a Toolkit: Extracting Reusable Platformer Architecture](/posts/2026/09/platformer-engine-architecture-guidebee-game-engine/) — the broader engine-boundary design this VM plugs into
- [Giving NPCs a Brain: Enemy Design and Reverse-Engineered Actor Rules](/posts/2026/09/actors-behavior-rules-reverse-engineered-enemies/) — the actor behaviors Engine 2 is built to eventually execute

## Connect

- **GitHub**: [guidebee](https://github.com/guidebee)
- **LinkedIn**: [James Shen](https://www.linkedin.com/in/james-shen-5190926/)

---

*This is post #41 in the Super Morse series. Same throughline as the rest: evidence before code,
and a shared core that earns its abstractions by proving them against a second real consumer
before calling them settled.*
