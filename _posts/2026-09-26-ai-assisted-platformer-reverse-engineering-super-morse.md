---
layout: single
title: "Building a Mario-Like Platformer — and Resurrecting Three Lost Android Games — With AI as My Co-Developer"
date: 2026-09-26
permalink: /posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/
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
excerpt: "A side project that started as 'port an old Mario clone to a modern Android engine' turned into something stranger: decompiling three abandoned mobile platformers byte by byte and rebuilding their game design from the evidence, with an AI assistant doing most of the archaeology."
---

![Title screen of the reconstructed platform-adventure game](../images/supermorse_title_screen.png)

## TL;DR

- **Super Morse** is a personal Android project built on a 2013-era fork of libGDX I call the
  Guidebee Game Engine (GGE) — the same engine that already runs a Flappy Bird clone and a
  Battle City clone in the same app.
- The headline addition is a full **Mario-style platformer port**: 8 worlds, 55 levels, three
  power states, a dozen enemy types — built from an old desktop Java game and re-hosted on GGE.
- The stranger addition: I also **decompiled three old, abandoned Android platformers**
  (Super Bob Go, Super Bucky Adventure, and Super Bobby's Adventure — all built on the same
  DHZSoft-derived engine) and am rebuilding one of them as an original game, using the recovered
  binary as *evidence*, not as source code to copy.
- None of this would have been a realistic solo side project. The interesting part isn't "AI
  wrote a platformer" — it's that an AI assistant turns out to be unusually good at the specific,
  tedious, evidence-grading discipline that reverse-engineering an undocumented binary format
  actually requires.
- This is post #36, and the first of five about this project — the others go deep on engine
  architecture, movement physics, level design at scale, and actor/enemy behavior.

---

## Two Projects Wearing One App

Super Morse is one Android app with three unrelated games sharing an engine, the same shape my
[Flappy Bird and Battle City clones](https://github.com/guidebee) already used. What made this
one different is that I pointed the same AI-assisted workflow at two very different kinds of
problem in the same repository:

1. **A straightforward port.** Take an old desktop Java Mario clone I'd built years ago, and
   re-host its gameplay on GGE's `microedition` (`LayerManager`/`Sprite`/`TiledLayer`) API — the
   same MIDP-style layer the other two games already use. This is "normal" game development:
   design the actor tree, tune the physics, build the levels.
2. **A forensic reconstruction.** Three old Android platformers — *Super Bob Go*, *Super Bucky
   Adventure*, *Super Bobby's Adventure* — all sat as decompiled, obfuscated APKs on my disk,
   built on some studio's own bespoke game engine with binary scene/actor data and a custom rule
   VM. I wanted to build a new, original platform-adventure game that reused what I could
   *recover* from that engine — real jump physics, a real actor roster, real level layouts — as
   a design reference, not a licensing headache. That meant treating a decompiled `.smali`/DEX
   tree the way you'd treat a court exhibit: read it, cite it, grade your own confidence in it,
   and never present a guess as a fact.

The second problem is the one I didn't expect an AI assistant to be good at. It turned out to be
the better half of the story.

---

## What "AI-Assisted" Actually Looked Like Here

If you've read [how I built the Solana trading
system](/posts/2026/03/ai-assisted-development-how-i-built-solana-trading-system/) or [how I
revived iRobot](/posts/2026/09/reviving-irobot-old-project-ai-agents/), the shape of this will be
familiar: I stay in charge of product decisions and scope; Claude Code does the volume work of
reading, writing, and — this time especially — *cross-checking its own claims against source*.

One day's git log from this project tells the story better than I can summarize it:

```
09:15  queue hidden controllers and spawn templates in the placement audit
09:27  decode the resource 75 bubble emitter; pin the qipao rig facts
11:16  let Kuba's axe-route bridge fall finish before the stage clears
11:33  Fix the four stale unit tests; correct the boss fireball to rules 1-5
11:45  draw Kuba's fireballs, throw both per attack, and let them hurt
13:54  enable per-rule scene repeat (R06) for gameplay
14:21  Count landings on Level 1's number lift; collapse and rebuild it
15:48  classify the readiness gaps of the device-certified stages 1-16
17:27  scene 332 tripwire puzzles; correct scene actions 6, 30 and 35
20:28  reconcile ten more pipe/exit-only scenes for the rescan cadence
21:04  reconcile 67 more scenes, leaving only 2 of 85 programs blocked
22:21  close and archive - scene 332 was its last open item
```

Fifty commits, roughly thirteen hours, one person driving. Every one of those commits reads like
a small, falsifiable claim — "Kuba's fireball rules are 1-5," "67 more scenes reconciled" — not
"implemented boss AI." That specificity isn't incidental; it's the whole point of how this had to
be built. When your source of truth is an obfuscated decompiled class named
`p408t3.YRXWvt1034675`, "roughly implemented" isn't good enough — either you can point at the
rule that proves a mushroom's stomp behavior, or you're inventing one.

---

## The Discipline That Made the Reverse-Engineering Half Work

Early on I made Claude adopt — and it stuck to, which is the part I didn't fully expect — a
three-tier evidence system for every claim about the original games' behavior:

- **Recovered** — read directly from the decompiled class or the decoded binary rule data. Not a
  guess.
- **Inferred** — a reading that fits every observed rule and play test, but that the code itself
  doesn't name.
