# Shared-tree cookbook

Concrete commands for the shared-tree adaptation in `agent-trunk-flow`.
Assumes several agent sessions share one working tree on one machine, and
a remote repo (e.g. GitHub) holds the record.

## 1. Initialize the tree as a real repo

Run once, from the canonical tree, with no other session writing:

```sh
cd /path/to/canonical-tree
git init -b main
git remote add origin <repo-url>
git add -A
git -c user.name="agent" -c user.email="agent@localhost" \
  commit -m "Initialize trunk from canonical tree

Performed-by: <agent-name> (<session-label>)."
git push -u origin main
```

## 2. Local main as a read-only mirror

After every push, and before starting risky work, refresh the mirror and
diff against it. Never commit on `main` directly from a session; routine
changes land on it only through the push transport after checks.

```sh
git fetch origin
git diff main...origin/main   # what the mirror is behind
git diff origin/main --stat   # what this tree changed since the mirror
```

If the tree and `origin/main` disagree about what is newest, stop and
reconcile before pushing. Pushing over a newer remote reverts someone
else's work.

## 3. Session checkpoint refs

Mark what a session produced without creating an integration branch:

```sh
git add -A
git commit -m "<imperative subject>

Performed-by: <agent-name> (<session-label>)."
git tag agent/<session>/<slug>   # or push a ref of the same name
```

For risky work that needs CI before landing on `main`, push the ref as a
real branch, wait for green CI, merge, then delete it:

```sh
git push origin HEAD:agent/<session>/<slug>
# wait for CI green, merge via the forge UI or CLI, then:
git push origin --delete agent/<session>/<slug>
```

## 4. Push mutex

One lock, one push at a time. A lockfile with a stale timeout is enough:

```sh
LOCK=/path/to/canonical-tree/.push-mutex.lock
if ! mkdir "$LOCK" 2>/dev/null; then
  echo "Another session is pushing; waiting."
  exit 1
fi
trap 'rmdir "$LOCK"' EXIT
# run checks, then push
```

Stale locks (holder died mid-push) need a timeout: record the holder's
PID and start time in the lock dir, and break the lock if the holder is
gone.

## 5. Drift check (scheduled)

The check that keeps the repo honest. Run hourly or nightly from a
scheduler, read-only:

```sh
cd /path/to/canonical-tree
git fetch origin
BEHIND=$(git rev-list --count HEAD..origin/main)
AHEAD=$(git rev-list --count origin/main..HEAD)
UNTRACKED_CHANGED=$(git status --porcelain | wc -l)
if [ "$BEHIND" -gt 0 ] || [ "$AHEAD" -gt 0 ] || [ "$UNTRACKED_CHANGED" -gt 0 ]; then
  echo "DRIFT: tree and repo differ (behind=$BEHIND ahead=$AHEAD dirty=$UNTRACKED_CHANGED)"
  # alert: open an issue, send a message, page the owner, per team convention
fi
```

Divergence means either unpushed session work (push it) or a remote change
the tree never pulled (reconcile before the next push).

## 6. Session brief snippet

Paste into every agent brief that touches the tree:

> Canonical tree: /path/to/canonical-tree (the only tree you may change).
> Local `main` is a read-only mirror of origin/main; never commit on it.
> Keep changes small and atomic. Run the test suite before committing.
> Every commit carries `Performed-by: <your-name> (<session-label>).`
> Push through the mutex. If the drift check is red, reconcile first.
