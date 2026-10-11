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

GitHub doesn't pass secrets to workflows triggered by pull requests from forks, so the action stops with an error there. Either run it on `push`, or skip it with `if: github.event.pull_request.head.repo.fork == false`.

## License

MIT