- **Designed replacement** — an explicit *my* decision, made because the source is missing,
  incomplete, or just not worth reproducing exactly.

That grading shows up everywhere in the project's docs, right down to individual sentences:
"stomp bounce is **recovered** as 28; whether it stacks with a Star is **inferred**." It sounds
pedantic until you remember the actual failure mode it prevents: an AI assistant that's
confidently, plausibly wrong about a binary format is much more dangerous than one that says "I
don't know yet, here's the undecoded byte." I'd rather have forty scenes marked "blocked, reason
recorded" than four hundred silently wrong ones.

The same discipline paid off in the more conventional Mario port, just pointed at a different
kind of ambiguity — not "what does this byte mean" but "should this become shared engine code or
stay game-specific." More on that in the [architecture post](#) below.

---

## What Actually Got Built

**The Mario port** — all 8 worlds, 55 levels, on today's Nintendo-derived placeholder art pending
an original-art reskin — is functionally complete: three power states (Small/Big/Fire), a dozen
enemy types with a consistent stomp-or-hurt contract, water levels with genuinely different
physics, pipe-warp secrets, and the full castle-boss template. I go deep on how that's built in
the [engine architecture](/posts/2026/09/platformer-engine-architecture-guidebee-game-engine/) and
[movement physics](/posts/2026/09/tuning-jump-arc-platformer-physics-ai/) posts.

**The reverse-engineered platformer** — codenamed Super Bob Go internally, after the game it's
recovering evidence from — is mid-flight: Level 1's full vertical slice (load → play → death/
revive → objectives → finish) is built and device-verified, and of the game's 85 scene programs
that gate the first sixteen displayed stages, all but 2 are now reconciled against recovered
data. Its actor registry alone is 316 resources; getting a readable behavior contract out of each
one — "this is a crumbling one-way platform with a 10-tick fuse," not "actor type 101, rules
undecoded" — is its own story, in the
[actors post](/posts/2026/09/actors-behavior-rules-reverse-engineered-enemies/).

Both games also gave me a genuinely reusable byproduct: a **level atlas system** that generates
minimaps, tile/enemy/checkpoint breakdowns, and design notes directly from shipped level data,
so the documentation can never silently drift from what's actually in the APK. That's the subject
of the [level design post](/posts/2026/09/level-design-at-scale-atlases-tile-catalogs/).

---

## Why This, and Why Now

Same honest answer as the iRobot post: I had the time, and the domain scratched a specific itch.
I'd built the original desktop Mario clone years ago and always meant to give it a proper mobile
home; the three decompiled APKs had been sitting in a `C:\workspace\` folder for even longer,
half-explored, because doing this kind of forensic reading by hand — tracing one obfuscated
method at a time through a DEX tree — is exactly the kind of task that's individually easy and
collectively enormous. That's a different bottleneck than the one iRobot hit (not enough hours),
but the same class of problem: work that's real and worth doing, but that doesn't survive contact
with a solo schedule until something changes the unit economics of doing it.

What changed, concretely: I could ask "what does resource 101's rule 10 actually do" and get back
a cited answer — decompiled class name, rule numbers, a plain-English contract — in the time it
used to take me to open the right `.smali` file. Fifty small, evidence-graded commits in a day
isn't a claim about typing speed. It's a claim about how much verified ground you can cover when
reading obfuscated bytecode stops being the bottleneck.

---

## What's Next

The reverse-engineered platformer's own roadmap is explicit about what's left, in the same
evidence-graded terms as everything else: boss encounters, mounts, the meta-progression economy,
and a genuine Spine 3.6 animation runtime (the original rig format current tooling can't read
yet) are all flagged **open**, not silently skipped. The Mario port's own next step is a visual
reskin — replacing every Nintendo-derived placeholder sprite with original art — which is
deliberately gated behind finishing a platformer-toolkit extraction first, so the reskin isn't
fighting a moving architecture underneath it at the same time.

---

## Conclusion

The part of this project I'm proudest of isn't the platformer itself — side-scrolling run/jump
games are a well-trodden genre, and I didn't invent any of its mechanics. It's that "go read an
obfuscated decompiled Android game and tell me, with citations, what its jump physics actually
are" turned out to be a task an AI assistant could do rigorously enough to trust the output. That's
a different kind of leverage than "write me a function" — it's leverage on the kind of tedious,
error-prone forensic reading that used to be the real cost of a project like this.

---

## Related Posts

- [Reviving iRobot: An Old Android Project, Six Years Later, With AI Agents as Co-Developers](/posts/2026/09/reviving-irobot-old-project-ai-agents/) — the same methodology, a different old project
- [How I Built a Solana Trading System with AI as My Co-Developer](/posts/2026/03/ai-assisted-development-how-i-built-solana-trading-system/) — where the evidence-graded, specialist-role workflow started

## Connect

- **GitHub**: [guidebee](https://github.com/guidebee)
- **LinkedIn**: [James Shen](https://www.linkedin.com/in/james-shen-5190926/)

---

*This is post #36. Same throughline as the last: AI agents doing the heavy lifting on reading,
writing, and — this time — evidence-grading; a human still deciding what's worth building and
what counts as proof. Four more posts on this project follow: engine architecture, movement
physics, level design at scale, and actor/enemy behavior.*
