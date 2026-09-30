---
name: agent-trunk-flow
description: >
  Trunk-based development for AI agent teams: one main branch, atomic
  commits at every checkpoint, CI as the reviewer, branches that live
  hours not days. Includes the shared-tree adaptation for fleets of agent
  sessions working in one working tree (local main as a read-only mirror,
  session refs, a push mutex, attribution trailers) and anti-patterns such
  as nightly batch merges. Use when setting up or running a multi-agent
  development workflow, or whenever the user says "agent-trunk-flow" or
  "trunk flow".
---

# agent-trunk-flow

Trunk-based development, adapted for teams of AI agents.

## Why it fits agent teams

Big orgs that deploy constantly run on a single trunk: Google lands on the
order of 80k commits a week on one branch, Meta reviews stacked diffs
instead of giant PRs, Shopify merges many times a day. Long-lived branches
exist to catch bad integrations between human developers. An agent team on
one tree rarely has that problem, so the branches add friction without
adding safety.

Agents make trunk-based flow more natural: no egos, no "my branch", and
superhuman speed rewards small, frequent integration. But agents need
explicit machinery where humans use judgment: when to branch, when to push,
and how to stay inside CI's speed envelope.

## Principles

1. **Single trunk.** `main` is the only long-lived branch and is always
   deployable.
2. **Atomic commits.** One logical change per commit, with its tests.
3. **Integrate at every checkpoint.** Agents have no "end of day", so push
   at every logical completion. Never batch into nightly merges.
4. **CI is the reviewer.** Humans do not scale to agent speed; green CI is
   the hard gate.
5. **Branches live hours, not days.** A branch older than one working
   session is a smell.
6. **Hide unfinished work behind flags or config, not branches.** Feature
   flags, config toggles, and draft states keep `main` shippable while
   work lands incrementally.
7. **Auto-revert on breakage.** If `main` breaks, the fix is a revert
   first and a diagnosis second.

## The shared-tree adaptation

Many agent fleets run several sessions against ONE working tree (common on
agent VMs). Classic trunk flow assumes one tree per developer, so adapt it:

- **The tree IS the trunk.** Git on the tree is a local safety net: diff,
  blame, revert, stash.
- **Keep a local `main` as a read-only mirror of `origin/main`.** Never
  commit on it. Pull it, diff against it, treat it as a reference.
- **Session checkpoints go on `agent/<session>/<slug>` refs.** They mark
  what a session produced; they are not integration branches.
- **Keep each change small** so sessions do not clobber each other's
  files. Small diffs merge; big rewrites collide.
- **Serialize pushes with a mutex.** Two sessions pushing at once will
  race; one lock, one push at a time.
- **Every push carries an attribution trailer** so the log shows which
  session shipped what.
- **A push transport ships finished work to remote** (API, CLI, or plain
  `git push`). The local tree is the workspace; the remote repo is the
  record.

Exact commands are in `references/shared-tree-cookbook.md`.

## Superhuman-speed adaptations

- **Conflicts arrive faster, so changes get smaller and integration more
  frequent.** If two sessions keep touching the same files, split the work
  by file ownership, not by time.
- **CI must stay under ~10 minutes or agents outrun it.** A slow suite
  means agents stack unpushed work on a green assumption. Trim the suite,
  parallelize it, or split fast checks (lint, unit) from slow ones (e2e)
  and gate pushes on the fast set.
- **Unpushed work, not stale branches, is the main risk.** Run a scheduled
  drift check that diffs the tree against the repo and alerts on
  divergence. The repo must never lag the tree: a stale repo recreates
  stale-checkout reverts, where a session redeploys an old snapshot over
  finished work.

## Branch policy

Branch ONLY for: bulk rewrites (50+ files), broad refactors, risky
migrations, or experiments you might throw away.

- Name: `agent/<session>/<slug>`, e.g. `agent/nightly-seo/copy-refresh`.
- CI must be green before merging.
- Merge within hours, then delete the branch.
- Default to direct-to-main for everything else. The solo-team insight:
  integration branches catch disagreements between developers; one agent
  team on one tree has none, so the branch is pure overhead.

## Commit conventions

- Atomic: one logical change, with its tests, per commit.
- Subject line: imperative mood, under 72 chars. `Add French hreflang to
  insight pages`, not `Added stuff`.
- Attribution trailer on every commit, for fleet audit trails:
  `Performed-by: <agent-name> (<session-label>).`
  Example: `Performed-by: Alexa (cron:daily-insight).`

## Review gate

1. The pushing agent always runs a `git diff` self-review first and reads
   the full diff, not just the file list.
2. CI (build, tests, linters, policy-as-code checks) is the hard gate and
   must be green before merge. No green, no merge, no exceptions.
3. For risky changes, a second fresh agent reviews: maker-checker. The
   reviewer receives only the diff plus the requirements, never the build
   transcript. A builder auditing its own work finds nothing.

## Anti-patterns

- **Nightly batch merges.** They turn `main` into a stale backup of the
  tree.
- **Long-lived dev branches.** They accumulate merge risk instead of
  catching it.
- **Mega-commits bundling unrelated changes.** Unreviewable, unrevertable.
- **Pushing without CI green.** The one rule that is never negotiable.
- **Requiring a human approval per change.** It kills agent autonomy; the
  gate is CI plus attribution.
- **Letting the repo drift from the tree.** The drift check exists to
  catch this.

## Quickstart

1. `git init` (or clone) in the canonical working tree; set `origin` to
   the repo.
2. Enable branch protection on `main`: require CI green before merge, no
   direct bypass for agents.
3. Adopt the attribution trailer convention in every commit.
4. Add a scheduled drift check (hourly or nightly) comparing tree vs repo;
   alert on divergence.
5. Add a push mutex if more than one session can push.
6. Document the tree path as canonical in the team's standing notes; every
   session brief names it explicitly.
