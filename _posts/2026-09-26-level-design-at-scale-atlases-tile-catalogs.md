---
layout: single
title: "Level Design at Scale: Atlases, Tile Catalogs, and 356 Decompiled Scenes"
date: 2026-09-26
permalink: /posts/2026/09/level-design-at-scale-atlases-tile-catalogs/
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
excerpt: "55 hand-authored levels and 356 decompiled scenes are two completely different level-design problems. One needed documentation that could never drift from the shipped JSON; the other needed a map from a marketing-facing level number to a buried binary scene ID."
---

![A generated level minimap, decoded straight from shipped level data](../images/mario_level_11_minimap.png)

Post #39 in the [Super Morse](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/)
series. Level design sounds like one discipline, but this project has two genuinely different
level-design problems living side by side: authoring and cataloging 55 original levels I have
full control over, and *identifying* 356 scenes I didn't author at all, recovered from a
decompiled binary where even the numbering scheme is its own small mystery.

## TL;DR

- Mario's 55 levels across 8 worlds are documented with a **generated** atlas — schematic
  minimaps, tile/enemy/checkpoint counts, and pacing notes, all produced directly from the
  shipped level JSON, so the docs can never silently drift from the actual game.
- The reverse-engineered game has 356 decompiled **scene programs** but no obvious mapping from
  "the level number a player sees" to "which scene resource that actually is" — recovering that
  formula was its own small research task.
- A generated **tile catalog** (315 recovered image assets) and a **level atlas** for the first
  16 stages turn a pile of decoded binary into something a human can actually browse and reason
  about.
- The common thread: whenever documentation *could* be hand-maintained prose, I made it generated
  data instead — because hand-maintained prose about game content drifts, and generated catalogs
  don't.

---

## Mario: Every Minimap Is Regenerable, on Purpose

The Mario level atlas is explicit about one rule: it's built directly from
`app/src/main/assets/mario/levels/*.json`, "not hand-transcribed, so it can't drift from the
shipped content." Every one of the 55 levels gets a schematic minimap — column = x position, row
= y (height), oriented exactly like the in-game camera — plus a full data breakdown:

![Minimap legend: checkpoints, pipe warps, spawn point](../images/mario_level_legend.png)

