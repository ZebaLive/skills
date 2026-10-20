# Verifier prompt

Fill in the `<...>` fields and pass the text below as the subagent prompt.

---

You verify one code-review finding. Do not judge severity. Do not suggest fixes. Do not look for other bugs. Check this one claim and report what you find.

**Repository:** `<owner>/<repo>`
**PR:** #<n>, head SHA `<sha>`, base SHA `<base sha>`
**How to read code at the head:** <e.g. `git show "<sha>:<path>"` in `<clone path>`; keep the quotes, because zsh misreads `$SHA:`, or `gh api "repos/<o>/<r>/contents/<path>?ref=<sha>" -q .content | base64 -d`; keep the quotes, because zsh treats `?` as a glob>
**Diff:** `gh pr diff <n> -R <owner>/<repo>`. For a claim about removed code, read that code at the base SHA.

**Claim:** <file>:<start>-<end>. <one-sentence claim>
**Failure scenario given by the reviewer:** <concrete input or state, and the wrong result>

Work through these checks in order. Judge the claim's substance, not the precision of its citation.

1. Read `<file>` at the head SHA. Find the code the claim describes. If the cited lines are off, locate the code by its identifiers and use the right lines. Report the correction in EVIDENCE. Refute only if the described code does not exist. For a claim about removed code, look at the base SHA instead.
2. Read the whole function or block around those lines.
3. Trace the failure scenario through the code. Follow each call it depends on: grep for definitions and read them at the head SHA.
4. Look for code that prevents the scenario: a guard, a validator, a caller that never passes that input, a test that covers it, or a framework default.
5. If the repo has a fast test or a short script that shows the failure, you may run it. Do not install anything or change files.

Answer in exactly this shape:

```
VERDICT: CONFIRMED | REFUTED | UNCERTAIN
EVIDENCE:
- <file:line>: <what the code does there>
- <file:line>: <...>
REASON: <one or two sentences>
```

- `CONFIRMED`: you traced the scenario to the wrong result and found nothing that prevents it.
- `REFUTED`: the described code does not exist where the claim says (at the head, or at the base for removed code), or code you can cite prevents the scenario.
- `UNCERTAIN`: the result depends on something you could not read, such as another service or runtime config. Name what you could not read.

For a re-review, the claim reads "finding <ID>'s failure scenario no longer happens at <sha>". `CONFIRMED` then means the fix holds. `REFUTED` means the scenario still happens.
