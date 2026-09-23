# Setup Harness CLI

[![CI](https://github.com/harness/setup-harness-cli/actions/workflows/ci.yml/badge.svg)](https://github.com/harness/setup-harness-cli/actions/workflows/ci.yml)

A GitHub Action that installs and authenticates the [Harness CLI (`hc`)](https://github.com/harness/harness-cli). After the action runs, `hc` is on `PATH` and authenticated. You can use _any `hc` command_ in the subsequent steps.

## Prerequisites

- A Harness account
- A Harness Personal Access Token (PAT) stored as a GitHub secret

## Quick Start

```yaml
steps:
  - uses: actions/checkout@v4

  - name: Setup Harness CLI
    uses: harness/setup-harness-cli@v1
    with:
      api-url: ${{ secrets.HARNESS_URL }}
      account: ${{ secrets.HARNESS_ACCOUNT_ID }}
      token: ${{ secrets.HARNESS_PAT_TOKEN }}

  - name: Push artifact
    run: hc artifact push generic my-registry build.tar.gz --name my-app --version ${{ github.sha }}
```

## Examples

### Push multiple artifact types

```yaml
- uses: harness/setup-harness-cli@v1
  with:
    api-url: ${{ secrets.HARNESS_URL }}
    account: ${{ secrets.HARNESS_ACCOUNT_ID }}
    token: ${{ secrets.HARNESS_PAT_TOKEN }}

- run: hc artifact push generic my-registry build.tar.gz --name my-app --version 1.0.0
- run: hc artifact push python my-registry dist/my_pkg-1.0.0.whl
- run: hc artifact push npm my-registry my-pkg-1.0.0.tgz
- run: hc artifact push rpm my-registry package.rpm
```

### Reuse credentials across multiple jobs

Set credentials at the workflow level using environment variables so that every job picks them up without repeating `with:`:

```yaml
env:
  HARNESS_URL:        ${{ secrets.HARNESS_URL }}
  HARNESS_ACCOUNT_ID: ${{ secrets.HARNESS_ACCOUNT_ID }}
  HARNESS_PAT_TOKEN:  ${{ secrets.HARNESS_PAT_TOKEN }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: harness/setup-harness-cli@v1
      - run: hc artifact push generic my-registry build.tar.gz --name my-app --version 1.0.0

  publish-python:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: harness/setup-harness-cli@v1
      - run: hc artifact push python my-registry dist/my_pkg-1.0.0.whl
```

> Note: `hc` is installed once and cached by version and operating system. Subsequent jobs restore the binary from cache instead of downloading it again.

### Pin to a specific CLI version

```yaml
- uses: harness/setup-harness-cli@v1
  with:
    api-url: ${{ secrets.HARNESS_URL }}
    account: ${{ secrets.HARNESS_ACCOUNT_ID }}
    token: ${{ secrets.HARNESS_PAT_TOKEN }}
    hc-version: v1.3.43
```

To always install the latest release, omit `hc-version` (or set it to `latest`). 

### Use the resolved version in later steps

```yaml
- name: Setup Harness CLI
  id: setup
  uses: harness/setup-harness-cli@v1
  with:
    api-url: ${{ secrets.HARNESS_URL }}
    account: ${{ secrets.HARNESS_ACCOUNT_ID }}
    token: ${{ secrets.HARNESS_PAT_TOKEN }}

- run: echo "Running with hc ${{ steps.setup.outputs.hc-version }}"
```

## Inputs

You can provide the credentials as action inputs or as environment variables. 

> Note: Inputs take precedence when both action inputs and environment variables are set.

| Input | Environment Variable | Required | Default | Description |
|-------|---------|----------|---------|-------------|
| `api-url` | `HARNESS_URL` | yes | - | Harness API base URL, no trailing slash. |
| `account` | `HARNESS_ACCOUNT_ID` | yes | - | Harness account ID. |
| `token` | `HARNESS_PAT_TOKEN` | yes | - | Harness PAT token. Always pass using `${{ secrets.* }}` - the action masks it from logs automatically. |
| `org` | `HARNESS_ORG_ID` | no | - | Harness Organization ID. |
| `project` | `HARNESS_PROJECT_ID` | no | - | Harness Project ID. |
| `hc-version` | - | no | `latest` | Harness CLI release tag to install (for example, `v1.3.43`). Defaults to the latest release. |
| `github-token` | - | no | `${{ github.token }}` | Token used to call the GitHub API when resolving the latest `hc` release. The default workflow token is sufficient. |

## Outputs

| Output | Description |
|--------|-------------|
| `hc-version` | The resolved `hc` version that was installed (for example, `v1.3.43`). |

## How it works

1. **Resolves version** - if `hc-version` is `latest`, fetches the current release tag from the GitHub Releases API.
2. **Checks PATH** - if a matching `hc` version is already on `PATH`, skips install entirely.
3. **Restores cache** - looks for a cached binary keyed by version + OS/arch. On a hit, restores without downloading again.
4. **Installs** - on a cache miss, downloads the release tarball from GitHub Releases, verifies the SHA-256 checksum against the published `checksums.txt`, then extracts and caches the binary.
5. **Authenticates** - runs `hc auth login` with the provided credentials. The PAT token is masked before any logging.
6. **Health check** - runs `hc version` to confirm the CLI is usable before reporting ready.
7. **Cleanup** - after the job finishes (even on failure), runs `hc auth logout` to remove stored credentials from the runner.

## Security

- The PAT token is masked using `core.setSecret` before any command runs. The token will never appear in the logs.
- The `hc` binary is verified against a SHA-256 checksum published alongside the release before it is executed.
- Credentials are automatically removed from the runner at the end of each job through the post-step cleanup.

## Contributing

Refer [CONTRIBUTING.md](https://github.com/harness/harness/blob/main/CONTRIBUTING.md).

## License

Apache License 2.0. Refer [LICENSE](https://github.com/harness/harness/blob/main/LICENSE).
