---
name: code-clairvoyant
description: Static analysis + ML-powered bug prediction engine that identifies potential runtime failures, race conditions, memory leaks, and security vulnerabilities before code executes
version: 2.1.0
author: OpenClaw Team
tags:
  - prediction
  - bugs
  - prevention
  - static-analysis
  - ml
dependencies:
  - python>=3.9
  - nodejs>=16.0.0
  - semgrep>=1.50.0
  - bandit>=1.7.5
  - saas-ai-predictor>=0.8.2
  - typescript>=5.0.0
  - esbuild>=0.19.0
  - flamegraph>=0.3.0
requires_openclaw: ">=2.0.0"
---

# Code Clairvoyant Skill

## Purpose

Code Clairvoyant analyzes codebases to predict potential bugs and vulnerabilities before they manifest at runtime. It combines pattern matching, dataflow analysis, and ML models trained on millions of bug-fix commits to identify:

- **Race conditions** in async/await and multi-threaded code
- **Memory leaks** in resource allocation patterns
- **Null dereference** risks from incomplete null-safety
- **Security vulnerabilities** (injection, XSS, path traversal)
- **Performance antipatterns** (O(n²) loops in hot paths, unnecessary re-renders)
- **Type inconsistency** that TypeScript compiler might miss
- **API misuse** (unhandled rejections, missing error handling)
- **Configuration drift** (environment-dependent bugs)

### Real Use Cases

1. **Pre-commit Gate**: Run automatically on `git commit` to block bug-prone code
2. **CI/CD Integration**: Fail builds when high-severity predictions exceed threshold
3. **Refactor Planning**: Identify risky hotspots before major refactoring
4. **Onboarding**: New developers learn project-specific pitfalls
5. **Code Review Augmentation**: Surface issues reviewers might miss
6. **Technical Debt Quantification**: Track prediction density over time

## Scope

### Commands

- `code-clairvoyant analyze [path]` - Full scan with all detectors
- `code-clairvoyant predict --type=<type> [files]` - Targeted prediction for specific bug class
- `code-clairvoyant suggest <prediction-id>` - Generate fix suggestions with code diffs
- `code-clairvoyant verify <prediction-id> --test` - Validate prediction with property-based testing
- `code-clairvoyant review <pr-number> --repo=<url>` - Analyze PR diff only
- `code-clairvoyant profile --cache` - Generate performance profile of analysis itself
- `code-clairvoyant baseline` - Establish current prediction baseline (for tracking)
- `code-clairvoyant export --format=json|csv|sarif` - Output for external tools
- `code-clairvoyant train --dataset=<path>` - Retrain models on custom bug dataset
- `code-clairvoyant revert <prediction-id>` - Rollback auto-applied fixes

### Configuration

Create `~/.openclaw/skills/code-clairvoyant/config.yaml`:

```yaml
thresholds:
  critical: 0.85    # Confidence score to report (0-1)
  warning: 0.65
  info: 0.40
detectors:
  - race_conditions: true
  - memory_leaks: true
  - null_deref: true
  - security: true
  - performance: true
  - type_errors: true
exclude:
  - "**/node_modules/**"
  - "**/dist/**"
  - "**/.git/**"
ml_model: "default-v3"  # Options: default-v3, custom, ensemble
max_files: 10000        # Limit for memory-constrained environments
cache_ttl: 3600        # Seconds to cache analysis results
```

## Work Process

### 1. Environment Setup

Automated on first run. Installs:
```bash
# Dependencies installed into ~/.openclaw/skills/code-clairvoyant/.venv/
pip install -r requirements.txt
npm ci --only=production
```

Download ML models (~250MB):
```bash
saas-ai-predictor download-model --name=bug-classifier-v3 --output=./models/
```

### 2. Code Collection

```bash
# Build file list respecting .gitignore and config excludes
code-clairvoyant collect --source=./src --output=manifest.txt
```

Parallel file reading with memory limit:
```bash
# Uses Rust-based scanner for speed
clairvoyant-scanner --manifest=manifest.txt --max-mb=500 --threads=8
```

### 3. Static Analysis Phase

**Phase 3a: Pattern Matching (Semgrep rules)**
```bash
# Custom OpenClaw ruleset (1200+ rules)
semgrep --config=./rules/clairvoyant.yaml --json --quiet src/
```

