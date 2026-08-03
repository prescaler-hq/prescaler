# Prescaler development workflow

Canonical process for `prescaler-api`, `prescaler-admin`, `prescaler-app`, `prescaler-web`, and `prescaler-ide-v2`.

## Branches

| Branch | Purpose | Deploys |
|---|---|---|
| `main` | Production-ready code | Production |
| `develop` | Integration and preview testing | Preview/staging |
| `release/production` | Release candidate when needed | Manual candidate |
| `release/YYYY-MM-DD` | Temporary dated release branch | Manual candidate |
| `feature/<ticket>-<slug>` | One bug, feature, or epic | None |
| `hotfix/<ticket>-<slug>` | Urgent production fix from `main` | None |

Use the same branch name in every affected repository. Keep `main` and `develop` protected; merge through pull requests.

## Change size

- Minor bug or small feature: branch from `develop`; one focused PR.
- Major feature or broad refactor: branch from `develop`; use one epic branch plus smaller PRs when practical.
- Production incident: branch from `main` as `hotfix/...`; merge to `main`, then merge the same fix into `develop`.
- Database, billing, auth, or API contract change: update `prescaler-api` first and update every affected client in the same change set.

## Start work

```bash
cd /Users/loveveersingh/Downloads/Prescaler/<repo>
git status --short --branch
git fetch origin --prune
git switch develop
git pull --ff-only origin develop
git switch -c feature/<ticket>-<slug>
```

For a hotfix:

```bash
git fetch origin --prune
git switch main
git pull --ff-only origin main
git switch -c hotfix/<ticket>-<slug>
```

## Work across repositories

```bash
for repo in prescaler-api prescaler-admin prescaler-app prescaler-web prescaler-ide-v2; do
  cd /Users/loveveersingh/Downloads/Prescaler/$repo
  git switch -c feature/<ticket>-<slug> develop
done
```

Only create the branch in repositories that actually change. Keep the branch name identical across repositories that do change.

## Before committing

```bash
git diff --check
git status --short
```

Run the checks for the repository:

```bash
# prescaler-api
npm run check:env && npm run typecheck && npm test && npm run build

# prescaler-admin
npm run check:gateway && npm run typecheck && npm run build

# prescaler-app
npm run check:gateway && npm run typecheck && npm run lint && npm test && npm run build

# prescaler-web
npm run typecheck && npm run lint && npm test && npm run build

# prescaler-ide-v2
npm run build
```

## Commit and push

```bash
git add <specific-files>
git commit -m "fix(scope): describe the change"
git push -u origin HEAD
```

Use conventional prefixes: `fix`, `feat`, `chore`, `refactor`, `docs`, `test`, `perf`.

Never commit `.env.local`, `.env.prod`, secrets, service keys, database passwords, or generated build output.

## Pull request order

1. Open PRs from feature/hotfix branches.
2. Merge API contract, migration, or billing changes first.
3. Merge dependent app/admin/web/IDE changes after API checks pass.
4. Merge to `develop` for preview testing.
5. Promote the tested commits to `main` through a PR.

## Database migrations

Apply to test before production:

```bash
cd /Users/loveveersingh/Downloads/Prescaler/prescaler-api
supabase link --project-ref <test-project-ref>
supabase migration list
supabase db push --dry-run
supabase db push --yes

supabase link --project-ref <production-project-ref>
supabase migration list
supabase db push --dry-run
supabase db push --yes
```

Migrations must be additive and rerunnable. Never edit an already-applied migration to change its meaning; add a new migration instead. Reconcile migration-history drift before using `--include-all` or repair commands.

## Production release

1. Confirm test migrations and preview deployment are healthy.
2. Merge the release PR to `main` in each affected repository.
3. Deploy `prescaler-api` first.
4. Deploy dependent admin/app/web/IDE services.
5. Verify `/v1/health`, authentication, billing, AI requests, usage, and admin access.
6. Monitor logs, Stripe webhooks, rate limits, and error rates.

Vercel production should follow `main`; preview should follow `develop` or PR deployments. Do not paste test values into production or live Stripe values into local environments.

## Rollback

For code:

```bash
git revert <bad-commit>
git push origin main
```

For Vercel, redeploy the last known-good deployment. Do not rewrite shared branch history. For database changes, use a forward corrective migration; do not reset production.

## Branch cleanup

After a PR is merged:

```bash
git fetch origin --prune
git branch -d feature/<ticket>-<slug>
git push origin --delete feature/<ticket>-<slug>
```

Only delete a branch after confirming its commits are merged and no deployment still uses it.
