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

## Contributing

1. Create a feature branch from `main`.
2. Make your changes and open a pull request.
3. Get a review before merging. Do not push directly to `main`.
