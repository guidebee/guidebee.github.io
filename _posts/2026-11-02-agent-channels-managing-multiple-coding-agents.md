---
layout: single
title: "Agent Channels: Running an Architect, Developers and Reviewers as One Team"
date: 2026-11-02
permalink: /posts/2026/11/agent-channels-managing-multiple-coding-agents/
categories:
  - blog
tags:
  - ai-tools
  - claude-code
  - codex
  - mcp
  - agents
  - code-review
  - workflow
  - side-project
excerpt: "How the Super Morse project coordinates Claude Code, Codex, Copilot, OpenCode, Kimi and DeepSeek through agent-bus: a Teams-style channel with threads, roles, asks, verdicts and a dispatcher that wakes agents, so one person can manage an architect, developers, testers and independent reviewers."
---

Post #46 in the [Super Morse](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/)
series. The [previous post](/posts/2026/10/super-morse-reskin-process-skins-and-filters/) was about art.
This one is about the team that does the work: several coding agents, and the chat channel that keeps
them from stepping on each other.

![Agent channels: seven agents with roles around the agent-bus channel](../images/agent_channels_cover.jpg)

![The #reviews channel: a review request, a claim, and a verdict](../images/agent_channel_messages.jpg)

*The `#reviews` channel. `codex-senior` asks for a spot-check, `claude-review1` claims it and replies with a
verdict. The request is marked `resolved`.*

## TL;DR

- **One chat channel for every agent.** `agent-bus` is a single Python file (standard library only) that
  gives Claude Code, Codex, OpenCode, Copilot, Kimi Code and DeepSeek a shared, Teams-style space. Agents
  reach it through MCP; I watch and post from a local web page.
- **Roles, not products.** An agent is a `solution-architect`, `senior-developer`, `junior-developer`,
  `tester` or `reviewer`. `@reviewer` reaches everyone with that role, and each role has a brief that is
  loaded when the agent connects.
- **Asks have a life cycle.** `review-request`, `help`, `question`, `proposal` and `handoff` posts move
  from `open` to `in-progress` (claimed) to `answered` to `resolved`. Nothing is lost in a scrollback.
- **A dispatcher wakes idle agents.** MCP cannot wake a sleeping agent, so a small watcher starts the
  right client in headless mode when the channel has work for it, with a grace period, a per-agent hourly
  cap and a timeout.

## The problem: agents are good alone and bad together

With one agent the loop is simple: ask, wait, read, commit. With six it falls apart:

- Two agents edit the same files.
- A review is requested by pasting a diff into another terminal, and the answer is pasted back.
- The agent that wrote a change also reviews it, so the same blind spots approve it.
- Nobody knows which questions are still open.
- I am the message bus, copying text between windows all day.

What I wanted was what a human team already has: a place to ask, a way to claim work, and a record of who
said what. So the project got one.

## What agent-bus is

```
  Claude Code     Codex      Copilot     OpenCode     Kimi      DeepSeek
  (architect)  (senior dev) (junior dev) (reviewer)  (tester)  (senior dev)
       \            |           |           |           |          /
        \           |           |           |           |         /
         +-------- MCP (stdio), one agent_bus.py server per client -------+
                                      |
                          messages.jsonl  (append-only)
                                      |
                  +-------------------+-------------------+
                  |                                       |
           web page (you)                        dispatcher (wakes agents)
        127.0.0.1:8765, post as `owner`          headless client runs
```

Every client runs its own copy of the same server, pointed at the **same absolute** `AGENT_BUS_DIR`. The
only shared state is one append-only log, `messages.jsonl`. Each line is an event: a post, reply, reaction,
status change or channel. It is never rewritten, a line torn by a crash is skipped, and a hundred
concurrent posts from four writers is one of the tests.

That is the whole architecture. No daemon, no database, no install. Python 3.8 or later.

## Channels

| Channel | Use |
|---|---|
| `#general` | Announcements, daily plans, anything else |
| `#reviews` | `review-request` posts; reviewers reply with `finding` or `verdict` |
| `#help` | `help` and `question` posts when an agent is stuck; anyone may answer |
| `#decisions` | `proposal` and `question` posts for the owner or the architect |
| new name | Created on first use, such as `#w7combat` or `#w7hero` for a work lane |

The left panel of the screenshot shows the result: `decisions`, `general`, `help`, `reviews`, and one
channel per work lane (`w7combat`, `w7hero`, `w7walkers`). Lane channels keep the traffic of one slice of
work in one place, so the people and agents who are not on that slice do not have to read it. The numbers
beside `Open asks` and `reviews` are the asks nobody has finished yet.

## Roles: who the agent is

Each agent registers with a name, roles and the channels it follows. Names must be unique, because read
positions are stored per name.

| Agent | Roles | Follows |
|---|---|---|
| `opus-architect` (Claude Code, Opus 5.5) | `solution-architect` | general, decisions, reviews |
| `claude-review` (Claude Code) | `reviewer` | general, reviews, help |
| `codex-senior` (Codex) | `senior-developer`, `w7Combat` | general, help, w7combat |
| `deepseek-hero` (DeepSeek) | `senior-developer`, `w7Hero` | general, help |
| `copilot-junior` (Copilot) | `junior-developer` | general, help |
| `kimi-tester` (Kimi Code) | `tester` | general, reviews, help |

Roles do two jobs:

- **Routing.** `@codex-senior` reaches one agent. `@reviewer` or `@w7Combat` reaches everyone holding that
  role. `@all` reaches everyone, and each agent acknowledges with a short reply or a `+1`.
- **Briefing.** If `docs/agents/roles/<role>.md` exists, the agent receives it when it connects, and can
  reread it with `get_role_brief`. A fresh session therefore knows what a reviewer is supposed to check
  without me re-explaining it.

Registering one is a single command. For a Claude Code reviewer:

```bash
claude mcp add agent-bus --scope local \
  -e AGENT_NAME=claude-review -e AGENT_ROLES=reviewer \
  -e AGENT_CHANNELS=general,reviews,help \
  -e AGENT_BUS_DIR=C:/Users/<you>/.agent-bus/super-morse \
  -- py -3 C:/workspace/super-morse/tools/agent_bus/agent_bus.py serve
```

The `local` scope keeps the identity out of git. Codex uses `~/.codex/config.toml`, OpenCode and Copilot use
JSON, and DeepSeek is registered in whichever client runs it. The command and environment are the same
everywhere.

### Two agents, one product

The same product can hold two identities. Claude Code passes its environment to the MCP server, so two
Claude sessions can share a checkout: register the server with only the bus directory, keep the default
identity in a git-ignored `.agent-identity.json`, and start the second one from a PowerShell window that
sets the variables:

```powershell
$env:AGENT_NAME='opus-architect'; $env:AGENT_ROLES='solution-architect'
claude --model claude-opus-5-5
```

That is how one machine runs an architect on Opus 5.5 next to a reviewer on the default model. In the
screenshot, `claude-review` and `claude-review1` are two such reviewers.

## Asks: work that cannot be forgotten

A chat that only scrolls loses requests. Agent-bus tracks the first five post kinds as **asks**:

```
open  ->  in-progress (claimed, with the claimer's name)  ->  answered  ->  resolved
                                                                      \->  wont-do
```

- Anyone can claim an ask or mark it answered.
- Only the poster or the owner can resolve it, close it as `wont-do`, or reopen it.
- Reply kinds are `reply`, `suggestion`, `recommendation`, `answer`, `verdict` and `finding`.
- Reactions are `+1`, `-1`, `eyes` (looking at it), `check` (done), `thanks` and `question`.

An open ask unclaimed for more than 24 hours is highlighted and sorted to the top of **Open asks**.
Resolved asks are archived after 24 hours, one JSONL file per thread, so the active channel stays short.

## Walkthrough: one review, start to finish

The screenshot is a real thread from today. Here is what happened.

**1. The request (`#310`).** `codex-senior`, the senior developer on the combat lane, finished a set of
checklist rows and posted a `review-request`:

> W7-07 D22 checklist rows 65, 66, 69, 73, 150 and 209 at `51127a8d0`. @reviewer Please independently
> spot-check the newly explicit rows. Each row maps a named production positive, an edited/nonmatch case,
> a capability/rollback case and a retry test.

Three things make it a good request, and they are the pattern worth copying:

- **A pinned commit.** The reviewer checks exactly what was submitted, not whatever the branch is now.
- **The exact command**, with the result already stated (106 tests, 0 failures, 0 skips) and the
  consistency checks that passed.
- **A boundary.** It says which cells to flag ("any cell the existing test or cited review does not
  actually prove") and what is out of scope (resource 160 is under a separate request, `#308`). Refs are
  attached as chips so the thread can be found later by task ID or commit with `search`.

**2. The claim.** The `@reviewer` mention reaches every agent holding that role. `claude-review1` marks the
ask `in-progress`, and the thread shows *"claude-review1 marked it in-progress"*. The other reviewer sees
it is taken and does something else.

**3. The work.** The reviewer does not trust the numbers in the request. It checks out a clean worktree at
the pin, builds in isolation and reruns the tests.

**4. The verdict (`#365`).** The reply is a `verdict` with evidence:

- **Run:** clean worktree at the pin, own isolated build, 8 classes, 96 tests, 0 failed, 0 skipped, and all
  37 test names cited in the six rows exist.
- **Per row:** pass for 65, 66, 69, 73, 150 and 209, with an honest limit: for five of the rows it checked
  that each cited test *exists and runs green*, not each assertion.
- **Resource 73, read in full:** the nonmatch, edited exit height and retry cases, each tied to the state
  it sets (`S44`, `S49`, the local `58`).
- **Evidence file:** the verdict is written to a document under `docs/engine2/evidence/`.

**5. Resolution.** The poster marks the thread `resolved`. The decision to record a status transition
stays with the owner, which is why the original request says "no record transition is requested here."

Note the numbers. The reviewer ran 96 tests where the author reported 106: it pinned a different commit
(`7014d5956`, which includes a later transfer-wiring change) and its own isolated run. Differences like that
are exactly what an independent review is for, and the thread keeps both figures visible.

## Why the reviewer should be a different vendor

A Claude reviewer approving a Claude author's change shares the author's habits. The project's rule (D24):

> For architect items, prefer a reviewer on another vendor's model.

So the routing table matters as much as the model choice:

| Role | Model | Why |
|---|---|---|
| solution architect | Opus 5.5, or Codex's strongest model at high effort | Cross-lane trade-offs and long context; few, high-value calls |
| reviewer | A strong model from a **different vendor** than the author | Independent eyes catch different mistakes |
| senior developer | A strong model (Opus 5.5, Sonnet 5.5, Codex) | Shared host code and hard slices |
| junior developer | An economical model (Sonnet 5.5, Haiku 5.5, DeepSeek, Kimi, Copilot) | Well-specified rows and bookkeeping |
| tester | An economical model | Running suites and route scripts, precise reporting |

The model is chosen in the client, not in the channel. `claude --model claude-opus-5-5`, a Codex
`--profile architect`, or an OpenCode `model` setting each work, and the channel does not care.

**Two architects** is a deliberate option. Run Opus 5.5 and Codex both as `solution-architect`: one
`@solution-architect` mention reaches both, and each replies with a `recommendation`. Keep exactly one of
them recording decisions in `#decisions`, and let the owner choose between conflicting recommendations,
otherwise the decision records disagree.

## The dispatcher: waking idle agents

MCP has a limit: a server cannot wake an idle agent. An agent sees new activity only when it calls
`get_activity` or `wait_for_activity`. The project `AGENTS.md` tells every agent to check at the start of a
session, after every push and before going idle, but a session that is not running reads nothing.

`dispatcher.py` closes the gap. It watches the log and, when a post concerns an agent, runs that agent's
client once, headless, with a short prompt: call `get_activity`, handle the listed messages, reply in the
threads, and finish.

```bash
py -3 tools/agent_bus/agent_bus.py --dir C:/Users/<you>/.agent-bus/super-morse dispatch
```

The message text never goes through a command line. The prompt only says to read the agent's own feed, so
the details come through its bus identity.

**What wakes an agent:**

- a post or reply that mentions it, one of its roles, or `@all`;
- a new ask or handoff in a channel it follows;
- a reply in a thread it started or claimed;
- a status change on a thread it started, so it can resolve it.

**What does not:** reactions, plain notes, announcements and replies in a thread the agent only took part
in. Without that last rule a busy thread would set off a chain of runs.

**Safety rules**, because an unattended agent that can start itself is a bill and a risk:

| Rule | Value |
|---|---|
| One run at a time per agent | Activity during a run gets one follow-up run if still unread |
| Grace period | 20 s: if a live session reads the activity first, no run starts |
| Runs per hour | 4 per agent |
| Timeout | 45 minutes, and the whole process tree is killed |
| Phone access | `platform-tools` removed from `PATH`; the prompt forbids adb |
| Instances | One dispatcher per bus directory, enforced with an OS file lock |

Each client has its own headless command in `dispatch.json`: `claude -p ... --allowedTools mcp__agent-bus`,
`codex exec -C <repo> ...`, `opencode run --standalone ...`, `copilot -p ... --allow-tool agent-bus -s`
and `kimi -p ...`. Two details are worth knowing. Codex needs
`default_tools_approval_mode = "approve"` or every bus call is refused. And Kimi's prompt mode approves
file writes without asking (checked 2026-11-01), so it earns less trust than the others. DeepSeek runs
through OpenCode with an override config, because its own harness asked for a separate sign-in in headless
mode.

A dispatched run is a fresh session. It knows only `AGENTS.md`, its role brief and the thread, so **put
task IDs, files and the expected result in the post**. The review request above worked because it did.

## Your view: the owner's web page

```bash
py -3 tools/agent_bus/agent_bus.py --dir C:/Users/<you>/.agent-bus/super-morse web
```

The page at `127.0.0.1:8765` is what the screenshot shows: channels with open-ask counts on the left,
threads with replies, reactions and status history in the middle, and the **Agents** panel at the bottom
left with each agent's role and when it was last seen. It refreshes every 3 seconds, follows the system
light or dark theme, and works at phone width. The composer at the bottom posts as `owner`, with a kind
selector and `@name` or `@role` mentions.

An agent leaves the active roster after 30 minutes without an MCP tool call, but its follows and history
stay, and it returns when it next connects.

The page listens on `127.0.0.1` only. It refuses other host names and cross-site posts, so another site
open in the browser cannot post through it.

There are also command-line equivalents for scripting:

```bash
py -3 tools/agent_bus/agent_bus.py post --channel general \
  --subject "Priorities today" --body "W7-07 S5 first, then W7-13. @all"
py -3 tools/agent_bus/agent_bus.py reply --thread 12 --kind answer --body "Approved; record it as D29."
```

## Keeping the record

The log is permanent, and it can be exported to Markdown and committed beside the code:

```bash
py -3 tools/agent_bus/agent_bus.py export \
  --out docs/engine2/agent-log/2026-11-01.md --since 2026-11-01T00:00:00Z
```

Verdicts still belong in the evidence files and the task results. The exported log shows how they were
reached, which is what I want when a decision looks wrong two weeks later.

## Rules that made it work

These are the habits that matter more than the tool:

1. **Pin the commit.** Every review request names a hash. "Latest" is not a thing a reviewer can verify.
2. **Put the check in the post.** Command, expected result, and what to flag. A fresh session cannot guess.
3. **Claim before you work.** `in-progress` stops two reviewers doing the same job.
4. **Different vendor for the reviewer.** At least for architect items and shared host code.
5. **One recorder of decisions.** Two architects may advise; one writes `#decisions`.
6. **Stage your own paths.** Agents in one checkout run `git add <paths>`, never `git add -A`, and
   coordinate edits in the channel.
7. **Cheap models for specified work.** Junior and tester roles take rows with clear acceptance checks.
8. **Reviewers state their limits.** "Exists and runs green, not each assertion" is more useful than a bare
   pass.

## Testing the bus itself

The tool has 41 tests (`py -3 -m unittest tools/agent_bus/test_agent_bus.py tools/agent_bus/test_dispatcher.py`):
24 for the bus and 17 for the dispatcher. The dispatcher tests use a fake agent client and cover wake rules,
the grace period, one run at a time, the hourly cap, the timeout kill, the `PATH` filter, restart and the
single-instance lock. The bus tests cover the status rules, mention parsing, a three-agent collaboration over
the MCP stdio protocol, the web API's cross-site protections and export. It was also exercised with the
official MCP Python client, and the page was checked in Chromium in light, dark and phone layouts.

## Impact and Next Steps

What changed in practice:

- I no longer copy text between terminals. I read the channel and answer the few things that need an owner.
- Reviews are independent and traceable: who asked, who claimed, what was run, what was found.
- Open work is visible in one list, and stale asks surface by themselves.

Limits I still live with:

- MCP cannot push, so an agent without a dispatcher run only reads on its next check.
- Headless permission modes differ by client, so each one needs a one-time setup.
- Role briefs are only as good as I write them.

Next steps are to tune each role brief from the verdicts that came back wrong, and to try a second
architect on a different vendor for the next big design decision.

## Conclusion

Managing several agents is a team problem, not a prompt problem. A shared channel, named roles, asks that
must be claimed and closed, reviewers from another vendor, and a small dispatcher were enough to turn six
separate sessions into something I can manage from one browser tab.

---

## Related Posts

- [Super Morse: Eight Skins, Sixteen Filters, and the Process Behind Them](/posts/2026/10/super-morse-reskin-process-skins-and-filters/): the previous post, with the audit-gated process these agents also follow
- [Super Morse: Game Architecture, Engine 2 Integration and Debug Tools](/posts/2026/10/super-morse-game-architecture-engine2-economy-debug-panel/): the Engine 2 work behind the W7 slices reviewed here

## Connect

- **GitHub**: [guidebee](https://github.com/guidebee)
- **LinkedIn**: [James Shen](https://www.linkedin.com/in/guidebee/)

---

*This is post #46 in the Super Morse series. One channel, named roles, claimed asks, and reviewers who do not share the author's model.*
