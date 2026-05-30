# genvid-public-ci

Shared GitHub Actions workflows for Genvid's **public npm packages** (the
`@genvid` scope). One reusable validation gate, plus copy-paste templates and
the onboarding runbook for standing up a new public package.

## What's here

- **`.github/workflows/node-gate.yml`** — reusable workflow (`workflow_call`)
  that runs the validation gate: `npm ci` → lint → typecheck → test → build,
  plus an optional non-failing `npm publish --dry-run`. Each package's `ci.yml`
  and `publish.yml` call it. It receives **no secrets and no `id-token`**.
- **`templates/ci.yml`**, **`templates/publish.yml`** — drop-in workflow files
  for a new package. They reference the gate at `@main` and need **no
  per-package edits**.

## How publishing works

Publishing uses **npm OIDC trusted publishing**. Each package publishes to npm
from its **own** `publish.yml` using a short-lived credential minted from the
GitHub OIDC token (`id-token: write`) — **no long-lived npm token is stored
anywhere**, and provenance is automatic. The shared gate never publishes,
never receives secrets, and never gets `id-token`, so it is safe to call from
fork PRs.

> Why the publish step lives in each package (not in this shared repo): npm's
> trusted-publisher matching against a *reusable* workflow's `job_workflow_ref`
> across repositories is not documented/verified. Registering the trusted
> publisher against the package's own `publish.yml` is the configuration npm
> unambiguously supports. If npm later confirms cross-repo matching, the publish
> step could move here (a one-line `uses:` change).

## Onboarding a new public `@genvid/<pkg>` package

### 1. Repo prerequisites

- The package repo must be **public** (required for npm provenance).
- Node 22, npm, with a committed `package-lock.json` (the gate runs `npm ci`).

### 2. package.json requirements

- `"name": "@genvid/<pkg>"`
- `"publishConfig": { "access": "public", ... }` (scoped packages are private by
  default)
- `"repository": { "type": "git", "url": "https://github.com/genvid-holdings/<repo>.git" }`
  (HTTPS form — npm provenance matches against it)
- `"prepack": "npm run build"` (so the published tarball always contains built
  output; required if `dist/` is gitignored)
- npm scripts `lint`, `typecheck`, `test`, `build` — the gate calls these.

### 3. Copy the workflow templates

Copy both files from `templates/` into the package repo's `.github/workflows/`:

```sh
cp templates/ci.yml      <pkg-repo>/.github/workflows/ci.yml
cp templates/publish.yml <pkg-repo>/.github/workflows/publish.yml
```

They are identical across packages — no edits needed.

### 4. One-time bootstrap (claim the name + register the trusted publisher)

npm's OIDC flow **cannot perform a package's first publish**, so the name must
be claimed once with a short-lived token before OIDC takes over.

1. Create a **granular npm token** (Read **and** write to `@genvid`), shortest
   expiry, used **locally only** — never stored in GitHub or 1Password.
2. Claim the name with a throwaway placeholder so the first OIDC-provenanced
   publish is a clean version:
   ```sh
   # in the package repo, on a clean checkout
   npm version X.Y.Z-bootstrap.0 --no-git-tag-version
   npm publish --access public            # uses the token from step 1
   npm deprecate "@genvid/<pkg>@X.Y.Z-bootstrap.0" "bootstrap placeholder"
   git checkout package.json package-lock.json   # restore the real version
   ```
   (Alternative: `npx --yes setup-npm-trusted-publish @genvid/<pkg>`.)
3. On **npmjs.com → the package → Settings → Trusted Publisher**, add a GitHub
   Actions publisher:
   | Field | Value |
   |-------|-------|
   | Organization or user | `genvid-holdings` |
   | Repository | `<repo>` |
   | Workflow filename | `publish.yml` |
   | Environment | *(blank)* |
4. **Revoke** the token from step 1. No long-lived token remains.

### 5. Cutting a release

- Bump `version` in `package.json`, merge to `main`.
- Tag and push:
  ```sh
  git tag vX.Y.Z
  git push origin vX.Y.Z
  ```
- `publish.yml` re-runs the gate, verifies the tag matches `package.json`
  `version`, then publishes with `--provenance` via OIDC. Confirm the provenance
  badge on npmjs.com.

## Notes

- The publish job runs `npm install -g npm@latest` because OIDC trusted
  publishing needs **npm ≥ 11.5.1**, while Node 22 ships npm 10.x.
- The **tag ↔ version guard** in `publish.yml` fails the run if the tag (minus
  the `v` prefix) doesn't equal `package.json` `version` — this catches
  "forgot to bump".
- The release tag glob is `v*.*.*`, which also admits prereleases like
  `v0.4.0-rc.1` (the guard still enforces tag↔version equality).
- Callers pin the gate at **`@main`**. Because the gate gets no secrets and no
  `id-token`, the supply-chain risk of a floating ref is low. If this repo
  adopts release tagging, switch callers to a pinned tag/SHA.

## Reference implementation

`genvid-holdings/c3source` was the first package migrated to this setup — see
its `.github/workflows/ci.yml` and `publish.yml` for a working example.
