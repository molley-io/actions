# actions

Molley shared actions.

| Action | Path | Purpose |
|---|---|---|
| Molley Code Analyzer | `/` (repo root) | Analyze repository features and sync results back to Molley |
| Molley Progress Report | `/report` | AI-written progress report from git history + Molley activity, committed to `/reports` |

## Molley Progress Report (`/report`)

Digs into the full git history of the repo (designed for monorepos), pulls the
last N days of product activity from Molley via the [Molley MCP
server](https://www.npmjs.com/package/@molley/mcp-server), and has Claude write
a progress report: blockers, successes, where the product is heading.

The report is **about the product, never about people** — no individual is
named, credited, or blamed. It reports where the product is moving, stuck,
stalling, or winning. Information, not blame.

The finished report is committed to the `develop` branch if it exists,
otherwise `main` (overridable), under `/reports/<UTC timestamp>-<style>.md`,
e.g. `reports/2026-07-10-0600Z-developer.md`. If a Slack, Teams, and/or Google
Chat incoming webhook is configured, the **entire report** is also posted to
the channel as a single message, converted to each platform's formatting
(Slack mrkdwn blocks, Teams Adaptive Card, Google Chat card). No links out to
GitHub. Only if a report exceeds the platform's message size limit is it
truncated, with a note pointing to the committed file path.

### Report styles

| `report_style` | Audience | Emphasis |
|---|---|---|
| `slt` | Senior leadership | Momentum, risk, strategic direction — plain business language, three-minute read |
| `sales` | Sales & marketing | Customer-visible progress, what's coming, go-to-market activity, talking points |
| `developer` | Engineering | Per-area codebase movement, churn hotspots, stalls, delivery flow, tech-debt signals |

### Usage

This is a base action: each repo imports it in its own workflow and configures
the style, schedule, and delivery there. A composite action cannot define its
own triggers in GitHub Actions, so **the weekly schedule (which day, which
time) is set with the `on.schedule` cron in your workflow** — cron format is
`minute hour day-of-month month day-of-week` in UTC:

- `'0 6 * * 1'` — Mondays at 06:00 UTC
- `'30 16 * * 5'` — Fridays at 16:30 UTC

```yaml
name: Weekly progress report
on:
  schedule:
    - cron: '0 6 * * 1' # runs automatically every Monday 06:00 UTC — change day/time here
  workflow_dispatch: # also allow running it manually on demand
    inputs:
      report_style:
        description: slt | sales | developer
        default: developer

permissions:
  contents: write

jobs:
  report:
    runs-on: ubuntu-latest
    steps:
      - uses: molley-io/actions/report@v1
        with:
          report_style: ${{ inputs.report_style || 'developer' }}
          molley_project_id: ${{ vars.MOLLEY_PROJECT_ID }}
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          MOLLEY_API_KEY: ${{ secrets.MOLLEY_API_KEY }}
          # Optional outward delivery — set any combination:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
          TEAMS_WEBHOOK_URL: ${{ secrets.TEAMS_WEBHOOK_URL }}
          GOOGLE_CHAT_WEBHOOK_URL: ${{ secrets.GOOGLE_CHAT_WEBHOOK_URL }}
```

### Multiple reports in one run (`routes`)

To generate several styles in a single cron run and send each to its own
channels, use the `routes` input instead of `report_style`. One line per
report: `<style>: <platform>:<ENV_VAR>, ...` where platform is `slack`,
`teams`, or `gchat`, and `ENV_VAR` names an env var (usually a secret) holding
that webhook URL. A style may list any number of destinations.

```yaml
      - uses: molley-io/actions/report@v1
        with:
          routes: |
            slt: slack:SLACK_URL_A, slack:SLACK_URL_B, slack:SLACK_URL_C
            sales: gchat:GCHAT_URL_Y
            developer: slack:SLACK_URL_DEV
          molley_project_id: ${{ vars.MOLLEY_PROJECT_ID }}
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          MOLLEY_API_KEY: ${{ secrets.MOLLEY_API_KEY }}
          SLACK_URL_A: ${{ secrets.SLACK_WEBHOOK_LEADERSHIP_A }}
          SLACK_URL_B: ${{ secrets.SLACK_WEBHOOK_LEADERSHIP_B }}
          SLACK_URL_C: ${{ secrets.SLACK_WEBHOOK_LEADERSHIP_C }}
          GCHAT_URL_Y: ${{ secrets.GCHAT_WEBHOOK_SALES }}
          SLACK_URL_DEV: ${{ secrets.SLACK_WEBHOOK_DEV }}
```

All reports in a run are generated from the same git/Molley evidence window and
committed together in one commit (each as its own timestamped file). When
`routes` is set, `report_style` and the legacy `SLACK_WEBHOOK_URL` /
`TEAMS_WEBHOOK_URL` / `GOOGLE_CHAT_WEBHOOK_URL` env vars are ignored.
If one style's generation fails, the others still commit and deliver.

### Inputs

| Input | Default | Description |
|---|---|---|
| `report_style` | `developer` | `slt`, `sales`, or `developer` (ignored when `routes` is set) |
| `routes` | — | Multi-report routing, one `style: platform:ENV_VAR, ...` line per report |
| `days` | `7` | Look-back window in days (max 14, limited by the Molley activity API) |
| `max_words` | — | Hard word cap: one number for all reports (`'600'`) or per style (`'slt: 400, developer: 900'`). Unset styles use built-in defaults: slt 500 words, sales/developer uncapped |
| `molley_project_id` | — | Molley project passport (same value the analyzer action uses) |
| `reports_dir` | `reports` | Directory the report is committed into |
| `output_branch` | auto | Target branch; empty = `develop` if it exists, else `main` |
| `model` | `claude-opus-4-6` | Claude model used to write the report |

### Environment

| Variable | Required | Description |
|---|---|---|
| `ANTHROPIC_API_KEY` | yes | API key used by the Claude Code CLI |
| `MOLLEY_API_KEY` | no | Enables the Molley MCP server; without it the report is git-only and says so |
| `MOLLEY_COMPANY_ID` | no | Only needed if it can't be resolved from the project passport |
| `SLACK_WEBHOOK_URL` | no | Slack incoming-webhook URL — posts the full report as mrkdwn blocks |
| `TEAMS_WEBHOOK_URL` | no | Microsoft Teams incoming-webhook URL — posts the full report as an Adaptive Card |
| `GOOGLE_CHAT_WEBHOOK_URL` | no | Google Chat space incoming-webhook URL — posts the full report as a card |

### Outputs

| Output | Description |
|---|---|
| `report_paths` | Space-separated repo-relative paths of the committed report(s) |
| `report_branch` | Branch the report(s) were pushed to |

The three legacy env vars (`SLACK_WEBHOOK_URL`, `TEAMS_WEBHOOK_URL`,
`GOOGLE_CHAT_WEBHOOK_URL`) still work in single-style mode: every configured
one receives the report.
