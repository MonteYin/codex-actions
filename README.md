# Codex Actions

A collection of GitHub automation workflows powered by OpenAI Codex CLI, providing AI-powered PR code review, issue triage, smart responses, and more.

## Features

| Workflow | Job Name | Description |
|----------|----------|-------------|
| `reusable-pr-review.yml` | PR Review | AI code review with inline comments |
| `reusable-pr-label.yml` | PR Label | Auto-label PRs based on content |
| `reusable-pr-description.yml` | PR Description | Auto-enhance PR descriptions |
| `reusable-issue-triage.yml` | Issue Label / Issue Dedupe | Classification + duplicate detection |
| `reusable-issue-auto-response.yml` | Issue Response | Auto-respond to new issues |
| `reusable-mention-responder.yml` | Mention Response | Respond to @codex mentions |
| `reusable-issue-stale-cleanup.yml` | Issue Stale Cleanup | Clean up inactive issues |

## Versioning

Use version tags for stability:

| Reference | Description |
|-----------|-------------|
| `@v1` | **Recommended** - Latest v1.x.x (stable, auto-updates) |
| `@v1.0.0` | Locked to exact version |
| `@main` | Latest (may include breaking changes) |

## Quick Start

### 1. Configure Secrets

Add the following secrets to your repository:

```
Settings → Secrets and variables → Actions → Repository secrets
```

| Secret | Required | Default | Description |
|--------|----------|---------|-------------|
| `OPENAI_API_KEY` | ✅ | - | OpenAI API key |
| `OPENAI_BASE_URL` | ❌ | - | Custom API endpoint |
| `CODEX_MODEL` | ❌ | `gpt-5.2` | Model for Codex |
| `CODEX_REASONING_EFFORT` | ❌ | `xhigh` | Reasoning effort level |

### 2. Add Workflow

#### Option A: Full Setup (Recommended)

Copy [`examples/codex-automation.yml`](examples/codex-automation.yml) to your repo:

```bash
mkdir -p .github/workflows
curl -o .github/workflows/codex.yml \
  https://raw.githubusercontent.com/MonteYin/codex-actions/main/examples/codex-automation.yml
```

#### Option B: Minimal Setup

Only PR review and issue triage:

```bash
mkdir -p .github/workflows
curl -o .github/workflows/codex.yml \
  https://raw.githubusercontent.com/MonteYin/codex-actions/main/examples/codex-minimal.yml
```

#### Option C: Direct Workflow Calls

```yaml
# .github/workflows/pr-review.yml
name: PR Review

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]

jobs:
  review:
    uses: MonteYin/codex-actions/.github/workflows/reusable-pr-review.yml@v1
    secrets: inherit
```

```yaml
# .github/workflows/issue-triage.yml
name: Issue Triage

on:
  issues:
    types: [opened]

jobs:
  triage:
    uses: MonteYin/codex-actions/.github/workflows/reusable-issue-triage.yml@v1
    secrets: inherit
```

## Available Reusable Workflows

### PR Review

```yaml
uses: MonteYin/codex-actions/.github/workflows/reusable-pr-review.yml@v1
secrets: inherit
```

### PR Label

```yaml
uses: MonteYin/codex-actions/.github/workflows/reusable-pr-label.yml@v1
secrets: inherit
```

### PR Description

```yaml
uses: MonteYin/codex-actions/.github/workflows/reusable-pr-description.yml@v1
with:
  min_description_length: 300  # optional
secrets: inherit
```

### Issue Triage

```yaml
uses: MonteYin/codex-actions/.github/workflows/reusable-issue-triage.yml@v1
with:
  enable_dedupe: true  # optional
secrets: inherit
```

### Issue Response

```yaml
uses: MonteYin/codex-actions/.github/workflows/reusable-issue-auto-response.yml@v1
secrets: inherit
```

### Mention Response

```yaml
uses: MonteYin/codex-actions/.github/workflows/reusable-mention-responder.yml@v1
with:
  mention_trigger: '@codex'  # optional
secrets: inherit
```

### Issue Stale Cleanup

```yaml
uses: MonteYin/codex-actions/.github/workflows/reusable-issue-stale-cleanup.yml@v1
with:
  stale_days: 30   # optional
  close_days: 14   # optional
secrets: inherit
```

## Configuration

### Secrets

| Secret | Required | Default | Description |
|--------|----------|---------|-------------|
| `OPENAI_API_KEY` | ✅ | - | OpenAI API key |
| `OPENAI_BASE_URL` | ❌ | - | Custom API endpoint |
| `CODEX_MODEL` | ❌ | `gpt-5.2` | Model for Codex |
| `CODEX_REASONING_EFFORT` | ❌ | `xhigh` | Reasoning effort level |

### Optional Inputs

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `runner` | string | `ubuntu-latest` | Custom runner |

## Directory Structure

```
codex-actions/
├── .github/
│   ├── workflows/
│   │   ├── codex.yml           # This repo's workflow
│   │   └── reusable-*.yml      # Reusable workflows (7)
│   ├── prompts/                # AI prompts
│   └── ISSUE_TEMPLATE/         # Issue templates
├── examples/
│   ├── codex-automation.yml    # Full caller workflow
│   └── codex-minimal.yml       # Minimal caller workflow
└── README.md
```

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

### Development

1. Fork the repository
2. Create your feature branch
3. Make your changes
4. Submit a pull request

## License

MIT
