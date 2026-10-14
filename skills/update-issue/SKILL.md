---
name: update-issue
description: Use when the user wants to clarify and rewrite one or more GitHub issues before implementation — e.g. "/update-issue 123", "update issue #45", "clean up issues 12 34 56", "the wording on #90 is unclear, can you fix it up". Interviews the user to resolve ambiguity in an issue's intent, weighs whether the ask is worth doing, then proposes a rewritten title/body, labels, and milestone and asks before pushing to GitHub. Does not implement the issue or touch assignees/state.
---

# Update issue → clarify, then rewrite

> Source: [Jolls/claude-skills](https://github.com/Jolls/claude-skills) (`skills/update-issue`)
> — specialize downstream copies per-project; sync generic fixes both ways.

Your job: take a vague or stale GitHub issue and turn it into a title/body that unambiguously says what's wanted, with the user's sign-off at each step. **Do not implement anything, and do not touch assignees/state (open/closed)** — this skill clarifies and rewrites issue text, labels, and milestone only.

Uses the current repo (`gh` infers it from the working directory) unless the user names a different one. Label taxonomy and the "worth doing" signal come from AGENTS.md → "Issue triage" and ROADMAP.md.

## Steps (repeat per issue number given)

1. **Read the issue.**
   ```
   gh issue view <N> --json title,body,labels,comments,number
   ```
   Use `--json`; plain-text `view` can silently return nothing.

2. **Form your own reading.** Work out what you think the issue is asking for — bug report, feature request, task — and whether it reads as one clear thing or several plausible things. Use the comments too; often the real ask evolved there and the original body is stale.

   Then do a bounded sanity check against the current codebase (a handful of grep/read calls): confirm the files/functions/entities the issue names still exist, and check whether an adjacent variant already in the app (an existing edit path, alternate format, related batch flow) leaves the request silent on a case that determines what's being asked. This is about understanding the *request*, not designing the fix. Skip it for pure wording/process cleanup with no code-facing claim.

3. **Show the issue as-is, then clarify.** Immediately before your first `AskUserQuestion` round, print the current title and body back to the user as a clean markdown block, e.g.:
   ```
   ### #<N>: <title>
   > <body, quoted>
   ```
   Include existing comments the same way if there are any (each as its own quoted block with its author). Echo the text as-is, not a paraphrase, so the user answers against the actual wording. Show it once per issue, right above the first questions.

   Then use `AskUserQuestion` for anything the issue leaves ambiguous or missing: which interpretation is meant (when there's more than one), scope boundaries, acceptance criteria, edge cases the issue is silent on. Don't silently pick an interpretation and don't pad this with questions you can already answer from the issue/comments — only ask what's genuinely unresolved. Keep going, question after question, until you're confident you could hand the rewritten issue to someone else and they'd build the right thing. It's fine for this to take several rounds on a messy issue and zero rounds on a clear one.

4. **Weigh the benefit.** Once the ask is unambiguous, pause and think about whether it's actually worth doing — not just what it says, but why it matters: who's affected, how often, what it costs to leave unfixed/unbuilt versus what it costs to build. If the clarified ask now reads as low-value, redundant with something else, or out of step with the project's stated direction (roadmap doc, milestone plan, or similar, if the project has one), say so plainly and raise it with the user (e.g. via `AskUserQuestion` — proceed anyway, close instead, or descope) rather than quietly rewriting a weak issue into a polished one. This is a sanity check, not a veto — most issues will clear it without any back-and-forth.

   **Already clear?** If the title and body already say unambiguously what's wanted and the labels fit, and the issue has a milestone (or the repo has none), report "no changes needed" and go to the next issue, skipping steps 5-8.

5. **Present the rewrite.** Show the user the proposed title and body as you'd actually write it back — GitHub-flavored markdown, concise, structured (e.g. problem/expected-behavior for a bug, motivation/proposal for a feature). This is a rewrite of the issue's own text reflecting what you both just clarified, not a design doc or implementation plan. Alongside it, propose label changes: check the repo's existing label set (`gh label list`) and this issue's current labels, and suggest additions/removals that now fit the clarified ask (severity, area/component, type) — carry forward labels that still fit, drop ones that no longer do, and add ones that are clearly missing. If the repo has no discernible label convention, skip this rather than inventing one.

6. **Ask about milestone.** Fetch the repo's open milestones (`gh api repos/{owner}/{repo}/milestones --jq '.[].title'`, filling in the current repo) and ask the user which one this issue belongs to via `AskUserQuestion`. **Always include "No milestone" as an option**, even when the issue currently has one assigned — moving an issue off its milestone is a legitimate answer, not just leaving it unset. If there are more milestones than fit in one question, offer the most plausible few (favor the issue's current milestone and whichever milestone its clarified scope fits) plus "No milestone"; the user can always type a different one via "Other". If the repo has no milestones at all, skip this step.

7. **Confirm before writing.** `AskUserQuestion`: update the issue now (text + labels + milestone), revise the rewrite further, or drop this issue and move to the next one. Loop back to step 5 on a revise.

8. **Push the update.** On approval, write the body to a temp file first — inline heredocs/here-strings for multi-line `gh` args are prone to getting garbled by the shell, so go through `--body-file` rather than passing the body inline. Apply the title/body, label, and milestone changes together:
   ```
   gh issue edit <N> --title "<new title>" --body-file <tmpfile> --add-label "<label>" --remove-label "<label>" --milestone "<name>"
   ```
   Repeat `--add-label`/`--remove-label` per label changed; omit either flag entirely if there's nothing to add or remove. For "No milestone", use `--remove-milestone` instead of `--milestone`. Use a scratch/temp directory for the body file, not the repo itself.

9. **Move to the next issue number**, if more were given. Each issue gets its own full pass through steps 1-8 — don't batch clarification or confirmation across issues; a user answer about issue #12 shouldn't be assumed to apply to #34.

## Guardrails

- Never touch assignees or state (open/closed) — title, body, labels, and milestone only.
- Never call `gh issue edit` without explicit approval in step 7, even if earlier answers seemed to imply consent.
- If `gh` isn't authenticated or the issue number doesn't exist, say so and stop rather than guessing.
