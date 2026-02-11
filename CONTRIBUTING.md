# Contributing to gh-repo-stats-plus-action

Thank you for your interest in contributing! Here are some guidelines to help you get started.

## Getting Started

1. Fork the repository
2. Clone your fork locally
3. Create a feature branch from `main`

```bash
git checkout -b feature/my-feature
```

## Making Changes

- Keep changes focused and scoped to a single concern
- Update `action.yml` if you are adding or modifying inputs, outputs, or steps
- Update `README.md` to reflect any changes to inputs, outputs, or usage
- Test your changes by referencing the action locally in a workflow (`uses: ./`)

## Action Dependencies

Third-party actions used within this composite action are pinned to specific commit SHAs (not tags) to support GitHub Enterprise environments and avoid caching issues. When adding or updating a dependency, always pin to a full commit SHA and include a comment with the version:

```yaml
uses: actions/setup-node@6044e13b5dc448c55e2357c09f80417699197238 #v6.2.0
```

## Submitting a Pull Request

1. Push your branch to your fork
2. Open a pull request against `main`
3. Provide a clear description of the change and why it is needed
4. Ensure the CI workflow passes

## Reporting Issues

If you find a bug or have a feature request, please [open an issue](https://github.com/mona-actions/gh-repo-stats-plus-action/issues/new).

## Code of Conduct

Be respectful and constructive in all interactions.
