---
layout: single
title: "Inside the Company: How 25 AI Agents Plan, Build, Review and Audit Each Other"
date: 2026-11-19
permalink: /posts/2026/11/inside-the-company-how-ai-agents-work-together/
categories:
  - blog
tags:
  - ai-tools
  - agents
  - claude-code
  - codex
  - deepseek
  - agent-hub
  - code-review
  - workflow
  - side-project
excerpt: "Part 3 of Agent Channels. The company site now has an 'Inside the company' view of the real Agent Hub team. Here is one afternoon from it: an independent reviewer failing a slice, a developer fixing it, an architect ruling on the next one, work handed over when a plan ran out, an auditor checking it all, and the team finding a bug in its own dispatcher."
---

Post #48 in the [Super Morse](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/)
series, and part 3 of [Agent Channels](/posts/2026/11/agent-channels-managing-multiple-coding-agents/).
[Part 2](/posts/2026/11/agent-hub-v0-coordination-hub-for-ai-coding-agents/) was about the plumbing: one
SQLite file, a task pool and cost-aware routing. This part is about what that plumbing carries: how the
agents actually work **with each other**, using one afternoon taken from the live
[Inside the company](https://jpeducationmaster.com/operations) view of my consultancy site.

![The Inside the company overview: 25 AI professionals, 1,064 tasks, 20,532 messages and 6,537 commits](../images/inside_company_overview.jpg)

## TL;DR

- **The site shows the real team, read-only.** An exporter pushes the hub's channels, claims, handoffs and
  metrics to the website every five minutes. Nobody on the site can post, and no agent can be started from it.
- **Nobody marks their own homework.** Every slice is built by one agent and reviewed by a different one. In
  the afternoon below, a reviewer failed a slice, the developer fixed it, and a second reviewer picked it up.
- **Work moves between agents without me.** When a subscription plan ran out the hub posted a handoff note
  with a checkpoint and the next agent carried on from the previous agent's commit.
- **An auditor reads the whole team every hour** and writes a verdict with numbers, including which "stalled"
  items are really declared waits.
- **The team found a bug in its own dispatcher** through its cost guard. The fix is written and merged. It is
  waiting for me to deploy it.

## The cast

The [team page](https://jpeducationmaster.com/team) lists 25 named AI professionals. The names are labels for
roles; behind them are Claude, Codex and DeepSeek accounts that the hub assigns work to.

![The team page: named AI professionals with roles such as project manager, reviewer, auditor and developer](../images/inside_company_team.jpg)

The roles that matter for this story:

| Role | Does | Examples |
|------|------|----------|
| Project managers | Turn my goals into tasks, route work, answer "who is free?" | Luca, Petra |
| Architects | Rule on design questions before code is written | Nora (acting architect), Hugo |
| Developers | Claim a task, build it, push a commit with a receipt | Gia, Hana, Milo, Wade |
| Reviewers and testers | Review a commit they did not write; run the tests | Vera, Ravi, Kira, Tess, Sam |
| Auditors | Check the whole team against its own rules, hourly | Iris, Ines |
| The owner | Me. Deploys, logs in to accounts, makes the calls nobody else may | James |

The rule that holds it together is short: **an agent that authored something cannot review it.** When Nora
ruled on a design question, the team log says so in as many words: "I authored this ruling, so I am not its
reviewer." The hub does not enforce it by magic; the agents' written rules do, and the auditor checks.

## One afternoon on the Super Morse engine

The task is W7-12, removing the legacy rule engine from Super Morse. It is split into slices. The channel
shows how one slice, **2a**, moved through the team.

![The Team chat view: a developer's fix receipt, a task claimed by a reviewer, and the other open conversations](../images/inside_company_team_chat.jpg)

### 1. A review that fails

Gia built slice 2a: a replay recorder for the game's explorer. Then **Vera**, a different agent, reviewed the
commit and wrote this verdict:

- **Pass:** the bounded mechanism wiring, the unchanged fingerprint extraction, and the existing tests
  (9 of 9 green).
- **Fail, medium (F1):** the recorder keeps references to mutable command objects. If an input object is
  reused later, the recorded history changes behind the recorder's back. The tests were green only because
  none of them ever mutated a recorded command.
- **Unknown, out of scope:** full explorer-state equivalence. She marked it unknown rather than guessing.

The finding was posted as a separate message addressed to Gia, with the exact failing case and the regression
test it wanted. Green tests were not accepted as proof; the reviewer wrote a case the tests did not cover.

### 2. A fix with a receipt

Gia's reply came back in the same thread: the `prefix` field made private so no caller can mutate it, a
regression test for input reuse, and a receipt: build tag, three test classes (5/5, 3/3, 5/5, 0 failed, 0
skipped), commit hashes for the code and for the record. A **receipt** is the unit of trust in this team. A
claim of "fixed" without one is not a fix. Then **Ravi** claimed the re-review task, which is the last line
in the screenshot above.

### 3. The next slice, and a ruling

Slice 2b-i got the same treatment. Ravi reviewed it and found two medium defects: a mismatch between the
production tick clock and the explorer's counter (reproduced at scene 36: 7636 against 7640), and a
dropped quality flag when a search hits its cap. Again the tests passed and did not detect either.

Before the next slice (2b-ii) could start, a design question needed an answer. **Nora**, acting as architect,
wrote a ruling with binding conditions: key the rows by spawn order within the controller, not by raw
index, fail loudly on a mismatch, record the cap and mark the result "approximate" rather than "exact", and
fall back to a smaller reduction only for a named site if spawn order cannot be proven. She also corrected
herself: an earlier commit of hers rested on a wrong premise, found by Gia's reading of the code, and the
ruling says so.

![Tasks and scheduling: resolved, in review and in progress columns, with owners, dependencies and a link to each task's conversation](../images/inside_company_tasks.jpg)

## When the plan runs out

Part 2 described cost-aware routing. This afternoon is what it looks like from inside.

- The **Claude** accounts reached their weekly close: 98% and 97% used, reset on 22 November. Six agents on
  that lane were out for about three days.
- A **Codex** login went missing, which took five more agents offline until I signed in again.
- The hub posted **cover**: Nora (architect and reviewer) and Iris (auditor) kept running on DeepSeek. By
  design the specialists have no cover. They wait.
- When Codex capacity came back, the hub posted an automatic handoff from one agent to another. The note
  carries a checkpoint number, what was completed, the exact next action, and the worktree and commit the
  new agent must build on: "Preserve the previous owner's work: build on its commit."

One of those handoffs is for a task that **must not** proceed. W6-12 is blocked until W7-16 records a passing,
verified cutover (rule D7, which is mine). The handoff note says the work is prohibited until then, and the
agent that received it did what the note said: it waited, and wrote down what it would check when the
dependency cleared.

## The auditor

Every hour **Iris** runs an audit script over the last 60 minutes and posts a verdict. The one at 11:45Z
reads, in part: 26 runs (6 idle, 0 failed), 12 commits (8 docs, 4 test), nothing unpushed, 17 tasks (5
accepted, 1 blocked, 3 in progress, 6 parked, 2 planned), and the verdict: **AMBER (low)**.

What makes it useful is what it refuses to call a problem:

- A task untouched for nine hours is "a held wait, not a stall", because it depends on another task that is
  not finished.
- A developer showing as "running" is productive if commits are landing; the auditor checks the commits.
- The Claude lane being out is "a weekly close, not a five-hour dip", with the quota file and the console
  log as evidence.

And it flags what is real: a developer's run that started on three unread wakes, finished cleanly and read
none of them.

## The team found its own bug

The **metered cost guard** posts a message whenever a metered run costs more than US$0.10. Mostly those are
a developer doing a legitimate slice. Luca, the project manager, inspected one that cost US$0.23 and found
something else: the dispatcher had started a full run for a task that a reviewer had already closed as
"won't do". The run did produce correct work in the end, so it was not a loss. The cost was the dispatcher's
bug, not the agent's.

Related: the dispatcher keeps re-delivering handled owner mail. The fix is merged, and an agent (Wade) has
claimed the follow-up task, "consume handled dispatcher wake batches and cap retries".

The merged fix is not running. The live copy of Agent Hub is a different checkout, and deploying it is an
owner action. Every agent knows this and has written it down, so none of them touched it. That is the safety
rule working and the throughput cost of it, in the same message thread.

## What the numbers say

![Metrics: 635 runs in 24 hours, 5 failed (0.8%), 153 commits, US$14.39, 1.27 billion tokens, 96.4% cache share](../images/inside_company_metrics.jpg)

Across all history: 25 professionals, 1,064 tasks, 1,670 claims, 20,532 messages, 669 review findings and
6,537 commits. In the last 24 hours: 635 runs, 5 failed (0.8%), 153 commits, 35.3 agent-hours, and an
estimated US$14.39 including a share of the plan fees. About 96% of the 1.27 billion tokens came from the
prompt cache.

Two of those numbers matter more than the rest. **669 review findings** against 6,537 commits says that
review is not a rubber stamp. And **0.8% failed runs** says the failures that matter are not the ones that
crash.

## Limits, honestly

- **The site is a read-only view of a bounded export.** Conversations show the last 24 hours (219 of 4,034
  total threads); the totals and metrics cover everything. Run logs, account names and credentials are
  never exported, and secret-shaped strings and email addresses are masked.
- **Messages are what the agents say, not independent evidence.** That is why a review has to be written by
  someone else and why receipts name commits and test counts.
- **None of this proves the software is better** than one careful agent would write. It shows the team runs,
  checks itself and stops where the rules say stop.
- **The team still needs an owner** for deploys, logins and any rule change. This afternoon, the dispatcher
  fix and two account sign-ins were waiting for me.

## Conclusion

The part of an agent team that does not show up in a demo is the part that makes it trustworthy: a reviewer
with the standing to say fail, a developer who answers with a receipt, an architect who corrects herself in
writing, a handoff that carries a checkpoint, an auditor who separates waiting from stalling, and a cost
guard that catches the team's own mistakes. None of those needs a smarter model. They need roles, records
and rules, and a place to read them. The [Inside the company](https://jpeducationmaster.com/operations)
view is that place, open to anyone.

---

## Related Posts

- [Agent Hub v0: A Day of Claude, Codex and DeepSeek Working as One Team](/posts/2026/11/agent-hub-v0-coordination-hub-for-ai-coding-agents/): part 2, the store, task pool, routing and metrics behind this view
- [Agent Channels: Running an Architect, Developers and Reviewers as One Team](/posts/2026/11/agent-channels-managing-multiple-coding-agents/): part 1, the channel, roles and asks

## Connect

- **GitHub**: [guidebee](https://github.com/guidebee)
- **LinkedIn**: [James Shen](https://www.linkedin.com/in/guidebee/)

---

*This is post #48 in the Super Morse series. Authors do not review their own work, handoffs carry a checkpoint, and an auditor reads the team every hour.*
