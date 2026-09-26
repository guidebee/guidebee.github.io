---
layout: single
title: "Giving NPCs a Brain: Enemy Design and Reverse-Engineered Actor Rules"
date: 2026-09-26
permalink: /posts/2026/09/actors-behavior-rules-reverse-engineered-enemies/
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
excerpt: "One game's enemies follow a rule I wrote down myself: jump on top to defeat, touch the side and it hurts. The other game's 316 actor resources follow rules nobody wrote down for me — they had to be pulled out of an obfuscated rule VM, action by numbered action."
---

![Field guide to the Mario port's enemy roster](../images/mario_enemies.png)

Post #40, the last in the [Super Morse](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/)
series. Everything so far — architecture, physics, levels — sets the stage for the part of a
platformer players actually spend most of their attention on: what the things trying to kill you
actually do. One game's enemy roster I designed myself, with a consistent rule I could just
write down. The other game's 316-resource actor registry had no such rule available to write down
— it had to be decoded from an obfuscated rule VM, one numbered action type at a time.

## TL;DR

- Mario's enemy field guide follows one consistent rule — **stomp from above to defeat, touch
  from the side and it hurts you** — with a handful of deliberate, clearly-labeled exceptions
  (`Spikey` never dies to a stomp; `Helmet` shrugs off fireballs).
- The reverse-engineered game's 316 actor resources had **no such rule available up front** — each
  one's behavior lives in a numbered, per-resource rule program that had to be decoded and
  translated into a plain-English contract, graded by confidence the whole way.
- The single biggest unlock was building a **shared rule vocabulary** — one table mapping "action
  type 25" to "spawn a pooled clone, place it, set it active" — so that decoding resource #202
  didn't mean starting from zero after decoding resource #97.
- Worked example: a push-button-and-bridge mechanism (resources 97/104/105) went from "three
  numbers in a registry" to a fully specified contract — including a subtlety the original
  handbuilt port had gotten *wrong* — once its rules were actually read rather than guessed at.

---

## Mario: One Rule, Named Exceptions

The enemy roster's whole design fits in one sentence, and the field guide states it exactly that
plainly: **jump on top of most enemies to defeat them; touching them from the side hurts you** —
unless you have a Star, in which case any touch destroys them.

| Enemy | Stompable? | Notes |
|---|---|---|
| `EnemyMushroom` | Yes | The most common enemy by far; walks forward, turns at walls, falls into pits |
| `EnemyTurtle` | Yes, leaves a shell | Kick the shell to send it sliding — it defeats *other* enemies, but also hurts you |
| `Helmet` / `HelmetShell` | Yes, **fireball-immune** | Same shell-kick trick works; Fire Mario's fireballs bounce off |
| `Spikey` / `SpikeyEgg` | **No — never** | Every touch hurts, from any side, unless starred |
| `Boss` | Only with a Star | A stomp or side-touch just hurts you; six fireballs or one Star kill it |

Every exception in that table is a deliberate genre convention I chose, not an accident of
implementation, and it's stated as such: `Enemy`'s contract
(`onStomped`/`onTouchedSide`/`onDefeatedByProjectile`/`bouncesOffEnemies`) is a genuinely
game-agnostic three-verb reaction interface — only the *default* implementations encode Mario's
specific rules, and even those defaults are "reasonable platformer-genre defaults," explicitly
flagged as the kind of thing a hypothetical second game could override without touching the
interface itself. That's the same architectural boundary from the [architecture
post](/posts/2026/09/platformer-engine-architecture-guidebee-game-engine/) in this series, just
seen from the enemy side instead of the player side.

## The Reverse-Engineered Game: No Rulebook, Just a Rule VM

