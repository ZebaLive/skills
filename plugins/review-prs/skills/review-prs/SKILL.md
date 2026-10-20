---
name: review-prs
description: Review one or more GitHub pull requests and write one validated review file, with findings sorted HIGH/MEDIUM/LOW, each tied to a file and line range with a depthfirst-style finding and fix suggestion. Use whenever the user asks to review a PR, several PRs, a PR stack, or PR links/numbers (e.g. "review these PRs", "/review-prs", "check #227", "look over this pull request", "review PRs across repos"), even if they don't say "skill" or "file". Also use it for follow-ups after the author pushes fixes ("fixes are in", "re-review", "check if the findings were addressed", "take another look"). Reviews multiple PRs both individually and as a whole. Does not post to GitHub unless asked.
---

# Review PRs

Review the given pull requests. Validate every finding with an independent verifier. Write one markdown file.

The user reads this file to decide what to fix and what to post. A false finding wastes their time and their colleague's time. A missed finding ships a bug. Validation exists for the first problem; the review angles exist for the second.

## Step 0: New review or re-review?

Look for an earlier review of the same PRs: a file in `tmp/review/` or `/tmp/pr-review/` whose `review-prs` marker lists them, or a review written earlier in this conversation. If one exists and the user says fixes are in, or asks you to check again, follow "Re-review" at the end of this file. Otherwise start at Step 1.

## Step 1: Check access before anything else

Run `gh pr view <n> -R <owner>/<repo> --json title,body,baseRefName,headRefOid,state,mergedAt,files` for each PR. Keep the output; Step 2 uses it.

If it fails, stop and tell the user which fix applies. Do not guess. Known failures:

- `Resource protected by organization SAML enforcement`: the user must authorise the token at `https://github.com/orgs/<org>/sso`, or run `gh auth refresh`.
- `gh` hangs or returns `HTTP 401`, and `gh auth token` prints nothing: the system keyring is locked. The user must unlock it in their own terminal.
- SSH `device not found`: the hardware security key is unplugged.

Also offer the fallback: the user can put diffs at `/tmp/prrev/<repo>-<n>.diff` or point to local clones.

## Step 2: Gather each PR

For each PR, collect:

- Title, body, base branch, head SHA (`headRefOid`), and changed files.
- The diff: `gh pr diff <n> -R <owner>/<repo>`.
- The code at the PR head. Review the head, not your memory of the diff. Use a local clone if one exists: `git fetch origin pull/<n>/head:refs/remotes/origin/pr-<n>`. Read files with `git show "${SHA}:<path>"`. Quote it, because zsh reads `$SHA:` as a history modifier. Without a clone, use `gh api "repos/<o>/<r>/contents/<path>?ref=<sha>" -q .content | base64 -d`. Quote the URL, because zsh treats `?` as a glob.
- Whether the PR is already merged. Review a merged PR at its head all the same. For each finding, read the cited file on `baseRefName` the same way. Say whether the fault is still present there.

Record each head SHA. You need it again in Step 5.

Ignore merge conflicts that GitHub reports unless the user asks about them.

## Step 3: Find candidates

Read every hunk. Then read the whole function around each hunk. Cite file line numbers at the head SHA, not positions in the diff. Bugs in unchanged lines of a touched function are in scope.

Work through these angles for each PR:

1. **Line scan.** For each line, ask what input, state, timing, or platform makes it wrong.
2. **Removed behaviour.** For each deleted line, name the guard it enforced. Find where the new code enforces it again.
3. **Callers and callees.** Grep for each changed symbol. Check every call site against the new contract.
4. **Security and authorisation.** Check scope checks, trust boundaries, tokens, and data exposed to callers who should not see it.
5. **Error paths.** Check swallowed exceptions, retry and conflict handling, and partial writes.
6. **Races and lifecycle.** For each row, file, lock, or cache entry the change reads and later writes, ask what else can delete, prune, expire, or change it in between. Check other jobs, schedulers, and workers that do not share the lock. For each status or pointer the change sets, ask what clears it if the next step fails. A value that nothing clears leaves the record stuck.
7. **Tests.** Check that the tests would fail if the change broke. A test that passes with the bug present gives no protection.
8. **Conventions.** Read the repo's `CLAUDE.md`, `AGENTS.md`, or contributing guide. Flag breaks only when they have a consequence.

