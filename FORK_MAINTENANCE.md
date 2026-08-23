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

## Cloudflare Workers deployment

Production runs on the `broadcast-channel` Worker. The custom domain
`blog.r3.net.eu.org/*` is declared in `wrangler.jsonc`, and the independent
validation URL is `https://broadcast-channel.lev1s.workers.dev`.

The legacy `telegram-channel-blog` Pages project has been removed. Do not deploy
this Astro SSR application to Pages: the upstream project supports Cloudflare
Workers only.

Runtime variables and the automatically provisioned `SESSION` KV binding live
in Cloudflare. `keep_vars` preserves them on later deployments; never commit
their values to this repository.

Cloudflare Workers Builds is connected to `Leev1s/BroadcastChannel` and deploys
the `main` branch with the upstream project's official commands:

```sh
SERVER_ADAPTER=cloudflare_workers pnpm build
pnpm exec wrangler deploy
```

A push to `main` is the production deployment trigger. The CI workflow remains
a separate verification gate; it does not deploy. Use the same commands from an
authenticated local machine only when diagnosing a build or intentionally
performing a manual recovery. Verify both the workers.dev URL and the custom
domain before considering a deployment complete.

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
SERVER_ADAPTER=cloudflare_workers pnpm build
pnpm exec wrangler deploy --dry-run --outdir /tmp/broadcast-channel-dry-run
```

After reviewing the range `upstream/main..HEAD`, update the deployment branch
with an exact force-with-lease guard:

```sh
git push --force-with-lease=main:<expected-origin-sha> origin HEAD:main
git branch -f main HEAD
git switch main
```

Wait for the Workers Builds deployment triggered by the `main` push. Keep the
dated local backup until the custom domain, workers.dev URL, Workers build, and
GitHub Actions run are all healthy. Do not use GitHub's **Sync fork** button or
merge `upstream/main` into this fork.
