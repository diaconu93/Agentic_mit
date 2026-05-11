# Design Spec — `mit:log-hours` Skill

**Date:** 2026-05-11
**Plugin:** `mit-dev` v0.1.x (first skill)
**Owner:** Vasile Diaconu (`diaconu93` / `vasile.diaconu@mit-dev.com`)
**Status:** Approved for implementation

## Context

MIT Consulting developers don't consistently log work hours in Jira. In May 2026, only 5 of 20 active users logged any hours (`worklogDate >= "2026-05-01"`). The gap isn't motivation — it's friction: opening Jira, finding the right issue, computing how long you spent, repeating per day. Most devs already capture the same information in commit messages (every MIT commit carries a Jira key like `AIC-215`).

`mit:log-hours` closes this gap: it reads commit history from the current repo, maps commits to Jira issues, estimates daily hour distribution, and posts worklogs in batch with per-day human review.

Out of scope (deliberately): non-MIT Jira instances, YouTrack, multi-repo scanning, automatic running on a schedule, bulk CSV import, editing/deleting existing worklogs.

## High-Level Behavior

```
$ /mit:log-hours [--since YYYY-MM-DD] [--until YYYY-MM-DD] [--dry-run]
```

- Default window: last 30 calendar days (yesterday inclusive, today exclusive).
- Workdays only: Monday–Friday. Weekends ignored.
- Cap: 8h per day, hard. Never proposes >8h on any single day.
- Per-day confirmation. Bulk posting only after all days reviewed.

## Architecture

Single-file skill. No external scripts, no native code, no dependencies beyond what the plugin already provides.

```
plugins/mit-dev/skills/log-hours/SKILL.md
```

Tools used (all already available via the plugin's `.mcp.json` + Claude Code's built-ins):

| Tool | Purpose |
|---|---|
| `Bash` | `git config user.email`, `git log` |
| `mcp__claude_ai_Atlassian_Rovo__searchJiraIssuesUsingJql` | Validate issue keys exist; find active issues for backfill |
| `mcp__jira__jira_get` | Per-issue worklog list with `startedAfter`/`startedBefore` (for conflict detection) |
| `mcp__claude_ai_Atlassian_Rovo__addWorklogToJiraIssue` | Post each worklog |
| `AskUserQuestion` | Per-day confirmation; no-commit-day input |

Constants hardcoded in the skill body:
- `cloudId = "a5748c05-2894-4347-936f-3cfe6897717a"`
- `host = "https://mitconsulting.atlassian.net"`

## Data Flow

### Phase 1 — Init

1. `git rev-parse --is-inside-work-tree` — error out clearly if not a git repo.
2. `git config user.email` → `$AUTHOR_EMAIL`. If empty, ask user.
3. Compute `$FROM` / `$UNTIL` from args or defaults (last 30d, today exclusive).
4. Validate: `$UNTIL <= today` (no future dates).

### Phase 2 — Scan commits

```bash
git log \
  --author="$AUTHOR_EMAIL" \
  --since="$FROM" --until="$UNTIL" \
  --no-merges \
  --format='%H|%aI|%s%n%b%n---END---'
```

For each commit:
- Extract Jira keys via regex `[A-Z]+-\d+` from subject + body. Deduplicate within a commit.
- Date = `%aI` truncated to YYYY-MM-DD (commit-author date, local time).
- Filter: keep only Mon–Fri commits.

Build:
```
commits_by_day = {
  "2026-05-04": {
    "AIC-215": [{hash, subject}, {hash, subject}],
    "BRON-499": [{hash, subject}],
  },
  "2026-05-05": { ... },
  ...
}
```

Track separately:
- `unmapped_commits` — commits with no Jira key.
- `multi_key_commits` — commits with ≥2 keys (each key gets a proportional share).

### Phase 3 — Validate issues

Single batch query:
```
searchJiraIssuesUsingJql(
  cloudId,
  jql='key in (KEY-1, KEY-2, ...)',
  fields=['summary', 'status'],
  maxResults=100
)
```

If any key missing from response → mark as `unmapped` and exclude its commits from estimation. Report to user.

### Phase 4 — Detect existing worklogs (skip rule)

For each issue with proposed hours, fetch worklogs in window:
```
jira_get(
  path=f"/rest/api/3/issue/{KEY}/worklog",
  queryParams={
    "startedAfter": epoch_ms(FROM),
    "startedBefore": epoch_ms(UNTIL),
    "maxResults": "1000"
  },
  jq="worklogs[?author.accountId=='<MY_ID>'].{started: started, seconds: timeSpentSeconds}"
)
```

