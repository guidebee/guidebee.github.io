---
layout: single
title: "Introducing Engine 2: A Rule Engine for Game Behavior"
date: 2026-10-05
permalink: /posts/2026/10/engine2-rule-engine-introduction-guidebee-game-engine/
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
  - virtual-machine
  - side-project
excerpt: "A tour of Engine 2, the game-independent rule language and VM inside the Guidebee Game Engine: its four-layer architecture, how a rule is compiled and executed, what the specification pins down, and why game behavior belongs in data instead of code."
---

Post #42 in the [Super Morse](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/)
series. Earlier posts touched on Engine 2 in passing. This one is the introduction I should have
written first: what it is, how it is put together, how it runs, what the spec guarantees, and
the question that matters most, which is *why a rule engine at all?*

## TL;DR

- **Engine 2** is a **game-independent rule language and execution system**. Actor and scene
  behavior is written as typed, ordered rules, compiled to a project-owned binary format
  (`.e2b`), and executed by a small virtual machine. The game keeps its mechanics (movement,
  collision, rendering, audio); the rules only *sequence* them.
- The design is **four layers**: an engine-neutral core, a per-game profile, a per-game host,
  and a per-engine platform adapter. Dependencies point down only.
- The runtime is **two small machines** (actor and scene), driven by integer ticks, with
  bounded budgets, deterministic behavior and source-mapped diagnostics.
- It is specified before it is trusted: a 1.0 specification, byte-exact test vectors, and a
  reference implementation of the core in the `gameengine` module. **Seven of eleven delivery
  phases are closed**; the first game profile and host integration are next.

![Gameplay from the platformer that Engine 2 is being built to drive](../images/supermorse_gameplay_hud.png)

## 1. What Engine 2 is

Think of how a programming language relates to the programs written in it. Engine 2 relates to
games the same way. It defines:

- a small **language** (`.e2`) for describing what an actor or a scene does,
- a **typed model** and a **binary format** that every rule compiles to,
- a **VM** that executes that format, and
- a narrow **host boundary** through which rules affect the world.

What it deliberately does *not* define is anything about a particular game. There is no jump
height, no enemy, no tile in the core. Those arrive through a **profile**, which is the
contract saying what a game's rule operations mean.

Here is the entire shape of an authored actor, taken from the specification's normative example:

```text
engine2 1;
profile "spec.example";
actor Counter resource 1 {
    local count: int at 0;
    local route: int at 1; local out: int at 2;
    rule first when self.count == 0 { self.count = 1; }
    else rule second when self.count == 0 { self.count = 2; }
    else rule third when self.count == 0 { self.count = 3; }
    rule route when self.route == 0 { self.route = 1; goto finish; }
    rule skipped { self.out = 9; }
    rule finish { self.out = 7; return; }
}
```

Each **rule** has an optional branch form (`else`, `return`, `goto`), a list of **conditions**
and a list of **actions**. Conditions and actions are not free-form code. Each one is a call to
an **operation registered by the game's profile**, with a typed signature. That is the whole
trick: the language is tiny and fixed, and the vocabulary is supplied per game.

Scenes use the same model with two additions, **activation** and **repeat**, plus the ability to
wait:

```text
scene Intro resource 2 {
    program opening at 0 {
        rule start activation always repeat once when system.phase == 0 {
            system.phase = 1;
            wait ticks(1);
            system.phase = 2;
        }
        else rule alternate activation always repeat forever when system.phase == 0 {
            system.phase = 9;
        }
    }
}
```

That `wait ticks(1)` is how a cutscene, a tutorial prompt or a camera move is sequenced without
the rule author writing a state machine by hand.

## 2. Architecture: four layers

