---
name: deep-pr-review
description: "Adversarial, evidence-based pull request review. Verifies every claim by executing the shipped artifact against a real engine on a sandbox, A/Bs against the base branch, and calibrates severity with production data before posting. Use for deep/thorough/senior review requests, or any PR touching queues, migrations, concurrency, or a hot path. Trigger: /deep-pr-review <PR-number-or-URL>"
---

# Deep PR Review

A review that reads code produces opinions. A review that **runs** code produces findings. This skill is the second kind.

**Core rule: the PR description is a hypothesis list, not evidence.** Every claim in the body — "this is idempotent", "existing rows retain legacy behaviour", "this commit is separable", "the migration is additive" — is something to test, not something to accept. Treat the author's framing as the thing most likely to hide the defect, because it is where they stopped looking.

## When to use

Any PR touching queues, claims/locking, schema migrations, concurrency, deploy seams, or a request-path hot spot. Also whenever the user asks for a deep, thorough, careful, or "senior/lead architect" review.

For a quick correctness pass on a small diff, this is overkill — read the diff and comment.

---

## Phase 0 — Ground truth

Read the repo's own conventions first (`AGENTS.md`, `CONTRIBUTING.md`, `CLAUDE.md`). Then pull the PR's title, body, base branch and head branch with whatever CLI or API your code host provides, and fetch the branches:

```bash
git fetch origin                   # retry a few times on transient auth failure
git log --oneline origin/<base>..origin/<head>
git diff --stat origin/<base>...origin/<head>
```

- **Do not trust the host's mergeability flag blindly.** Some hosts report a stale or constant value. Check conflicts yourself with a trial merge in the worktree.
- **Review in an isolated worktree, never the working copy.** The user's checkout stays untouched and you can cherry-pick freely:
  ```bash
  git worktree add ../pr<N>wt origin/<head-branch> --detach
  ```
  On Windows, keep the worktree path short — deep dependency folders can exceed the path-length limit and make `worktree add` fail.
- Save the full diff to a file and read it in chunks. Do not review from `--stat`.

## Phase 1 — Map the blast radius

- Write down every claim the PR body makes. Each is a test you owe.
- Find **all** call sites of every changed function, not only the ones the PR touched:
  ```bash
  grep -rn "ChangedClass::method\|changed_function(" .
  ```
- **Search tools that honour `.gitignore` can be blind to live code.** Generated, vendored, or legacy trees that are ignored but still executed return a clean, empty result — which looks exactly like "no other callers". Use `grep -rn` or `rg --no-ignore` for anything outside the tracked source.
- Ask the **deploy-seam question** on every schema or config change: *who runs this, versus who needs it?* Producer and consumer are often different processes with different deploy units. A migration that only runs in the worker's bootstrap, while the web request that enqueues work already needs the new column, is a production outage that no unit test will catch.

## Phase 2 — Execute the shipped artifact

Do not reason about what the SQL/query/template will do. Capture the exact string the code emits and run it.

**The capture shim.** Stub the boundary the code writes through, record the payload, and retarget it at a sandbox. Example in PHP:

```php
// Capture the real SQL the repository builds, without a database.
if (!class_exists('Db', false)) {
    class Db {
        public static string $sql = '';
        public static function query(string $sql) { self::$sql = $sql; return null; }
        public static function lastError(): string { return ''; }
    }
}
require_once $worktree.'/src/Queue/JobRepository.php';
JobRepository::claimBatch($workerId, $limit);
$sql = str_replace('app.jobs', 'sandbox_jobs', Db::$sql);  // retarget
```

Now execute `$sql` for real. You are testing the shipped string, not your transcription of it.

**Sandbox policy — non-negotiable:**
- Against **real tables: read-only probes only.** A `SELECT <new column list> FROM <table> WHERE 1=0` proves a binding error without writing anything.
- For anything that mutates, create a **sandbox table** that mirrors the real schema plus the PR's new columns and indexes. It must be visible to more than one connection so you can test concurrency:
  - SQL Server: a `##global` temp table (a `#local` one is invisible to the second connection).
  - PostgreSQL: a table in a scratch schema.
  - MySQL: a table in a scratch database.
- Confirm which environment you are pointed at **before** the first statement (`SELECT @@SERVERNAME, DB_NAME()` / `SELECT current_database()` / `SELECT DATABASE()`). Local dev setups often point at a shared QA database.
- Drop the sandbox when done.

**Test concurrency with actual concurrent processes**, never by reasoning about isolation levels. Spawn workers, run the real claim/update in a loop from each, and have the parent sample for invariant violations:

```php
$procs[$w] = proc_open("\"".PHP_BINARY."\" worker.php $workerId $table $mode $secs",
                       [1=>['pipe','w'],2=>['pipe','w']], $pipes[$w]);
// parent samples: SELECT key, COUNT(*) FROM t WHERE status = 'CLAIMED' GROUP BY key
// any COUNT(*) > 1 is a mutual-exclusion violation
```

