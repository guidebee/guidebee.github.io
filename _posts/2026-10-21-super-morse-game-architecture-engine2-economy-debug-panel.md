---
layout: single
title: "Super Morse: Game Architecture, Engine 2 Integration, Debug Tools and the Game Economy"
date: 2026-10-21
permalink: /posts/2026/10/super-morse-game-architecture-engine2-economy-debug-panel/
categories:
  - blog
tags:
  - ai-tools
  - claude-code
  - java
  - android
  - game-development
  - software-architecture
  - rule-engine
  - game-economy
  - debugging
  - testing
  - side-project
excerpt: "How Super Morse is put together as a product: the frame pipeline, the Engine 2 host and authority selection, an in-app rule inspector and debug panel, a logged wallet economy with ranks, chests and a Star Road, and the test and device gates that keep it honest."
---

Post #43 in the [Super Morse](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/)
series. The [last post](/posts/2026/10/engine2-rule-engine-introduction-guidebee-game-engine/)
introduced Engine 2, the rule language and VM. This one is about the *game* that runs it: how a
frame is put together, how Engine 2 is plugged into a real Android game without a big-bang rewrite,
the debug tooling we built so we can see what the rules are doing on a phone, and the game economy
that turns a level runner into something you keep coming back to.

## TL;DR

- **One frame, one contract.** Every tick runs a fixed, documented list of phases (hero body, world,
  spawned actors, shots, portals, contacts, programs, camera), and a test asserts that order under
  both the Java and the Engine 2 owner.
- **Engine 2 is adopted one actor and one scene slot at a time.** An authority table decides, at the
  start of each attempt, who owns each resource. Preflight failures keep the Java owner, and a
  single switch rolls everything back.
- **We can see the rules running on a phone.** A debug panel and an in-app **rule inspector** pause
  the game, step one tick, show which rules matched and why, and export the trace as JSONL.
- **The economy is a service, not a scattering of `gold += n`.** One wallet logs every grant and
  spend. On top of it sit level boosters, a Treasure hub (daily calendar, wheels, chests), a free Star
  Road, military ranks with a Quartermaster, and a Treasure Challenge. Every random draw has an
  odds panel.
- **Progress:** 48 product tasks: 21 done, 19 partial, 8 open (about 64% weighted). The rules
  foundation is 100% done; the large actor and scene waves are next.

![A mounted hero in the campaign, with the debug button in the HUD](../images/supermorse_mounted_hero.png)

## 1. The shape of the game

Super Morse is a 2D side-scrolling platformer for Android, written in Java on the Guidebee Game
Engine. Seen from a distance it has four layers:

```text
┌───────────────────────────────────────────────────────────────┐
│ Product layer   screens, hero roster, economy, store, menus   │
│                 EconomyService · HeroRoster · Treasure · Ranks│
├───────────────────────────────────────────────────────────────┤
│ Session layer   GameScreen → StageAttempt (one attempt = one  │
│                 run of a level; retry creates a new attempt)  │
├───────────────────────────────────────────────────────────────┤
│ World layer     MorseWorld: actors, contacts, portals, shots  │
│                 Actor state store (one writer per field)      │
├───────────────────────────────────────────────────────────────┤
│ Behavior layer  Java actor classes  ◄─ authority ─►  Engine 2  │
│                 (legacy owners)          table      modules    │
└───────────────────────────────────────────────────────────────┘
```

The key idea is the **attempt**. A `StageAttempt` is one run of one level. It owns the world, the
input, the scene programs and the reward staging. Retry throws it away and builds a new one, so
nothing that happened in a failed run leaks into the next. That boundary turns out to matter for
almost everything below: Engine 2 selection, booster carry-over and how coins are banked.

## 2. The frame pipeline: order is behavior

In a platformer, the order things run in *is* the gameplay. If contacts resolve before the hero
moves, a stomp becomes a hit. If a rail moves after its rider, the rider jitters. So the first
product task after the Engine 2 core was to write down the tick order once and test it.