For every workday in window, if **any** worklog by current user exists on that day (across any issue), mark the entire day as `skip — already logged`. Skip rule applies regardless of total hours on that day.

### Phase 5 — Build plan

For each workday in window NOT skipped:

**If commits exist:**
- Total commits that day = `T` (count of distinct commit hashes — multi-key commits count as 1).
- Each commit contributes weight `1.0` distributed equally across its `k` keys (so a 2-key commit contributes `0.5` to each of its keys). Total weight summed across all keys on the day = `T`.
- For each issue touched that day:
  - `weight_for_issue = W` (sum of fractional weights from all commits referencing this key).
  - `hours = 8.0 * (W / T)`.
- Round each issue's hours to nearest 0.25h (15min).
- Adjust the last issue's hours up/down to make daily sum exactly 8h (absorb rounding drift).

Worked example: day has 3 commits. Commit A has key `AIC-215`. Commit B has key `AIC-215`. Commit C has keys `AIC-215` and `BRON-499` (multi-key).
- `T = 3`.
- `W(AIC-215) = 1 + 1 + 0.5 = 2.5` → `hours = 8 * 2.5/3 ≈ 6.67h` → rounded `6.75h`.
- `W(BRON-499) = 0.5` → `hours = 8 * 0.5/3 ≈ 1.33h` → rounded `1.25h`.
- Sum = `8.0h` after drift absorbed in last issue.

**If no commits:**
- Mark `needs_user_input: true`.

Plan structure:
```
plan = [
  {
    date: "2026-05-04",
    needs_input: false,
    entries: [
      {key: "AIC-215", hours: 5.0, comment: "AIC-215 fix dashboard pagination (a3f2b1c)\nAIC-215 update query filter (b8d4e2f)"},
      {key: "BRON-499", hours: 3.0, comment: "BRON-499 fix login redirect (c1d3e5f)"},
    ],
  },
  {
    date: "2026-05-06",
    needs_input: true,
    entries: [],  // populated during review
  },
  ...
]
```

### Phase 6 — Review & Post (per-day loop)

For each day in plan, chronologically:

**If `needs_input`:**
1. Fetch active issues:
   ```
   searchJiraIssuesUsingJql(
     jql='assignee = currentUser() AND status not in (Done, Closed, Resolved) ORDER BY updated DESC',
     maxResults=15
   )
   ```
2. `AskUserQuestion`:
   - Question: "Wednesday 2026-05-06 — no commits. Pe ce ai lucrat?"
   - Options: numbered list of active issues `[KEY] Summary (status)` + "Skip — zi liberă" + "Other (enter manually)".
3. If user picks issues, follow-up question for hours per issue (free text, e.g. `2h 30m`). Validate sum ≤ 8h.
4. If "Skip" → record day as skipped with reason "user-skipped", move on.

**Confirmation step (every day, both with-commits and after-input):**
1. Display table preview:
   ```
   Date: 2026-05-04 (Monday)
   ┌──────────┬────────┬─────────────────────────────┐
   │ Issue    │ Hours  │ Comment                     │
   ├──────────┼────────┼─────────────────────────────┤
   │ AIC-215  │ 5h 00m │ (2 commits)                 │
   │ BRON-499 │ 3h 00m │ (1 commit)                  │
   └──────────┴────────┴─────────────────────────────┘
   Total: 8h
   ```
2. `AskUserQuestion`: **Confirm** / **Edit** / **Skip day**.
   - **Confirm** → POST all lines for the day.
   - **Edit** → free text input to adjust hours per issue, re-render preview, re-ask.
   - **Skip** → discard day's plan, move on.

**Posting:**
For each confirmed entry:
```
addWorklogToJiraIssue(
  cloudId,
  issueIdOrKey=entry.key,
  timeSpent=format_jira_duration(entry.hours),   // e.g. "5h" or "5h 30m"
  started=f"{date}T09:00:00.000+0000",            // 09:00 UTC default
  comment=entry.comment
)
```

Retry on rate limit (2s, 5s, 10s). Max 3 retries; then abort current day and continue with next.

**Final summary:**
```
Posted:
  2026-05-04: AIC-215 5h, BRON-499 3h  ✓
  2026-05-05: AIC-215 8h               ✓

Skipped:
  2026-05-06: user chose Skip
  2026-05-07: existing worklog

Failed:
  2026-05-08: AIC-300 3h — 403 Forbidden (issue archived?)

Unmapped commits (no Jira key or invalid key):
  ad8f291 "refactor utils"
  c1b3e8f "FAKE-999 test"  ← key not in MIT Jira
```