The other game's 316 actor resources have no field guide to start from — their behavior lives in
a per-resource, per-instance rule program (`levels/<id>/collision.mo123.bin`, despite the
misleading path — it's an actor *behavior* program, not level collision data), executed by a
shared runtime class evaluating ordered `if`/`else-if`/jump rules over the actor's own state
arrays.

The single move that made this tractable instead of a 316-times-repeated slog was building one
shared vocabulary table, once, mapping numeric action/condition types to plain meanings — with
every entry graded by evidence tier:

| Action type | Meaning | Tier |
|---:|---|---|
| 0 | Move the actor by its speed field, direction 0-3 | field layout recovered |
| 23 | `startAscent(v)` — begin or adjust a jump/gravity impulse | recovered |
| 25 | **Spawn**: take a pooled clone, place it, activate it, optionally bind it to a target's touch-list | recovered |
| 45 | **Bind query**: refresh "who is touching my rectangle right now," up to *n* actors, by several selection modes | recovered from a three-game DEX cross-reference |
| 56 | **Grid stamp**: write a material value into the level's collision layer — 1 solid, 2 one-way, 5 low-friction, 66 water, 99 lethal | recovered |
| 6, 7, 9, 15... | Spawn/attack configuration, particle effects, rare setters | still undecoded, printed raw rather than guessed |

That last row matters as much as the recovered ones — the vocabulary table is honest about what
it *doesn't* know yet, and a rule that hits an undecoded action type gets printed raw rather than
silently treated as a no-op. An unsupported term staying visibly unsupported is the whole point;
the alternative is a behavior that looks implemented and just quietly does nothing.

### Worked Example: A Button, a Bridge, and a Bug the Port Already Had

The most satisfying single result from this decoding effort was a mundane-sounding push-button
mechanism — three actor resources (97 the button, 104/105 the two states of a bridge block it
controls) — because decoding it *found and fixed* a real behavioral gap in logic I'd already
hand-built:

- **Resource 97** refreshes a "who's touching me" list every tick from its own rectangle. While
  that list is non-empty it sets an internal flag; a rising edge plays a "pressed" animation with
  a sound, a falling edge plays "released" the same way.
- **Resources 104 and 105** each mirror that same flag from 97 via a direct actor-to-actor
  compare — one is solid at rest, the other is its exact inverse, and each rising edge flips
  which one is solid, pushing anything standing on the newly-solid block 80 units aside.

The correction that fell out of actually reading rule 0 rather than assuming: the press condition
is *"any untagged active actor touching the plate's rectangle,"* not *"the hero standing on it
while grounded."* My own hand-built `SwitchNetwork` implementation had assumed the latter — a
completely reasonable design guess, and close enough that nobody had noticed it was wrong — right
up until the recovered rule showed it also fires for an actor touching the plate mid-air, or for
something other than the hero entirely. That's a small gap, but it's exactly the kind of thing
that would have shipped quietly wrong forever without ever going back to the source and checking.

### The Boss Fight That Needed a Whole Cross-Reference Table

Not every actor decoded this cleanly. **Kuba**, one of the game's bosses, needed its own dedicated
research pass — separating a decorative "story" version of the boss from the one whose health
(system variable `S[30]`) actually drives a real fight, resolving that its on-screen health bar
loses exactly 10 per qualifying hit, and pinning down that its defeat trigger reads a *different*
per-frame attack-target field than its normal hurtbox does. None of that was guessable from
playing the original game — the story boss and the real boss look identical on screen — it only
became legible by reading the rule numbers directly and cross-checking them against which of the
game's 24 scenes actually place the "real" resource versus the decorative one.

## The Same Discipline, Applied to a Harder Target

Everything in this post is really the [evidence-tier discipline from the overview
post](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/) — recovered,
inferred, undecoded — pointed at the hardest target in the whole project: 316 resources, most of
them reused across hundreds of scene placements, each capable of hiding a subtlety like the
button's mid-air press or Kuba's split identity. What made an AI assistant genuinely useful here
wasn't writing enemy code — Mario's enemy roster didn't need any archaeology, and got built the
normal way. It was the willingness to grind through a numbered rule table, resource by resource,
without quietly rounding an unresolved condition type into a plausible guess just to keep the
narrative moving. That's tedious in a way that's easy to shortcut and hard to catch after the
fact — which is exactly why grading every claim by its evidence tier, out loud, in the docs
themselves, turned out to be the load-bearing habit for this whole side of the project.

---

## Related Posts

- [Building a Mario-Like Platformer — and Resurrecting Three Lost Android Games](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/) — the series overview
- [Tuning the Jump Arc: Platformer Physics, Recovered and Designed](/posts/2026/09/tuning-jump-arc-platformer-physics-ai/) — the same recovered-rules discipline, applied to the player instead of enemies
- [Level Design at Scale: Atlases, Tile Catalogs, and 356 Decompiled Scenes](/posts/2026/09/level-design-at-scale-atlases-tile-catalogs/) — where these actors actually get placed

## Connect

- **GitHub**: [guidebee](https://github.com/guidebee)
- **LinkedIn**: [James Shen](https://www.linkedin.com/in/james-shen-5190926/)

---

*This is post #40, closing out the Super Morse series — five posts on building an AI-assisted
platformer and reverse-engineering three abandoned Android games alongside it.*
