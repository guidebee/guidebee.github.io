---
layout: single
title: "Super Morse: A Proposed Game Economy and Its Reference Implementation"
date: 2026-10-22
permalink: /posts/2026/10/super-morse-game-economy-design/
categories:
  - blog
tags:
  - ai-tools
  - claude-code
  - java
  - android
  - game-development
  - game-economy
  - game-design
  - testing
  - side-project
excerpt: "The proposed Super Morse economy: one logged wallet, a hero roster with weapon mastery, fixed-price boosters, a Treasure hub with visible odds, a free Star Road, military ranks and a Quartermaster. The design is subject to change, and the screens are a reference implementation, not the final UI."
---

Post #44 in the [Super Morse](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/)
series. The [previous post](/posts/2026/10/super-morse-game-architecture-engine2-economy-debug-panel/)
covered the game architecture, Engine 2 integration and the debug tools. This one covers the
game economy on its own.

> **Status: proposal, subject to change.** Everything in this post (the currencies, prices, odds,
> drop rates, reward tables and even which systems exist) is a **proposed design**. None of it is
> final balance, and any of it may change or be cut after playtesting.
>
> **The screens are a reference implementation.** The UI you see below is a plain, functional
> layer built to prove that each system works end to end on a phone: reachable from the game,
> saved correctly, with odds shown. It isn't the final art, layout or flow. The real UI
> will be designed separately and will sit on the same service underneath.

## TL;DR

- **One wallet, one log.** Every balance change goes through a single `EconomyService` that records
  a reason, and every multi-step action commits in one save flush.
- **Proposed systems on top:** a hero roster with weapon mastery, fixed-price level boosters, a
  Treasure hub (daily calendar, wheels, chests), a free Star Road, a Treasure Challenge after boss
  clears, and military ranks with a Quartermaster.
- **Fairness rules are built in, not bolted on:** every random draw has an odds panel, no paid product
  contains a random item, and debug or preview runs never pay rewards.
- **The economy can't change how a level plays.** In-level drops come from an observer that reads
  the world after each tick and never writes gameplay state.

## 1. Why build the foundation first

It's tempting to start an economy with the shop screen. We started with the save file instead.
The old save cleared everything whenever its schema changed. That's fine for a prototype, but
fatal once players have coins and unlocks they care about. So the first task was a **real
migration** that keeps gold, unlocks and progress, followed by one service through which
**every** balance changes:

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

```text
   gameplay (StageAttempt)              product screens
   coins picked up, enemies             boosters, treasure, ranks,
   defeated, level cleared              hero roster, Star Road
            │                                   │
            ▼                                   │
   staged rewards ──(victory/retry/abandon)──►  │
            │                                   │
            └───────────────┬───────────────────┘
                            ▼
                ┌─────────────────────────┐
                │      EconomyService     │  grant / spend + reason
                │  wallet · event log     │  one flush per action
                └────────────┬────────────┘
                             ▼
                      save (migrated)
```

Device testing paid for this design immediately. A coin warp spent 500 coins while the level
stayed locked, because the unlock write was never flushed. Now the unlock and the debit **share one
preferences flush**, and the same rule applies to every economy action.

## 2. Hero roster and weapon mastery

*Proposed.* Ten hero skins map to rigs, signature weapons or skills, tiers, coin prices and piece
counts in `hero-roster.json`. A new save starts with the starter hero and the fireball. Other
heroes unlock with coins or collected hero pieces, and you pick a **goal hero** that piece rewards
go towards.

Coins don't buy new weapons. Each hero keeps its own signature weapon, and coins buy **weapon
mastery levels** that add starting ammo. The coin-magnet hero attracts nearby coins without
needing a mount.

## 3. Level entry: fixed-price boosters

![Reference UI: level entry with Shield, Heart, Magnet and Guardian boosters](../images/supermorse_level_entry_boosters.png)

*Proposed prices.* Before a level you can buy a Shield (absorbs the first ordinary hit), an extra
Heart, a Magnet or a Guardian boost, or a Permanent Guardian. Prices are fixed and shown up front,
and **retries keep unused boosters**, so a failed run doesn't burn what you paid for.

## 4. Treasure hub

![Reference UI: Treasure hub with the Daily Calendar, Lucky Wheel and three chest tiers](../images/supermorse_treasure_hub.png)

*Proposed.* The Treasure hub groups the reward loops in one place:

- a **seven-day Daily Calendar** with a Daily Spin,
- a **Lucky Wheel**, one key per spin, with a visible jackpot counter,
- **wooden, silver and gold chests**.

Every card has a **?** that opens its odds and guarantees, and the footer states the rule
plainly: *earned from play only*.

