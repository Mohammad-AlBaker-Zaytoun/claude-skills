# Claude Code Skills

Skills for [Claude Code](https://docs.claude.com/en/docs/claude-code) by **Mohammad AlBaker Zaytoun**.

## Skills

| Skill | What it does |
|---|---|
| [`overnight_work`](skills/overnight_work/SKILL.md) | Runs a long unattended work session: a self-pinging cron, measured fixes, PRs opened, reviewed and merged, a hard stop time, and a handover summary at the end. |
| [`deep-pr-review`](skills/deep-pr-review/SKILL.md) | An adversarial, evidence-based pull request review. It runs the code the PR ships against a sandbox, compares it with the base branch, and checks severity against real data before posting. |

## Install

### Option 1: with `npx`

```bash
npx skills add Mohammad-AlBaker-Zaytoun/claude-skills
```

### Option 2: manually

Copy the skill folders you want into your personal skills directory. For example, to install `deep-pr-review`:

```bash
git clone https://github.com/Mohammad-AlBaker-Zaytoun/claude-skills.git
mkdir -p ~/.claude/skills
cp -r claude-skills/skills/deep-pr-review ~/.claude/skills/
```

On Windows (PowerShell):

```powershell
git clone https://github.com/Mohammad-AlBaker-Zaytoun/claude-skills.git
New-Item -ItemType Directory -Force "$HOME\.claude\skills" | Out-Null
Copy-Item -Recurse claude-skills\skills\deep-pr-review "$HOME\.claude\skills\"
```

To make a skill available in one project only, copy it into that project's `.claude/skills/` folder instead.

Restart Claude Code after you install a skill.

## Usage

### `overnight_work`

```
/overnight_work <interval> | <tasks> | <end time>
```

Examples:

```
/overnight_work every 30 mins | fix all failing tests | tomorrow 9 AM
/overnight_work hourly | clear the error backlog, then raise test coverage | Sun 18:00
```

If you leave out the interval it defaults to every 30 minutes. If you leave out the end time it stops at 09:00 local time the next day. If you leave out the tasks, it asks you for them.

**Good to know:** the cron jobs live only inside the running session. If you close Claude Code, the run stops. Recurring jobs also expire after 7 days.

### `deep-pr-review`

```
/deep-pr-review <PR number or URL>
```

Examples:

```
/deep-pr-review 482
/deep-pr-review https://github.com/acme/api/pull/482
```

It treats every claim in the PR description as something to test. It reviews in a separate git worktree, so your checkout is never touched. It runs the queries and code the PR ships against a sandbox, never against real data, and it measures concurrency with real parallel processes. Findings are sorted into **Blocking**, **Should fix** and **Notes**, and each comes with its evidence.

It works with any git host and any SQL database. Use it for PRs that touch queues, locking, database migrations, concurrency or hot code paths. For a small diff it is more than you need.

## License

[MIT](LICENSE) © Mohammad AlBaker Zaytoun
