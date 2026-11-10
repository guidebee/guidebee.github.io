---
layout: single
title: "Agent Hub v0: A Day of Claude, Codex and DeepSeek Working as One Team"
date: 2026-11-10
permalink: /posts/2026/11/agent-hub-v0-coordination-hub-for-ai-coding-agents/
categories:
  - blog
tags:
  - ai-tools
  - claude-code
  - codex
  - deepseek
  - mcp
  - agents
  - sqlite
  - cost
  - workflow
  - side-project
excerpt: "Part 2 of Agent Channels. The agent-bus prototype became Agent Hub v0: a per-project SQLite store, a hub-computed task pool, cost-aware routing that spends prepaid Claude and Codex quota before metered DeepSeek, automatic handoffs when an account runs out, and a metrics dashboard. Here is what one day of an unattended team looks like in numbers."
---

Post #47 in the [Super Morse](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/)
series, and part 2 of [Agent Channels](/posts/2026/11/agent-channels-managing-multiple-coding-agents/).
Part 1 was a chat channel that let several coding agents work as a team. This part is about what happened
when I let that team run on its own for a day, and what I had to build to trust it.

![Agent Hub: Claude, Codex and DeepSeek around one laptop, with the team channel, MCP server, dispatcher and owner views](../images/agent_hub_cover.jpg)

## TL;DR

- **The prototype became a project.** `agent-bus` moved out of the Super Morse repository into **Agent
  Hub**, with its git history. The core is still standard-library Python.
- **One SQLite file per project.** Every message, read position, presence record and run record now lives in
  `hub.db`. A database transaction replaced the lock file, and the live channel moved over with no lost events.
- **The hub computes the work, not the agents.** A task pool reports what is ready for each agent. An idle
  agent is only started when something is ready for it.
- **Cost-aware routing.** Claude and Codex are prepaid subscriptions; DeepSeek bills every request. The hub
  now sends work to the prepaid agents first and to DeepSeek when they are out of quota or busy.
- **Limits are handled, not suffered.** When a plan runs out, the hub marks the whole account offline, posts
  a notice, hands over the claimed tasks and tries the account again after the reset, or as soon as I log
  in to another account.
- **A dashboard shows what it costs.** In the last 24 hours: 851 runs, 70.6 agent-hours, 211 of 213 asks
  closed, a median of 1.1 minutes to claim, and about US$30 of metered DeepSeek usage.

**Agent Hub does not try to make models smarter. It tries to make teams of models operationally manageable.**

## At a glance

```
        Claude Code                      Codex
   (architect, reviewer,         (developers, tester,
        auditor)                       manager)
              \                         /
               \                       /
             +-----------------------------+
             |          Agent Hub          |
             |-----------------------------|
             |  Channels    who said what  |
             |  Claims      who owns what  |
             |  Routing     who runs next  |
             |  Costs       what it spent  |
             |  Experience  (next step)    |
             +-----------------------------+
                            |
                         DeepSeek
                 (metered safety net, same roles)
```

The first four layers run today and are what this post is about. **Experience**, meaning lessons from earlier
work that an agent can reuse, is the experiment that comes next.

## What changed since part 1

Part 1 ended with a single Python file and an append-only JSONL log. It worked, and the team used it every
day. Three problems showed up as the team grew:

1. **Agents paid to look for work.** An idle agent woke up every 15 minutes, re-read a Markdown task index,
   and decided from prose rules whether anything was ready. Most of the time nothing was.
2. **Usage limits stopped the team.** When Codex hit its limit, every Codex agent kept failing, one run at a
   time, until a person noticed.
3. **I had no idea what it cost.** DeepSeek charges per token. Claude and Codex are subscriptions with a
   5-hour window and a weekly cap. Nothing measured either.

The answers are three architecture decisions, recorded as ADRs in the new repository:

| ADR | Decision | State |
|---|---|---|
| ADR-027 | v0 is the extracted `agent_bus`: project settings in `.agent-hub/project.json`, a portable project home, and a task pool computed by the hub | Implemented |
| ADR-028 | Storage split by owner: one SQLite `hub.db` per project for operational data; later, one central database for identity and experience | Stage 1 implemented |
| ADR-029 | Cost-aware routing: prepaid subscription capacity first, metered API second | Implemented |

## The shape of Agent Hub v0

```
  Claude Code        Codex             DeepSeek           you
  (architect,        (developers,      (developers,       (owner)
   reviewer,          tester, manager)  tester, manager)
   auditor)
       \                 |                  /                |
        +---- MCP (stdio), one agent_bus.py server per client      web view / desktop console
                         |                                   |
               <project home>/hub.db  (SQLite, WAL)  <-------+
                         |
            dispatcher: wake rules, task pool, cost-aware routing,
            account outages, claim handoffs, run records
                         |
            headless runs: claude -p / codex exec / dsh --profile headless
```

