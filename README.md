# agent-trunk-flow

Trunk-based development, adapted for teams of AI agents.

Big orgs that deploy constantly run on a single trunk: Google lands on the
order of 80k commits a week on one branch, Meta reviews stacked diffs
instead of giant PRs, Shopify merges many times a day. This skill adapts
that practice for agent fleets: one main branch, atomic commits at every
checkpoint, CI as the reviewer, branches that live hours not days.

It also covers the **shared-tree adaptation** for fleets of agent sessions
working in one working tree: the tree IS the trunk, local git is the
safety net (diff, revert, stash), a push transport ships to remote, and a
scheduled drift check keeps the repo from lagging the tree.

## Install

With any skill manager that reads `SKILL.md`:

```sh
npx skills add movahedi-ca/agent-trunk-flow
```

Or copy `SKILL.md` (plus `references/`) into your agent's skill directory.

## What's inside

- `SKILL.md` — the full workflow: principles, shared-tree adaptation,
  superhuman-speed adjustments, branch policy, commit conventions,
  review gate (CI plus maker-checker), anti-patterns, quickstart.
- `references/shared-tree-cookbook.md` — concrete commands for the
  shared-tree setup.

## License

MIT. Copyright (c) 2026 Mohammad Movahedi.
