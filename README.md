# shared-workflow

Reusable GitHub Actions workflows shared across BHS-BS repositories.

## Usage

Call a workflow from another repository with `workflow_call`:

```yaml
jobs:
  example:
    uses: BHS-BS/shared-workflow/.github/workflows/<workflow>.yml@main
    secrets: inherit
```

## Workflows

| Workflow | Use it for |
|---|---|
| `maven-build.yml` | Maven build; uploads the jars as `<BUILD_ARTIFACT>.zip`. `JAVA_VERSION` defaults to `21`. |
| `npm-build.yml` | npm build (`npm ci` + `npm run <BUILD_SCRIPT>`); uploads `OUTPUT_DIR` as `<BUILD_ARTIFACT>.zip`. `NODE_VERSION` defaults to `22`. |
| `docker-build.yml` | Builds the image and pushes it to `ghcr.io/<owner>/<SERVICE_NAME>`, tagged with the branch name, or the git tag name on a tag push. PRs build only. |

### npm projects

- **The Dockerfile runs the npm build itself** (multi-stage): call only `docker-build.yml`,
  without `BUILD_ARTIFACT`.
- **The Dockerfile copies a prebuilt bundle:** call `npm-build.yml`, then `docker-build.yml`
  with the same `BUILD_ARTIFACT`.

**Private `@<owner>` packages** (e.g. `@bhs-bs/ui`): set `NPM_REGISTRY_SECRET: true` and give the
calling job `packages: read` (`docker-build.yml` already needs `write`). The caller's
`GITHUB_TOKEN` is used, so there is no secret to create. In `docker-build.yml` it reaches the build as the `npmrc` build secret, never as
an ARG, so it is not recorded in the image:

```dockerfile
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
```

Each repo that installs a private package must be granted read access to it, under the package's
**Package settings → Manage Actions access**. Without it, `npm ci` fails with 403 or 404.

**Build-time environment** (`npm-build.yml`): a caller can't set `env:` on a reusable workflow, so
pass `BUILD_ENV`, one `KEY=VALUE` per line.

Example caller for an npm project whose Dockerfile copies the built bundle:

```yaml
jobs:
  build-npm:
    permissions:
      contents: read
      packages: read          # needed only with NPM_REGISTRY_SECRET
    uses: BHS-BS/shared-workflow/.github/workflows/npm-build.yml@develop
    with:
      SERVICE_LOCATION: ./my-ui
      BUILD_ARTIFACT: my-ui
      OUTPUT_DIR: build
      NPM_REGISTRY_SECRET: true
      BUILD_ENV: |
        CI=false
        REACT_APP_API_URL=/api

  build-docker:
    needs: build-npm
    permissions:
      contents: read
      packages: write
    uses: BHS-BS/shared-workflow/.github/workflows/docker-build.yml@develop
    with:
      SERVICE_LOCATION: ./my-ui
      SERVICE_NAME: my-ui
      BUILD_ARTIFACT: my-ui
```

## Contributing

1. Create a feature branch from `main`.
2. Make your changes and open a pull request.
3. Get a review before merging. Do not push directly to `main`.
