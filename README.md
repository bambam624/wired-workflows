# wired-workflows

Shared GitHub Actions workflows for the Wired Builder portfolio.

**This repo is public on purpose.** The sites that call it live under three
different GitHub owners (`bambam624`, `WiredBuilder`, `sunhouse-hq`), and GitHub
only allows a repo to call a reusable workflow from another owner when the source
is public. Nothing secret lives here: workflow files contain no credentials, and
every secret is passed in by the calling repository at run time.

## Available workflows

| Workflow | For |
|---|---|
| `astro-worker-deploy.yml` | Astro sites on Cloudflare Workers with static assets |

## Using it

A calling repo's entire `.github/workflows/deploy.yml` becomes:

```yaml
name: Deploy
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
  workflow_dispatch:

jobs:
  deploy:
    uses: bambam624/wired-workflows/.github/workflows/astro-worker-deploy.yml@v1
    with:
      production-url: https://example.com
      smoke-text: Example Business
      verify-files: dist/index.html dist/sitemap-index.xml dist/robots.txt
    secrets:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

### Inputs

| Input | Default | Purpose |
|---|---|---|
| `node-version` | `22` | Keep the portfolio on one version |
| `verify-files` | `dist/index.html` | Space-separated paths that must exist after build |
| `production-url` | none | Smoke-checked after deploy; empty disables |
| `smoke-text` | none | Text that must appear in the production response |
| `post-deploy` | none | Command after a successful deploy, e.g. an IndexNow ping |

### Build-time variables

`PUBLIC_POSTHOG_PROJECT_TOKEN` and `PUBLIC_POSTHOG_HOST` are read from the
**calling** repo's Actions *variables* (not secrets) and exported in both jobs.
No input to set: a repo that defines them gets them, a repo that does not gets
empty strings.

They are set on the deploy job as well as the check job because a repo that
sets `build.command` in its wrangler config gets a second build inside
`wrangler deploy`. Exporting them only on the check job would ship an artifact
with the token and then overwrite it with a rebuild without one.

Astro inlines `PUBLIC_*` at build time, so a missing value is not a runtime
error. It is analytics that quietly stops reporting, which is why this is
documented rather than left to be rediscovered.

### What it does

1. **check** (also on pull requests): install, `typecheck --if-present`,
   `test --if-present`, build, assert the expected output files exist, then
   upload `dist/` as an artifact.
2. **deploy** (main only): download that artifact and `wrangler deploy`. The
   deploy job runs on a fresh runner, so without the hand-off it would have
   nothing to publish for any repo that does not set `build.command`.
3. **smoke check**: request the production URL, retry three times, assert HTTP
   200 and the expected text.

A failed smoke check fails the job loudly and prints the rollback command. It
does not roll back automatically: an auto-rollback that misfires on a flaky
check is worse than none.

## Versioning: why you pin a tag

Calling repos reference `@v1`, never `@main`. Editing `main` here changes
nothing for anyone until a tag moves. That is what stops one bad edit breaking
every site at once.

To ship a change:

1. Commit to `main`.
2. Tag it `v2`.
3. Move **one low-stakes site** to `@v2` and watch a real deploy.
4. Promote the rest only after that canary has been clean for a week.

To roll back, move the affected repo's `uses:` line back to `@v1`. One line,
per repo, instant.

## Cloudflare accounts

The account comes from `account_id` in each repo's wrangler config, not from a
secret. Worker names are not unique across Cloudflare accounts, so the pin in
the config is the authoritative answer to where a site publishes. A repo that
has not pinned will fail here rather than guess.

## What is deliberately not here

- **Pages deploys.** The three remaining Pages sites are scheduled to migrate to
  Workers; encoding a deprecating product in shared automation is not worth it.
- **Anything secret.** This repo is public.