**Phase 3b: Dataflow Analysis**
```bash
# Build interprocedural control flow graph (ICFG)
pyopera analyze --cfg --taint src/ --output=icfg.dot
```

**Phase 3c: ML Inference**
```bash
# Extract features from AST, feed to model
saas-ai-predictor infer --model=models/bug-classifier-v3.pt --features=ast_features.json --batch-size=64
```

### 4. Prediction Aggregation

Merge results from all phases, deduplicate by location, assign confidence scores based on:
- Number of detectors that flagged the issue
- ML model probability
- Historical false positive rate for that pattern

### 5. Suggestion Generation

For each prediction:
```bash
# Generate up to 3 fix variants using template-based + LLM synthesis
code-clairvoyant suggest --prediction-id=12345 --strategy=minimal-change
```

### 6. Verification (Optional)

Property-based test generation with Hypothesis:
```bash
code-clairvoyant verify --prediction-id=12345 --test-framework=pytest --iterations=1000
```

### 7. Reporting

Output formats:
- **Terminal**: Color-coded with file:line:col, severity, confidence
- **JSON**: Full prediction objects with suggested diffs
- **SARIF**: GitHub Code Scanning integration
- **HTML**: Interactive report with filter/sort

Example terminal output:
```
src/utils/auth.js:45:12 [CRITICAL] Possible missing await on async validateToken()
  Detected: async function calls without await
  Confidence: 0.92
  Suggestion: await validateToken(user) before proceeding
  ML reasoning: "Pattern: async function returns promise but not awaited"
  Run: code-clairvoyant suggest c49a8f3e
```

## Golden Rules

1. **Never auto-fix without explicit `--force`**: Even high-confidence predictions require human review
2. **Always show reasoning**: Include which detector fired and ML feature attribution
3. **Respect project baselines**: If predictions exceed baseline by >20%, investigate root cause
4. **Cache aggressively**: Reuse results unchanged files (SHA256 hash)
5. **Fail fast on syntax errors**: Invalid code causes undefined behavior in detectors
6. **Respect .gitignore**: Never analyze generated files or vendor dependencies
7. **Guard secrets**: Never log or cache code containing detected API keys/tokens
8. **Rate limit LLM suggestions**: Max 5 suggestions per minute to avoid API throttling
9. **Always provide rollback**: Auto-applied fixes must be reversible in one command
10. **Document false positives**: Add `# clairvoyant-ignore: <rule-id>` with justification

## Examples

### Example 1: Find race conditions in async functions

```bash
# Analyze entire src/ directory
code-clairvoyant analyze src/ --detectors=race_conditions --format=terminal

# Output:
src/handlers/api.js:78:5 [CRITICAL] Non-atomic check-then-act on shared state
  Race condition: checking user.sessionActive then calling user.refresh()
  Confidence: 0.94 (pattern: CHECK_THEN_ACT_ASYNC)
  Suggest: Use mutex lock or atomic compare-and-swap
  code-clairvoyant suggest --id=rc9f2a1b

src/components/Dashboard.tsx:125:15 [WARNING] Missing await on setState inside loop
  Async setState may cause race in React 18+ concurrent mode
  Confidence: 0.71 (pattern: ASYNC_SETSTATE_IN_LOOP)
```

### Example 2: Predict memory leaks in resource-heavy operations

```bash
# Target specific file types
code-clairvoyant predict --type=memory_leaks --include="**/*.go" --threshold=0.8

# Output in JSON for IDE integration:
{
  "predictions": [
    {
      "id": "ml_87b3c4d",
      "file": "server/connection_pool.go",
      "line": 203,
      "severity": "critical",
      "confidence": 0.89,
      "bug_type": "unclosed_http_body",
      "description": "http.Response.Body may not be closed on error path",
      "suggestion": {
        "strategy": "defer_close",
        "diff": "203: +defer resp.Body.Close()\n 204: +if resp.StatusCode != 200 { return fmt.Errorf(...) }"
      }
    }
  ]
}
```

### Example 3: CI/CD integration

```yaml
# .github/workflows/clairvoyant.yml
jobs:
  bug-prediction:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Code Clairvoyant
        run: |
          code-clairvoyant analyze . --format=sarif --output=results.sarif
          # Upload to GitHub Code Scanning
          github/codeql-action/upload-sarif@v2 --sarif-file=results.sarif
        env:
          CLAIRVOYANT_THRESHOLD: "0.75"  # Fail below this confidence
      - name: Check for critical predictions
        run: |
          CRITICAL=$(code-clairvoyant export --format=json | jq '[.predictions[] | select(.severity=="critical")] | length')
          if [ "$CRITICAL" -gt 5 ]; then
            echo "Critical predictions exceed threshold: $CRITICAL"
            exit 1
          fi
```

