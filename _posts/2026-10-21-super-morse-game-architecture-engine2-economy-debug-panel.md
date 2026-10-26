---
layout: single
title: "Super Morse: Game Architecture, Engine 2 Integration and Debug Tools"
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
  - debugging
  - testing
  - side-project
excerpt: "How Super Morse is put together: the frame pipeline, the Engine 2 runtime model end to end, how behavior moves from Java to Engine 2 one actor family and one scene slot at a time, a worked example on scene 30, and the in-app rule inspector and debug panel."
---

Post #43 in the [Super Morse](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/)
series. The [last post](/posts/2026/10/engine2-rule-engine-introduction-guidebee-game-engine/)
introduced Engine 2, the rule language and VM. This one is about the *game* that runs it: how a
frame is put together, how Engine 2 runs inside it, how behavior moves from Java to Engine 2
without a big-bang rewrite, and the debug tooling we built so we can see what the rules are doing
on a phone. The proposed game economy has its own post:
[A Proposed Game Economy and Its Reference Implementation](/posts/2026/10/super-morse-game-economy-design/).

## TL;DR

- **One frame, one contract.** Every tick runs a fixed, documented list of phases, and a test asserts
  that order whichever implementation is in charge.
- **Engine 2 runs inside the frame, not beside it.** One game thread, synchronous visits at fixed
  points, waits implemented as saved continuations, and hard per-visit budgets.
- **Behavior moves to Engine 2 one actor family and one scene slot at a time.** An authority table
  decides, at the start of each attempt, whether Java or Engine 2 runs each one. Anything that fails
  its checks stays on Java, and a single switch rolls everything back.
- **Scene 30 is the worked example:** four program slots, two of them guide prompts, one of them run
  by Engine 2 while the other three stay on Java in the same attempt.
- **We can see the rules running on a phone.** A debug panel and an in-app **rule inspector** pause
  the game, step one tick, show which rules matched and why, and export the trace as JSONL.
- **Progress:** 48 product tasks: 21 done, 19 partial, 8 open (about 64% weighted). The rules
  foundation is 100% done; the large actor and scene waves are next.

![A mounted hero in the campaign, with the debug button in the HUD](../images/supermorse_mounted_hero.png)

## A few terms first

Engine 2 reuses a few everyday words with precise meanings. This post uses them consistently:

| Term | Meaning in this post |
|---|---|
| **Attempt** | One run of one level. Retry throws it away and starts a new one. |
| **Tick** | One step of the game simulation (60 per second). |
| **Authority** | *Which implementation* runs an actor family or a scene slot: the **Java authority** (the existing hand-written code, also called *legacy*) or the **Engine 2 authority** (a compiled rule module on the VM). |
| **Scene owner** | The *placed actor in a level* that drives a scene program. Not the same thing as authority. |
| **Program slot** | The number of one program inside a module. A scene module has many; an actor module has exactly one, at the reserved slot 65535. |
| **Variable slot** | A numbered integer variable the game owns (for example system variable 331). Rules read and write it through the host. |
| **Profile** | The contract that says what each rule operation means for this game. |
| **Host** | The game-side code that carries out operations for the VM. |

## 1. The shape of the game

Super Morse is a 2D side-scrolling platformer for Android, written in Java on the Guidebee Game
Engine. From a distance it has four layers, and the arrows show who calls whom:

```text
┌──────────────────────────────────────────────────────────────────────────┐
│ Product    menus, hero roster, economy screens                           │
└───────────────┬──────────────────────────────────────────────────────────┘
                │ starts and ends attempts, reads results
┌───────────────▼──────────────────────────────────────────────────────────┐
│ Session    GameScreen → StageAttempt  (one attempt = one run of a level) │
└───────────────┬──────────────────────────────────────────────────────────┘
                │ runs the frame pipeline once per tick
┌───────────────▼──────────────────────────────────────────────────────────┐
│ World and  MorseWorld · hero body (input, physics, firing) · contacts ·  │
│ mechanics  camera · ActorStateStore · system variables        (Java)     │
└───────────────┬───────────────────────────────────────▲──────────────────┘
                │ visits each actor / scene slot        │ calls services,
                │                                       │ reads and writes state
┌───────────────▼───────────────────────────────────────┴──────────────────┐
│ Behavior   per actor family and per scene slot, exactly one of:          │
│            Java authority (class / scene interpreter)                    │
│            Engine 2 authority (VM + .e2m module, through the hosts)      │
└──────────────────────────────────────────────────────────────────────────┘
```

Two things to notice. First, behavior sits *under* the world: the world loop visits it, and it calls
back up into world services. Behavior never owns the loop. Second, the **hero** lives in the world
and mechanics layer and is all Java today. Its decisions (locomotion, weapons, forms, mounts, death
and revival) are planned to move into rules later.

The key idea is the **attempt**. A `StageAttempt` owns the world, the input, the scene programs and
the reward staging. Retry throws it away and builds a new one, so nothing from a failed run leaks
into the next. That boundary turns out to matter for almost everything below.

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

Engine 2 doesn't get its own phase in this list. An Engine 2 actor is visited **at exactly the
position its Java class would have ticked** (2a or 2b), and an Engine 2 scene slot runs inside
phase 3 where the Java scene interpreter would have. Completion events (a dialogue closed, a
transition finished) are seen at the next tick's phase 3.

To keep this from drifting, a debug hook `TickPhases` (one null check when off) records phase
entries at the ten boundaries, and `TickPhaseOrderTest` asserts the order for scene 30 over 120
ticks. It also checks that scene 65 records **the same phases tick for tick** for 200 ticks under
the Java authority and under the Engine 2 pilots.

