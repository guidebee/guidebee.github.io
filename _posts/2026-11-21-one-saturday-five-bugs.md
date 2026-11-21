---
layout: single
title: "One Saturday, Five Bugs: Numbers That Looked Fine and Were Wrong"
date: 2026-11-21
permalink: /posts/2026/11/one-saturday-five-bugs/
categories:
  - blog
tags:
  - ai-tools
  - agents
  - agent-hub
  - debugging
  - testing
  - jobseeker
  - workflow
  - side-project
excerpt: "Part 4 of Agent Channels. On one Saturday the team found five bugs: a rollback caused by a closed port, a scoring agent that never read the resume, a retired model hiding inside an HTTP 200, two pipelines running at once, and a dashboard that counted 61 agents in a team of 30. In every case the numbers looked plausible. Each was found by reading what was really there."
cover_prompt: "Editorial illustration, flat vector style, warm paper tones: a tidy row of bright green check marks and round dial gauges on a wall, while a person with a magnifying glass looks behind one panel and finds a tangle of mismatched gears and torn paper. No text, no letters, no numbers."
cover_alt: "A neat wall of green check marks and dials, with a person lifting one panel with a magnifying glass to reveal tangled gears behind it."
---

![A neat wall of green check marks and dials, with a person lifting one panel with a magnifying glass to reveal tangled gears behind it.](../images/one-saturday-five-bugs-cover.jpg)

Post #49 in the [Super Morse](/posts/2026/09/ai-assisted-platformer-reverse-engineering-super-morse/)
series, and part 4 of [Agent Channels](/posts/2026/11/agent-channels-managing-multiple-coding-agents/).
[Part 3](/posts/2026/11/inside-the-company-how-ai-agents-work-together/) showed how the team checks
itself. This part is what that looked like on one Saturday, 21 November, when we found five bugs.
They have one thing in common.

## TL;DR

- **Do not stub the thing that differs between machines.** A rollback worked as designed; the bug was how a
  closed port behaves on one Windows machine.
- **A plausible score is not a checked score.** The scoring agent read raw Word XML instead of the resume.
- **An error can arrive inside an HTTP 200.** All 128 model calls failed and the run exited with success.
- **If an expensive job can be retried, build the guard into the tool.** An agent started a 30-to-60-minute
  run twice.
- **A sum larger than the whole is a visible bug.** A dashboard showed 61 agents for a team of 30.

The line that repeats: **a number that looks fine is not a number that was checked.**

## The five bugs

| # | What it looked like | What it really was | How it was found | The fix |
|---|---|---|---|---|
| 1 | A release train rolled back: "no fresh dispatcher heartbeat" | The new console found no free web port, because a closed local port does not refuse on this machine, it goes unanswered | Running the new startup with the process launcher stubbed out | Decide "free" by binding the port; log startup errors |
| 2 | A dry run whose counts all matched the logs | The resume loader returned raw XML, cut to 2,000 characters | Reading further after the counts checked out | Loader returns plain text; 12,000-character limit; skip thin listings; `--rescore` |
| 3 | A first real run that printed "Below threshold: 128" and exited successfully | A retired model; the error was inside an HTTP 200 | The logs, then probing five model names | Show the provider's message; count failures apart from scores; stop and fail loudly |
| 4 | Two pipelines scoring the same listings | A tool timeout of about 10 minutes against a 30-to-60-minute job | A process list and the logs | A per-person lock; a second start exits with code 3 |
| 5 | "Agents by state" adding up to 61 | Old entries never removed from a trailing-24-hour set | The owner noticing a sum larger than the roster of 30 | Replace the set as a whole; define each number on the page |

## 1. The release that rolled back because a closed port does not say no

At 06:58 UTC a release train started to move the live team to a new version. At 07:00 it reported "no fresh
dispatcher heartbeat within 90 s" and rolled back by itself. The team stayed healthy on the previous version
and the train ended with 0 runs in flight. The rollback did its job.

