# workflows

Shared GitHub Actions workflows for the vinicunca organization.

This repository does not publish an npm package. The release workflow below is only a `workflow_call` entry point. Pushing to this repo, or merging a pull request here, does not run `pnpm publish`.

## Release

Reusable workflow: [`.github/workflows/release.yml`](.github/workflows/release.yml)

Package repositories pin it as:

```yaml
uses: vinicunca/workflows/.github/workflows/release.yml@v1
```

`eslint-config`, `perkakas`, and `unocss-preset-vinicunca` can switch from `sxzz/workflows` by changing only that `uses` line. Keep `with: publish: true` and the same permissions.

### Caller

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    uses: vinicunca/workflows/.github/workflows/release.yml@v1
    with:
      publish: true
    permissions:
      contents: write
      id-token: write
```

`contents: write` lets `changelogithub` create the GitHub release. `id-token: write` lets the job request a GitHub OIDC token. npm trusted publishing requires `id-token: write` on the caller job and on this reusable workflow. This workflow sets `contents: write` and `id-token: write` on its `release` job. The caller has to grant those same permissions, as in the example above.

### Inputs

| Input | Type | Default | Effect |
| --- | --- | --- | --- |
| `github-release` | boolean | `true` | Run `pnpx changelogithub` to open the GitHub release. A failure there is ignored (`continue-on-error`) so publishing can still continue. |
| `publish` | boolean | `false` | Publish with pnpm. |
| `stage` | boolean | `false` | Use `pnpm stage publish` instead of `pnpm publish`. |
| `build` | string | `''` | Shell command run before publish. It runs only when it is non-empty and `publish` is `true`. |
| `tag` | string | `''` | Passed through as `--tag <value>` when set. |

Leave `build` empty when the package already builds from `prepublish` or `prepublishOnly`. `pnpm publish` runs those scripts.

### What the job does

The job runs on `ubuntu-latest`, a GitHub-hosted runner. npm trusted publishing does not support self-hosted runners.

1. Check out the caller repository with full git history (tags and commits), so `changelogithub` can build the release notes.
2. Install the pnpm version from the caller repository's `packageManager` field, then install dependencies with `pnpm install --frozen-lockfile`. The store is not cached.
3. When `github-release` is true (the default), run `pnpx changelogithub` with `GITHUB_TOKEN`.
4. When `build` and `publish` are both set, run the build command.
5. When `publish` or `stage` is true, run:

   ```sh
   pnpm -r publish --access public --no-git-checks
   ```

   `stage: true` inserts the `stage` subcommand (`pnpm -r stage publish ...`). A non-empty `tag` appends `--tag <value>`.

Publish authenticates with npm trusted publishing. The workflow does not set `NODE_AUTH_TOKEN`, `NPM_TOKEN`, or `NPM_ID_TOKEN`, and it does not pass a registry token into `actions/setup-node`. pnpm (11 or newer; the current vinicunca libraries use pnpm 12) requests the GitHub OIDC token itself and exchanges it for a short-lived npm credential. For a public package from a public repository, pnpm attaches provenance from that token. No `--provenance` flag and no long-lived npm token are required.

Use a `packageManager` of pnpm 11 or newer. Older pnpm releases do not perform this exchange.

Node.js is the current LTS. Trusted publishing requires Node.js 22.14 or newer.

## npm trusted publisher

Configure one trusted publisher on each package (`npmjs.com` → package → **Settings** → **Trusted Publisher** → **GitHub Actions**).

npm's current trusted-publishing docs: when `npm publish` runs inside `workflow_call`, npm matches the **calling** workflow's filename, not the file that contains the publish command. The publish command lives in this shared workflow, but the filename to register is the caller file in the package repository (`release.yml`). Registering `workflows` or this shared file will not match, and publish fails with `ENEEDAUTH`.

`id-token: write` is required on both sides:

- the caller job (`permissions` on `jobs.release` in the package repo)
- this workflow's `release` job (already set)

### Fields

| Field | Value |
| --- | --- |
| Organization or user | `vinicunca` (the GitHub org or user that owns the package repository) |
| Repository | The package repository name, such as `eslint-config`. Not `workflows`. |
| Workflow filename | `release.yml`. The filename only, including `.yml`. This is the caller file in the package repository, not a path and not this shared workflow. |
| Environment name | Leave empty unless the caller job sets `environment`. The caller above does not. |
| Allowed actions | Allow **npm publish**. Configurations created after 3 September 2026 allow `npm stage publish` automatically; direct `pnpm publish` still needs **npm publish** enabled. |

The workflow file named in that form must exist at `.github/workflows/release.yml` in the package repository.

| Package repository | Organization or user | Repository | Workflow filename | npm publish |
| --- | --- | --- | --- | --- |
| `vinicunca/eslint-config` | `vinicunca` | `eslint-config` | `release.yml` | allow |
| `vinicunca/perkakas` | `vinicunca` | `perkakas` | `release.yml` | allow |
| `vinicunca/unocss-preset-vinicunca` | `vinicunca` | `unocss-preset-vinicunca` | `release.yml` | allow |

Each package's `repository.url` must match that GitHub repository. Provenance is generated only for a public package published from a public repository.

This repository (`vinicunca/workflows`) is not a trusted publisher and has nothing to publish.