## 5. Star Road: the non-random track

![Reference UI: Star Road, a free reward every 15 stars](../images/supermorse_star_road.png)

*Proposed.* The **Star Road** is deliberately the opposite of a wheel: a free, known reward every 15
stars (a hero, coins, keys, goal-hero pieces, a gold chest). Nothing on it is random or paid. Claims
go in order and commit in one flush.

## 6. Treasure Challenge

![Reference UI: Treasure Challenge, one free pick from three face-down cards](../images/supermorse_treasure_challenge.png)

*Proposed.* A boss's first clear pays a Gold Star and one **Treasure Challenge**: three face-down
cards, one of each kind (coins scaled by worlds cleared, goal-hero pieces, or keys), in a shuffled
order. You get one free pick, then the other two are revealed. There is no paid re-pick.

## 7. Ranks and the Quartermaster

![Reference UI: Army, Navy and Air Force ranks, Gold Stars, Quartermaster trades and a daily transfer](../images/supermorse_ranks_quartermaster.png)

*Proposed.* Defeating enemies and breaking bricks can drop **insignia**. The branch comes from the
actor family:

| Branch | Earned from |
|---|---|
| Army | walkers, frogs, piranha plants, bricks |
| Navy | pufferfish, squid |
| Air Force | flying fish |

Each drop adds rank progress, which is never spent (*ranks never fall*), and supply, which you
spend at the Quartermaster on coins, ammo, goal-hero pieces or an outfit. A free daily transfer
moves 2 supply between branches. Promotions pay once each, plus a Gold Star.

The proposed drop rule is a pity timer:

```text
chance starts at 10 in 150
each miss:  +1
each drop:  reset to 10 in 150     (4% of drops are Gold Stars)
caps:       4 per attempt in an uncleared scene
            1 per scene per day once the scene is cleared
```

A 3,000-roll test checks the average lands at roughly one drop per 9 to 10 defeats.

**Keeping the economy out of gameplay.** The in-level side is an `InsigniaObserver` that **reads**
the world after each tick and reports enemies that went from live to defeated or captured, plus
newly broken bricks. It never writes gameplay state and never touches the rule random streams, so
turning the economy on or off can't change how a level plays. Script removals (an actor set dead
with no stomp, knockout or capture) don't count.

## 8. Principles we intend to keep

The numbers above will move. These rules are meant to stay:

```text
✓ every balance change goes through EconomyService and is logged with a reason
✓ every multi-step action commits in one flush
✓ every random draw has an odds panel
✓ no paid product contains a random item
✓ debug, preview and scene-test runs never pay rewards
✓ observers read the world; they never write it
```

The current direction is **in-app purchases only, no ads**, with fixed-price products whose price is
shown. That's also a proposal, and the store itself is not built yet.

## 9. How it was verified

- **Unit tests** for migration, wallet grants and spends, a clock moved backwards, staged rewards,
  ranks (drop chance, caps, promotions, recipe limits, daily transfer), the Star Road, chests and
  the Treasure Challenge (30 seeds). The full app suite stays green alongside.
- **Device layout checks on a Pixel** for every screen above, at 2340×1080, confirming nothing is
  clipped and every control is reachable.
- **Save safety.** Checks that would change a save ran on a scratch wallet. Afterwards the real
  739-byte profile was restored and verified by SHA-256, and no purchase or claim was kept.

## What's still open

1. **Balance pass**: a measured balance sheet, then re-cut prices, odds and reward tables.
2. **Store and billing**: fixed-price products and the purchase flow.
3. **Collection book** for hero cards and outfits.
4. **Telemetry** of grants and spends by reason, to drive the balance pass.
5. **Final UI**: replacing the reference screens with designed ones.

## Conclusion

The economy is a proposal sitting on a foundation that isn't. The wallet, the event log, the
single-flush commits, the odds panels and the read-only observer are the parts we expect to keep.
The prices, rates and screens are a first draft, implemented end to end so they can be played,
measured and changed.

---

## Related Posts

- [Super Morse: Game Architecture, Engine 2 Integration and Debug Tools](/posts/2026/10/super-morse-game-architecture-engine2-economy-debug-panel/): the game this economy runs in
- [Introducing Engine 2: A Rule Engine for Game Behavior](/posts/2026/10/engine2-rule-engine-introduction-guidebee-game-engine/): the rule engine behind the actors that drop insignia

## Connect

- **GitHub**: [guidebee](https://github.com/guidebee)
- **LinkedIn**: [James Shen](https://www.linkedin.com/in/guidebee/)

---

*This is post #44 in the Super Morse series. Proposed numbers, reference screens, and a wallet
that logs every coin so the numbers can be changed with evidence.*