| # | Phase | What runs |
|--:|---|---|
| 0 | start | clear per-frame actor state, count the tick |
| 1 | `stage.body` | moving mechanisms (beams carry the hero first), crushers, hero body (input, skills, swim, firing), switches, brick groups |
| 2 | `stage.world` | boss intro hold, then the world update |
| 2a | `world.instances` | per-instance loop in serialized order: bosses, walkers, plants, maces, coins, keys, blocks, finish |
| 2b | `world.spawnedActors` | runtime-spawned actors (effects, clones), expiry, dropped keys and coins |
| 2c | `world.heroShots` | hero weapon shots move and hit |
| 2d | `world.portalTransit` | portal capture and transit |
| 2e | `world.contacts` | every contact from the loop resolved in one pass: damage, grants, finish |
| 2f | `stage.mountAndItemContacts` | balloons, bubbles, wings, mount pickups |
| 3 | `stage.programs` | pipes, tripwires, scene programs, batches, camera pans, teleports |
| 4 | `stage.camera` | camera follows the page |

Engine 2 doesn't get its own slot in this list. An Engine 2 actor is visited **at exactly the
position its Java class would have ticked**, and an owned scene slot runs inside phase 3 where the
scene program host used to. Completion events (a dialogue closed, a transition finished) are seen
at the next tick's phase 3.

To keep this from drifting, a debug hook `TickPhases` (one null check when off) records phase
entries at the ten boundaries, and `TickPhaseOrderTest` asserts the order for scene 30 over 120
ticks. It also checks that scene 65 records **the same phases tick for tick** for 200 ticks under
the Java owner and under the Engine 2 pilots.

## 3. Plugging Engine 2 into a live game

Rewriting every actor at once would have stalled the game for months. Instead, Engine 2 is
adopted **per actor resource and per scene slot**, behind one selection path.

### Owners: Java (legacy) versus Engine 2

Throughout this post, and the codebase, the word **owner** means *the one piece of code that
decides what a thing does this tick*. Every actor instance and every scene-program slot has
exactly one owner for the whole attempt. There are two kinds.

**The Java owner** (also called the **legacy owner**) is the behavior the game shipped with:

- for **actors**, a hand-written Java class per family (`Actor61Effect`, `FishSpawner`, the walker,
  plant and block classes) whose `tick()` method runs inside `MorseWorld`'s instance loop;
- for **scenes**, the Java scene-program interpreter in `SceneProgramHost`, which steps each slot's
  program in phase 3 of the frame.

"Legacy" doesn't mean broken. It means *this is the version we are migrating away from*. It
is still the default for everything Engine 2 has not taken over, and it's the reference that
every Engine 2 port is compared against.

**The Engine 2 owner** is a compiled rule module (`.e2m`) running on the Engine 2 VM. The VM itself
knows nothing about Super Morse. It reaches the game through the **hosts** (`Engine2ActorHost`,
`Engine2SceneHost`), whose operation handlers call into the same world and scene services the
Java owner uses.

```text
                    MorseWorld instance loop (phase 2a) / scene programs (phase 3)
                                         │
                          who owns this actor / slot?
                       ┌─────────────────┴─────────────────┐
                       ▼                                   ▼
               Java owner (legacy)                   Engine 2 owner
          FishSpawner.tick(), Actor61Effect,    VM visits the .e2m module
          SceneProgramHost.tickProgram()        ──► Engine2ActorHost / SceneHost
                       │                              op handlers
                       │                                   │
                       └──────────────┬────────────────────┘
                                      ▼
            shared game mechanics: movement, collision, contacts, camera,
            dialogue, transitions, system variables, actor state store
```

This is the split Engine 2 was designed around: **rules sequence, code does mechanics**. Swapping
the owner changes *who decides* (when the fish launches, which rule kills it). It doesn't change
*how* the fish moves or collides, because both owners call the same mechanics in the same
frame phase.

Three rules keep the two owners from stepping on each other:

1. **One owner per actor or slot, fixed for the attempt.** Selection happens at attempt start;
   retry re-selects. The `OwnerLedger` faults if both owners tick the same actor in one tick.
2. **The Java owner is the fallback, not a safety net mid-run.** If a module fails to load or
   preflight, the Java owner keeps the slot and the `SupportReport` says why. A VM fault *during*
   a run does not quietly switch back, so bugs surface instead of hiding.
