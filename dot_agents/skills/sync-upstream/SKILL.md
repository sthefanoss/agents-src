---
name: sync-upstream
description: Fetches upstream/main in ~/.agents-src, reports the incoming commits, changed paths under dot_agents/, and any overlap with local adaptations, and merges on approval while refreshing the ~/.agents, ~/.claude, and ~/.gemini/config symlinks. Judging an outside instruction source item by item belongs to absorb. Runs only when typed.
disable-model-invocation: true
---

# Sync upstream

## Inspect before touching the tree

Run `git -C ~/.agents-src fetch upstream` first so `upstream/main` reflects the
current remote state, and check `git -C ~/.agents-src rev-list --count
HEAD..upstream/main`. When the count is `0`, report that `HEAD` already includes
`upstream/main` and stop.

When `upstream/main` has new commits, read these outputs without modifying the
working tree:

- `git -C ~/.agents-src status --short`
- `git -C ~/.agents-src log --oneline HEAD..upstream/main`
- `git -C ~/.agents-src log --oneline upstream/main..HEAD`
- `git -C ~/.agents-src diff --name-status HEAD...upstream/main -- dot_agents/ .gitignore`
- `git -C ~/.agents-src diff --name-status upstream/main...HEAD -- dot_agents/ .gitignore`

Report the incoming commit count, the incoming added, modified, renamed, and
deleted paths under `dot_agents/`, the local commits and paths changed since the
merge base, and every path modified on both sides. End the turn there and wait
for approval before merging, because merging alters live instruction files that
concurrent sessions load.

## Merge and preserve local adaptations

After approval arrives, check `git -C ~/.agents-src status --short` and stop and
report any uncommitted path if it prints anything, so another session's work
stays out of the merge commit.

Run `git -C ~/.agents-src merge --no-commit --no-ff upstream/main` and reconcile
the local delta between this fork and `upstream/main`:

- Keep `.gitignore` entries and skill directories under `dot_agents/skills/`
  added in `HEAD` (`git -C ~/.agents-src checkout --ours .gitignore && git -C
  ~/.agents-src add .gitignore`).
- In `dot_agents/AGENTS.md`, take `upstream/main`'s file and re-apply the local
  delta from `git -C ~/.agents-src diff upstream/main...HEAD --
  dot_agents/AGENTS.md` (`~/.agents-src` and `$HOME` symlinks in place of
  chezmoi and `~/code/dotfiles`, no `using-the-wiki.md` row in the read table,
  and the second-language correction paragraph).
- Remove any path deleted in `upstream/main` that conflicted only because an
  earlier local commit had edited it (`git -C ~/.agents-src rm -f <path>`),
  unless the approval kept that path.

Confirm `git -C ~/.agents-src diff --name-only --diff-filter=U` prints nothing,
then commit the merge.

## Refresh symlinks and verify

Remove broken symlinks that pointed at paths deleted or renamed upstream:

```sh
find -H ~/.agents ~/.claude ~/.gemini/config -type l ! -exec test -e {} \; -print -delete
```

Link every top-level entry in `$HOME/.agents-src/dot_agents` except `agents`
into `~/.agents/`, and link every `.md` file in
`$HOME/.agents-src/dot_agents/agents` into `~/.agents/agents/`. Confirm that
`~/.claude` links `CLAUDE.md`, `rulebook`, `skills`, and `agents` to their
`~/.agents/` counterparts, and that `~/.gemini/config` links `AGENTS.md` and
`skills` to `~/.agents/AGENTS.md` and `~/.agents/skills`.

Verify that `git -C ~/.agents-src status --short` prints nothing and that `find
-H ~/.agents ~/.claude ~/.gemini/config -type l ! -exec test -e {} \;` prints
nothing. Report the merge commit hash and the paths updated under `dot_agents/`.