### Cross-PR pass (two or more PRs)

Review the PRs as one change after the individual passes. Look for:

- Contract mismatches: one PR sends a field, shape, error code, or header that another PR does not accept.
- Ordering: one PR must deploy or merge before another, and nothing enforces it.
- Duplicated or conflicting logic in two PRs.
- A gap that no single PR owns.

### Inherited faults

The PR may touch code that was already wrong. Report the fault, but label it **pre-existing** and suggest a follow-up PR. Do not blame this PR for it.

## Step 4: Validate every candidate

Spawn one verifier subagent per candidate. Use `model: "haiku"`. The alias resolves to the newest Haiku. The verifier only checks; you make every decision.

Send all verifier calls in one message, in the foreground (`run_in_background: false`). They run in parallel and their replies come back directly. Do not search transcript files for verdicts: the prompt text also contains "VERDICT".

Give each verifier the prompt in `references/verifier-prompt.md`, filled in with the candidate. Include the head SHA, the base SHA, and how to read code at each.

Act on the verdicts:

- `CONFIRMED`: keep the finding. Use the verifier's evidence to sharpen the text.
- `REFUTED`: drop the finding. List it under "Dropped candidates" with the reason, so the user sees what was checked.
- `UNCERTAIN`: read the code yourself. Keep the finding only if you can show the failure path. Otherwise drop it.

Write nothing to the file that a verifier refuted.

Candidates you rule out yourself before verification also go under "Dropped candidates". Cite the code that rules each one out.

If the PR body already admits a confirmed fault, keep the finding. Say that the author knows, and judge whether the PR body describes the cost correctly.

## Step 5: Re-check the head

Run `gh pr view` again and compare each `headRefOid` with Step 2. If a head moved, re-verify the findings against the new head. Drop findings the author already fixed. Say in the file which SHA you reviewed.

## Step 6: Assign severity

- 🔴 **HIGH**: data loss, security or authorisation hole, crash on a normal path, wrong money or compliance result. Block the merge.
- 🟡 **MEDIUM**: wrong behaviour on a reachable but less common path, a broken contract between services, a missing test for risky logic. Fix before merge.
- ⚪ **LOW**: edge cases, pre-existing faults, thin tests, convention breaks with a small consequence. Fix or follow up.

Severity follows the consequence, not the size of the fix.

## Step 7: Write the review file

Write to `tmp/review/<slug>.md` in the current working directory. Use `/tmp/pr-review/<slug>.md` outside a git repo. Build the slug from the repos and PR numbers, e.g. `api-94_web-24`. Create the directory if needed.

Use this template exactly:

```markdown
# PR review: <short title>

<!-- review-prs: <owner>/<repo>#<n>@<full head sha> <owner>/<repo>#<n>@<full head sha> -->

Reviewed: <date>
| PR | Title | Head SHA |
| --- | --- | --- |
| [<repo>#<n>](<url>) | <title> | `<sha7>` |

## Verdict

<One paragraph. Say whether each PR is safe to merge and what blocks it. Give counts: N high, N medium, N low.>

## Cross-PR findings

<Only for two or more PRs. Same finding format as below. Write "None." if there are none.>

## <repo>#<n>: <title>

### 🔴 HIGH

<findings, or "None.">

### 🟡 MEDIUM

<findings, or "None.">

### ⚪ LOW

<findings, or "None.">

<repeat the section for each PR>

## Dropped candidates

- `<file>:<line>`: <claim>. Dropped by <verifier | reviewer>: <reason and the code that shows it>.
```

### Finding format

Follow the depthfirst-app style. Each finding:

````markdown
#### <ID> · `<path/to/file.py>:<start>-<end>`

<emoji> **Severity: <LEVEL>**

<Two to four sentences. Name the input or state that triggers the fault, what the code does, and the consequence. Quote identifiers in backticks. Say "pre-existing" if the PR inherited it.>

<details>
<summary>💡 Fix Suggestion</summary>

**Suggestion:** <What to change and why that fixes it. Name the test that should cover it.>

```suggestion
<replacement lines, only when the fix is local to the quoted lines>
```

</details>
````

Give each finding an ID: `H`, `M`, or `L` plus a number, counted across the whole file (`H1`, `M1`, `M2`, `L1`). IDs never change or get reused. A re-review uses them to report status.

