# sync-to-mirror

A composite GitHub Action that force-pushes branches from the current repository
to a **mirror repo** using an SSH deploy key. It is the single source of truth
for the per-repo "sync to mirror" workflows (replaces the hand-copied
`sync-to-personal.yml` / `sync-to-private.yml` files).

## Usage

```yaml
name: Sync to Mirror

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: sync-to-mirror-${{ github.ref }}
  cancel-in-progress: false

jobs:
  sync:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: aixelindonesia/sync-to-mirror@v1
        with:
          target-repo: git@github.com:OWNER/REPO.git
          sync-branches: main
          git-user-name: Aixel
          git-user-email: aixelindonesia@gmail.com
          ssh-private-key: ${{ secrets.PERSONAL_REPO_SSH_KEY }}
```

> The action does its own `checkout` (full history) — the caller workflow needs
> no separate checkout step.
>
> The SSH secret name is the **caller's** choice — pass whatever secret the repo
> already has (`PERSONAL_REPO_SSH_KEY`, `PRIVATE_REPO_SSH_KEY`, etc.). The action
> never hardcodes it, so no secret needs renaming when adopting this action.

## Inputs

| Input             | Required | Default                     | Description                                              |
| ----------------- | -------- | --------------------------- | -------------------------------------------------------- |
| `target-repo`     | yes      | —                           | SSH URL of the mirror, `git@github.com:owner/repo.git`   |
| `ssh-private-key` | yes      | —                           | Private key; pass the repo's existing SSH secret         |
| `sync-branches`   | no       | `main`                      | Space-separated branch list, e.g. `"main staging"`       |
| `git-user-name`   | no       | — (skips commit)            | Committer name for the empty sync commit; needs `git-user-email` too    |
| `git-user-email`  | no       | — (skips commit)            | Committer email; needs `git-user-name` too                              |

## One-time setup per consuming repo

1. Generate a keypair (no passphrase):
   ```bash
   ssh-keygen -t ed25519 -C "<repo>-sync" -f /tmp/<repo>-sync -N ""
   ```
2. **Mirror repo** → Settings → Deploy keys → add the **public** key with **write** access.
3. **Source repo** → Settings → Secrets and variables → Actions → add the
   **private** key as a secret (any name), and reference that name in
   `ssh-private-key`. If the repo already has a sync secret, reuse it — no
   rename needed.
4. Add the caller workflow above as `.github/workflows/sync-to-mirror.yml`.

## Notes

- When both `git-user-name` and `git-user-email` are set, each run appends an
  empty `chore: sync` commit, so the mirror's history intentionally diverges
  from origin by these markers. Leave both empty to mirror branches verbatim
  with no extra commit.
- The push is `--force` — the mirror branch is overwritten; treat it as one-way.
- Keep `on.push.branches` in sync with `sync-branches`.

## Versioning

Consumers pin `@v1`. Releases are cut as `vX.Y.Z` tags with a moving `vX` tag.
