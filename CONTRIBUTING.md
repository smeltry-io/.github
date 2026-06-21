# Contributing to Smeltry

Thank you for your interest in contributing!

## Before You Start

- Read the [GOVERNANCE.md](GOVERNANCE.md) to understand how decisions are made.
- Check existing [issues](https://github.com/smeltry-io/.github/issues) and
  [pull requests](https://github.com/smeltry-io/.github/pulls) before opening new ones.
- For significant changes, open an issue first to discuss the approach.

## Developer Certificate of Origin (DCO)

All commits **must** include a `Signed-off-by` line. This certifies that you wrote
or have the right to submit the code under the project's Apache 2.0 license.

```
git commit -s -m "your commit message"
```

This appends:
```
Signed-off-by: Your Name <your@email.com>
```

Commits without a DCO sign-off will not be merged. The DCO check runs
automatically on every pull request.

## Code Style

- **Go**: follow standard Go conventions and `kubebuilder` patterns.
- **Comments and documentation**: English only.
- Run `go test ./...` before submitting a PR.

## Commit Messages

Use [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add ClusterClaim controller
fix: release Netbox IPs on finalizer
docs: update SiteConfig CRD reference
chore: rename smelt → smeltry
```

## Pull Request Process

1. Fork the relevant repository and create a branch.
2. Make your changes with signed commits.
3. Open a PR against `main`.
4. Address reviewer feedback.
5. A maintainer will merge once approved.

## Code of Conduct

By participating, you agree to abide by the
[CNCF Code of Conduct](https://github.com/cncf/foundation/blob/main/code-of-conduct.md).