![Engine 2's four layers and the boundary rules between them](../images/engine2_layers.png)

| Layer | Owns | Must not own |
|---|---|---|
| **Core** (`gameengine`, package `com.guidebee.game.engine2`) | Language, compiler, typed model, binary writer and loader, validator, actor and scene machines, scheduler contract, diagnostics, trace, snapshots, abstract host ports | Any game resource ID, Android type, renderer or physics |
| **Game profile** (per game) | Operation registry with typed signatures, capabilities, resource catalog, selector variants, policies | Control flow, parsing, another game's data |
| **Game host** (per game) | World state, collision, physics, animation, camera, HUD/UI, saves, the `RenderSnapshot` and audio events | A second behavior authority |
| **Platform adapter** (per engine) | Device input, rendering, audio, assets and storage, clock and lifecycle | Any gameplay decision |

A few boundary rules do most of the work of keeping this honest:

1. **Rules never draw or play anything.** Effects leave through typed host ports. Rendering
   reads an immutable snapshot and never writes actor state; audio consumes semantic events
   ("play this clip") rather than calling an audio API.
2. **One behavior authority** per actor instance and scene program. Two writers is a defect,
   and authority only switches at a stage-attempt boundary.
3. **The core never touches the platform.** Time, randomness, storage and input reach it
   through injected ports, and the core and game host are single-threaded.
4. **Unsupported means blocked.** If a program needs a capability the host has not
   implemented, the *whole program* is disabled with a source-mapped diagnostic. It is never
   enabled as a silent no-op.

The practical effect of this layering is that porting is well-scoped. The **portable asset is
the specification and the `.e2b` byte format**, not the Java classes. The first target is the
JVM (Android, with the libGDX-derived Guidebee Game Engine); a Unity host is planned later, and
golden traces from the Java reference host are the conformance target any second core must
reproduce event for event.

## 3. From source to a running rule

![The build, load and enable pipeline](../images/engine2_pipeline.png)

The pipeline has a deliberate gate in the middle:

1. The **compiler** lowers `.e2` source to the typed model, checking every call against the
   game's profile and recording a source map so a runtime event can be traced back to a line.
2. The **binary writer** emits an `.e2b` module. Output is **reproducible**: identical models
   produce identical bytes, with no timestamps or absolute paths. The header carries a SHA-256
   **digest of the profile's canonical contract**, so a module can only run against the profile
   it was compiled for.
3. The **loader** validates bytes, digests and signatures. It never calls an effect.
4. **Enabling** is a separate step. Every capability the module needs must be reported by the
   host at an equal major version, and every resource it references must exist in the game's
   catalog with the right type. Only then is an immutable binding published.

Validation alone grants no authority. A missing capability or resource blocks the program with a
diagnostic, and a change in the resource catalog deauthorizes a program before its next visit.

## 4. The runtime: two machines

![Actor machine and scene machine side by side](../images/engine2_rules.png)

Engine 2 has two machines that share values, tracing and host services, but keep **separate
control flow and separate operation namespaces**. A scene action and an actor action that happen
to share a number mean unrelated things, and the registry is keyed by
`(family, role, id)` so they can never be confused.

**The actor machine** runs a visit from rule 0 on every tick. The pseudocode from the spec is
short enough to quote:

```text
pc = 0
while pc < ruleCount:
    charge one rule-visit              // fault E2_BUDGET if over 4096
    r = rules[pc]
    if r.branch is an else form:
        if status[pc-1] in {matched, skipped}:
            status[pc] = skipped; pc += 1; continue
    matched = evaluate(r.conditions, r.join)   // ordered, short-circuit
    if not matched: pc += 1; continue
    for a in r.actions:
        charge one action-dispatch     // fault E2_BUDGET if over 16384
        dispatch(a)
    if r.branch is a return form: return
    if r.branch is a jump form: pc = r.jumpTarget; continue
    pc += 1
```

**The scene machine** keeps a cursor per program slot and starts **at most one rule match per
tick**. Its gates are evaluated in a fixed order: else-eligibility, remaining repeat, activation,
then conditions. When an action returns `SUSPEND(token)`, such as a `wait`, the action list
pauses and resumes at the *next* action later. It does not rematch the rule, and it does not
consume the repeat count again.

A few properties make the runtime something you can trust and test:

| Property | How it is achieved |
|---|---|
| **Determinism** | Time is an integer tick; randomness is an injected, seedable, recordable service. No wall-clock reads. |
| **Boundedness** | Rule-visit (4,096) and action-dispatch (16,384) budgets per invocation. Exhaustion is a *fault*, never a false condition. |
| **Honest failure** | A fault stops the selected program, keeps earlier writes, and *latches*. No rollback, no automatic retry. |
| **Order preservation** | The compiler may not reorder, duplicate or fold predicates, because a predicate can read state or randomness. |
| **Stale-token safety** | A new attempt invalidates continuations; a stale token faults with no host write (`E2_STALE_CONTINUATION`). |
| **Tracing** | A bounded trace ring buffer, off by default in release builds, records rule and action events with their source. |

One more design point: **the tick rate is an output of the core as well as an input.** A rule
can change the frame period during play, so the scheduler is a contract between the core and
the host rather than a fixed loop.

## 5. The specification

Engine 2 is specified in nine chapters, and the spec is the design source for every
implementation phase:

| # | Chapter | Defines |
|---|---|---|
| 1 | Architecture | Purpose, scope, layers, platform boundary rules, portability |
| 2 | The `.e2` language | Grammar, declarations, rules, control flow, lowering, diagnostics |
| 3 | Typed model and binary format | Canonical model, profile digest, `.e2b` bytes, versioning, validation, test vectors |
| 4 | Runtime | Machines, state lifetimes, waits and tokens, budgets, scheduler, tracing |
| 5 | Profiles, hosts and adapters | Profile ABI, enablement, host-port tiers, authority and state ownership |
| 6 | First game profile | Concrete operations and semantics for the first supported game |
| 7 | Conformance and delivery | Conformance levels, test strategy, delivery phases, decision register |
| 8 | Studio authoring | Designer-authoring goal for Guidebee Studio |

Three habits keep a spec like this from drifting into fiction:

- **Every statement has a status label**: *Normative* (a requirement, in the RFC 2119 sense),
  *Design*, *Derived*, *Proposed*, *Open* or *Reserved*. A reader always knows how much weight
  a sentence carries.
- **Byte-exact vectors.** The module 1.1 format ships with an exact 1,280-byte actor vector
  and a 699-byte scene vector, plus ten negative paths. The loader must load and **re-encode
  them byte for byte**.
- **A decision register.** Conflicts are resolved by recording a numbered decision, not by
  silently preferring one document.

Equivalence is stated in tiers rather than as a vague promise. For a given game, the *functional
core* (rule order, branch semantics, ordered short-circuit evaluation, wait boundaries, spawn and
remove visibility) must always match; some quirks only matter where real content reaches them;
and exception paths and invalid ranges are free to differ.

## 6. Why a rule engine?

This is the question that decides whether any of the above is worth building. The alternative is
the usual one: write each actor and each scene as Java, one class or one big switch at a time.
Engine 2 argues against that for six reasons.

| Benefit | What it means in practice |
|---|---|
| **Separate *what* from *how*** | Rules pick and sequence mechanics. Movement, collision and animation stay in typed, tested host code that every rule calls identically. |
| **Scale content without scaling code** | A new enemy or cutscene is data compiled against a profile, not another class. The code surface does not grow with the content. |
| **Reuse across games** | Games share the language, compiler, binary format and VM. Each supplies only a profile and a host. |
| **Tooling and diagnostics** | Typed rules can be validated before they run, and every runtime event maps back to a source line. A designer gets an error message, not a crash. |
| **Determinism** | Integer ticks, seeded randomness, bounded work, traces and snapshots make behavior reproducible, which makes regressions *provable*. |
| **Safety** | A module cannot run arbitrary code. Content stays bounded in time and size (at most 1 MiB, 4,096 rules, 256 calls per rule). |

There is a seventh reason that matters to me personally as a one-person shop: **designer
authoring**. The goal in Chapter 8 is that a non-developer can build levels, cutscenes and
tutorials in Guidebee Studio as data that runs on any host implementing the game's profile, with
no game-specific engine code. That only works if behavior is data.

### The costs, stated honestly

A rule engine is not free, and the spec lists what it accepts:

- **Hidden semantics must be written down.** Numeric IDs are opaque, so every operation needs
  a name, a signature and a source map.
- **Rule systems tend to grow into poor programming languages.** Engine 2 fights this on
  purpose: it is *not* a general-purpose scripting language. No loops, no user functions, no
  dynamic strings, no arbitrary code. The language grows only by versioned increments, and
  reserved features are rejected with a diagnostic until they exist.
- **An extra layer to debug.** This is why source-mapped diagnostics, traces and golden traces
  are first-class requirements rather than afterthoughts.

## 7. Where it stands

The delivery plan has eleven phases, and the first seven gates are closed:

| Phase | Status |
|---|---|
| P01 Research and corpus | Done |
| P02 Specification (module 1.1, exact vectors) | Done |
| P03 Typed model, importer, validator | Done |
| P04 `.e2` source compiler | Done for the executable 1.1 subset |
| P05 Binary writer, loader, package, binding | Done: vectors re-encode byte for byte |
| P06 Actor VM | Done for the 1.1 subset |
| P07 Scene VM and scheduler | Done for the one-owner-per-slot subset |
| P08 First game profile and host integration | **Next** |
| P09 to P11 Device pilot, corpus expansion, reconciliation | Planned |

The core lives in `gameengine/src/main/java/com/guidebee/game/engine2/`, with direct JVM tests
passing for the model, compiler, binary format and both machines. Nothing game-specific is in
the core, and that is the point: the next milestone is the first profile and host, the place
where the layering either proves itself or doesn't.

## Impact and Next Steps

**What this buys:**

- New behavior ships as data, validated at compile time and bounded at run time.
- The spec and `.e2b` format, not the Java code, are the portability contract for Unity later.
- Failures are explicit, source-mapped and reproducible.

**What's next:**

1. **P08**: the first game profile, host and adapters, and getting a real scene running
   end to end through the host ports.
2. **Multi-owner scenes**: the scene machine currently supports one bound owner per program slot.
3. **Device pilot**: run compiled modules on a phone and compare traces to the reference host.
4. **Studio authoring**: let a designer produce `.e2` from a visual editor.

## Conclusion

Engine 2 is a bet that game behavior is better treated as *data with a contract* than as code
with conventions. The core is small on purpose: ordered rules, typed calls, two machines, integer
ticks, hard budgets. Everything else, including every mechanic the player actually feels, stays in
the game host where it can be tested like ordinary code. If the bet pays off, adding the next
enemy, cutscene or even the next game is a matter of writing rules and a profile, not touching
the engine.

---

## Related Posts

- [Building a Mario-Like Platformer with AI](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/): the series overview
- [From One Game to a Toolkit: Extracting Reusable Platformer Architecture](/posts/2026/09/platformer-engine-architecture-guidebee-game-engine/): the engine boundary Engine 2 plugs into
- [Engine 2: Designing a Rule VM](/posts/2026/09/engine2-rule-vm-design-guidebee-game-engine/): the earlier, design-stage deep dive

## Connect

- **GitHub**: [guidebee](https://github.com/guidebee)
- **LinkedIn**: [James Shen](https://www.linkedin.com/in/james-shen-5190926/)

---

*This is post #42 in the Super Morse series. Rules for sequencing, code for mechanics, and a
specification that has to prove itself against byte-exact vectors before anyone calls it done.*