## 3. The Engine 2 runtime model

### 3.1 End to end, in one picture

Before the details, here is the whole path from a rule someone writes to a fish moving on screen:

```text
AUTHOR AND BUILD (repo, CI)        LOAD (each attempt start)            RUN (every tick)

 operation table (data)                                               world loop / phase 3
        │                                                                     │
        ▼                                                                     ▼
 product profile ──┐                                                   VM machine
 (digest pinned)   │                                                 (actor or scene)
                   ▼                                                          │
 .e2 source ──► compiler ──► .e2m ──► load ──► preflight ──► authority ───────┘
                             + manifest       against the      table:          │ operation id
                             (SHA-256)        capability       Java or         ▼
                                              registry         Engine 2     host handlers
                                                                               │
                                                                               ▼
                                                                  game services and state
                                                                  (movement, contacts, camera,
                                                                   dialogue, ActorStateStore,
                                                                   system variables)
```

Everything left of the authority table happens once. Everything right of it happens every tick,
and only for the actors and slots that Engine 2 actually runs.

### 3.2 Engine 2's four layers in this game

The previous post described Engine 2 as four layers with dependencies pointing down only. Here is
where each one lives in Super Morse:

| Layer | What it is | Where it lives in Super Morse |
|---|---|---|
| **Core** | language, compiler, `.e2m` format, VM machines, budgets, traces; knows no game | `gameengine` module, packages `com.guidebee.game.engine2.*`, including the game-free `ActorStateStore` |
| **Profile** | what each operation means for this game: ids, signatures, capabilities, policies | `supermorsego.product`, generated from a machine-readable **operation table** (92 reachable operations); its digest is pinned in tests |
| **Host** | carries out operations: one small handler per operation, dispatched by id | app package `…morse.engine2` (`Engine2ActorHost`, `Engine2SceneHost`, `Engine2Authority`, `Engine2Capabilities`) |
| **Platform adapter** | the actual game and engine services the handlers call | `MorseWorld`, `StageAttempt`, the scene services (camera, dialogue, transitions, system variables) |

The portability story follows from this split. The **specification and the `.e2m` format**, not the
Java code, are the contract. Modules and the profile can move to another engine (Unity is the
candidate) unchanged. What has to be rebuilt there is the VM (checked against byte-exact test
vectors) and the host and adapters for that engine.

### 3.3 Where state lives, and who may write it

The VM **never touches storage**. That's the rule that lets Java and Engine 2 share one world
safely:

```text
            Engine 2 VM                                  Java code
                │                                            │
   read:  readSlot(domain, index) ──┐          ┌── reads and writes through
   write: profile action, e.g.      │          │   the same view
          setSystemSlot(0, 1)       │          │
                │                   ▼          ▼
                └──────────► host ──► ActorStateView ◄── every write names its StateWriter
                                          │                (rules, physics, contacts, …)
                                          ▼
                              ActorStateStore (one per attempt)
                              locals · flag words · base info · runtime slots ·
                              properties · extras · runtime-clone id range
                              + scene system variables in the scene services
```

- **Reads** happen through `readSlot`. The VM reads every slot argument itself, in order, before it
  dispatches a call.
- **Writes** happen only through profile actions. A rule can't compute an address or write
  arbitrary memory.
- **One writer per field.** The store's write guard (strict in all unit tests) checks that each
  field is written only by the writer class allowed to write it. If rules and physics both try to
  own a field, the guard catches it.
- **Snapshots.** The store is part of the attempt checkpoint, so restore and retry don't need
  special cases.

### 3.4 One thread: Engine 2 is not a background task runner

A slot can `wait ticks(80)` in the middle of a cutscene while the game keeps running, so it's
natural to picture Engine 2 as a background thread. It isn't. There is **one game thread and one
frame pipeline**, and Engine 2 runs *inside* it, synchronously, at fixed points:

```text
game thread, one tick:
  phase 1   hero body, mechanisms                      Java
  phase 2a  instance loop ── actor visit ──► Engine 2 actor machine (returns before the next actor)
  phase 2e  contacts                                   Java
  phase 3   scene programs ── each slot ──► Engine 2 scene machine (resume or scan, then return)
  phase 4   camera                                     Java
  ── render ──
```

What makes waits possible is the **continuation**, not a thread. When a scene rule hits
`wait ticks(n)` or a long action like a fade, the machine saves where it stopped (slot, rule, next
action, due tick) and *returns*. On a later tick, phase 3 sees the continuation is due and resumes
from that point. It's the same idea as coroutines or `async`/`await` on a single-threaded event
loop: the code reads as if it blocks, but it only ever runs in short steps inside the tick.