Each project keeps everything in one folder, the **project home**:

```text
<home>/hub.db         events, read cursors, agent presence, runs, accounts, routing decisions
<home>/config/        dispatch.json (dispatcher), roster.json (desktop roster)
<home>/dispatch/      state.json, runtime.json, dispatch.lock
<home>/runs/          <agent>/*.log  (raw run transcripts)
<home>/logs/          console-*.log
```

Nothing inside the home records its own path, so a home can be copied to another folder or machine. The
project's repository keeps only `.agent-hub/project.json`. That file names the home, the policy file
(`AGENTS.md`), the roles folder, the task pool and the routing.

## A task pool the hub computes

In part 1 every idle agent read the task index itself. Now the project supplies a command that prints the
unfinished tasks with their readiness as JSON, and the hub matches them to agents:

- a task the agent already holds, with nothing recorded as waiting, is work to **resume**;
- a ready task in the agent's lane (`w7Combat`, `w7Hero`, ...) goes to that lane;
- a ready task outside any lane goes to any developer.

Agents see the same list through two new MCP tools, `list_ready_tasks` and `claim_task`. The dispatcher
starts an idle check only when the pool has something for that agent. When it does not, the log says so and
no run starts:

```
idle checks held for codex-senior: nothing ready for it in the task pool or channel
```

## One SQLite file per project

The JSONL log was simple, and it had a limit: every MCP process replayed the whole file at start, and every
write took a lock file that was deleted once it looked stale. A slow writer could lose its lock.

`hub.db` is SQLite in WAL mode. A write is a `BEGIN IMMEDIATE` transaction: check, insert, commit. A process
reads only the rows after the last one it has seen. Events are append-only: triggers refuse an update or a
delete.

Archived threads changed too. A resolved ask whose last status change is over a day old used to be copied
into an archive file. Now it is filtered out of the active views by a query, using the same rule, and its
messages stay searchable.

### Moving a live channel

The Super Morse channel was busy when it moved. The steps:

1. **Rehearse on a copy.** Migrate a copy of the home and compare it with the original. Every event, cursor,
   agent and run matched, and the bus opened in 0.10 s instead of 0.40 s.
2. **Stop everything.** Dispatcher, web view, desktop console, MCP servers.
3. **Migrate.** `agent-hub-store migrate` imported 7,874 events, 24 cursors, 24 agents and 386 run records.
   It merged 801 archived events from 59 files, checked the import with hashes and `PRAGMA integrity_check`,
   and only then moved the old files to a backup folder.
4. **Restart and watch.** After an hour of normal running with no errors and no gaps in the event numbers, I
   deleted the backup.

One detail almost caused trouble. The DeepSeek Harness desktop app restarts its MCP server by itself, and
it did so a minute *before* the migration, so that server opened the old files. I stopped it again right
after the migration, and the copy Harness started next found `hub.db`.

## Cost-aware routing

This is the part that changed how the team behaves. The three clients are paid for differently:

| Client | Billing | Cost of one more run | Limit |
|---|---|---|---|
| DeepSeek | Metered API, per token; cache hits are cheaper than misses | Always above zero | Account balance |
| Codex | Subscription | Zero while the plan has quota | 5-hour window and a weekly cap, per account |
| Claude Code | Subscription | Zero while the plan has quota | 5-hour window and a weekly cap, **shared with my own interactive sessions** |

Prepaid capacity left unused in a 5-hour window expires. So the routing rule is:

1. **Subscription first.** Codex and Claude agents get the work while their plan has quota.
2. **Metered second.** DeepSeek takes over when they are out of quota or busy. It has no spending cap, on
   purpose: it is the safety net that keeps the channel moving.
3. **Limits belong to accounts.** All seven Codex identities share one account, so one limit hit takes the
   whole account offline until the reset time read from the error message.
4. **The hub posts the notice.** An agent that is out of quota cannot post: its client refuses the request
   before the model runs. The hub reads the run's output, so it posts `codex offline (usage limit)` in
   `#general` with the reset time.
5. **Claims follow availability.** A task claimed by an offline agent moves to an available agent with the same
   roles. When the prepaid agent comes back, the claim moves back, but only after the DeepSeek owner has
   been quiet for 30 minutes. The hub cannot see uncommitted edits, so recent activity stands in for
   unfinished work.

Before this, Codex usage limits had produced 55 failed runs out of 72. In the 24 hours of the dashboard
below, 57 usage-limit failures came in the hours before the routing went live. After that came 6 failures,
over 3 outages:
- each outage cost one failed run;
- when the error message gave no reset time, the hub sent one probe every 15 minutes until the account
  answered again.

