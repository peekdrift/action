# Peekdrift GitHub Action

Visual regression testing for Storybook. The action uploads your built Storybook to [Peekdrift](https://peekdrift.com). Peekdrift screenshots every story, compares each one with the accepted baseline, and reports the result as a check on your pull request.

```yaml
name: Visual tests
on: [push, pull_request]

jobs:
  peekdrift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
      - uses: actions/setup-node@v7
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - uses: peekdrift/action@v1
        with:
          projectToken: ${{ secrets.PEEKDRIFT_PROJECT_TOKEN }}
```

`fetch-depth: 0` lets Peekdrift walk your git history to find the right baseline. It also lets approvals carry over when a pull request is merged.

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `projectToken` | (required) | Project token from the project's settings page. |
| `storybookBuildDir` | | An already built Storybook. Skips the build step. |
| `buildScriptName` | `build-storybook` | The package.json script that builds Storybook. |
| `workingDir` | `.` | Directory to run in, for monorepos. |
| `config` | | Path to `peekdrift.config.json`. |
| `onlyChanged` | `false` | Only screenshot stories affected by changed files. The others are carried over from the baseline and don't count toward your snapshot limit. |
| `externals` | | With `onlyChanged`: newline-separated globs that affect every story, such as `public/**`. |
| `untraced` | | With `onlyChanged`: newline-separated globs whose changes never matter. |
| `exitOnceUploaded` | `false` | Finish once the Storybook is uploaded. The GitHub check still reports the result. |
| `exitZeroOnChanges` | `false` | Succeed even when changes need review. |
| `skip` | | `true` passes the check without snapshots; a glob such as `dependabot/**` skips only matching branches. |
| `junitReport` | | `true` (or a file path) writes a JUnit XML report: one test case per snapshot. |
| `commit`, `branch`, `pullRequest` | | Report the build for this commit, branch and pull request instead of the ones the workflow ran on (see Forks). |
| `version` | `latest` | The `peekdrift` CLI version. |

## Outputs

`buildUrl`, `buildNumber`, `status`, `code`, `changeCount`, `newCount`, `removedCount`, `unchangedCount`, `pendingCount`.

```yaml
      - uses: peekdrift/action@v1
        id: peekdrift
        with:
          projectToken: ${{ secrets.PEEKDRIFT_PROJECT_TOKEN }}
      - run: echo "Review at ${{ steps.peekdrift.outputs.buildUrl }}"
```

## Screenshots in your own CI

To capture in your own runners instead of on Peekdrift, run `npx peekdrift capture` and then `npx peekdrift upload --wait`. The [CLI documentation](https://www.npmjs.com/package/peekdrift) explains how.

## Forks

GitHub doesn't pass secrets to workflows triggered by pull requests from forks, and giving a fork's code your project token would let it read the token. Split the work in two instead: the fork's workflow builds the Storybook without secrets, and a second workflow that only runs your repository's own code uploads the built files. A Storybook is static files; Peekdrift renders them in a sandbox with outside requests blocked, so the fork's code never runs where your secrets are.

`.github/workflows/storybook.yml` builds the Storybook for every pull request, forks included:

```yaml
name: Storybook
on: pull_request

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v7
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run build-storybook -- --output-dir storybook-static
      - uses: actions/upload-artifact@v5
        with:
          name: storybook
          path: storybook-static
          retention-days: 1
```

`.github/workflows/visual-tests.yml` uploads it with your token once that workflow finishes:

```yaml
name: Visual tests
on:
  workflow_run:
    workflows: [Storybook]
    types: [completed]

permissions:
  contents: read
  actions: read
  pull-requests: read

jobs:
  peekdrift:
    if: github.event.workflow_run.conclusion == 'success' && github.event.workflow_run.event == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      # Your repository's code only: never check out or run the pull request's.
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
      - uses: actions/download-artifact@v5
        with:
          name: storybook
          path: storybook-static
          run-id: ${{ github.event.workflow_run.id }}
          github-token: ${{ github.token }}
      # The commit and branch come from GitHub, not from the artifact; the pull request is the open one for that commit.
      - id: pr
        env:
          GH_TOKEN: ${{ github.token }}
          SHA: ${{ github.event.workflow_run.head_sha }}
        run: |
          number="$(gh api "repos/${GITHUB_REPOSITORY}/commits/${SHA}/pulls" --jq '[.[] | select(.state == "open")][0].number // empty')"
          echo "number=${number}" >> "$GITHUB_OUTPUT"
          # The commit's history (not its files) lets Peekdrift find the right baseline.
          if [ -n "$number" ]; then git fetch --no-tags origin "pull/${number}/head"; fi
      - uses: peekdrift/action@v1
        with:
          projectToken: ${{ secrets.PEEKDRIFT_PROJECT_TOKEN }}
          storybookBuildDir: storybook-static
          commit: ${{ github.event.workflow_run.head_sha }}
          # Prefixed with the fork's owner: a fork's "main" must never count as your default branch.
          branch: ${{ github.event.workflow_run.head_repository.owner.login }}:${{ github.event.workflow_run.head_branch }}
          pullRequest: ${{ steps.pr.outputs.number }}
          exitOnceUploaded: true
```

`onlyChanged` is not available in this setup, because the second workflow does not have the pull request's files.

## License

MIT