Things that genuinely happen on other threads, such as a dialogue closed by a UI tap or audio
finishing, **never touch rule state directly**. They only post an event into a lock-free inbox
carrying the attempt number and a continuation token. The next tick drains that inbox in phase 3,
and a stale token (from a previous attempt, or for a scene owner that's gone) is rejected.

This is deliberate:

- **Determinism.** The same inputs produce the same ticks, which is what makes 5,000- and
  10,000-tick Java-versus-Engine 2 comparisons possible at all.
- **No locks.** Rules and mechanics never run at the same time, so they share the state store and
  system variables without synchronization.
- **Bounded cost.** Each visit has a budget for rule visits and action dispatches.

### 3.5 What about frame rate?

If everything runs on the game thread, what happens when the rules get expensive? Does Engine 2
skip rules, or does the game lag?

It **never skips a rule** to save time. Every rule that's due runs completely, every tick, so an
overloaded tick would show up as a late frame (lag), not as missing behavior. The design keeps
that from happening in four ways.

**1. A runaway rule faults instead of hanging the frame.** Each visit to an actor or slot is
capped at **4,096 rule visits and 16,384 action dispatches**. Going over raises `E2_BUDGET`, which
stops that one program and records where. Running out of budget is *a fault, never a quiet
"condition false"*. The cap exists to catch bugs such as an infinite jump loop. It isn't a
performance throttle.

**2. Scene work has a known worst case.** Scene programs can't jump backwards, and at most one rule
match starts per slot per tick, so the maximum work per program is fixed at compile time. Across
all **701 scene modules**, the worst needs **17 rule visits and 163 calls**, far below 25% of the
caps. Long sequences don't pile up in one tick either: `wait ticks(n)` spreads a cutscene
across many ticks.

**3. The real cost is how many actors there are, not how complex the rules are.** One level can
hold thousands of blocks, about 13,700 block instances across all levels. The JVM benchmark
(medians over five separate processes, with a no-op host) made that clear:

| Path | Cost per 1,000 blocks per tick |
|---|---|
| Java `BlockDispenser.tick()` | 4.15 µs |
| Engine 2, visiting every block | ~866 µs (projected **11.8 ms** for every block) |
| Engine 2, skipping dormant blocks | **1.81 µs** |

At 60 Hz a frame has 16.7 ms, so visiting every block would eat most of it. The answer is to
**skip actors that can't act**, not rules. `Engine2ActorVisitScheduler` keeps a stable-order
roster: a block with no pending event and nothing to do isn't visited at all, and it wakes when
something happens to it, such as the hero bumping it. The skip is deterministic, and it mirrors the
frozen active roster the Java version already uses. It's recorded as a proposed decision.

**4. Loading cost is small too.** Loading and checking the largest scene's modules (16.8 KB of
binary) takes a median of **7.4 ms** on the JVM. That happens once per level load, not per tick.

**Still open:** device numbers. The target is Engine 2 costing no more than 10% above Java on the
heaviest scene, with no extra dropped frames. Frame time and memory still need to be measured
on the test phones. A CI guard already fails the build if module count or total module bytes grow
past their recorded limits.

## 4. Migrating to Engine 2

Rewriting every actor at once would have stalled the game for months. Instead, behavior moves to
Engine 2 **per actor family and per scene slot**, behind one selection path.

### 4.1 Java authority versus Engine 2 authority

Every actor instance and every scene program slot has exactly one **authority** for the whole
attempt: the one implementation that decides what it does each tick.

**The Java authority** (also called **legacy**) is the behavior the game shipped with:

- for **actors**, a hand-written Java class per family (`Actor61Effect`, `FishSpawner`, the walker,
  plant and block classes) whose `tick()` method runs inside `MorseWorld`'s instance loop;
- for **scenes**, the Java scene-program interpreter in `SceneProgramHost`, which steps each slot's
  program in phase 3 of the frame.

"Legacy" doesn't mean broken. It means *this is the version we are migrating away from*. It's still
the default for everything Engine 2 has not taken over, and it's the reference every Engine 2 port
is compared against.

**The Engine 2 authority** is a compiled rule module (`.e2m`) running on the Engine 2 VM, reaching
the game only through the hosts.

```text
                    MorseWorld instance loop (phase 2a/2b) / scene programs (phase 3)
                                         │
                       which authority runs this actor / slot?
                       ┌─────────────────┴─────────────────┐
                       ▼                                   ▼
               Java authority (legacy)              Engine 2 authority
          FishSpawner.tick(), Actor61Effect,    VM visits the .e2m module
          SceneProgramHost.tickProgram()        ──► Engine2ActorHost / SceneHost
                       │                              operation handlers
                       │                                   │
                       └──────────────┬────────────────────┘
                                      ▼
            shared game mechanics: movement, collision, contacts, camera,
            dialogue, transitions, system variables, actor state store
```

This is the split Engine 2 was designed around: **rules sequence, code does mechanics**. Switching
authority changes *who decides* (when the fish launches, which rule kills it). It doesn't change
*how* the fish moves or collides, because both authorities call the same mechanics in the same
frame phase.

Three rules keep the two authorities from stepping on each other:

1. **One authority per actor or slot, fixed for the attempt.** Selection happens at attempt start;
   retry selects again. The `OwnerLedger` makes every tick site claim the actor for Java or Engine 2,
   and both in one tick is a fault, not a silent double update.
2. **Java is a load-time fallback only.** If a module fails to load or preflight, Java keeps the
   actor or slot and the `SupportReport` says why. A VM fault *during* a run does not quietly switch
   back, so bugs surface instead of hiding.
3. **Same inputs, same outputs.** Where a family has both implementations, the Engine 2 host is handed
   the same random stream the Java class would have used, so the two can be compared tick for tick.

The end state is **cutover**: once a family's Engine 2 module matches its Java version and passes the
device gates, Engine 2 becomes its default authority. Once every family is across, the legacy Java
behavior classes are retired, and only the mechanics they called remain.

### 4.2 Hosts and capabilities

- **Hosts.** `Engine2ActorHost` and `Engine2SceneHost` dispatch by operation id through a
  registry of small handler objects rather than one giant `switch`. Unknown operations fault with
  `E2_HOST_FAULT`. Scene handlers call the *same* service methods the Java scene interpreter
  already used, so behavior is shared, not copied.
