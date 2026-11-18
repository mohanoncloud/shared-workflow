# CLAUDE.md

Guidance for Claude Code (claude.ai/code) and the Claude PR review in this repository. Cross-repo context (service map, template, other repos) is in `../CLAUDE.md` when this repo is cloned in the BHS-BS workspace.

## Project context

- **What it is:** reusable GitHub Actions workflows for every BHS-BS repository. BHS-BS's own repo, not a fork; it replaces the upstream `mosip/kattu` workflows the MOSIP forks used to call.
- **Part of:** shared CI tooling. It belongs to none of the My Pass groups (platform, eSignet, Inji, apps); any BHS-BS repo can call it (e.g. audit-manager, keycloak).
- **Remotes:** `origin` is `github.com/BHS-BS/shared-workflow`.
- **Blast radius:** callers use `uses: BHS-BS/shared-workflow/.github/workflows/<file>.yml@develop`, so **a merge to `develop` changes CI in every calling repo immediately**.

## Architecture overview

Reusable workflows (`on: workflow_call`) in `.github/workflows/`:

| Workflow | Does | Key inputs |
|---|---|---|
| `maven-build.yml` | `mvn -U -B package` (tests run), zips executable jars, uploads artifact `<BUILD_ARTIFACT>` (1-day retention) | `SERVICE_LOCATION`, `BUILD_ARTIFACT`, `JAVA_VERSION` (21), `MAVEN_NON_EXEC_ARTIFACTS`, `PACKAGE_TARGET_DIR`, `POST_BUILD_SCRIPT` |
| `npm-build.yml` | `npm ci` + `npm run <BUILD_SCRIPT>`, zips `OUTPUT_DIR`, uploads artifact | `SERVICE_LOCATION`, `BUILD_ARTIFACT`, `NODE_VERSION` (22), `OUTPUT_DIR`, `BUILD_ENV`, `NPM_REGISTRY_SECRET` |
| `docker-build.yml` | Downloads the artifact (optional), builds the image, pushes `ghcr.io/<owner>/<SERVICE_NAME>:<branch or tag>`; PRs build only | `SERVICE_LOCATION`, `SERVICE_NAME`, `BUILD_ARTIFACT`, `BASE_SERVICE_NAME`, `NPM_REGISTRY_SECRET` |
| `helm-lint-publish.yml` | Lints and renders every chart in `CHARTS_DIR` if any changed; on a branch push publishes them to `oci://ghcr.io/<owner>/charts/<chart>` | `CHARTS_DIR`, `IGNORE_CHARTS`, `CHART_REGISTRY_PATH`, `DEPENDENCY_REPOS`, `HELM_VERSION` |
| `tag.yml` | Creates a git tag + GitHub release on the caller's commit; refuses existing, SNAPSHOT and unflagged alpha/beta/RC tags | `TAG`, `BODY`, `PRE_RELEASE`, `DRAFT` |
| `verify-keycloak-init.yml` | Checks every client `saroles` entry is in the realm `roles` of a keycloak-init values file | `kc_values_file_path` |

Runs in this repo only:

- `helm-mirror.yml` copies the third-party charts in `helm-mirror/charts.txt` (`<repo URL> <chart> <version>`, e.g. Bitnami `common 1.17.1`) to `oci://ghcr.io/<owner>/charts/`. PRs only check the download; pushes to `develop` publish. Existing versions are skipped.
- `examples/chart-lint-publish.yml` is the caller to copy into a module repo.

How to call each workflow (example caller YAML, required permissions, input details) is documented in `README.md` and in the comment block at the top of each workflow file. Update those when a workflow changes; don't copy them into this file.

## Coding standards and conventions

- Workflow and input names: inputs are `UPPER_SNAKE_CASE` (except the legacy `kc_values_file_path`). Every input has a comment saying what it is for and its default.
- **Never break callers.** New inputs must be optional with a default that keeps today's behaviour. Renaming or removing an input, artifact name or image/chart path is a breaking change: update every caller in the same rollout and say so in the PR.
- Pass inputs and secrets to scripts through `env:`, not `${{ }}` inside `run:` (see `npm-build.yml` `BUILD_ENV`, `docker-build.yml` npm token). Tokens never go into build ARGs.
- Use the caller's `GITHUB_TOKEN`; no PATs. Request the least `permissions:` the job needs, and document what the caller must grant.
- Set `timeout-minutes` on every job (callers can't set it on a `uses:` job).
- Pin actions to a major version (`@v4`); pin tool versions (`HELM_VERSION`, chart versions in `charts.txt`).
- Comment any step whose reason isn't obvious from the code, saying why it is done that way (e.g. the comment on `npm-build.yml`'s missing `permissions:` block). Don't comment what a step plainly does.

