# Research Tracker Design

## Overview

A GitHub Action that tracks AI/ML repositories daily to identify research opportunities. It analyzes commits, PRs, issues, and discussions to surface signals like new architectures, performance breakthroughs, exposed limitations, and overlooked contributions.

## Architecture

```
codex-actions/
├── .github/
│   ├── workflows/
│   │   ├── reusable-repo-tracker.yml        # Core: analyze single repo
│   │   └── reusable-tracker-dispatcher.yml  # Dispatcher: trigger all repos
│   └── prompts/
│       └── repo-tracker.md                  # AI analysis prompt
```

### Components

| Component | File | Responsibility |
|-----------|------|----------------|
| Dispatcher | `reusable-tracker-dispatcher.yml` | Parse repo list, trigger parallel analysis, retry failures, send summary webhook |
| Tracker | `reusable-repo-tracker.yml` | Analyze single repo, update Markdown, send per-repo webhook |
| Prompt | `prompts/repo-tracker.md` | AI instructions for identifying research opportunities |

## Data Flow

```
Daily cron trigger
    ↓
Dispatcher reads vars.WATCHED_REPOS
    ↓
Parallel trigger N repo-tracker instances
    ↓
Each tracker:
  → Shallow clone + GitHub API
  → Exclude noise files (lockfiles, generated)
  → Read reports/{owner}/{repo}.md history
  → Codex analysis (optimized prompt)
  → Update Markdown (backlog + ID + checkbox)
  → POST webhook (per-repo)
    ↓
Failure? → Retry up to 2x → Still failed? → Alert webhook
    ↓
Dispatcher POST summary webhook
```

## Configuration

### Repository Variables

```
WATCHED_REPOS = "langchain-ai/langchain,openai/openai-python,..."
WEBHOOK_URL = "https://your-webhook.com/..."
```

### Repository Secrets

```
OPENAI_API_KEY = "sk-..."
OPENAI_BASE_URL = "" (optional)
CODEX_MODEL = "gpt-5.3-codex" (optional)
```

## Workflow Specifications

### reusable-repo-tracker.yml

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `repo` | Yes | - | Target repository (owner/repo) |
| `webhook_url` | No | - | Webhook URL for per-repo report |
| `lookback_hours` | No | 24 | Hours to look back for activity |
| `runner` | No | auto | Runner to use |

**Steps:**

1. Checkout caller repository (for reports/)
2. Shallow clone target repository (`--depth 50 --since="24 hours ago"`)
3. Fetch repo activity via GitHub API (PRs, Issues, Commits)
4. Preprocess diffs (exclude lockfiles, generated files)
5. Load historical context from `reports/{owner}/{repo}.md`
6. Run Codex analysis with optimized prompt
7. Update Markdown report
8. Commit & Push changes
9. POST webhook (if configured)

### reusable-tracker-dispatcher.yml

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `repos` | Yes | - | Comma-separated repo list |
| `webhook_url` | No | - | Webhook URL for summary |
| `lookback_hours` | No | 24 | Hours to look back |

**Jobs:**

1. **prepare**: Parse repo list into matrix
2. **analyze**: Matrix job calling repo-tracker for each repo (`fail-fast: false`)
3. **collect-failures-1**: Identify failed repos
4. **retry-1**: Retry failed repos (first attempt)
5. **collect-failures-2**: Identify still-failed repos
6. **retry-2**: Retry failed repos (second attempt)
7. **handle-final-failures**: Alert webhook for repos that failed 3x, log to `reports/failures.md`
8. **summarize**: Aggregate results, POST summary webhook

## AI Prompt Design

Located at `.github/prompts/repo-tracker.md`.

### Key Features

1. **Noise Filtering**: Explicitly ignore CI/CD, docs formatting, version bumps
2. **Signal Categories**:
   - `new_architecture`: Paper implementations, novel approaches
   - `performance`: Benchmark improvements, optimizations
   - `limitation`: Failure cases, edge cases, architectural flaws
   - `hot_discussion`: High-engagement debates
   - `overlooked`: High-potential low-attention PRs
   - `upstream`: Breaking changes in dependencies
3. **Synthesis**: Combine related events into single opportunities
4. **History Comparison**: Avoid re-reporting known content
5. **Strict JSON Output**: No markdown formatting, no invented references

### Output Schema