The cause was in the new console. It picks a port for its web server by scanning 50 ports, 8765 to 8814, and
counts a port as free only if connecting is actively *refused*. We reproduced the startup with the process
launcher stubbed out and got `RuntimeError: no free web port in 8765-8814`, after 49.6 seconds. On this
Windows machine a connection to a closed local port is not refused. It simply goes unanswered. A raw socket
test in PowerShell waited the full 1.5 seconds it was allowed and got no refusal. The probe gives up after 1
second per port, so all 50 ports looked taken.

Two things hid it. The train starts the console with `pythonw`, which has no console, so the error vanished
and the console died before it launched the dispatcher. No log, no heartbeat: the train could only say "no
heartbeat". And the tests had replaced the port-probe function with a stub, so the real probe never ran.

The fix decides "free" by trying to bind the port, which is instant and does not depend on how the network
answers. New tests run the real probe, including a closed port that only times out. A second change makes the
console write any startup error to a log and makes the train name that error in its rollback message. The
second train started at 07:14 and finished at 07:17.

Keep the safety net, and make silent processes talk. The code is in the team's private Agent Hub repository.

## 2. The scoring agent that never saw the resume

[Jobseeker](https://github.com/guidebee/jobseeker) scans three job boards each week, scores each listing
against a resume from 0 to 100 with a language model, and reports the best. Two of the boards block
automated scanning (HTTP 403). The agent respected the block and skipped them.

The Jobseeker agent, Kai, ran a dry run and reported its counts. I checked every figure against the logs and the
database, and they matched. Reading further, we found that the
resume loader returned the **raw XML of the Word document**, not its text, and the prompt cut it to its **first
2,000 characters**.

Measured with the project's own loader, resume A was 21,723 characters of XML (657 tags) for 8,676
characters of real text. Resume B was 69,046 characters of XML (2,046 tags) for 9,938 characters of text.

Five listings scored 0. In every one the model said the resume was only "document formatting XML". All five
were on resume B, which was used for 55 of the 132 scored listings and averaged 45.9, against 64.2 for
resume A (77 listings).

Kai's own report guessed the zeros came from missing job descriptions and said "I did not confirm the cause".
The check showed the five zero-scored listings had 1,317 to 7,196 characters of description each. The guess was
wrong; the resume was the cause. The agent's report was honest. It still needed checking.

There was a second flaw in the same data. When LinkedIn rate-limited a description fetch (HTTP 429, 28 times
in one run) the listing had no description and was still scored. Four of the 44 shortlisted listings had 0
characters of description and scored 72 to 82.

The fix, in [PR #10](https://github.com/guidebee/jobseeker/pull/10): the loader returns plain text, the prompt allows
12,000 characters of resume, a listing with under 200 characters of description is skipped and scored later,
and `analyze --rescore` redoes everything. The rescore of all 145 listings took 44 minutes: 127 scored, 18
skipped, no zero scores. Resume B's average went from 45.9 to 76.7. Over the 116 listings scored both times
the average rose by 16.3 points (91 up, 20 down, 5 the same). The share of scored listings at 70 or more
went from 33% to 71%.

The old scores looked reasonable for weeks. Compare a pipeline's output with what its input really looked like.

## 3. The model that was retired, and an error hidden inside a "200 OK"

The agent's first real analysis made 128 model calls, and **all 128** failed with "no content in response".
Nothing was scored. The summary then printed "Below threshold: 128" and the run exited with success.

The scoring model's name had been retired. The provider returns that error as HTTP 200, with the message in a
`base_resp` field and no `choices`. The client ignored the field and threw the reason away: "model is no
longer available" (error 2069). We probed five likely model names with a one-word request. Four were retired;
only `MiniMax-M1` answered.

[PR #7](https://github.com/guidebee/jobseeker/pull/7) shows the provider's own message, defaults to the model that
answers, and honours the `MINIMAX_MODEL` setting the README had promised but the code ignored. It also counts
failures separately from scores, stops after five failures in a row, and exits with an error if nothing could
be scored.

An error can arrive inside a 200. And never count a failure as a score.

## 4. Two runs at once, and the process that survived being killed

A weekly run takes 30 to 60 minutes. The agent's shell tool gives up after about 10. At 19:37 Melbourne time
the agent started the command a second time, 20 minutes after the first. Two pipelines were scoring the
same listings, each paying for the same model calls.

The agent's report, verbatim: "I started the wrapper twice, and I did not intend to. My shell tool timed out
at about 10 minutes, and I started it again instead of checking runs/ first. ... This is my mistake."

I stopped the duplicate. The first kill removed the wrapper but not the pipeline underneath, which had a
different parent process and carried on into the analysis step. It had scored 11 listings when it was
spotted.

[PR #9](https://github.com/guidebee/jobseeker/pull/9) adds a per-person lock. A second start exits with code 3 and
"already active" and starts nothing. The lock is released when a run ends, even a failed one, and a lock left
by a crash is recognised as stale. The policy file now says a run takes 30 to 60 minutes and must be started
once. The tests include a real concurrent start.

Long jobs and short tool timeouts make agents retry. Do not forbid the second start: make it impossible.

## 5. The dashboard that counted 61 agents among 30

The home page of the team's website showed "Agents by state": 29 Unknown, 20 Active, 8 Standby, 2 Working,
1 Available, 1 Paused. That is 61 agents for a roster of 30.

Figures for a trailing 24 hours are recomputed on every export, but the store only added or updated entries
and never removed old ones, so a state seen earlier in the day lingered. The "29 unknown" were the moments
during the release trains when the dispatcher was restarting and every agent read as unknown.

Two more numbers confused me. "Messages exchanged: 25,357" sat next to message numbers above 31,000 on the
Messages page. The team's running log numbers every event, and at the time it held 31,809: 15,161 replies, 5,904 status
changes, 4,393 posts, 4,274 reactions, 1,834 checkpoints and a few hundred holds, decisions and channel
events. Posts plus replies are 19,554. The site had also counted status changes (25,458). And "AI
professionals" showed 47, because the live data lists every identity that ever posted, including 17 tool
and system senders. The named professionals are exactly 30.

The fixes, in the team's private website repository: a trailing-window set is now replaced as a whole,
"messages" are posts and replies, "AI professionals" are the named ones, and each page now says what its
number counts.

Every number on a public dashboard needs a definition, and "does it add up?" is a test.

## Scope, honestly

- **Jobseeker is a pilot for one person.** I am that person. It has no customers and no other users.
- **The pilot is a week old.** I have left out its other performance numbers because I have not verified
  them for publication.
- **The team is still learning.** Five bugs in a day is not a boast. The stubbed port probe and the scores
  nobody compared with their input were our mistakes.
- **The fixes do not prove the code is now correct.** They show that these five defects were found and
  have a test or a definition behind them.
- The [Jobseeker](https://github.com/guidebee/jobseeker) code is public. The Agent Hub code stays private.

## Conclusion

None of the five was found by a green check. Each was found by reading what was really there: the bytes of
the resume, the body of a 200, the process list, the roster against the sum.

---

## Related Posts

- [Inside the Company: How 25 AI Agents Plan, Build, Review and Audit Each Other](/posts/2026/11/inside-the-company-how-ai-agents-work-together/): part 3, how the team checks itself
- [Agent Hub v0: A Day of Claude, Codex and DeepSeek Working as One Team](/posts/2026/11/agent-hub-v0-coordination-hub-for-ai-coding-agents/): part 2
- [Agent Channels: Running an Architect, Developers and Reviewers as One Team](/posts/2026/11/agent-channels-managing-multiple-coding-agents/): part 1

## Connect

- **GitHub**: [guidebee](https://github.com/guidebee)
- **LinkedIn**: [James Shen](https://www.linkedin.com/in/guidebee/)

---

*This is post #49 in the Super Morse series. A number that looks fine is not a number that was checked.*
