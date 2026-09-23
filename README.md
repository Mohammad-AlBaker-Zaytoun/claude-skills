# Claude Code Skills

Skills for [Claude Code](https://docs.claude.com/en/docs/claude-code) by **Mohammad AlBaker Zaytoun**.

## Skills

| Skill | What it does |
|---|---|
| [`overnight_work`](skills/overnight_work/SKILL.md) | Runs a long unattended work session: a self-pinging cron, measured fixes, PRs opened, reviewed and merged, a hard stop time, and a handover summary at the end. |

## Install

### Option 1: with `npx`

```bash
npx skills add Mohammad-AlBaker-Zaytoun/claude-skills
```

### Option 2: manually

Copy the skill folder into your personal skills directory:

```bash
git clone https://github.com/Mohammad-AlBaker-Zaytoun/claude-skills.git
mkdir -p ~/.claude/skills
cp -r claude-skills/skills/overnight_work ~/.claude/skills/
```

On Windows (PowerShell):

```powershell
git clone https://github.com/Mohammad-AlBaker-Zaytoun/claude-skills.git
New-Item -ItemType Directory -Force "$HOME\.claude\skills" | Out-Null
Copy-Item -Recurse claude-skills\skills\overnight_work "$HOME\.claude\skills\"
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

## License

[MIT](LICENSE) © Mohammad AlBaker Zaytoun