- **Capability registry.** `Engine2Capabilities` is *computed from the hosts*: a capability exists
  only when **every** operation in it has a working handler. Having a row in the profile doesn't
  count. As handlers land, capabilities appear, and modules that need them start passing preflight.
  Nobody edits the registry by hand.

### 4.3 The authority table: how it's designed

"Authority table" is a convenient name, but it isn't one literal table. `Engine2Authority` combines
four inputs and makes one decision per candidate, once, at the start of each attempt:

```text
 Registrations (code, shipped)          Config (this run)
 what CAN run on Engine 2               what MAY run on Engine 2
 ─ actor-190, actor-061, actor-199      ─ actors = {…} or "all registered"
 ─ scene-030-slot-2, scene-030-slot-3   ─ scenes = {"30:2", …} or "all registered"
 ─ wave-1 bundles, scenes 30 to 45      ─ wave1  = {30, 31, …} (opt-in)
                                        ─ rollback = true | false
            │                                     │
            └──────────────┬──────────────────────┘
                           ▼
            Engine2Authority.select(context)  ◄── Capability registry (from the hosts)
                           │                  ◄── Resource catalog (what the game ships)
                           ▼
              SupportReport: one line per candidate, with a status and a reason
```

#### Registrations: what *can* run on Engine 2

A **registration** describes one selectable module. It's code, shipped with the app
(`PilotRegistrations`), and each one carries:

| Field | Meaning | Example |
|---|---|---|
| `name` | stable id, used in reports and dependencies | `actor-061`, `scene-030-slot-2` |
| `family`, `resource`, `slot` | what it replaces; actors always use slot 65535 | scene 30, slot 2 |
| `profile` | the contract the module must match | product profile, or a pilot profile |
| `loader` | where the `.e2m` comes from | packaged app assets (tests load files) |
| `installer` | the code that actually switches authority | `world.enableActor61Engine2(loaded)` |
| `applies` | does this candidate apply to *this* attempt at all? | "scene is 65", "this level has fish spawners" |
| `capabilities` *(optional)* | a module's own narrower capability set | the pilot profiles |
| `dependsOn` *(optional)* | only select if another registration was selected in the same pass | slot 3 depends on slot 2 |

There's also a **bundle** form: all the slots of one scene, selected **all or nothing**.

What's registered today:

| Registration | Applies when | Profile |
|---|---|---|
| `actor-190` (defeat puff) | every attempt | product |
| `actor-061` (contact effect) | scene 65 | product |
| `actor-199` (flying fish) | the level has fish spawners | product |
| `scene-030-slot-2` | scene 30, guide service available | scene 30 pilot |
| `scene-030-slot-3` | same, **and** slot 2 was selected | scene 30 pilot |
| wave-1 bundles, scenes 30 to 45 | that scene, only when opted in | product |

A wave-1 bundle for scene 30 covers all four slots and replaces the two single-slot guide
registrations. The other wave-1 scenes cover slots 0 and 1.

#### Config: what *may* run this time

The **config** narrows the registrations for a run. It has four parts:

```text
actors=61,190        only these actor families (omit = every registered actor)
scenes=30:2,30:3     only these (scene:slot) pairs (omit = every registered slot)
wave1=30,31          opt these whole scenes into bundles (scenes 30 to 45 only)
rollback=true        keep everything on Java
```

In a normal run the config is the default: **every registered module is allowed**, and wave-1
bundles are off. In debug builds a developer can pass a config string as an intent extra to try a
combination on a phone without rebuilding the APK. A malformed string is logged and ignored, and
the defaults apply. Release builds don't read the extra at all.

#### `select`: one decision per candidate, in order

For each registration, `select` walks the same checks in a fixed order and stops at the first one
that fails. Every outcome is written to the `SupportReport`:

| # | Check | If it fails, status is |
|--:|---|---|
| 1 | `applies` to this attempt? | *(skipped, no entry)* |
| 2 | rollback off? | `KEPT_LEGACY_ROLLBACK` |
| 3 | permitted by the config? | `KEPT_LEGACY_NOT_CONFIGURED` |
| 4 | `dependsOn` selected earlier in this pass? | `KEPT_LEGACY_NOT_CONFIGURED` (names the dependency) |
| 5 | still at tick 0 of the attempt? | `KEPT_LEGACY_LATE` ("a retry selects at its start") |
| 6 | module loads? | `KEPT_LEGACY_LOAD_FAILED` |
| 7 | preflight: profile matches, every capability present, every resource in the catalog | `KEPT_LEGACY_PREFLIGHT_FAILED`, listing what's missing |
| 8 | installer succeeds? | `KEPT_LEGACY_INSTALL_FAILED` |
| ✓ | all passed | `SELECTED`, with the module's SHA-256 and the profile digest |

One more status can appear later: `FAULTED_NO_FALLBACK`, when a selected module faults during the
attempt. As the name says, Java does not take over mid-attempt.

Four design choices sit behind this:

- **Fail closed to Java.** Every failure path keeps the Java authority, and `select` never throws
  for a module problem. The worst a bad module can do at load time is *not run*.
- **Nothing changes until the last step.** Loading and preflight have no side effects. Only the
  installer switches authority, for example `world.enableEffect190Engine2(loaded)`, or, for a scene
  slot, `programHost.replaceProgramAtAttemptStart(2, …)`, which swaps slot 2's program before the
  first tick.
- **Bundles are atomic.** For a wave-1 scene, every slot is loaded and preflighted first. If any one
  fails, **every** slot of that scene is reported as kept on Java, with the failing slot named. This
  matters because a scene's slots coordinate through shared system variables and one shared action
  list, so half a scene on each side is a combination nobody has tested.