## Edge Cases (confirmed during brainstorming)

| Case | Handling |
|---|---|
| cwd not a git repo | Immediate error, suggest `cd` to repo. |
| `git config user.email` missing | Ask user inline. |
| No commits in window | Skip Phase 2-5 results, jump to Phase 6 with all workdays as `needs_input`. |
| Commit with multiple keys | Each key gets `1/k` weight in proportional split. |
| Commit with valid-format but non-existent key | Excluded from estimation, listed in final summary as `unmapped`. |
| Commit with no key | Ignored entirely (no estimation), listed in summary. |
| Day with 1 issue and N commits | Full 8h on that issue. |
| Day with 1 commit at 10am only | Still 8h (algorithm assumes full workday; user can Edit during review). |
| Rounding drift | Last issue absorbs drift to keep daily sum exactly 8h. |
| Existing worklog by current user (any amount) | Day fully skipped. |
| Existing worklog by other user on same issue | Ignored (filter on `currentUser()` only). |
| Multiple worklogs same day, sum < 8h | Day still skipped (skip rule is binary, not cumulative). |
| OAuth Atlassian expired | Error message points to `/plugin atlassian` for reauth. |
| Issue archived/closed during POST | Single-line failure, continue with next entry. |
| Rate limit | 3 retries, exponential backoff, then abort day with clear message. |
| Partial POST success | No rollback. Final summary shows posted vs failed clearly. |
| `--since` in future / before window | Validation error before any work. |
| No-commit-day user input sum > 8h | Re-ask until ≤ 8h. |

## Constraints (firm)

- **Max 8h/day** — algorithm cannot generate >8h; user input is validated to ≤8h.
- **One day at a time** — confirmation is per-day, not bulk.
- **Skip-existing is strict** — any existing worklog by current user blocks the whole day. No top-up logic in v1.
- **Single repo, current cwd only** — no multi-repo scanning. User runs the skill per repo.
- **Jira keys must come from commit msg or branch name** — no fuzzy summary matching in v1.
- **Constants hardcoded** — cloudId, host, JQL templates. No `.local.md` config file in v1.

## Testing Plan

Manual only (no unit tests for markdown-only skill).

| Test | Setup | Expected |
|---|---|---|
| Happy path | Real repo with 1 week of commits, all with keys | Skill proposes plan, you confirm each day, worklogs appear in Jira |
| `--dry-run` flag | Same as above + `--dry-run` | Skill shows plan but NO POST happens; verify via `jira_get` no new worklogs |
| Smoke on test issue | 1-2 commits on `AIC-TEST-1` for one day | Verify worklog via `jira_get /rest/api/3/issue/AIC-TEST-1/worklog` |
| Skip existing | Manually log 1m on a day, then run with `--since=that-day` | Day is skipped with reason |
| Invalid key | Commit `"FAKE-999 test"` | Listed in `unmapped` summary, no POST attempt |
| No-key commit | Commit `"refactor utils"` | Ignored from estimation, listed in summary |
| No-commit day | Run on a workday with zero commits | AskUserQuestion shows active issues list |
| Skip day | At review, choose "Skip" | No POST, day in skipped summary |
| Not a git repo | `cd /tmp && /mit:log-hours` | Error message, exit |
| Future `--since` | `/mit:log-hours --since 2027-01-01` | Validation error |

Roll-out: author tests on own commits for 1 week, then announce to team.

## Open Questions (deferred to v2+)

- Top-up mode: if a day has 4h logged, propose remaining 4h. (User chose `skip` for v1.)
- Multi-repo scanning. (User chose `cwd-only` for v1.)
- Configurable holiday list. (User chose `M-F only, ignore holidays` for v1.)
- Per-entry confirmation granularity. (User chose `per-day` for v1.)
- Scheduled / cron mode ("log my last week every Friday at 5pm").
- Alternative estimation heuristics (session clustering, time-since-previous).

## References

- Reference skill pattern: `~/.claude/plugins/cache/claude-plugins-official/atlassian/9b52fb18e184/skills/triage-issue/SKILL.md`
- Jira worklog API: `POST /rest/api/3/issue/{KEY}/worklog`
- ADF for comment: handled by `addWorklogToJiraIssue` MCP tool (we pass plain string).
- Active users in workspace (for context): 20 distinct, only 5 logged hours in May 2026.