### Example 4: PR review bot integration

```bash
# Analyze only changed files in PR
code-clairvoyant review --repo=https://github.com/owner/repo --pr=456 --token=$GITHUB_TOKEN

# Auto-comment on PR:
/run-code-clairvoyant --detectors=all --threshold=0.7

# Bot responds:
Code Clairvoyant found 3 issues in this PR:
1. src/lib/db.js:87 - Possible SQL injection via unescaped parameter
2. src/api/upload.js:45 - Missing file type validation
3. src/utils/payment.js:201 - Unhandled promise rejection
See full report: https://clairvoyant.openclaw.io/pr/456
```

### Example 5: Train custom model for project-specific patterns

```bash
# Export historical bugs from Jira/GitHub to training format
code-clairvoyant export-bugs --since="2023-01-01" --output=training-data/

# Format: training-data/<bug-id>/buggy_code.ext, training-data/<bug-id>/fixed_code.ext

# Train new model
code-clairvoyant train --dataset=training-data/ --epochs=50 --validation-split=0.2

# Output:
Epoch 1/50: loss=0.342, val_loss=0.415
Epoch 50/50: loss=0.078, val_loss=0.092
Model saved: models/custom-project-v1.pt
Accuracy on validation: 94.2%

# Deploy custom model
code-clairvoyant config set ml_model=custom-project-v1
```

### Example 6: Baseline tracking for tech debt

```bash
# Establish baseline on main branch
git checkout main
code-clairvoyant baseline --name=v1.0 --output=baseline.json

# Later, compare feature branch
git checkout feature/new-api
code-clairvoyant baseline --compare=../main-baseline.json

# Output:
Compared against baseline (v1.0):
  New predictions: +12
  Resolved predictions: -3
  Net change: +9 (tech debt increase)
  Prediction density: 3.4 per KLOC (baseline: 2.1)
  
  Top new hotspots:
    src/api/v2/user_service.go:45:45 [CRITICAL] Missing rate limiter
    src/api/v2/user_service.go:78:12 [WARNING] Unencrypted PII in logs
```

## Rollback Commands

### 1. Rollback a single auto-applied fix

```bash
# List applied fixes
code-clairvoyant history --applied-only

# Output:
Applied fixes (last 10):
  ID           Applied At             File
  fix_9f2a1b   2026-03-07 14:22:01   src/auth.js
  fix_3c8d4e   2026-03-07 14:20:45   src/api/client.ts

# Revert specific fix
code-clairvoyant revert fix_9f2a1b
# Restores src/auth.js from git HEAD, creates backup at src/auth.js.clairvoyant-backup
```

### 2. Revert all fixes from a session

```bash
# Revert last N fixes
code-clairvoyant revert --last=5

# Revert fixes applied before specific timestamp
code-clairvoyant revert --before="2026-03-07 13:00:00"
```

### 3. Bulk rollback with verification

```bash
# Dry run first
code-clairvoyant revert --pattern="*.go" --dry-run

# Shows: Would revert 8 fixes across 5 files

# Execute with confirmation
code-clairvoyant revert --pattern="src/memory/**" --confirm
```

### 4. Git-based recovery

```bash
# If revert command fails, restore from git
git checkout HEAD -- src/modified/file.js

# Or restore specific commit
git show <commit-before-clairvoyant>:src/file.js > src/file.js
```

### 5. Configuration rollback

```bash
# If configuration changes cause issues
code-clairvoyant config reset --to-default

# Or restore backup
cp ~/.openclaw/skills/code-clairvoyant/backups/config-20260307.yaml ~/.openclaw/skills/code-clairvoyant/config.yaml
```

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `CLAIRVOYANT_THRESHOLD` | No | 0.65 | Minimum confidence score (0-1) to report predictions |
| `CLAIRVOYANT_CACHE_DIR` | No | `~/.cache/clairvoyant` | Override cache location |
| `CLAIRVOYANT_MODEL_PATH` | No | `./models/` | Path to custom ML models |
| `CLAIRVOYANT_MAX_WORKERS` | No | `auto` | Worker threads for analysis (1-N) |
| `CLAIRVOYANT_LLM_API_KEY` | No | - | API key for suggestion generation (OpenAI/Anthropic) |
| `CLAIRVOYANT_DISABLE_ML` | No | `false` | Set to `true` to use only rule-based detectors |