A representative entry (World 1, Level 2 — the game's first secret):

> **Theme:** UnderGround / GreenAndTrees · **Size:** 310×16 tiles
> **Enemies:** EnemyMushroom ×14, EnemyTurtle ×3, EnemyTurtlePatrol ×1
> **Checkpoints:** horizontal pipe → level 13; four vertical pipes → levels 98, 41, 31, and
> **21 — the World-1 secret warp room**
> **Ground structure:** a 7-wide gap crossed by a falling lift; a 7-wide gap crossed by a rising
> lift; enemy encounters concentrate in the opening third (14/4/0 opening/middle/closing)

Two things about that last line matter more than they look like they should. First, "enemy
encounters concentrate in the opening third" isn't a design note I wrote by eyeballing the level
— it's the output of an automated pass over the level's own tile positions, counting enemy
placements across three equal horizontal bands. Second, the doc is explicit about a failure mode
it deliberately avoids: gaps too wide to honestly call a single jump — which usually mean a
multi-tier floor, not a real chasm — are flagged as such rather than reported with false
precision. That's a small thing, but it's exactly the kind of overclaiming an automated
"describe this level" pass could produce if nobody was watching for it, and worth catching
before it becomes a permanent, quietly-wrong sentence in the documentation.

Zoomed out, the eight worlds follow one repeating shape almost every game in this genre uses —
worth naming plainly rather than rediscovering level by level:

| World | Setting | New mechanic |
|---|---|---|
| 1 | Grassy hills → caves → castle | The whole core loop: bricks, pipes, first boss, first secret |
| 2 | Grassy hills → **underwater** → castle | Sea physics, non-gravity-affected enemies |
| 3 | Grassy hills → sky garden → castle | Patrol enemies, hammer-throwing `Monkey`, seesaw lifts |
| 4 | Caves → sky → **maze castle** | First castle built around pipe warps instead of one bridge |
| 8 | Grassland ×3 → **5-room castle gauntlet** → underwater → finale | Genuine dead-end loops; the real ending |

World 1–7 all share the same four-level shape (grassland → themed second level → sky/lift level →
castle boss); World 8 deliberately breaks the pattern for the finale. Naming that pattern
explicitly, once, in the docs is what makes it obvious when a *new* level accidentally breaks it
without meaning to.

## The Reverse-Engineered Game: Finding the Map Before Drawing the Map

The other game's level-design problem starts one step earlier: before I can catalog a scene, I
need to know *which* scene a player-facing level number actually is. The original game's own
numbering has two disjoint regimes, recovered rather than assumed:

```text
Displayed level 1..281:
  scene  = displayedLevel + 29
  difficulty = 0

Displayed level 282+:
  n = displayedLevel - 282
  scene = 50 + (n % 261)
  difficulty = floor(n / 261) + 1
```

That's not a guess dressed up as a formula — it's normative, with boundary tests recorded right
alongside it (level 1 → scene 030/difficulty 0; level 282 → scene 050/difficulty 1; level 543 →
scene 050/difficulty 2), specifically so a future implementation change can be checked against
fixed points instead of re-derived from scratch.

Once scenes are addressable, they get the same generated-catalog treatment as Mario's levels, at
a very different scale: **356 decoded scenes**, **46,816 serialized actor records**, and **315
recovered image assets** across the shared asset buckets. A generated tile catalog turns that
pile of binary into something browsable — contact sheets per asset bucket, one row per recovered
image, with an honest gap called out explicitly rather than papered over:

> "Only assets whose original filename is a real word... have an actual name. The plain numeric
> IDs have **no recovered name** — I checked the decompiled code for a tile-ID-to-name lookup
> table and found none, so the Name column is left blank for those."

![The reconstructed game's campaign/world map, generated from recovered level data](../images/supermorse_campaign_map.png)

A **level atlas** for the currently-certified first 16 displayed stages layers design intent on
top of that same recovered data — required actor clusters per stage (the objects that *must*
resolve for a level to be playable at all: player spawn, camera targets, coins, key objective,
finish flag, revival bubble), read against real scene 030 records rather than "a manually
convenient position."

## Canonical Sources, Not Just a Style Guide

The thing that keeps both of these level-design efforts honest is the same explicit precedence
rule, stated once at the top of the docs and then actually followed:

1. Parsed binary evidence and generated JSON, always first.
2. A recovered-facts document, for anything binary evidence alone doesn't fully explain.
3. My own product/design decisions, for anything the source doesn't have an opinion on.
4. Everything else — architecture docs, implementation recipes — last.

That ordering matters specifically because level design is where "what did the original game
actually do" and "what do I want this new game to do" collide most often. A level atlas that
quietly let design intent outrank recovered fact would drift into fiction one convenient
assumption at a time; one that only ever asserts recovered fact would never let me actually
design anything new. Stating the order once, and pointing an AI assistant at it before every
research task, is what keeps forty-five decoded scenes and eight new-game design decisions from
turning into forty-five decoded scenes and eight silently-blended guesses.

---

## Related Posts

- [Building a Mario-Like Platformer — and Resurrecting Three Lost Android Games](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/) — the series overview
- [Giving NPCs a Brain: Enemy Design and Reverse-Engineered Actor Rules](/posts/2026/09/actors-behavior-rules-reverse-engineered-enemies/) — the actors that populate these levels
- [Tuning the Jump Arc: Platformer Physics, Recovered and Designed](/posts/2026/09/tuning-jump-arc-platformer-physics-ai/) — the movement these levels are built around

## Connect

- **GitHub**: [guidebee](https://github.com/guidebee)
- **LinkedIn**: [James Shen](https://www.linkedin.com/in/james-shen-5190926/)

---

*This is post #39 in the Super Morse series — five posts on building an AI-assisted platformer
and reverse-engineering three abandoned Android games alongside it.*
