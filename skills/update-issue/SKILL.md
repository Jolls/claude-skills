---
name: update-issue
description: Use when the user wants to clarify and rewrite one or more GitHub issues before implementation — e.g. "/update-issue 123", "update issue #45", "clean up issues 12 34 56", "the wording on #90 is unclear, can you fix it up". Interviews the user to resolve ambiguity in an issue's intent, weighs whether the ask is worth doing, then proposes a rewritten title/body, labels, and milestone and asks before pushing to GitHub. Does not implement the issue or touch assignees/state.
---

# Update issue → clarify, then rewrite

> Source: [Jolls/claude-skills](https://github.com/Jolls/claude-skills) (`skills/update-issue`)
> — specialize downstream copies per-project; sync generic fixes both ways.

Turn a vague or stale GitHub issue into a title/body that unambiguously says what's wanted, with the user's sign-off at each step. Change only title, body, labels, and milestone: **no implementing, no assignees, no open/closed state.**

Uses the current repo unless the user names another. Label taxonomy and the "worth doing" signal come from AGENTS.md → "Issue triage" and ROADMAP.md.

## Multiple issues: prep in parallel, ask one at a time

With **two or more** issues, don't run the steps serially. First:

1. Fetch the label list and open milestones once.
2. Launch one Agent per issue in a single message, at a cheap model/effort (read-and-draft work). Each does steps 1, 2 and 4, must **not** ask the user or run `gh issue edit`, and writes a draft to a temp file outside the repo containing: the issue's current title/body/comments verbatim; the genuinely ambiguous points as questions with concrete options and a recommended one; any benefit-check flag; a proposed rewrite assuming the recommended options; proposed labels. It returns only the draft path.
3. Then go through the issues **one at a time**, in the order given: read that issue's draft and run steps 3 and 5-8 from it. Never put two issues' questions in one `AskUserQuestion` round, and don't start the next issue's questions until this one is pushed or skipped.

If an answer changes an issue's scope enough that the draft no longer fits, revise the rewrite yourself rather than relaunching the agent. With a single issue, skip this section.

## Steps (a full pass per issue number; never carry one issue's answers to another)

1. **Read the issue:** `gh issue view <N> --json title,body,labels,comments,number`. Use `--json`; plain-text `view` can silently return nothing.

2. **Form your own reading.** Decide whether it's a bug, feature, or task, and whether it reads as one clear thing or several. Read the comments; the real ask often evolved there.

   Then sanity-check the current code with a handful of grep/read calls: do the files/functions/entities it names still exist, and does an adjacent variant already in the app (an existing edit path, alternate format, batch flow) leave the request silent on a case that changes what's being asked? Understand the *request*, not the fix. Skip for pure wording/process cleanup.

3. **Show the issue as-is, then clarify.** Right before the first `AskUserQuestion`, echo the current title, body, and comments (each with its author) verbatim as quoted markdown, once per issue:
   ```
   ### #<N>: <title>
   > <body, quoted>
   ```
   Then `AskUserQuestion` only for what's genuinely unresolved: which interpretation, scope boundaries, acceptance criteria, edge cases. Don't guess and don't ask what the issue or comments already answer. Continue until someone else could build the right thing from the rewrite; that may be several rounds or none.

4. **Weigh the benefit.** Is it worth doing: who's affected, how often, cost of leaving it versus building it? If it now looks low-value, redundant, or off the project's direction, say so and ask the user (proceed, close, or descope) instead of polishing a weak issue. This is a flag, not a veto; most issues pass without discussion.

   **Already clear?** If title and body are already unambiguous, labels fit, and it has a milestone (or the repo has none), report "no changes needed" and move to the next issue, skipping steps 5-8.

5. **Present the rewrite:** the proposed title and body as you'd write them back (concise GitHub markdown; problem/expected behavior for a bug, motivation/proposal for a feature), not a design doc. Propose label changes from `gh label list` and the issue's current labels: keep what fits, drop what no longer does, add what's clearly missing. If the repo has no label convention, skip labels.

6. **Ask about milestone.** List open milestones (`gh api repos/{owner}/{repo}/milestones --jq '.[].title'`) and ask via `AskUserQuestion`. **Always offer "No milestone"**, even if one is set. If there are too many to list, offer the most plausible few (current one first); the user can type another via "Other". Skip if the repo has no milestones.

7. **Confirm before writing.** `AskUserQuestion`: update now, revise the rewrite (back to step 5), or drop this issue.

8. **Push the update.** Write the body to a temp file outside the repo and use `--body-file` (inline multi-line args get garbled by the shell):
   ```
   gh issue edit <N> --title "<new title>" --body-file <tmpfile> --add-label "<label>" --remove-label "<label>" --milestone "<name>"
   ```
   Repeat `--add-label`/`--remove-label` per label, omit either when unused, and use `--remove-milestone` for "No milestone".

## Guardrails

- Never touch assignees or state (open/closed).
- Never run `gh issue edit` without explicit approval in step 7, even if earlier answers seemed to imply it.
- If `gh` isn't authenticated or the issue doesn't exist, say so and stop.
