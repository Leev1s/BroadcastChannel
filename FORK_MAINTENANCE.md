# Fork maintenance

This fork intentionally carries a small set of local customizations on top of
`miantiao-me/BroadcastChannel`. Upstream updates are applied manually so they
can be reviewed and tested before replacing the deployment branch.

## Repository layout

- `origin`: `Leev1s/BroadcastChannel`
- `upstream`: `miantiao-me/BroadcastChannel`
- `main`: the Cloudflare deployment branch
- `src/styles/app/fork.css`: isolated visual overrides

Automatic upstream-sync workflows must not write to `main`. They can merge
unreviewed upstream changes into the deployed branch and make the fork history
harder to rebase.

## Safe upstream update

Start from a clean worktree and record the expected remote commit before doing
anything destructive:

```sh
git fetch origin upstream --prune
git switch main
git branch codex/backup-main-YYYYMMDD
git switch -c codex/update-upstream-YYYYMMDD
git rebase upstream/main
```

Resolve conflicts by preserving upstream behavior first, then reapplying the
small fork-specific overrides. Validate the rebased branch:

```sh
pnpm install --frozen-lockfile
pnpm lint
pnpm typecheck
pnpm test
pnpm build
```

After reviewing the range `upstream/main..HEAD`, update the deployment branch
with an exact force-with-lease guard:

```sh
git push --force-with-lease=main:<expected-origin-sha> origin HEAD:main
git branch -f main HEAD
git switch main
```

Keep the dated local backup until the Cloudflare deployment and GitHub Actions
runs are healthy. Do not use GitHub's **Sync fork** button or merge
`upstream/main` into this fork.