3. **Same inputs, same outputs.** Where a family has both owners, the Engine 2 host is handed the
   same random stream the Java class would have used, so the two can be compared tick for tick.
   That's what the parity harness and the Java-versus-Engine 2 differential tests measure.

The end state is **cutover**: once a family's Engine 2 module matches its Java owner and passes the
device gates, Engine 2 becomes its default owner. Once every family is across, the legacy Java
behavior classes are retired, and only the mechanics they called remain.

### A concrete example: scene 30

Scene 30 is the first campaign level (LEVEL 1). It has four scene-program slots, and they show
the owner model well because **one scene runs with two kinds of owner at the same time**.

| Slot | What it does | Rules |
|--:|---|---|
| 0 | the opening story cutscene: scripted animations, actors moved off-screen in batches, a fade out and back in | 273 lines of `.e2` |
| 1 | further scene logic | 54 lines |
| 2 | tutorial prompt at the first marker | 1 rule |
| 3 | tutorial prompt at the second marker | 1 rule |

![Scene 30's intro on a Pixel, with slot 2 owned by Engine 2](../images/supermorse_scene30_engine2_intro.png)

**Slot 2 as rules.** This is the whole program for slot 2, in readable Engine 2 source:

```text
engine2 1.1;
profile "supermorsego.scene030";
system guideClosed: int at 331;

scene CastleIntro resource 30 {
  program tutorial2 at 2 {
    rule prompt repeat once
      when scene.actorPositionCompare(leftActor:1, leftAxis:7, comparison:0,
                                      rightActor:36, rightAxis:7)
       and system.guideClosed == 0
    {
      scene.guideStep(step:0, count:1);
    }
  }
}
```

In words: *once, when the hero (actor 1) reaches the x position of marker actor 36, and the
player hasn't dismissed the guide (system variable 331 is 0), show the first guide step.* Slot 3
is the same rule against marker 37.

**The same slot under each owner:**

```text
                         scene 30, slot 2, phase 3 of every tick
                                        │
              ┌─────────────────────────┴──────────────────────────┐
     Java owner (legacy)                                  Engine 2 owner
SceneProgramHost.tickProgram()                    scene machine visits
interprets slot 2's program data                  scene-030-guide-2.e2m
   │                                                 │
   │ position check in Java                          │ op: actorPositionCompare ─┐
   │ reads system var 331 in Java                    │ op: system read 331       │ Engine2SceneHost
   │ calls guide service                             │ op: guideStep             ┘ handlers
   │                                                 │
   └──────────────────────────┬──────────────────────┘
                              ▼
       same services: actor positions, system variables, guide/dialogue UI
```

Neither owner draws the guide bubble or tracks the hero's position. Both ask the same services.
The only thing that changes is *who evaluates the condition and decides to fire*.

**Selection at attempt start.** With Engine 2 enabled for slot 2 only, the `SupportReport` for a
scene-30 attempt on the phone reads:

```text
scene-030-slot-2 SELECTED module=6367ee51ea89
scene-030-slot-3 KEPT_LEGACY_NOT_CONFIGURED (not in the configured set)
```

So slot 2 is driven by the compiled module (identified by its hash), and slot 3, the identical rule
one marker later, stays with the Java interpreter in the same attempt. Slots 0 and 1 also stay
on the Java owner. If the module had failed to load or preflight, slot 2 would have reported
`KEPT_LEGACY` with the reason and the level would have played exactly as before.

**How we checked it.**

- **Frame order:** `TickPhaseOrderTest` runs scene 30 for 120 ticks and asserts all ten phases
  run in contract order whichever owner holds the slots.
- **On the phone:** the scene-30 intro was captured on a Pixel twice, once with slot 2 on Engine 2
  and once with Engine 2 selection omitted, and showed no visible difference. That's the
  screenshot above.
- **Long runs:** scene 30 is part of the 5,000-tick Java-versus-Engine 2 comparisons. It's also
  where one of the two remaining differences lives: movement diverges at the moment one scene
  rule replaces another, and the opening transition is not yet an exact match. Until that's fixed,
  the story slots stay on the Java owner, which is exactly what per-slot ownership is for.

That's the whole model in one level: the frame order is fixed, the mechanics are shared, and
each slot independently says *who decides*, with the Java owner keeping any slot that
Engine 2 can't yet prove.

### Host, capabilities and authority

```text
                attempt start
                      │
                      ▼
        ┌───────────────────────────┐
        │  Engine2Authority.select  │
        └─────────────┬─────────────┘
       for each registered resource / (scene, slot):
                      │
      load module ──► preflight against ──► install
          │           capability registry       │
          │                 │                    │
      failed?           missing cap?          failed?
          └────────┬────────┴──────────┬─────────┘
                   ▼                   ▼
         keep Java owner      SupportReport entry
                              (logcat -s ENGINE2)
```

- **Capability registry.** `Engine2Capabilities` starts empty. A capability only exists when
  *every* operation in it has a working handler. Having a row in the profile doesn't count.
- **Hosts.** `Engine2ActorHost` and `Engine2SceneHost` dispatch by operation id through a
  registry of small handler objects rather than one giant `switch`. Unknown ops fault with
  `E2_HOST_FAULT`. Scene handlers call the *same* service methods the Java scene host already
  used, so behavior is shared, not copied.
- **Authority table.** Actors are a set of resource ids; scenes are a set of `(sceneId, slot)`
  pairs. Defaults come from a generated ledger and can be overridden in debug builds with an
  intent extra, for example `actors=61,190 scenes=30:2`, without rebuilding the APK.

- **One owner, always.** An `OwnerLedger` makes every tick site claim the actor for either Java or
  Engine 2; both in one tick is a fault, not a silent double update.
- **Rollback.** A single boolean forces every selection back to Java at the next attempt start.
- **Selection is stable inside an attempt.** Retry re-selects; a VM fault mid-attempt does *not*
  fall back silently.

### Where it stands

Today Engine 2 owns real content, not just demos:

- All **34 scene-program slots of scenes 30 to 45** import (116 rules, 744 calls), package into
  byte-reproducible binaries and pass host preflight. A production driver runs each of the 16
  scene bundles for 5,000 idle ticks.
- **48 five-thousand-tick comparisons** between the Java and Engine 2 owners (idle, run right,
  synthetic run-and-jump) produced 46 exact matches. The two that still differ are being worked on.
- **All 42 wave-A actor modules** (effects, controllers, story, finish) are packaged with readable
  sources: 199 rules and 863 calls, every one of their 37 operations with a handler.
- The flying fish, coin magnet and mushroom families run on Engine 2 in the live world, and the
  fish matches its Java implementation for **10,000 ticks**.

Every ported rule also has a readable `.e2` source: all 810 generated sources recompile with
zero byte mismatches, so the source, not the binary, is what a designer edits.

## 4. Debug panel and rule inspector

Parity tests on the JVM tell you *that* something diverged. On a phone you need to see *why*. The
debug build has two layers of tooling.

### The debug panel

The purple **DEBUG** button in the HUD opens a side menu: scene picker, previous/next scene, hop
left/right through a level, pause, a materials overlay, and a live status strip with role, HP,
attempt deaths, coins, keys, blocks exhausted, hero physics state and session state.

![Debug panel over scene 45: status strip, hop controls and scene navigation](../images/supermorse_debug_overlay.png)

Every preview run is labelled `PREVIEW, NOT CERTIFIED`, and debug runs (god mode, scene tests,
previews) **never pay economy rewards**. This was a deliberate rule: a debug tool that can mint
coins will eventually corrupt somebody's save.

### The rule inspector

Tap an actor marker and the inspector opens on it. It shows:

- the actor id, resource, position, state and **owner** (Java, Engine 2 placement, Engine 2 runtime clone),
- the module hash and the current tick,
- every rule in order, **matched or not**, with each condition's evaluated arguments and result,
- each action with its resolved operands, and
- the actor's locals, flags, base info and extras **after** the tick.

![The rule inspector paused on a live fish clone: rule 27 matched, rules 0 to 26 did not](../images/supermorse_rule_inspector.png)

The controls are **RESUME / OLDER / NEWER / STEP 1 TICK / OWNER < / OWNER >**, plus **EXPORT**.
Under the hood:

```text
RuleTraceSink (Java owners)         Engine2ActorTraceRing (one actor)
        │                                    │
        ▼                                    ▼
 RuleTraceRing: last N ticks, grouped by tick, zero cost when detached
        │
        ├──► overlay: one tick at a time, step while paused
        └──► EXPORT: JSON lines (same schema the parity harness reads)
```

Because the export uses the same schema as the parity harness, a trace captured on a phone can
be fed straight into the comparers. The exit check for this task was concrete: on a Pixel, pause
the slice scene, select the fish, and read which rule launched it and which rule killed it. The
exported trace shows rule 16's collision and state predicates evaluating true at tick 2425 and its
defeat actions firing.

## 5. The game economy

The economy was the biggest product addition this month. It's built in two layers: a boring,
well-tested foundation and the screens on top.

### Foundation: one wallet, one log

The old save cleared everything on a schema change. The first job was a **real migration** that
keeps gold, unlocks and progress, then one service through which **every** balance changes:

```java
// EconomyService: the only way a balance changes
public synchronized int grant(String resource, int amount, String reason) { ... }
public synchronized boolean spend(String resource, int amount, String reason) { ... }
```

- The wallet holds coins, keys, hero pieces per hero, a goal hero, insignia and supply, stars,
  ownership (heroes, outfits, mounts, weapon mastery), jackpot and chest counters, daily state and
  an event log.
- Rewards are **staged during the attempt** and only applied on victory, retry or abandon.
  `LevelRewards` applies first-clear, replay, star, perfect and failure-retention rules.
- Tests cover migration, a clock moved backwards, and two devices spending the same coins.

One device check paid for itself immediately: a coin warp spent 500 coins while the level stayed
locked, because the unlock write was never flushed. Now the unlock and the debit **share one
preferences flush**, and every economy action commits in a single flush.

### Hero roster and weapons

Ten hero skins map to rigs, signature weapons or skills, tiers, coin prices and piece counts in
`hero-roster.json`. A new save starts with the starter hero and the fireball. Coins buy **weapon
mastery levels**, which add starting ammo, not new weapons. The coin-magnet hero attracts coins
without needing a mount.

### Level entry: fixed-price boosters

![Level entry: Shield, Heart, Magnet and Guardian boosters at fixed prices](../images/supermorse_level_entry_boosters.png)

Before a level you can buy a Shield, an extra Heart, a Magnet or a Guardian boost, or a Permanent
Guardian. Prices are fixed and shown, and **retries keep unused boosters**.

### Treasure hub, Star Road and the Challenge

![Treasure hub: Daily Calendar, Lucky Wheel and three chest tiers, each with an odds panel](../images/supermorse_treasure_hub.png)

The Treasure hub has a seven-day Daily Calendar with a Daily Spin, a Lucky Wheel (one key per spin,
jackpot counter), and wooden, silver and gold chests. Every card has a **?** that opens its odds and
guarantees. The footer says it plainly: *earned from play only*.

![Star Road: a free reward every 15 stars](../images/supermorse_star_road.png)

The **Star Road** is the deliberately non-random track: a free reward every 15 stars (a hero,
coins, keys, goal-hero pieces, a gold chest). Claims go in order, in one flush.

![Treasure Challenge after a first boss clear: one free pick from three face-down cards](../images/supermorse_treasure_challenge.png)

A boss's first clear pays a Gold Star and one **Treasure Challenge**: three face-down cards, one of
each kind (coins scaled by worlds cleared, goal-hero pieces, or keys), in a shuffled order. You get
one free pick, then the other two are revealed. There is no paid re-pick.

### Ranks and the Quartermaster

![Ranks and Quartermaster: Army, Navy and Air Force branches, Gold Stars, trades and a free daily transfer](../images/supermorse_ranks_quartermaster.png)

Defeating enemies and breaking bricks can drop **insignia**. The branch comes from the actor family:
walkers, frogs, plants and bricks are Army; pufferfish and squid are Navy; flying fish are Air
Force. Each drop adds rank progress (never spent; *ranks never fall*) and supply, which you spend
at the Quartermaster on coins, ammo, goal-hero pieces or an outfit.

The drop rule is a pity timer: 10 in 150, +1 per miss, reset on a drop, 4% of drops are Gold
Stars. It's capped at 4 drops per attempt in an uncleared scene and 1 per scene per day once
cleared. A 3,000-roll test checks the average lands at roughly one drop per 9 to 10 defeats.

The in-level side is an `InsigniaObserver` that **reads** the world after each tick and reports
enemies that went from live to defeated, plus newly broken bricks. It never writes gameplay state
and never touches the rule random streams, so the economy can't change how a level plays.
Script removals (an actor set dead with no stomp, knockout or capture) don't count.

### Economy principles

```text
✓ every balance change goes through EconomyService and is logged
✓ every multi-step action commits in one flush
✓ every random draw has an odds panel
✓ no paid product contains a random item
✓ debug, preview and scene-test runs never pay rewards
✓ observers read the world; they never write it
```

## 6. Levels, campaign and certification

New levels come in as **authored scene packages**. A conversion tool produces a package plus a
record of every adaptation decision, and `AuthoredScenePackage` loads it through the same
`StageAttempt` as a built-in level. Authored scenes can place more than 35 typed actor families:
springs, cloud and crumbling platforms, rail movers, ice strips, question and gift blocks, pipes,
turtles, squid, piranha plants, mace chains, cannons, ice spikes and more, each with retry
handling.

Two authored packages are now mapped to campaign levels 544 and 545, behind a **release
certification gate**: an exact-hash certification matrix and a release manifest that stays empty
until every gate (completion, retry, persistence and Continue, death, visuals) is signed off. Mount
campaign certification is done: the hero picks up a mount, rides it, and finishes the level on a
Pixel.

## 7. How we verify

Each task has written exit criteria, and "done" means tests green **and** a device check where the
task touches the player:

- **JVM suites:** about 1,945 app tests and 175 `gameengine` tests, run in both normal and
  forced-Engine 2 authority modes.
- **Differential tests:** Java owner versus Engine 2 module, tick for tick, from 2,000 to 10,000 ticks.
- **Device checks on a Pixel**, hash-bound to the APK, with screenshots kept as evidence.
- **Save safety:** device checks on the economy run on a scratch save, then the owner's real
  739-byte profile is restored and verified by SHA-256.

## Impact and Next Steps

**What this buys:**

- Engine 2 now owns real scene slots and actor families in a shipping-shaped game, with a one-flag
  rollback.
- When a rule misbehaves on a phone, we can pause, step and read *why*, then replay the exported
  trace on the JVM.
- The economy is auditable: one wallet, one log, atomic commits, visible odds.

**What's next:**

1. **Scene waves 2 and 3**: scenes 46 to 310 and the non-campaign scenes.
2. **Actor waves C to E**: mechanisms, enemies and hazards, projectiles and bosses.
3. **Hero port**: locomotion, weapons, forms, mounts, transitions, death and revival as rules.
4. **Store and billing**: fixed-price products, the collection book, telemetry by reason.
5. **Performance and memory budgets**, then cutover with Engine 2 as the default owner.

## Conclusion

The theme of this stretch was making the game *observable and accountable*. The frame order is
written down and tested, Engine 2 ownership is an explicit, reported decision per slot, the rule
inspector shows what the VM did on a real device, and every coin has a reason in the log. With
those in place, the rest of the port and the rest of the economy can be added one wave at a time
without guessing.

---

## Related Posts

- [Introducing Engine 2: A Rule Engine for Game Behavior](/posts/2026/10/engine2-rule-engine-introduction-guidebee-game-engine/): the language and VM this post integrates
- [From One Game to a Toolkit: Extracting Reusable Platformer Architecture](/posts/2026/09/platformer-engine-architecture-guidebee-game-engine/): the engine boundary
- [Level Design at Scale: Atlases and Tile Catalogs](/posts/2026/09/level-design-at-scale-atlases-tile-catalogs/): the level pipeline

## Connect

- **GitHub**: [guidebee](https://github.com/guidebee)
- **LinkedIn**: [James Shen](https://www.linkedin.com/in/james-shen-5190926/)

---

*This is post #43 in the Super Morse series. Write the frame order down, make every owner explicit,
log every coin, and give yourself a way to watch the rules run on a real phone.*