Leave out the `<details>` block when you have no fix. Leave out the `suggestion` block when the fix spans several places; describe it in prose instead. A `suggestion` block replaces exactly the lines in the heading, so keep its indentation identical to the source. If the fault spans more lines than the fix touches, name the fault's range in the prose and keep the heading on the fix's range.

Example:

````markdown
#### M1 · `src/billing/refunds.py:88-89`

🟡 **Severity: MEDIUM**

`refund()` checks `can_refund` against `request.account_id`, which the caller controls, instead of the invoice's account. A support user with refund rights on one account can refund invoices on any account by changing `account_id` in the request body.

<details>
<summary>💡 Fix Suggestion</summary>

**Suggestion:** Load the invoice first and check `can_refund` against the invoice's own account. Add a test where a user with refund rights on account A tries to refund an invoice on account B and gets a 403.

```suggestion
    invoice = Invoice.get(invoice_id)
    require(user.can_refund(invoice.account_id))
```

</details>
````

## Step 8: Report

Tell the user the file path, the verdict, and the counts by severity. Name the HIGH findings in one line each.

Do not post to GitHub. If the user asks you to post, use inline review comments for line findings and one PR comment for cross-PR and overall findings. GitHub accepts inline comments only on lines in the diff. Put findings on other lines in the PR comment, with `file:line` in the text. Each comment uses the same finding format. Re-check the head SHA before posting.

## Re-review

Use this flow when the author pushed fixes after an earlier review. Review only what changed. Do not repeat the full review.

1. Read the earlier review file. Take each PR's old head SHA from the `review-prs` marker and the list of finding IDs.
2. Run Step 1 for each PR. Fetch the new head SHA. If it equals the old one, tell the user nothing changed and stop.
3. Get the change since the last review: `git diff "<old>..<new>"` in a clone, or `gh api repos/<o>/<r>/compare/<old>...<new>`. If the author force-pushed and the old SHA is gone, diff the files the findings name, plus the files changed since the base, at the new head. If the range merges the base branch in, limit the diff to files in `gh pr diff`. Changes that came from the base were reviewed elsewhere.
4. Set a status for each earlier finding:
   - **Fixed**: the failure scenario no longer happens at the new head.
   - **Partly fixed**: some paths still fail. Say which.
   - **Open**: the code at the cited lines did not change, or the change does not stop the failure.
   - **Obsolete**: the code was removed or rewritten, so the finding no longer applies.
   - **Won't fix**: the author or the user decided to leave it. Quote where they said so.
   - **Fixed elsewhere**: still present at the PR head, but another merged PR fixed it on the base branch. Name that PR.
5. Validate statuses with Haiku verifiers, using `references/verifier-prompt.md`:
   - **Fixed**: claim "finding <ID>'s failure scenario no longer happens at <new sha>". `REFUTED` means **Partly fixed** or **Open**, as the evidence shows.
   - **Partly fixed**: claim "the fix covers <fixed paths>, and <remaining paths> still fail at <new sha>".
   - **Open**: first grep the new diff for changes to the finding's callers, guards, and validators. A fix can land away from the cited lines. Mark **Open** only if no change touches the failure path.
   - **Obsolete**: set it from the diff alone.
6. Review the new commits for new faults. Run Step 3 on the change since the last review only, then Steps 4 and 6. New findings take the next free IDs.
7. Update the same review file:
   - Update the marker and the head SHA table to the new heads.
   - Update the `Reviewed:` date.
   - Add a section above "Verdict": `## Re-review <n>: <date>, <old sha7>..<new sha7>`. It holds only a table with the columns `ID | Finding | Status | Evidence`.
   - Put new findings in the PR's HIGH, MEDIUM, or LOW section with the next free ID. Add `**Status: New (re-review <n>)**` under each heading.
   - Add new dropped candidates to "Dropped candidates".
   - Rewrite the verdict for the new head. Count only findings that are open, partly fixed, or new.
   - Leave the earlier findings in place. Add `**Status: <status> (re-review <n>)**` under each heading.
8. Report the counts by status, any new findings, and whether anything still blocks the merge.

If the user asks for comments, post only for **Open**, **Partly fixed**, and new findings. Reply in the thread of an existing comment instead of opening a new one.

