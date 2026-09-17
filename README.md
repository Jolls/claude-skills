# claude-skills

Reusable [Claude Code](https://claude.com/claude-code) skills, generalized from project-specific
versions so they can be dropped into any repo.

## Usage

Copy the skill folder(s) you want into your project's `.claude/skills/`, e.g.:

```
cp -r skills/done <your-project>/.claude/skills/done
```

## Skills

- **done** — after a PR merges and its remote branch is deleted, sync local `main` and remove the
  stale local branch(es).
- **evaluate-issue** — scope a GitHub issue and recommend the model/reasoning level (and whether
  sub-agents help) for the session that implements it. Recommends only, doesn't implement.
- **implement-issues** — given a batch of GitHub issues, plan and implement them sequentially on
  one branch, landing in a single combined PR.
- **update-issue** — clarify a vague/stale GitHub issue through an interview with the user, weigh
  whether it's worth doing, then rewrite its title/body and labels. Confirms with the user before
  pushing at each step; doesn't implement or touch milestone/assignees/state.

`evaluate-issue`, `implement-issues`, and `update-issue` are project-agnostic but read better once
you tell them (via your project's CLAUDE.md or similar) what counts as "schema," "security-sensitive,"
your label taxonomy, etc. in your codebase — see the "Adapting this skill" note at the bottom of each.
