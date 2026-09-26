---
layout: single
title: "From One Game to a Toolkit: Extracting Reusable Platformer Architecture With AI"
date: 2026-09-26
permalink: /posts/2026/09/platformer-engine-architecture-guidebee-game-engine/
categories:
  - blog
tags:
  - ai-tools
  - claude-code
  - android
  - java
  - game-development
  - software-architecture
  - side-project
excerpt: "Two very different engine problems in one project: refactoring a working Mario port into a reusable platformer toolkit without breaking it, and designing a compiler/VM pair that can execute a decompiled game's own scripted behavior safely."
---

![In-game HUD and touch controls, running on the Guidebee Game Engine](../images/supermorse_gameplay_hud.png)

This is post #37 in the [Super Morse](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/)
series — the engine-architecture deep dive. If you haven't read the overview post, the short
version: one Android app, two problems — porting a Mario clone onto my own decade-old libGDX
fork, and executing behavior recovered from a decompiled, unrelated platformer safely. Both
problems turned out to be architecture problems before they were content problems.

## TL;DR

- The Mario port sits directly on the **Guidebee Game Engine (GGE)**, a ~2013-era fork of
  libGDX's MIDP-style `microedition` API (`LayerManager`/`Sprite`/`TiledLayer`) — the same layer
  my Flappy Bird and Battle City clones already use.
- With Phase 2 of the Mario port functionally done, the next move isn't "build a second game" —
  it's a **proposed extraction**: pull the genuinely reusable ~60% of Mario's code into a
  `platformer` toolkit package, so a *future* platform game doesn't start by copy-pasting
  `activity/mario/` and rewriting most of it.
- For the reverse-engineered game, the harder architectural problem was different: how do you
  safely *execute* behavior recovered from someone else's obfuscated bytecode, without either
  reimplementing their entire VM or trusting a translation layer you can't verify? The answer is
  a small, purpose-built compiler and two tiny VMs — not a general scripting language.
- Both designs share one instinct an AI assistant turned out to be unusually disciplined about
  enforcing: **never genericize the one thing that should stay hand-tuned** — movement feel in
  one case, Java-verified game mechanics in the other.

---

## Problem One: When Does "Working Code" Deserve a Refactor?

The Mario port's gameplay code (`activity/mario/`) sits on GGE the same way Flappy Bird and
Battle City do — nothing new there. What's new is asking: if I built a *second*, unrelated
platform game next, how much of Mario's code could I actually reuse unchanged?

The honest answer, audited class by class rather than guessed, was "less than it looks like, but
more than nothing." Sorting every class in `activity/mario/` by how much rework moving it to a
shared layer would need turned up three clean buckets:

**Already generic — move verbatim.** `TileMovement`'s wall/floor/ceiling clamp math, an
`OscillatorClock` timer utility, a `CameraController` that's parameterized entirely by
constructor arguments, and the "one static method per interaction pair, called in a fixed order"
collision-resolver *pattern* — none of these read a Mario-specific constant anywhere in their
body.

**Generic shape, game-specific data — extract as a parameterized base.** A score/coins/lives
controller is really "a counter with a coin→life threshold," three named constants away from
reusable. A save-state class is "a set of cleared level IDs behind `Preferences`," one hardcoded
string away. The pattern repeats across the menu screen, the HUD, and the debug panel.

**Genuinely game-specific, but worth a real extension point.** `Player`'s **physics constants**
(jump arc, run speed, water-mode gravity) are Mario's own tuned feel and stay put — deliberately.
Its **scaffolding** (invincibility timers, morph-transition flipbook machinery, checkpoint/respawn
logic) is a different story: none of it reads a Mario-specific constant either, so it becomes an
abstract `PowerStateActor<S extends Enum<S>>` base class with one abstract method,
`applyMovement`, that a future game's player character must supply itself:

```java
public abstract class PowerStateActor<S extends Enum<S>> extends Layer {
    protected S powerState;
    private float invincibleTimer, shieldTimer;
    private TextureRegion[] transitionFrames;   // morph-flipbook machinery
    private float checkpointX, checkpointY;     // respawn point
    // consumeDeath()/setInvincibleFor()/updateCheckpoint(), all moved verbatim —
    // none of it reads a Mario-specific constant

    /** The one thing a subclass must supply: how to actually move this frame. */
    protected abstract void applyMovement(PlatformerCommand command, float frames);
}
```

The highest-leverage single change in the whole proposal, though, is turning six hardcoded
`switch` statements — `LevelLoader.spawnBricks`/`spawnEnemies`/`spawnHazards`/... — into a
`TileTypeRegistry` a game populates instead of a loader a game forks:

```java
TileTypeRegistry registry = new TileTypeRegistry()
    .register("Brick", (tile, level, ctx) -> add(ctx, new Brick(tile.x, tile.y)))
    .register("EnemyMushroom", (tile, level, ctx) -> addEnemy(ctx, new EnemyMushroom(tile.x, tile.y)));
```

