# Setup odx

A composite action, for GitHub Actions and Forgejo Actions alike, that
installs [odx](https://github.com/getfluvo/odx), the CLI that aggregates,
locks, ports, migrates and lints Odoo projects from `pyproject.toml`; caches
its data; and, with `bot: true`, runs the nightly cycle of the upstream
dashboard and bot pull requests.

## Checks on pull requests

```yaml
# .github/workflows/odx.yml
on: pull_request
permissions:
  contents: read
jobs:
  odx:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with:
          fetch-depth: 0
      - uses: getfluvo/odx-action@v1
      - run: odx verify
      - run: odx check-deps
      - run: odx lint --only-changed --output-format github
```

`odx lint --output-format github` shows the findings as annotations on the
pull request; `sarif` feeds GitHub code scanning instead.

## The bot, nightly

```yaml
# .github/workflows/odx-bot.yml
on:
  schedule: [{ cron: "10 2 * * *" }]
  issues: { types: [edited] }
  workflow_dispatch:
permissions:
  contents: read
jobs:
  bot:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/create-github-app-token@v2
        id: app
        with:
          app-id: ${{ vars.ODX_APP_ID }}
          private-key: ${{ secrets.ODX_APP_KEY }}
      - uses: actions/checkout@v5
        with:
          token: ${{ steps.app.outputs.token }}
          fetch-depth: 0
      - uses: getfluvo/odx-action@v1
        id: odx
        with:
          bot: true
        env:
          GITHUB_TOKEN: ${{ steps.app.outputs.token }}
      - uses: actions/upload-artifact@v4
        with:
          name: odx-plan
          path: ${{ steps.odx.outputs.plan-json }}
```

With `apply: auto`, the default, a scheduled run applies (`odx bot run
--apply`) and a `pull_request` run only plans. The identity, the token
scopes and what the bot may write are described in
[The nightly bot](https://odx.readthedocs.io/en/latest/ci.html#the-nightly-bot).

On Forgejo the workflow lives in `.forgejo/workflows` and uses
`https://github.com/getfluvo/odx-action@v1`, with the token of a bot account
from a secret instead of the app.

## Inputs

| Input | Default | |
| --- | --- | --- |
| `version` | `latest` | the odx release from PyPI (`0.0.8`); `none` keeps the odx on `PATH` |
| `cache` | `true` | restore and save the odx cache, keyed on `odx.lock` |
| `working-directory` | `.` | the project directory |
| `bot` | `false` | run `odx --json bot run` |
| `apply` | `auto` | `true`, `false`, or `auto`: apply except on `pull_request` events |
| `series` | | only the dashboard of this series |
| `max-prs` | | pull requests a run opens at most |
| `remote` | `origin` | the git remote of the project repository |

## Outputs

| Output | |
| --- | --- |
| `version` | the installed odx version |
| `plan-json` | the path of the run's JSON, with `bot` |
| `dashboard-issue` | the number of the dashboard issue, with `bot` |

odx is installed with uv, pipx or pip, whichever the runner has, and
`ODX_CACHE_DIR` points at the runner's temporary directory unless the
workflow sets it. A project with `cache_dir` in `[tool.odx]` uses that
directory instead, which the action does not cache: set `cache: false` there.

## License

MIT