## Dependencies

### Python Requirements (`requirements.txt`)
```
semgrep==1.50.0
bandit==1.7.5
saas-ai-predictor @ git+https://github.com/openclaw/saas-ai-predictor@v0.8.2
networkx==3.2
astroid==3.0.1
hypothesis==6.82.0
pytest==7.4.0
flask==3.0.0
sqlalchemy==2.0.23
```

### Node.js Requirements (`package.json`)
```json
{
  "dependencies": {
    "typescript": "^5.0.0",
    "esbuild": "^0.19.0",
    "@typescript-eslint/parser": "^6.0.0",
    "eslint": "^8.0.0",
    "flamegraph": "^0.3.0"
  }
}
```

### System Dependencies
- `graphviz` (for CFG visualization)
- `ripgrep` (fast file scanning)
- `git` (for history/baseline comparison)

## Verification Steps

### Post-installation check

```bash
# Verify all components
code-clairvoyant check

# Expected output:
[OK] Python 3.11.5 found
[OK] Node.js 18.17.0 found
[OK] Semgrep 1.50.0 installed
[OK] ML model downloaded (247MB)
[OK] Cache directory writable
[OK] Configuration valid
All systems ready ✓
```

### Verify on known bug

```bash
# Create test file with intentional bug
cat > test_race.js << 'EOF'
let counter = 0;
async function increment() {
  const temp = counter;
  await new Promise(r => setTimeout(r, 1));
  counter = temp + 1; // Race condition
}
EOF

# Run analysis
code-clairvoyant analyze test_race.js --format=json

# Should detect:
# "race_condition" prediction on line 4 with confidence >0.8
```

### CI Verification

```bash
# Simulate CI run
code-clairvoyant analyze src/ --format=sarif --output=ci-results.sarif
cat ci-results.sarif | grep -q '"level":"error"' && echo "Critical issues found" || echo "Clean"
```

## Troubleshooting

### Issue: "MemoryError during analysis of large codebase"

**Solution**: Reduce worker count or enable incremental mode
```bash
code-clairvoyant analyze src/ --max-workers=2 --incremental
# Or increase cache TTL to reuse results
export CLAIRVOYANT_CACHE_TTL=7200
```

### Issue: "ML model not found / download failed"

**Solution**: Manually download or use rule-only mode
```bash
# Skip ML (less accurate but works)
code-clairvoyant analyze src/ --disable-ml

# Or manually download:
saas-ai-predictor download-model --url=https://cdn.openclaw.ai/models/v3/bug-classifier.pt
```

### Issue: "False positive rate >30%"

**Solution**: Adjust threshold or train custom model
```bash
# Raise threshold to reduce noise
code-clairvoyant analyze src/ --threshold=0.8

# Or collect false positives and retrain:
code-clairvoyant mark-fp --prediction-id=abc123 --reason="false positive: lock is actually held"
code-clairvoyant train --dataset=$(code-clairvoyant export-fp-dataset) --fine-tune
```

### Issue: "Too slow (>10 minutes)"

**Solution**: Enable caching, limit detectors, exclude files
```bash
# First run will be slow, subsequent runs cached
code-clairvoyant analyze src/ --cache-ttl=86400

# Disable expensive detectors
code-clairvoyant analyze src/ --detectors=race_conditions,security --skip=performance

# Exclude test files
echo "**/*.test.*" >> ~/.openclaw/skills/code-clairvoyant/.ignore
```

### Issue: "Detector crashes on invalid syntax"

**Solution**: Pre-validate all files, fix syntax errors first
```bash
# Find syntax errors
find src/ -name "*.js" -exec node --check {} \; 2>&1 | grep -v "SyntaxError"

# Or let clairvoyant skip invalid files:
code-clairvoyant analyze src/ --skip-invalid-syntax --strict=false
```

### Issue: "Git history operations too slow"

**Solution**: Disable baseline comparison or limit history range
```bash
code-clairvoyant baseline --days=7  # Only look at last week
# Or skip git entirely:
code-clairvoyant analyze src/ --no-git-context
```
```