A second, hypothetical game writes its own table for its own tile vocabulary — `Girder`, `Drone`,
`BatteryCell` — and the generic loader underneath needs zero changes to support it.

## The Part I Insisted On: This Is a Proposal, Not a Mandate

Here's the thing this design doc gets right that I want to call out specifically, because it's
the same instinct I leaned on in the iRobot node-graph design: it says, in its own words, *"this
restructuring buys zero player-visible benefit for Mario on its own... if there's no concrete
plan or strong intent to build a second platform game in this app, the honest recommendation is:
don't do this yet."*

That's not hedging — it's the correct answer to "should we abstract this," and it's a harder
answer for an AI assistant to hold onto than you'd think, because "propose an elegant reusable
architecture" is a much more satisfying deliverable than "here's a design, and here's why you
shouldn't build it yet." The whole proposal is still gated behind a Phase H: build a small,
throwaway second game on top of the toolkit *before* calling any of it stable — the standard
"wait for the second real use case" discipline for avoiding a plausible-but-wrong abstraction. As
of today, nothing in Phase A–H has actually shipped; the toolkit stays a design doc until a real
second game gives it something to prove itself against.

## Problem Two: Executing Someone Else's Recovered Behavior, Safely

The reverse-engineered game's engine problem looks nothing like the Mario one on the surface, but
it rhymes. The original game (call it Super Bob Go) isn't just Java actor classes — it's a
data-driven engine: binary **scene programs** coordinate stage-wide behavior, binary
**actor-resource rules** define reusable per-actor state machines, and a Java runtime turns
numeric condition/action type IDs into engine operations. I want to recreate that separation of
concerns — content as data, mechanics as verified Java — without copying the original's
obfuscated authoring format or its bugs.

The architecture that fell out of that goal is a compiler/VM pipeline:

```text
semantic scene/actor documents
          |
          v
validator + compiler + linker
          |
          v
typed immutable runtime IR
          |
          v
scene VM + actor VM ----> typed Java engine services
                              |
                              v
                 physics, collision, animation,
                 camera, audio, UI, spawning, saves
```

A few decisions here were non-negotiable early, and stayed that way through the whole design:

- **Rules are orchestration, not mechanics.** A rule can say "when the hero touches this,
  trigger a stomp" — it never re-implements what a stomp *is*. Physics, collision, and damage all
  stay behind typed Java services the rule layer merely calls.
- **Exactly one mutating authority per actor, ever.** A given actor placement is driven by
  hand-written Java, by interpreted *legacy* rules recovered from the original binary, or by new
  *authored* rules I write myself — never two at once. A `SHADOW_COMPARE` mode lets a candidate
  rule run non-mutating, side-by-side, purely to compare its intended output against the live
  authority — which is how a resource gets *certified* before its authority is ever switched.
- **Scene rules and actor rules get separate instruction registries.** Their legacy numeric IDs
  and execution contexts genuinely differ in the original engine; pretending they're one
  instruction set would just reintroduce the obfuscation I'm trying to get rid of, with extra
  steps.
- **The editor is not allowed a second interpretation of runtime semantics.** Anything the future
  authoring tool previews must invoke the exact same compiler and Java services the shipped game
  runs — never a parallel, editor-only physics implementation that can quietly drift from what
  actually ships.

This is still a proposed target architecture, not implemented — Phase 0 of its own six-phase
rollout is where things stand today: viewers stay read-only over the legacy programs, provenance
gets preserved, and coverage inventories get generated, before a single line of the actual VM
gets written.

## The Common Thread

Both designs above got the most valuable pushback from the same place: forcing an explicit
answer to *"would a second, unrelated consumer plausibly want this exact code, unchanged?"* For
the Mario toolkit, that's a hypothetical second game, deliberately deferred to a real Phase H
before being trusted. For the rule engine, it's sharper — the "second consumer" is a decompiled
binary I don't get to redesign, so the compiler has to swallow whatever the legacy format
actually did, correctly, while still giving new content a clean, semantic authoring surface on
top. Getting an AI assistant to hold both halves of that — respect the old evidence, don't let it
contaminate the new design — turned out to be the real architectural work.

---

## Related Posts

- [Building a Mario-Like Platformer — and Resurrecting Three Lost Android Games](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/) — the series overview
- [Tuning the Jump Arc: Movement Physics, Recovered and Designed](/posts/2026/09/tuning-jump-arc-platformer-physics-ai/) — where these architectural boundaries (movement feel vs. scaffolding) get put to use
- [Reviving iRobot: An Old Android Project, Six Years Later, With AI Agents as Co-Developers](/posts/2026/09/reviving-irobot-old-project-ai-agents/) — the same "propose it, then insist on proving it" discipline, applied to a node-graph editor instead

## Connect

- **GitHub**: [guidebee](https://github.com/guidebee)
- **LinkedIn**: [James Shen](https://www.linkedin.com/in/james-shen-5190926/)

---

*This is post #37 in the Super Morse series — five posts on building an AI-assisted platformer
and reverse-engineering three abandoned Android games alongside it.*
