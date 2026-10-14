---
name: implement-issues
description: Use when the user hands you a batch of GitHub issues (or a PR-slate grouping from an epic, e.g. "issue A → [B+C] → D") and wants each one updated, evaluated, planned, and implemented sequentially on a single branch, landing in one combined PR to main. Triggers on "evaluate and implement issues #NNN, #NNN", "work through the [epic] slate", "plan then implement these issues". Not for a single ad-hoc bug fix (just do it directly) or for planning without implementing (use /evaluate-issue or Plan Mode alone).
---

# Implement a batch of issues on one branch, one combined PR

> Source: [Jolls/claude-skills](https://github.com/Jolls/claude-skills) (`skills/implement-issues`)
> — specialize downstream copies per-project; sync generic fixes both ways.

Each **group** (one issue, or a bundle that ships together) gets a plan. Implementation is sequential in **one working tree on one branch**, left **uncommitted** until the human tests the cumulative tip and approves. Each group is built **test-first**, so the human only clicks through what tests can't prove.

## 0. Confirm groups and order

Respect the user's order (e.g. "752 → [753+755] → 754"); otherwise propose one (dependencies first, then ascending risk) and confirm before planning. Ask whether they want a **per-group checkpoint** (pause after each group for a manual check; recommend for large batches or lots of manual-only surface) or only the end-of-batch test.

## 1. Update each issue

Run `/update-issue <NNN>` on **every** issue, even clear ones, so each can be planned without judgment calls.

## 2. Evaluate each issue

Run `/evaluate-issue <NNN>` on every issue; it sets the model/level for each group's planning agent. For a bundle, combine: model = higher tier if *any* member trips it; level = the highest (or name both and pick).

## 3. Plan each group — parallel agents

Launch one Agent per group in a single message, at the recommended model/level. Prompt each with:
- The issue body/locations verbatim (no `gh` re-fetch).
- **"Provide a plan with specific file changes to implement the plan. Only enough context needed to implement. No judgement calls in the plan. If unsure, ask."**
- Plan path `docs/plans/<issue-ids>-<slug>.md`. It's a scratch file: step 5 moves it into the issue and deletes it, so it is never committed.
- Read the current code at every referenced location first; issue line numbers go stale.
- A **Test plan** with four parts:
  1. **Coverage audit:** existing tests covering what the change touches (file + test name).
  2. **Characterization tests:** new tests pinning *current* behavior the change could break that nothing covers. Must pass on unchanged code. Mark any that lock in a bug the change fixes.
  3. **Red tests:** acceptance criteria as tests that fail now and pass after. Name each, what it asserts, why it fails today.
  4. **Manual-only:** what can't be automated (visual layout, client JS, OS dialogs), as short bullets.

  List exact rows for any new fixture/seed data and note the human refresh step. If no tests are needed, write `No tests: <few-word reason>`.
- **Return only the plan path + open questions**, not the plan body or investigation notes.

## 4. Resolve open questions

Ask each open question with `AskUserQuestion`, one per item, concrete options (not yes/no), recommended first. Write the answer into the plan as a "Resolved decision" with the exact change spec, so no judgment calls remain. The user's go-ahead on the plans also approves the tests they list; `No tests:` needs no separate sign-off.

## 5. Implement sequentially — uncommitted

Continue in this session; no sub-agent. If a plan needs much higher effort than this session (e.g. schema/auth-heavy), ask the user rather than implementing it at low effort.

**Unplanned judgment call:** don't guess. If it only needs the user's call, use `AskUserQuestion`. If framing the options needs investigation, spawn one narrow high-effort Agent for *that question only*, then ask.

Branch off main once for the whole batch:
```
git checkout main
git checkout -b feature/<batch-slug>
```

**Move each plan into its issue** before the first group: `gh issue comment <N> --body-file docs/plans/<file>.md` (every member of a bundle), then delete the file. The comment is the plan from here on; edit it rather than recreating a file.

Then per group, in apply order:
1. **Characterization tests:** write and run against unchanged code. They must **pass**; a failure is an existing bug or wrong assumption, so stop and ask.
2. **Red tests:** write and run. They must **fail on an assertion** for the plan's reason. Stub new signatures first so a compile error doesn't count. A red test that already passes means the plan is wrong; stop and ask.
3. **Change:** apply the plan's edits until everything passes (flip any characterization test the plan marked as locking in a bug).
4. If the group touched schema/migrations or code covered by an integration suite, run that suite and report pass/fail. On a stale-fixture/seed failure, ask the user to refresh it; never do it yourself.
5. Run `/code-review low` on the group's incremental diff **before the next group**; fix what it flags and re-verify.
6. With per-group checkpoints: pause, give the group's manual-only items and golden path, and wait for the OK.
7. **Do not commit.**

`No tests` groups skip 1-2.

## 6. Human tests the tip

Before handing off, give a bullet **manual test-points list** for the whole batch, based on the actual diff. Lead with the plans' manual-only items, note which tests cover the rest, and skip anything already checked at a checkpoint unless a later group touched it. Cover:
- **Entry points touched** and how to reach each (route, nav path, CLI command).
- **New/changed UI elements** and the expected result of each interaction.
- **Golden path** per feature, end to end.
- **Edge cases:** empty, duplicate, or bad input, permission boundaries, anything the plan's success criteria named.
- **Cross-feature regressions:** features sharing the same data, handlers, or helpers.
- **Observable state** to check afterward (row written, audit entry) when the UI doesn't show it.
- Any special test mode/environment required.

Before any commit, recommend the checks the diff warrants: a deeper code review over the combined diff if it exceeds ~300 changed lines, spans multiple `internal/` packages, or touches anything AGENTS.md → "Issue triage" lists as schema/database or security-sensitive; integration tests if not already run; a `/simplify` pass if the batch introduced reuse or simplification opportunities. Run the chosen passes, summarize, fix. **Commit only on the user's go-ahead.**

## 7. On approval: commit, push, one combined PR

If **no file is touched by more than one group** (known from the plans), commit per group, staging only that group's files, with a message citing its issue. Otherwise make **one commit for the batch**; `git add -p` is interactive and whole-file staging can't split a shared file.

Commit hygiene:
- Changelog: follow AGENTS.md → "Keep the changelog": one new version heading for the PR, all groups' changes under it, grouped by category.
- Plan files were moved into the issues in step 5; don't commit them.
- Don't tag; tagging happens at release.

Write the PR body to a file (heredocs garble multi-line content), then:
```
git push -u origin feature/<batch-slug>
gh pr create --title "<summarize the batch>" --body-file <path>
```
PR body: one bullet per issue's change, `Closes #NNN` on its own line per issue (bare numbers after a comma don't auto-close), and a test-plan checklist of what was verified: tests per group (characterization + red → green), build/test and low-effort review per group, any deeper pass, and the human's manual test.

## 8. Clean up

Once the user confirms the PR is merged, run `/done`.