Count deadlocks explicitly (SQL Server 1205, PostgreSQL `40P01`, MySQL 1213) rather than assuming retry logic hides them.

## Phase 3 — A/B against the base branch

A number without a baseline is not a finding. Extract the base version of the changed file and run the identical workload both ways:

```bash
mkdir -p /tmp/baseroot/<same/relative/dir>
git show origin/<base>:<path/to/File> > /tmp/baseroot/<path/to/File>
```

Point the worker at one root or the other by mode. Report **deltas** (`148/240 vs 240/240`), not absolutes. Never hand-edit the new code to simulate the old — regex-stripping a guard silently fails and produces a fake baseline.

Run at least two workload shapes, because the interesting result is usually where they diverge — e.g. a fan-out shape (the change costs nothing) and a concentrated shape (the change costs 38%).

## Phase 4 — Calibrate severity with production data

**This phase is what separates a useful review from an alarming one.** Before calling anything a performance or correctness problem, measure the real operating point.

- Replay **real stored payloads** through the new code path rather than synthetic ones — read-only:
  ```sql
  SELECT raw_body, type FROM inbound_events
   WHERE type = 'message' ORDER BY id DESC LIMIT 3000;
  ```
  then run each body through the new function and count outcomes.
- Establish the real distribution: how much of the table is even affected? What is the realistic depth/rate?
- **Be willing to downgrade your own finding.** A theoretically real "multi-entity payloads lose the ordering key" hole can measure 0 in 3,000 on real traffic, and a linear-read regression can sit at the harmless end of its own curve when the busiest real key sees 11 events a minute. Report both honestly, labelled as tail risk.
- State plainly what you measured and what you could not. "Latent — currently zero rows exercise this path" is a finding; pretending it is live is not.

## Phase 5 — Verify the author's structural claims

- **"This commit is separable"** → cherry-pick it onto the base and run the suite:
  ```bash
  git worktree add ../prSep origin/<base> --detach
  cd ../prSep && git cherry-pick -n <sha>
  <run the test suite>
  ```
  Exclude failures caused only by the environment (a missing untracked config file, a service you cannot reach) and say which ones you excluded.
- **"Idempotent"** → run it twice.
- **"Additive / existing rows unaffected"** → check the deploy seam and the rollback path, not just the DDL.
- Run the PR's own tests, then ask what they actually assert. A test that counts substrings in a generated SQL string pins the *text*, not the *behaviour* — say so, and supply the behavioural test you already wrote in Phase 2.
- Mutation-check any test the PR adds: break the code it claims to guard and confirm the test fails. A test that still passes is asserting its own fixture.

## Phase 6 — Pre-flight before posting

- **Verify every line-number citation** (`grep -n` each one). A wrong line number discredits a correct finding.
- Check for line-ending noise inflating the diff:
  ```bash
  git diff --numstat origin/<base>...HEAD                    | awk '{s+=$1+$2} END{print s}'
  git diff --numstat --ignore-all-space origin/<base>...HEAD | awk '{s+=$1+$2} END{print s}'
  ```
- Scan added lines for secrets.
- **Give credit for what held up.** Lead with the verified-correct table. A review that only lists problems gets read defensively and its real findings get discounted.
- Order findings **Blocking / Should fix / Notes**, each with the evidence inline.
- Recommend a concrete path, not just objections.

## Phase 7 — Post and clean up

Write the body to a temp Markdown file, then post it as a formal review (approve, request changes, or comment) with your code host's CLI or API. Pass the body as a file rather than an inline string, so the shell does not mangle multi-line Markdown.

- Use the exact review-state value the host expects. Some APIs quietly save an unrecognised value as a draft or pending review instead of returning an error, so a typo looks like success.
- **Always read the response back** and check the review state matches what you sent.
- Then clean up: `git worktree remove <path> --force`, drop sandbox tables, confirm the user's checkout is back on its original branch.

---

## Severity bar

| Level | Meaning |
|---|---|
| **Blocking** | Proved broken, or breaks on a realistic deploy/rollback sequence. Must have executed evidence. |
| **Should fix** | Real defect or measured regression, but bounded or not yet reachable. Say what bounds it. |
| **Notes** | Latent, cosmetic, missing docs/tests, or behaviour changes the PR body failed to disclose. |

Never assign severity from code reading alone. If you could not test it, label it "unverified" and say why.

## Checklist

- [ ] Reviewed in an isolated worktree; user's checkout untouched
- [ ] Every PR-body claim enumerated and individually tested
- [ ] Shipped artifact executed, not transcribed
- [ ] Real tables read-only; all mutation in a sandbox; environment confirmed first
- [ ] Concurrency tested with real concurrent processes; deadlocks counted
- [ ] A/B against the base branch, deltas reported
- [ ] Severity calibrated against real production/QA data
- [ ] Author's separability/idempotency claims independently verified
- [ ] New tests mutation-checked
- [ ] All line-number citations checked
- [ ] What held up is credited, up front
- [ ] Review state confirmed after posting
- [ ] Worktrees removed, sandboxes dropped, branch restored