Every routing decision is recorded with its candidates and scores.

### Switching accounts in the middle of the day

Today I logged Codex into a second account that still had quota. The hub had saved the first account's
reset time, three days away, and a restart of the desktop console did not change that. The console attaches
to the running dispatcher, and the outage is saved in `state.json`.

The fix is a check that runs every 10 seconds while an account is offline. If the client's login file
(`~/.codex/auth.json`, or Claude's credentials) now names a different account, the hub tries one probe run
at once. Codex rewrites that file whenever it refreshes its tokens, so the hub compares a short hash of the
account id, not the file's time. The dispatcher log after the restart, condensed:

```
21:23:36 codex: login changed (~/.codex/auth.json); trying one probe run before the saved reset
21:23:56 codex-reviewer: started
21:24:22 codex-reviewer: finished (exit 0)
         -> #9807 "codex back online"
```

For the cases the hub cannot detect, `agent-hub-store account online codex` asks the running dispatcher to
try again.

## The one step that still needs a person

Some tasks end on a real phone: the port has to behave like the original game on the device. The project's
Pixel is a dedicated test device. The device runner, though, stopped for a person twice per beat: once to
confirm what was on screen, and once to tap EXPORT at the right moment.

So "run the Pixel session" could not be automatic. The project's prompt rule told every dispatched agent not
to touch a phone, and a headless run has no keyboard for the runner's prompts. I changed the rule for one
identity: only `deepseek-tester` may drive a device gate in a dispatched run, through the runner, under the
device lock. The team is now replacing the runner's human prompts with checks it can make itself.

The first unattended attempt stopped at its first beat on an empty inspector export. The runner refused to
file it, and the team recorded the abort as a result, not a failure. That is the behaviour I want from an
unattended device run.

## What a day looks like in numbers

The web view now has a dashboard at `/metrics`. It reads `hub.db` directly and refreshes every minute.

![Hub metrics: tiles, runs per hour by kind, and metered cost per hour with the peak-price hours shaded](../images/agent_hub_metrics_dashboard.jpg)

*The last 24 hours. Top: runs, cost, tokens, task flow, failures and commits. Middle: runs per hour by
kind. Bottom: metered DeepSeek cost per hour; the shaded bands are DeepSeek's peak-price hours.*

| Measure | Last 24 hours |
|---|---|
| Dispatched runs | 851 (about 35 an hour) |
| Agent working time | 70.6 hours, so about three agents busy at any moment |
| Asks opened and closed | 213 opened, 211 closed, 9 reopened |
| Time to claim an ask | median 1.1 min, 90th percentile 16 min |
| Time to answered or resolved | median 10 min, 90th percentile 39 min |
| Tokens | 2.74 billion, 97% of them cache reads |
| Failed runs | 76: 64 on usage limits, 4 killed at the timeout |
| Commits | 350 |
| Metered cost | about US$30, of which US$12.27 in peak hours |

![Per-agent table: runs, hours, tokens, cost, claims, closes, verdicts, posts and commits](../images/agent_hub_metrics_agents.jpg)

*Per agent. The DeepSeek identities carry the load today, because Codex was out of quota for most of the
day and Claude ran out twice. `deepseek-hero-2` is my own attended session, so it has no dispatched runs.*

### Checking the cost against the bill

DeepSeek's pricing page lists `deepseek-flash` at US$0.003 per million cached input tokens, US$0.15 for
fresh input and US$0.60 for output. The weekday peak windows (01:00 to 04:00 and 06:00 to 10:00 UTC) cost
twice as much. The run records hold every token count, so the hub can estimate each run's cost.

All DeepSeek runs since the records start, about 27 hours, come to **about US$34**, roughly ¥245. The
DeepSeek portal showed about ¥250 for the same period. The difference is most likely my own interactive
sessions, which bill the same key but are not dispatched runs.

Where the money went over those 27 hours surprised me:

| | Tokens | Share of tokens | Cost |
|---|---|---|---|
| Cache hits | 2,786 M | 98% | about US$12 |
| Fresh input | 49 M | 2% | about US$10 |
| Output | 18 M | 0.6% | about US$13 |

Cache hits are almost all the tokens and only a third of the cost. The peak window is a bigger lever: for
me it is 12:00 to 15:00 and 17:00 to 21:00 local time, and it accounted for 40% of the day's cost.

### What the numbers say

- **The hub works.** An ask is claimed in about a minute and closed in about ten. A usage limit now costs a
  failed run or two per outage instead of stopping the team.
- **Idle checks are the biggest waste.** 325 of the 851 runs were idle checks, with a median length of 1.2
  minutes, and most ended with "0 ready". They used 610 million tokens and about US$7 of the day's
  DeepSeek cost. When nothing is ready across the whole team, the team still pays to confirm it.
- **Coordination has a price.** The DeepSeek manager ran 168 times and posted 293 messages, with 4 commits.
  Some of that is the job; it is the agent to watch.
- **Output is mostly evidence.** Of the lines added in a day, about 92% were documentation and evidence,
  3,000 were app code and 5,700 were tests. That fits a migration that is verified row by row, but commit
  counts flatter the amount of product that moved.

### Two bugs the dashboard found

Measuring turned up two bugs that nothing else had:

1. **Fresh input was priced at zero.** The cost estimate subtracted cache reads from `input_tokens`, but
   Claude and DeepSeek already report cache reads separately. Nobody noticed, because no prices were
   configured.
2. **A live run looked finished.** The run-log parser trusted any `# exit … at …` line. A manager agent that
   was reading other agents' logs had their exit lines in its own output, so its live run showed −8.5 hours.
   Only the dispatcher's own trailing lines count now.

## Rules that made it work

The habits from part 1 still apply. These are the new ones:

1. **The hub decides what is ready, not each agent.** Readiness computed once is cheaper and more consistent
   than readiness re-derived from prose by every idle agent.
2. **Treat a usage limit as an account state.** One failure per outage, a notice in the channel, and a
   handoff, instead of a retry every five minutes.
3. **Spend prepaid quota first, and keep a metered safety net with no cap.** A stalled channel costs more
   than a day of DeepSeek.
4. **Never move a claim on top of unfinished work.** Handing work back to a recovered agent waits for the
   current owner to go quiet.
5. **Measure before tuning.** The dashboard showed where the money went, and found two bugs on the way.
6. **Keep the person for what needs a person.** A dedicated device can be driven by an agent; a judgement
   about evidence still comes back to the owner.

## Testing the hub itself

The test suite has grown from 41 tests in part 1 to 243. New tests cover:
- migrating a live home, including archive merging and refusing to start while a dispatcher runs;
- concurrent writers in separate processes;
- the SQL archive filter matching the old rule;
- account outages, re-probes after a login change, and claim handoffs in both directions;
- peak and off-peak pricing;
- the run-log parser;
- the dashboard's data and web routes.

The dashboard was also checked in a browser at desktop and narrow widths, in light and dark mode.

## Impact and Next Steps

What changed in practice:

- The team runs unattended through the day and through usage limits. I check the channel, answer the few
  owner questions, and look at the dashboard.
- Switching an agent's account is no longer a restart. Logging in is enough.
- I know what a day costs, and which part of it is waste.

Limits I still live with:

- Subscription quota is estimated, not read: neither client reports what is left.
- Idle checks still cost money when the whole team is waiting on me, as it was tonight for a phone session.
- Device runs still need a person until the runner's checks are automatic.

Next steps:

- **Cut idle-check spend** when the task pool is empty for everyone.
- **The central database (ADR-028 stage 2).** A stable identity for each agent across projects, so experience
  can be credited to an agent rather than to one project's seat names.
- **The original goal.** Can an agent improve on unseen tasks by retrieving validated experience from earlier
  work? The run records and usage meter built this week are the starting data for that experiment.

## Conclusion

Part 1 turned six sessions into a team. Part 2 is about letting that team run without me: a store that
cannot lose a message, a task pool the hub computes, routing that spends prepaid quota first and survives
running out of it, and a dashboard that says what all of it costs. The surprise was how much measuring
mattered. The dashboard was meant to show performance; it also found the waste and two bugs.

These numbers prove coordination at scale, not better software outcomes. Fast claims and closed asks show the
team runs; they do not show the code is better than one careful agent would write. The coordination,
routing, storage and observability layers are now real and useful. So the open question is no longer "can
agent teams run?" but "can validated experience make those teams measurably better?" That is the next
experiment.

---

## Related Posts

- [Agent Channels: Running an Architect, Developers and Reviewers as One Team](/posts/2026/11/agent-channels-managing-multiple-coding-agents/): part 1, the channel, roles, asks and dispatcher this post builds on
- [Super Morse: Eight Skins, Sixteen Filters, and the Process Behind Them](/posts/2026/10/super-morse-reskin-process-skins-and-filters/): the audit-gated process the agents follow

## Connect

- **GitHub**: [guidebee](https://github.com/guidebee)
- **LinkedIn**: [James Shen](https://www.linkedin.com/in/guidebee/)

---

*This is post #47 in the Super Morse series. Prepaid quota first, a metered safety net, and a dashboard that says what a day of agents costs.*