- **Order is part of the design.** Registrations are evaluated in registration order, which is how
  `dependsOn` works: slot 3 can only be selected if slot 2 already was, in the same pass.

#### How it's used

```text
GameScreen builds an attempt (first play, retry, or a pipe into a sub-stage)
  │
  ├─ Context(sceneId, ticksRun = 0, attempt) + services the installers need
  │     (scene 30 adds the guide service)
  ├─ report = engine2Authority.select(context)
  ├─ debug builds: Log.i("ENGINE2", report.dump())    → adb logcat -s ENGINE2
  └─ the attempt starts ticking; the choices are now fixed

every tick:  OwnerLedger: each tick site claims each actor for LEGACY or ENGINE2;
             both in one tick throws, and the ledger is cheap enough to stay on in production
on retry:    a new attempt and a new select, so a changed config or a fixed module takes effect
```

Tests use the same class rather than a test-only shortcut. For example, the wave-1 test helper
builds an authority with `wave1=<scene>` to run a whole scene on Engine 2, so production and
tests go through the **same** selection path.

#### Reading a report

On the Pixel, with the config `scenes=30:2`, the scene 30 lines of the report read:

```text
scene-030-slot-2 SELECTED module=6367ee51ea89
scene-030-slot-3 KEPT_LEGACY_NOT_CONFIGURED (not in the configured set)
```

Slot 2 passed every check, and its module hash is recorded. Slot 3 stopped at check 3, because
the config named only `30:2`. Had the config allowed slot 3 but slot 2 failed, slot 3 would stop at
check 4 instead, because it depends on slot 2. Candidates whose `applies` check is false, such as
`actor-061` outside scene 65, don't appear at all. With `wave1=30`, the same attempt would instead list all four
scene 30 slots together: all `SELECTED` from one bundle, or all kept on Java with the reason.

**From pilots to cutover.** Today the registrations are a short, hand-curated list, and the larger
wave-1 bundles are opt-in. Cutover is the same mechanism scaled up: each family that passes its
parity and device gates gets a registration that is selected by default, until the Java authority
has nothing left to own.

### 4.4 The attempt lifecycle

Selection, slot state, waits, faults and retry all hang off the attempt. In one timeline:

```text
ATTEMPT START
  ├─ select authority per actor family and per (scene, slot)  → SupportReport
  ├─ beginAttempt on each Engine 2 scene machine: reset rule status, repeat counters,
  │  cursors and continuations; clear any fault
  └─ bind actor state into the attempt's ActorStateStore

EVERY TICK (frame pipeline)
  ├─ phase 2a/2b: actor visits (Engine 2 or Java, never both)
  ├─ phase 3: drain the event inbox, then per scene slot: resume a due continuation, or scan
  └─ a fault latches on that one program; everything else keeps running

ATTEMPT END: victory, death, retry or quit
  └─ the attempt is discarded: continuation tokens become stale, staged rewards are applied or dropped

RETRY = a brand-new attempt: select again, reset again, stale events from the old one rejected
```

### 4.5 When Java and Engine 2 meet: a hero and a fish

Mixed authority is the normal state during migration, so the obvious question is what happens when
an Engine 2 actor and a Java actor interact. The flying fish is a good example. Its clones run on
Engine 2; the hero is all Java.

```text
a tick
  phase 1   hero body (Java): input and physics move the hero into the fish
  phase 2b  fish clone visit (Engine 2): its rules ask the host world questions
            (hero position, collision, its own state) and request movement, and
            MorseWorld carries the movement out; its contact is appended like any
            other actor's
  phase 2e  contacts (Java): every contact, including the hero's, is resolved once,
            in one pass, by the same Java code whichever authority runs the fish
a later fish visit
  phase 2b  fish (Engine 2): its defeat rule's collision and state predicates
            are true → defeat actions run through the host
```

Neither side calls the other. They meet only through the shared world, the state store and the
contact pass, at fixed phases. That's also why the rule inspector (section 6) can explain the
defeat: the exported trace shows rule 16's collision and state predicates evaluating true and its
defeat actions firing.

### 4.6 Failure modes, and what the player sees

| Failure | Where it's caught | What happens | Player sees |
|---|---|---|---|
| Module missing or won't load | attempt start | Java keeps it; `SupportReport` entry | nothing different |
| Missing capability (an operation without a handler) | preflight | Java keeps it; the report names the capability | nothing different |
| Both authorities tick one actor | `OwnerLedger`, every tick | fault | a bug, caught by tests |
| Runtime fault (host error, bad control flow, budget) | VM | that one program stops and stays stopped until the next attempt; earlier writes remain, no rollback | **open question** for release builds |
| Stale event (old attempt, scene owner gone) | inbox drain | rejected with `E2_STALE_CONTINUATION` | nothing |

The open row matters. In development and CI, stopping the faulted program is exactly right: it
fails loudly, with the source location in the trace. In a release build, a stopped program could
look like an enemy that stops making decisions or a cutscene that never finishes. The spec
deliberately leaves room for a **containment policy** declared by the profile (for example, abort
the visit and continue the frame), but none is defined yet. Choosing one is part of the cutover
work.

### 4.7 Source of truth and the build pipeline

Every Engine 2 module can be rebuilt from what is in the repository:

```text
operation table (92 operations, data) ──► product profile (digest pinned)
                                                │
existing level rule data ──► importer ──────────┤ maps every term through the table
                                                ▼
                                  corpus builder ──► .e2m modules + manifest.json
                                                     (name, bytes, SHA-256, rules, calls,
                                                      capabilities, source mode)
                                                │
                                                ▼
                                  port ledger: per program, imports / refused,
                                  preflight, parity, current authority
                                                │
                     promote ──► readable .e2 source in the repo  (source mode: imported → authored)
```

- **793 programs import** today: 92 actor families and all 701 scene slots. The ledger ranks
  whatever still blocks the rest.
- Output is **canonical**: sorted keys, no clock, no absolute paths. The same input gives the same
  bytes, and a CI check fails if the ledger is stale.
- Once a program is **promoted**, its `.e2` source becomes the master copy that designers edit. The
  build compiles it and refuses to run if the data it was imported from has changed underneath it.
  All 810 generated sources recompile with zero byte mismatches.

One detail for careful readers: the scene 30 example below is written against a small hand-made
**pilot profile** (`supermorsego.scene030`, language 1.1), which is easier to read. Everything
generated at scale, including the slot 1 rules quoted later, uses the full **product profile**
(`supermorsego.product`, language 1.2).

### 4.8 Where it stands

Engine 2 runs real content today, not just demos:

- All **34 scene-program slots of scenes 30 to 45** import (116 rules, 744 calls), package into
  byte-reproducible binaries and pass host preflight. A production driver runs each of the 16
  scene bundles for 5,000 idle ticks.
- **48 five-thousand-tick comparisons** between Java and Engine 2 (idle, run right, synthetic
  run-and-jump) produced 46 exact matches. The two that still differ are being worked on.
- **All 42 wave-A actor modules** (effects, controllers, story, finish) are packaged with readable
  sources: 199 rules and 863 calls, every one of their 37 operations with a handler.
- The flying fish, coin magnet and mushroom families run on Engine 2 in the live world, and the
  fish matches its Java implementation for **10,000 ticks**.

## 5. Worked example: scene 30

Scene 30 is the first campaign level (LEVEL 1). It shows the model well because **one scene runs
under two authorities at the same time**, and the unit that makes that possible is the
**program slot**.

### 5.1 What a program slot is

A scene's behavior isn't one big script. It's a set of **independent programs**, each tagged
with a number: its *program slot*. In Engine 2 source it's the `at N` in `program name at N { ... }`.
A scene module can hold up to 65,535 of them. An actor module, by contrast, always has exactly one
program, at the reserved slot 65535.

Think of a slot as a **channel number**, not a memory address. Each slot is its own small state
machine, and the runtime keeps this state **per slot**:

| Per-slot state | What it holds | Lifetime |
|---|---|---|
| Rule status | for each rule: not matched, matched, or skipped by an `else` | reset at every attempt |
| Repeat counters | how many more times each rule may fire (`repeat once` = 1, `forever` = -1) | reset at every attempt |
| Cursor | where the next scan of the rule list starts | the attempt |
| Continuation | a paused action list: which rule, which action next, and the tick it's due (set by `wait ticks(n)` or a long action like a fade or dialogue) | until it finishes, or the attempt ends |
| Fault | a latched error; a faulted slot stops until the next attempt | the attempt |

**How one slot runs in a tick** (inside phase 3 of the frame):

```text
tick(slot):
    if a continuation is pending:            // a wait or long action is in progress
        resume it if it's due; then stop     // never start a second scan this tick
    scan rules from the cursor:
        skip if an else-branch is not eligible
        skip if the repeat counter is 0      // "repeat once" already used
        check the activation predicate
        check the conditions                 // the `when` part
        first rule that passes: decrement repeat, start its actions, stop scanning
    at most ONE rule match starts per slot per tick
```

**Slots don't talk to each other directly.** Two slots never share state through their slot
numbers. They share it through **variable slots**, the numbered system integers the game owns. In
scene 30:

```text
scene 30 module
 ├─ program slot 0  (director)          reads/writes  system[0] phase, system[6], system[15], system[331], system[460]
 ├─ program slot 1  (finish)            reads/writes  system[15] ...
 ├─ program slot 2  (tutorial line 1)   reads         hero X, system[331]
 └─ program slot 3  (tutorial line 2)   reads         hero X, system[331]
        ▲ program slots: which program runs      ▲ variable slots: the shared integers they use
```

The director in slot 0 advances `system[0]` (the phase) as the story plays out. Other slots can
react to that value without knowing slot 0 exists.

**Two separate choices decide how a slot runs.** The level's placement data decides which **scene
owner** (a placed actor) drives which program slot; the slot is scanned when that actor is eligible.
Separately, the **authority table** decides whether Java or Engine 2 runs it, keyed by
`(scene, slot)`, for example `30:2`.

**Why slots are the right unit of migration.**

- **Small.** Many slots are a single rule (the tutorial prompts are). A slot can be ported,
  compared and switched without touching the rest of the scene.
- **Isolated.** Its status, repeats, cursor and continuation belong to it, so one slot can run on
  Engine 2 while its neighbors stay on Java in the same attempt.
- **Contained failure.** A fault latches on that slot only, until the next attempt.
- **Clear seams.** Because slots only meet through system variables and shared services, both
  authorities read and write the same integers through the same host.

There's one place where slots *do* interact, and it's the hardest part to port. The game keeps
**one current scene action list** shared across scene owners, and a later matching scene owner can
**replace** a blocked one, losing the replaced one's continuation. Engine 2 reproduces this as an
explicit scene-manager policy (with a `replace` trace event) rather than hiding it inside each
slot.

### 5.2 Scene 30's four slots

