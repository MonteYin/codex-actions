# Code Patterns and Conventions

## Workflow File Pattern (reusable-*.yml)

Every reusable workflow follows this structure:
1. `on.workflow_call` with standard inputs (`runner`) + secrets (`OPENAI_API_KEY`, `OPENAI_BASE_URL`, `CODEX_MODEL`, `CODEX_REASONING_EFFORT`)
2. Job runs on `node:20-bullseye` container with runner auto-detection
3. Steps: checkout target repo → checkout prompts (sparse) → setup node → install deps → optional data prep → run codex → post-process with `actions/github-script@v7`

## Codex CLI Setup Pattern

```bash
mkdir -p "$CODEX_HOME"
MODEL="${{ secrets.CODEX_MODEL }}"
MODEL="${MODEL:-gpt-5.4}"
EFFORT="${{ secrets.CODEX_REASONING_EFFORT }}"
EFFORT="${EFFORT:-xhigh}"
cat > "$CODEX_HOME/config.toml" << TOML
model = "$MODEL"
model_reasoning_effort = "$EFFORT"
TOML
echo "{\"OPENAI_API_KEY\": \"${{ secrets.OPENAI_API_KEY }}\"}" > "$CODEX_HOME/auth.json"

PROMPT=$(cat .codex-actions/.github/prompts/<name>.md)
codex exec --skip-git-repo-check --full-auto --output-last-message /tmp/codex-output.json "$PROMPT" || true
```

## Prompt Variable Substitution

Use `${VAR}` placeholders in prompt .md files, replace with bash:
```bash
PROMPT="${PROMPT//\$\{SCAN_PATHS\}/$SCAN_PATHS}"
```

## Post-Processing Pattern (actions/github-script@v7)

1. Check `/tmp/codex-output.json` exists
2. `JSON.parse` with try-catch (return on failure)
3. Ensure labels exist (getLabel → createLabel on 404)
4. Process results with GitHub API calls

## Deduplication Pattern (code-scan)

Embed fingerprint as HTML comment `<!-- fingerprint:xxx -->` in issue body. Query existing issues by label, regex-extract fingerprints, skip duplicates.

## Commit Message Style

```
<type>: <short description>
```
Types: `feat`, `chore`, `fix`. No Co-Authored-By lines.

## Versioning

- `v1` tag is a floating tag, force-updated on new features
- `v1.x.x` tags are immutable point releases
- Consumer workflows reference `@v1` for auto-updates
