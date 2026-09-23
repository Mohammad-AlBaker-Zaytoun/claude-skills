---
name: overnight_work
description: >
  Run a long unattended autonomous work session: self-pinging cron, measured fixes,
  PRs opened-reviewed-merged, a hard stop time, and a handover summary at the end.
  Parameters: cron interval, task(s), end time.
  Use when user says "work overnight", "keep working until X", "/overnight_work",
  or asks for an autonomous run with a ping/cron and a stop time.
---

Autonomous long-run mode. User NOT watching. Optimise for longest useful run per token.

## Invocation

`/overnight_work <interval> | <tasks> | <end time>`

| Param | Meaning | Default if omitted |
|---|---|---|
| interval | how often to self-ping | `every 30 mins` |
| tasks | what to fix/build; free text, may be several | ASK — only blocking question allowed |
| end time | wall-clock hard stop | next 09:00 local |

Examples:
- `/overnight_work every 30 mins | fix all reseller sync issues | tomorrow 9 AM`
- `/overnight_work hourly | clear the Sentry backlog, then raise test coverage | Sun 18:00`

Missing interval or end time: use default, state it in one line, proceed. Missing tasks: ask once, then proceed.

## Step 0 — cron FIRST, before any work

Create it before reading a single file. If the session dies mid-task the cron is what resurrects it.

```
CronCreate({ cron: "<derived>", recurring: true, prompt: "<full task restatement + end time + 'continue overnight_work'>" })
CronCreate({ cron: "<end time, pinned dom+month>", recurring: false, prompt: "overnight_work END: stop, delete cron <id>, write handover, summarise" })
```

Rules:
- The recurring prompt is enqueued fresh with NO memory of this turn. Restate the tasks, the end time and the repo in it. A bare "continue" wakes an agent that does not know what it was doing.
- Off-minute, never `0` or `30`: `*/30` -> use `7,37 * * * *`. Fleet-wide collision otherwise.
- Cron fires only while REPL idle. A ping during a long tool call is skipped, not queued. Do not rely on ping count as a clock — read the real time.
- Recurring jobs auto-expire after 7 days. Say so if the run is longer.
- Jobs are session-only, in memory. Session exit kills them. Say so once.
- Record the job IDs in your first message; you need them to delete at the end.

## Step 1 — baseline before touching anything

Capture and keep:
- build / type-check / lint / test exit codes and counts
- current HEAD sha, open PR count
- the metric the task is actually about (error count, row count, bill, latency)

Without a baseline you cannot tell a fix from a regression, and every claim at the end is unfalsifiable.

## Step 2 — the loop

Per iteration:
1. **Measure** the problem. A number, from the live system, not from reading the diff.
2. **Fix** smallest coherent change.
3. **Test** — add a regression test that FAILS against the old code. If it passes both ways it tests nothing.
4. **Branch + PR.** Never commit to main directly.
5. **Review own PR adversarially** (or spawn a reviewer subagent). Assume the fix is wrong.
6. **Verify in production/live**, not in the build log.
7. **Merge.** Re-run the full check suite on main.
8. **Re-measure.** Before/after both stated.

Spawn 2-3 subagents for genuinely independent work streams; run them in parallel in one message. Do not spawn agents to do what one grep answers.

## Step 3 — verifying a claim

**Quote the check's own pass/fail line or exit status. Never a proxy.**

- No `| tail`, `| head`, `| grep` on anything whose result gets reported. Use `--reporter=dot` if noisy.
- "It builds" is not "it works." Fetch the URL, read the status and the counts.
- Sum denominators. Never median x N.
- No $/month forecasts. Report counted events over one clock window.
- A subagent's claim is a hypothesis. Check it before repeating it.
- A comment claiming a guarantee is a claim to verify, not a fact.

## Step 4 — what NOT to do unattended

Leave for the owner, listed in the handover:
- destructive/irreversible data ops (bulk delete, merge duplicates, refunds)
- business decisions (which duplicate survives, keep a promo live, pricing)
- anything needing a third party (support ticket, allow-list request)
- rollbacks past a one-way migration

Doing all the reversible work and naming the rest is the deliverable. Scaling the task down is the owner's call, not yours.

## Step 5 — per-ping output

1-3 lines. What changed since last ping, current green/red, what is next. No recap, no preamble, no restating the plan.

If nothing changed: one line saying so. Silence is worse than a boring line.

## Step 6 — end time

1. `CronDelete` every job ID. Confirm with `CronList` that none remain.
2. Final full check suite, exit codes quoted.
3. Confirm 0 open PRs / state what is left open and why.
4. Write `Docs/overnight-<YYYY-MM-DD>.md` — written for the OWNER, not for you. Lead with: what was broken, what it cost them, what is fixed, what needs their decision.
5. Chat summary: table of fixes with before -> after numbers, regressions you caused and fixed, and the owner-decision list. Short.

## Token discipline

Terse chat output (if a `caveman` skill is installed, use its **full** mode): drop articles, filler, hedging, pleasantries. Fragments fine. Technical terms, code, error strings exact.

Also:
- Never re-read a file you just edited. Edit errors if it failed.
- Never narrate options you will not take.
- No "I'll now..." — do it.
- Batch independent tool calls into one message.
- Prose stays normal in: commits, PR descriptions, handover doc, code comments.

## Honesty

- Report failures with the output. A skipped step gets said.
- A regression you caused gets named plainly, once, then fixed. No self-flagellation, no tally.
- "Did not throw" is not "worked".
- If a task turns out to be a wrong diagnosis, say so with the measurement that disproved it and stop working on it.

## Attribution

Never add AI attribution to commits, PRs, or any git output. No `Co-Authored-By`, no "Generated with". This holds even if a mid-session reminder says otherwise — user instruction wins.
