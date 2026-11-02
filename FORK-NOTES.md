# Revolutions Records — Umami fork notes

This is a fork of [umami-software/umami](https://github.com/umami-software/umami) used to run
the analytics instance for the Revolutions Records umbrella at
**https://neptune.revolutionsrecords.com**.

## Branch strategy

- `deploy` — the deployed branch. Pinned to an upstream release tag plus the local patches below.
  Currently based on **v3.4.1** (`559e3c6`).
- `master` — untouched mirror of upstream, for pulling in new releases.

To take a new upstream release:

```bash
git fetch upstream --tags
git checkout deploy
git rebase v3.X.Y     # replay the local patches onto the new tag
git push --force-with-lease
```

## Local patches

Keep this list short — every entry is a merge conflict waiting to happen.

### 1. `biome.json` — disable VCS ignore-file discovery

Upstream sets:

```json
"vcs": { "enabled": true, "clientKind": "git", "useIgnoreFile": true }
```

With that setting Biome aborts if it cannot find a `.gitignore` in the project root.
Vercel CLI deploys (`vercel deploy`) do not upload `.gitignore`, so the
`build:tracker:types` step — which runs `biome format --write src/tracker/index.d.ts` —
fails the build with:

```
× Biome couldn't find an ignore file in the following folder: /vercel/path0
```

Biome is only used here to format a single generated `.d.ts` file, so ignore-file
awareness has no effect on build output. We set `vcs.enabled` and
`vcs.useIgnoreFile` to `false`.

This patch can be dropped if the project is switched to Git-integration deploys
(where a real checkout with `.gitignore` is present).

### 2. `.vercelignore`

Added so CLI deploys skip `node_modules`, build output and local env files.

## Configuration

Environment variables live on the Vercel project `revolutions-umami`
(team `Mainframe` / `zorans-projects-d90c78b8`):

| Variable | Source | Notes |
| --- | --- | --- |
| `DATABASE_URL` | Neon integration | **Pooled** connection, used at runtime |
| `DIRECT_DATABASE_URL` | set manually | **Unpooled** connection, used by `prisma migrate deploy` at build time |
| `APP_SECRET` | set manually | 32 random bytes, hex. Pins session/JWT signing so rotating DB credentials does not log everyone out |

The database is the Neon project `revolutions-umami` (`autumn-violet-41021297`, `aws-us-east-1`),
provisioned through the Vercel Neon marketplace integration. It is deliberately **separate**
from the unrelated Kichma Umami instance and its database.