```json
{
  "repo": "owner/repo",
  "analysis_date": "YYYY-MM-DD",
  "summary": "One-sentence narrative",
  "opportunities": [
    {
      "type": "new_architecture | performance | limitation | hot_discussion | overlooked | upstream",
      "title": "Concise title",
      "description": "Technical explanation",
      "evidence": ["PR #123", "Issue #456"],
      "research_potential": "high | medium | low",
      "suggested_action": "Concrete next step"
    }
  ],
  "notable_changes": [
    {
      "type": "commit | pr | issue | release",
      "ref": "#123 or commit hash",
      "summary": "Brief description"
    }
  ],
  "no_significant_activity": false
}
```

## Markdown Report Structure

**File:** `reports/{owner}/{repo}.md`

```markdown
# {repo} Research Tracker

## Active Research Backlog
*(Aggregated action items from daily logs)*

- **[HIGH] R-2025-01-15-01: Title**
  - Status: Todo/Researching/Done
  - Action: [ ] Specific task

---

## 2025-01-15
**Summary:** One-sentence summary

### Opportunities Identified

#### [HIGH] Title (ID: R-2025-01-15-01)
- **Type:** `type`
- **Evidence:** [PR #123](url), [Issue #456](url)
- **Hypothesis:** Why this matters
- **Suggested Action:**
  - [ ] Specific task

---

<details>
<summary>Archived (older than 7 days)</summary>

## 2025-01-08
...

</details>
```

### Features

- **Active Backlog**: Top section aggregates all pending research items
- **Opportunity IDs**: Unique identifiers (R-YYYY-MM-DD-NN) for tracking
- **Status Indicators**: Todo/Researching/Done
- **Checkboxes**: Actionable tasks
- **Collapsible Archive**: Old entries hidden in `<details>`

## Webhook Payloads

### Per-Repo Analysis

```json
{
  "type": "repo_analysis",
  "repo": "owner/repo",
  "analysis_date": "2025-01-15",
  "summary": "Summary text",
  "opportunities": [...],
  "report_url": "https://github.com/.../reports/owner/repo.md"
}
```

### Daily Summary

```json
{
  "type": "daily_summary",
  "analysis_date": "2025-01-15",
  "repos_analyzed": 5,
  "repos_failed": 0,
  "total_opportunities": {
    "high": 2,
    "medium": 5,
    "low": 3
  },
  "top_opportunities": [...],
  "run_url": "https://github.com/.../actions/runs/123"
}
```

### Failure Alert

```json
{
  "type": "alert",
  "level": "error",
  "analysis_date": "2025-01-15",
  "message": "Repos failed after 2 retries",
  "failed_repos": ["owner/repo"],
  "run_url": "https://github.com/.../actions/runs/123"
}
```

## Usage Example

### Caller Workflow

```yaml
# .github/workflows/research-tracker.yml
name: Research Tracker

on:
  schedule:
    - cron: '0 8 * * *'  # Daily at 8am UTC
  workflow_dispatch:

jobs:
  track:
    uses: MonteYin/codex-actions/.github/workflows/reusable-tracker-dispatcher.yml@v1
    with:
      repos: ${{ vars.WATCHED_REPOS }}
      webhook_url: ${{ vars.WEBHOOK_URL }}
    secrets: inherit
```

### Repository Configuration

```
Settings → Secrets and variables → Actions → Variables:
  WATCHED_REPOS = "langchain-ai/langchain,openai/openai-python"
  WEBHOOK_URL = "https://hooks.example.com/research"

Settings → Secrets and variables → Actions → Secrets:
  OPENAI_API_KEY = "sk-..."
```

## Diff Preprocessing

Exclude noise files before analysis:

```bash
git diff HEAD~50..HEAD \
  -- . \
  ':!package-lock.json' \
  ':!yarn.lock' \
  ':!pnpm-lock.yaml' \
  ':!poetry.lock' \
  ':!Cargo.lock' \
  ':!*.min.js' \
  ':!*.min.css' \
  ':!*.pb.go' \
  ':!*.generated.*' \
  > /tmp/processed_diff.txt
```

## Retry Strategy

| Attempt | Action |
|---------|--------|
| 1 | Initial analysis |
| 2 | First retry (after failure) |
| 3 | Second retry (final attempt) |
| - | Alert webhook + log to failures.md |

## Future Considerations

- Support for custom focus keywords per repo
- Trend detection across multiple days
- Integration with note-taking systems
- RSS feed generation