| Slot | Role | What it does | Size |
|--:|---|---|---|
| 0 | director | the opening story cutscene: scripted animations, actors moved off-screen in batches, a fade out and back in, and the story phase in `system[0]` | 273 lines of `.e2` |
| 1 | finish | end-of-level handling | 54 lines |
| 2 | tutorial line 1 | guide prompt at the first marker | 1 rule |
| 3 | tutorial line 2 | guide prompt at the second marker | 1 rule |

![Scene 30's intro on a Pixel, with slot 2 run by Engine 2](../images/supermorse_scene30_engine2_intro.png)

### 5.3 Where's the gameplay?

None of the four slots *is* the gameplay. Scene slots are the level's **director**: they sequence
story and presentation around the play. Every operation in scene 30's slots is a scene
operation. Slot 0 uses animations, screen transitions, actor movement batches, camera switches,
audio, text and system variables. Slot 1 uses a dialogue, a movement batch and some flags. Slots 2
and 3 each show one guide prompt. Nothing in them moves the hero or resolves a collision.

The play itself comes from two other places:

```text
                         one tick of scene 30
                                  │
  ┌───────────────────────────────┼────────────────────────────────┐
  ▼                               ▼                                ▼
MECHANICS (Java, always)    ACTOR PROGRAMS (per actor)       SCENE PROGRAM SLOTS
hero body: input, run,      one program per actor family,    slot 0 director (intro)
jump, physics, firing       always at slot 65535:            slot 1 finish
world loop, contacts,       walker, turtle, mushroom,        slot 2 tutorial line 1
damage, portals, camera     coin, fish, block...             slot 3 tutorial line 2
                            authority: Java or Engine 2,     authority: Java or Engine 2,
                            per actor family                 per (scene, slot)
  ◄────────────── the gameplay ──────────────►               ◄── story & presentation ──►
```

1. **Mechanics** (frame phases 1, 2 and 4): the hero's body, the world update, contacts, damage,
   portals and the camera. These stay as Java code permanently. That's the *rules sequence, code
   does mechanics* split.
2. **Actor programs** (phases 2a and 2b): each actor family has a module with a single program at
   the reserved slot 65535, visited once per tick for every placed actor. These decide what each
   enemy and pickup does. The flying fish, coin magnet and mushroom families already run on Engine 2.

That's why switching slot 2 to Engine 2 doesn't change how scene 30 *plays*. It only changes who
decides when the tutorial prompt appears.

### 5.4 How gameplay triggers the slots

Slots never call each other, and there's no "go to slot 2". Every active slot is scanned each tick,
and a rule fires when its conditions become true. Those conditions read live gameplay state, so
**gameplay is what drives the scene logic**. It reaches the slots in three ways:

1. **Conditions that read gameplay directly.** Slots 2 and 3 trigger only on the hero's position
   (`actorPositionCompare` against a marker). The hero gets there through input, physics and
   collision, not a script. When the player walks far enough, the condition flips and the prompt
   fires.
2. **Activation zones.** The profile provides activation predicates such as
   `referencePointInsideOwner` (a designated point, typically the hero, is inside the scene owner's
   bounds) and `referencePointInsideOwnerWithInput` (the same, while a particular input is held).
   Walking into an invisible trigger box, or pressing a button inside it, starts the slot's rules.
3. **Shared system variables.** Gameplay code and actor rules write numbered system variables, and
   slots watch them. Scene 30's slot 1 is a small chain built entirely on them:

```text
rule0  when system[15] == 1  → movement batch (moves the hero, actor 1), wait 25 ticks,
                               set hero animation, advance system[15]
rule1  when system[15] == 3  → clear a flag bit on the hero, set system[4] = 1
rule2  when system[4]  == 1  → show dialogue 0
```

It also works the other way. A slot can **take over** gameplay: slot 1 moves the hero with a
movement batch and changes a flag bit on it. While a movement batch, camera pan or transition is
running, that effect owns the world update and the slot's action list waits until it finishes,
which is how a cutscene takes over normal play. A slot can also switch to another scene. So the
loop is: gameplay changes state, a slot's condition becomes true, the slot's actions may take
control for a while, and then control returns to the player.

### 5.5 Slot 2, rule by rule

This is the whole program for slot 2, in readable Engine 2 source:

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

In words: *once (`repeat once`), when the hero (actor 1) reaches the x position of marker actor 36,
and shared system variable 331 is still 0, show the first guide step.* Slot 3 is the same rule
against marker 37.

Traced through the slot model: each tick, slot 2 scans its single rule. While the hero is left of
the marker, the condition is false and nothing happens. On the first tick the hero reaches it, the
rule matches, its repeat counter drops from 1 to 0, and `guideStep` starts. From then on the repeat
gate skips the rule without evaluating anything. A retry starts a new attempt, which resets the
counter, so the prompt can show again.

**The same slot under each authority:**

```text
                         scene 30, slot 2, phase 3 of every tick
                                        │
              ┌─────────────────────────┴──────────────────────────┐
     Java authority (legacy)                              Engine 2 authority
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

Neither authority draws the guide bubble or tracks the hero's position. Both ask the same services.
The only thing that changes is *who evaluates the condition and decides to fire*.

### 5.6 Selection and verification

With Engine 2 enabled for slot 2 only, the `SupportReport` for a scene-30 attempt on the phone reads:

```text
scene-030-slot-2 SELECTED module=6367ee51ea89
scene-030-slot-3 KEPT_LEGACY_NOT_CONFIGURED (not in the configured set)
```

So slot 2 is driven by the compiled module (identified by its hash), and slot 3, the identical rule
one marker later, stays on Java in the same attempt. Slots 0 and 1 also stay on Java. If the module
had failed to load or preflight, slot 2 would have reported `KEPT_LEGACY` with the reason, and the
level would have played exactly as before.

- **Frame order:** `TickPhaseOrderTest` runs scene 30 for 120 ticks and asserts all ten phases
  run in contract order whichever authority holds the slots.
- **On the phone:** the scene-30 intro was captured on a Pixel twice, once with slot 2 on Engine 2
  and once with Engine 2 selection omitted, and showed no visible difference. That's the
  screenshot above.
- **Long runs:** scene 30 is part of the 5,000-tick Java-versus-Engine 2 comparisons. It's also
  where one of the two remaining differences lives: movement diverges at the moment one scene
  owner replaces another (the shared action list case in 5.1), and the opening transition is not
  yet an exact match. Until that's fixed, the director slot stays on Java, which is exactly what
  per-slot authority is for.

## 6. Debug panel and rule inspector

Parity tests on the JVM tell you *that* something diverged. On a phone you need to see *why*. The
debug build has two layers of tooling.

### The debug panel

The purple **DEBUG** button in the HUD opens a side menu: scene picker, previous/next scene, hop
left/right through a level, pause, a materials overlay, and a live status strip with role, HP,
attempt deaths, coins, keys, blocks exhausted, hero physics state and session state.

![Debug panel over scene 45: status strip, hop controls and scene navigation](../images/supermorse_debug_overlay.png)

Every preview run is labelled `PREVIEW, NOT CERTIFIED`, and debug runs (god mode, scene tests,
previews) **never pay rewards** into the save. This was a deliberate rule: a debug tool that can
mint coins will eventually corrupt somebody's save.

### The rule inspector

Tap an actor marker and the inspector opens on it. It shows:

- the actor id, resource, position, state and authority (the overlay labels it `OWNER`: Java,
  Engine 2 placement, or Engine 2 runtime clone),
- the module hash and the current tick,
- every rule in order, **matched or not**, with each condition's evaluated arguments and result,
- each action with its resolved operands, and
- the actor's locals, flags, base info and extras **after** the tick.

![The rule inspector paused on a live fish clone: rule 27 matched, rules 0 to 26 did not](../images/supermorse_rule_inspector.png)

The controls are **RESUME / OLDER / NEWER / STEP 1 TICK / OWNER < / OWNER >**, plus **EXPORT**.
Under the hood:

```text
RuleTraceSink (Java authority)      Engine2ActorTraceRing (one actor)
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
defeat actions firing. That's the hero-and-fish interaction from section 4.5, seen from the
inside.

## 7. Levels, campaign and certification

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

## 8. How we verify

Each task has written exit criteria, and "done" means tests green **and** a device check where the
task touches the player:

- **JVM suites:** about 1,945 app tests and 175 `gameengine` tests, run in both normal and
  forced-Engine 2 authority modes.
- **Differential tests:** Java versus Engine 2, tick for tick, from 2,000 to 10,000 ticks.
- **Device checks on a Pixel**, hash-bound to the APK, with screenshots kept as evidence.
- **Save safety:** device checks that would change progress run on a scratch save, then the
  owner's real 739-byte profile is restored and verified by SHA-256.

## Impact and Next Steps

**What this buys:**

- Engine 2 now runs real scene slots and actor families in a shipping-shaped game, with a one-flag
  rollback.
- Mixed authority is safe by construction: one thread, one writer per field, one authority per
  actor or slot, and a shared contact pass.
- When a rule misbehaves on a phone, we can pause, step and read *why*, then replay the exported
  trace on the JVM.

**What's next:**

1. **Scene waves 2 and 3**: scenes 46 to 310 and the non-campaign scenes.
2. **Actor waves C to E**: mechanisms, enemies and hazards, projectiles and bosses.
3. **Hero port**: locomotion, weapons, forms, mounts, transitions, death and revival as rules.
4. **Release fault policy**: decide what a faulted program does in a shipping build.
5. **Performance and memory budgets** on the test phones, then cutover with Engine 2 as the
   default authority.

## Conclusion

The theme of this stretch was making the game *observable and accountable*. The frame order is
written down and tested. Engine 2 runs inside that frame on one thread, touches state only through
the host, and can be selected per actor family and per scene slot with a reported reason. The
rule inspector shows what the VM did on a real device. With those in place, the rest of the port
can be added one wave, and one slot, at a time without guessing.

---

## Related Posts

- [Super Morse: A Proposed Game Economy and Its Reference Implementation](/posts/2026/10/super-morse-game-economy-design/): the economy, split out into its own post
- [Introducing Engine 2: A Rule Engine for Game Behavior](/posts/2026/10/engine2-rule-engine-introduction-guidebee-game-engine/): the language and VM this post integrates
- [From One Game to a Toolkit: Extracting Reusable Platformer Architecture](/posts/2026/09/platformer-engine-architecture-guidebee-game-engine/): the engine boundary
- [Level Design at Scale: Atlases and Tile Catalogs](/posts/2026/09/level-design-at-scale-atlases-tile-catalogs/): the level pipeline

## Connect

- **GitHub**: [guidebee](https://github.com/guidebee)
- **LinkedIn**: [James Shen](https://www.linkedin.com/in/guidebee/)

---

*This is post #43 in the Super Morse series. Write the frame order down, make every authority
explicit, port one slot at a time, and give yourself a way to watch the rules run on a real phone.*