## Build and run

There is nothing to build. A workflow only runs when a caller (or, for `helm-mirror.yml`, this repo) triggers it on GitHub.

To try a change before merging, point one caller at your branch in a throwaway PR in that repo:

```yaml
uses: BHS-BS/shared-workflow/.github/workflows/docker-build.yml@MP-xx
```

Check both a PR run and, where the change affects publishing, a push run. Don't merge the caller change; revert it to `@develop`.

## Testing

No automated tests. Before a PR:

```bash
actionlint                     # from the repo root; https://github.com/rhysd/actionlint
```

- Today it reports two findings in `verify-keycloak-init.yml` (`actions/checkout@v3`, `actions/setup-python@v4` are too old). Don't add new ones.
- Then run the change from a branch in a real caller (above). For `helm-mirror.yml`, the PR run itself is the test.

## Code quality and linting

- No linter runs in CI for this repo. `actionlint` (with `shellcheck` installed, it also checks `run:` scripts) is the expected local check.
- CI here: only `helm-mirror.yml` (PRs that change `helm-mirror/charts.txt` or the workflow).

## Branch and PR conventions

- Working branch is `develop`; callers pin `@develop`. `main` holds only a README today.
- Branch from `develop`, named after the Jira key (e.g. `MP-206`).
- PR into `develop`, titled `[MP-xx] <summary>`. Commit messages start with the Jira key.
- Before merge: lead review (no CODEOWNERS yet) and a successful run from the branch in at least one caller. The Claude review (MP-83) is not yet set up as a workflow in this repo.
- Update this file and `README.md` in the same PR when a workflow, input or convention changes.

## BHS rules (apply to every BHS-BS repo)

- **GA versions only.** No `-SNAPSHOT` dependencies, parents or plugins, and no new snapshot repositories.
- **No secrets in code or config.** Passwords, client secrets and keys come from environment or config-server placeholders. Never commit real values, including in test resources, Helm values or scripts.
- **Keycloak realm is `mypass`.**
- **DB upgrade scripts chain without gaps.** Every `X_to_Y_upgrade.sql` has a matching `X_to_Y_rollback.sql`, and each step's `Y` is the next step's `X`.
- **Workflows trigger on real branches only** (`develop`, `main`, PRs; no upstream `MOSIP*`/`release*`/`1.*` patterns), and every job sets `timeout-minutes`.

## Pitfalls

- **`README.md` says `@main` and "branch from `main`".** Both are wrong today: `main` has no workflows, so `@main` fails. Use `@develop` (TODO: fix the README).
- **Missing `timeout-minutes`.** `maven-build.yml`, `npm-build.yml`, `docker-build.yml` and `verify-keycloak-init.yml` don't set it. A caller can't add it to a `uses:` job, so every calling repo breaks the BHS rule until it is fixed here.
- **`verify-keycloak-init.yml` hardcodes realm `mypass`.** The keycloak repo's values still use realm `mosip`, so pointing keycloak at this workflow fails with a `KeyError` until the realm is renamed there. keycloak still calls the `mosip/kattu` version.
- **`docker-build.yml` pushes on every non-PR run, tagged by branch name.** Callers must limit `push:` to real branches (`develop`, `main`); a `develop*` pattern publishes an image per work branch (see keycloak `push-trigger.yml`).
- **`BASE_SERVICE_NAME` on a PR uses the target branch's base image** (`<base>:develop`), which must already exist in GHCR.
- **`maven-build.yml` always runs tests** (`mvn package`, no skip input) and uses `-U`. Repos with `skipTests` hardcoded in surefire behave differently; check the caller's pom.
- **`npm-build.yml` has no `permissions:` block on purpose.** Adding `packages: read` would reject every caller that doesn't grant it.
- **Helm publish rules.** A release chart version that already exists fails the publish and asks for a bump; pre-release versions (`-develop`) are overwritten. Callers must not trigger publishing on `main`, or a `develop` → `main` merge republishes the same versions (see `examples/chart-lint-publish.yml`).
- **Mirrored chart access.** A repo whose charts depend on `oci://ghcr.io/bhs-bs/charts/common` needs read access to that package (**Package settings → Manage Actions access**); private npm packages need the same.
- **`DEPENDENCY_REPOS` still defaults to Bitnami** until every chart uses the GHCR mirror (MP-196).
