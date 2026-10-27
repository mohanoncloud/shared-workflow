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
calling job `packages: read` (or `write`). The caller's `GITHUB_TOKEN` is used, so there is no
secret to create. In `docker-build.yml` it reaches the build as the `npmrc` build secret, never as
an ARG, so it is not recorded in the image:

```dockerfile
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc npm ci
```

Each repo that installs a private package must be granted read access to it, under the package's
**Package settings → Manage Actions access**. Without it, `npm ci` fails with 403 or 404.

## Contributing

1. Create a feature branch from `main`.
2. Make your changes and open a pull request.
3. Get a review before merging. Do not push directly to `main`.
