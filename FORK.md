# Fork strategy

This repository is a downstream fork of
[`canonical/opensearch-dashboards-snap`](https://github.com/canonical/opensearch-dashboards-snap)
(git remote `upstream`), producing the **wazuh-dashboard** snap instead of the
upstream **opensearch-dashboards** snap.

## Branches

- **`main`** — the actively developed branch. It diverged from upstream at
  `15e296d` ("Switch Jira issue sync from workflow to bot (#17)", 2024-08-19)
  and has since accumulated its own ad-hoc history of Wazuh-specific changes
  (product rename, apt-package build, CI/Jira automation, etc.), not meant to
  be replayed commit-by-commit onto future upstream syncs.
- **`2/edge`** — the branch published to the snap store's `2/edge` channel.
  It is rebuilt to be **upstream's `2/edge` branch plus a small, well-scoped
  set of squashed Wazuh commits on top**, so it can be rebased cleanly as
  upstream evolves.
- **`2-edge-old-mirror`** — backup of the previous `2/edge`, which was a pure,
  unmodified mirror of `upstream/2/edge` (no Wazuh changes at all). Kept only
  for reference; safe to delete once the new `2/edge` is validated.

## Key commits

| What | Commit | Note |
|---|---|---|
| Divergence point (`main` vs. `upstream`) | `15e296d` | Last commit shared between `main` and the upstream `2/edge`/`3/edge` lineage. |
| Last upstream commit applied to `2/edge` | `351b06f` | Tip of `upstream/2/edge` at the time of this rebuild ("Merge pull request #50 ... add-security-events-permissions"). `2/edge` is based directly on this commit. |

## Squashed Wazuh commits on `2/edge`

On top of `351b06f`, `2/edge` carries exactly four commits, each a squashed,
self-contained rework of one concern (reconstructed from `main`'s 42-commit,
unsquashed history so future rebases have a small, predictable conflict surface):

1. **Build wazuh-dashboard from Wazuh apt packages** — replaces the OpenSearch
   Dashboards Launchpad tarball build with Wazuh's own apt package
   (`wazuh-dashboard=4.11.0-1`), renames the snap, updates paths/env vars.
2. **Update README and CONTRIBUTOR docs for Wazuh Dashboard**.
3. **Update CI and release workflows for the Wazuh Dashboard build**.
4. **Add GitHub-to-Jira sync automation and a security policy**.

The end result is verified to have a working tree identical to `main`.

## Syncing with upstream (rebase strategy)

When upstream publishes new commits on `2/edge` (or `3/edge`, following the
same pattern):

```sh
git fetch upstream

# See what's new upstream since our last-applied commit:
git log 351b06f..upstream/2/edge --oneline

# Rebase our 4 squashed commits onto the new upstream tip:
git checkout 2/edge
git rebase --onto upstream/2/edge 351b06f
```

Update the "last upstream commit applied" reference in this file to the new
`upstream/2/edge` tip once the rebase succeeds.

### Use `git rerere`

This repo relies on `git rerere` (reuse recorded resolution) to avoid
re-resolving the same conflicts on every rebase. Enable it locally once:

```sh
git config rerere.enabled true
git config rerere.autoupdate true
```

`rerere`'s cache (`.git/rr-cache`) is local to your clone and not committed.
The first time you resolve a conflict it is remembered; subsequent rebases
that hit the same conflict are resolved automatically. Because conflicts are
now isolated to four small, scoped commits instead of a 42-commit noisy
history, the conflict surface — and the value of `rerere` — stays high.

### Keep the squash structure

After a successful rebase, if any of the four commits grew messy conflict
resolution noise, squash it back down (`git rebase -i`) so the branch keeps
exactly the small set of scoped commits described above. Do not let `2/edge`
accumulate an unbounded, unsquashed patch history again — that is what made
the previous `main` history hard to maintain.

## Publishing

`2/edge` is force-pushed as a maintained fork branch (its history is expected
to be rewritten on every upstream sync). Coordinate with the team before
force-pushing to `origin/2/edge`, and make sure CI/release automation is
aware this branch's history is not append-only.
