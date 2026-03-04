---
name: rapid-executor
description: AI-powered executor for analysis tasks with parallel execution and intelligent retry logic
version: 1.2.0
author: OpenClaw Team
tags:
  - analysis
  - ai
  - automation
  - execution
  - performance
dependencies:
  - python>=3.9
  - jq>=1.6
  - parallel>=20161222
  - git
  - bash>=5.0
required_tools:
  - rapid-exec
  - rapid-analyze
  - rapid-cache
os:
  - linux
  - darwin
---

# Rapid Executor

AI-powered execution engine for analysis tasks with intelligent parallelization, dynamic batching, and adaptive retry strategies.

## Purpose

Execute complex analysis pipelines across multiple targets with optimal resource utilization. Designed for:
- Parallel codebase analysis across repositories (RPGCLAW, FlickClaw simultaneous scans)
- Batch processing of log files from multiple VPS servers
- Concurrent dependency vulnerability scans with automatic rate-limit handling
- Distributed test execution with result aggregation
- Performance benchmarking across environment variants

## Scope

### Primary Commands

```
# Execute analysis task with AI-driven batching
rapid-exec --task <task_name> --targets <targets_file> [--ai-optimize] [--max-parallel N]

# Analyze repository with smart detection
rapid-analyze --repo <path|url> [--depth N] [--include-tests] [--output-format json|tariff]

# Cache management for repeated executions
rapid-cache --policy <lru|ttl|size> --max-size <bytes>

# Distributed execution via SSH
rapid-exec --ssh-hosts <hosts_file> --command "<cmd>" [--env-file <env>]

# Smart retry with exponential backoff
rapid-exec --task <task> --retry-attempts N --retry-delay <seconds>
```

### Configuration Files

- `~/.config/rapid/executor.conf` - Main configuration
- `.rapidignore` - Exclude patterns (similar to .gitignore)
- `rapid-tasks.yaml` - Task definitions with AI-suggested parameters

## Detailed Work Process

### 1. Target Discovery Phase
- Scan input sources (file, stdin, --targets-file)
- Deduplicate targets using content-aware hashing
- Classify targets by type (repo, server, log, test-suite)
- Estimate resource requirements per target

### 2. AI Optimization Phase (if --ai-optimize enabled)
- Analyze historical execution data from rapid-cache
- Predict execution time per target based on:
  - Repository size (git rev-list --count HEAD)
  - File count (find . -type f | wc -l)
  - Dependency complexity (package.json/go.mod/pyproject.toml parsing)
- Generate optimal batch sizes to maximize parallelism without resource exhaustion
- Create execution plan with priority ordering

### 3. Execution Engine
```bash
# Real execution flow:
INPUT_TARGETS=$(cat $TARGETS_FILE | grep -v '^#' | grep -v '^$')
TOTAL=${#INPUT_TARGETS[@]}

# AI-optimized batch calculation
BATCH_SIZE=$(rapid-calc-optimal-batch --targets $TOTAL --memory $AVAIL_RAM --cpu $CPU_CORES)

# Parallel execution with progress tracking
parallel --jobs $MAX_PARALLEL --joblog rapid.log --progress \
  "rapid-run-single {} --context $EXECUTION_ID" ::: ${INPUT_TARGETS[@]}

# Real-time aggregation
tail -f rapid.log | rapid-aggregate --format json > results/$TASK_NAME.json
```

### 4. Result Processing
- Validate exit codes (non-zero = failure)
- Capture stdout/stderr to cache with target hash
- Generate executive summary with AI insights:
  - Slowest targets identified
  - Failure patterns detected
  - Resource bottlenecks highlighted
- Store in structured format (JSON with metadata)

### 5. Reporting
- Console output with spinner/progress bar
- `results/` directory with per-target outputs
- `summary.json` with aggregated metrics
- Optional webhook post to CI/CD system

## Golden Rules

1. **Never disable --ai-optimize for large sets (>50 targets)** - Manual batching causes resource starvation
2. **Always use SSH key authentication** - Password prompts break parallel execution
3. **Cache is authoritative** - Never delete `~/.cache/rapid/` without `rapid-cache --evict-all`
4. **Target file must be newline-separated, one per line** - Spaces/comments cause silent failures
5. **Exit code propagation is mandatory** - Always check `$?` after rapid-exec
6. **Memory limits must be set** - Unbounded parallel analysis crashes OOM
7. **Log files must be rotated** - rapid.log grows indefinitely; use `logrotate`
8. **Task definitions must be version-controlled** - Store `rapid-tasks.yaml` in repo
9. **SSH hosts file format: `user@host:port`** - Missing port defaults to 22
10. **Never run rapid-exec as root** - Creates permission issues in cache

## Examples

### Example 1: Analyze all RPGCLAW microservices
```bash
# Discover services
find services/ -name package.json -exec dirname {} \; > targets.txt

# Execute with AI optimization
rapid-exec \
  --task dependency-scan \
  --targets targets.txt \
  --ai-optimize \
  --max-parallel 8 \
  --retry-attempts 3 \
  --retry-delay 5

# Output:
# ✓Executed 42/42 targets in 3m17s (avg: 4.7s)
# ⚠ 2 targets exceeded 10s: user-service, payment-service
# ✗ Failed: legacy-auth (port 3000 not responding)
# Cache hit rate: 78%
```

### Example 2: Parallel log analysis across VPS fleet
```bash
# hosts.txt:
# admin@vps-rpgclaw-1.example.com:22
# admin@vps-rpgclaw-2.example.com:22
# admin@vps-flickclaw-1.example.com:3010

rapid-exec \
  --ssh-hosts hosts.txt \
  --command "journalctl --since '2 hours ago' | grep -i error" \
  --env-file .env.ssh \
  --output-format jsonlines \
  --max-parallel 4

# Result saved to: results/log-analysis-20240115.jsonl
# Each line: {host: "vps-rpgclaw-1", errors: 127, samples: [...]}
```

### Example 3: Auto-retry with backoff for flaky network tests
```bash
rapid-analyze \
  --repo ./tests/e2e \
  --include-tests \
  --retry-attempts 5 \
  --retry-delay 2 \
  --retry-backoff multiplicative \
  --max-parallel 2

# Uses AI to identify tests with intermittent failures
# Auto-reruns only failed tests
# Final report shows retry statistics per test
```

### Example 4: Cache-aware repeated execution
```bash
# First run (cold cache)
time rapid-exec --task lint --targets repos.txt
# real    5m23s

# Second run (warm cache, no code changes)
time rapid-exec --task lint --targets repos.txt
# real    1m48s  (66% faster via cache hits)

# Clear only old entries (>7 days)
rapid-cache --policy ttl --max-age 604800
```

### Example 5: Custom task with AI-suggested parameters
```yaml
# rapid-tasks.yaml
tasks:
  security-scan:
    command: "npm audit --json"
    ai_suggestion:
      max_parallel: "target_count < 20 ? target_count : 20"
      retry_attempts: 2
      timeout: 300  # seconds
      memory_per_target_mb: 256
```

rapid-exec --task security-scan --targets services.txt --ai-optimize
# AI calculates: 42 services → batch size 20, 3 parallel batches
```

## Rollback Commands

### Immediate Abort (SIGINT/SIGTERM propagation)
```bash
# Send SIGTERM to all child processes
pkill -P $$ rapid-run-single
# Or use built-in:
rapid-exec --abort --execution-id <id_from_log>
```

### Clean Partial Results
```bash
# Remove incomplete execution artifacts
rm -rf results/$TASK_NAME-*
rm -f rapid.log rapid.partial

# Clear cache for specific task
rapid-cache --evict-task $TASK_NAME --targets-file <file>
```

### Full Cache Invalidation
```bash
# Conservative: remove only entries older than X
rapid-cache --policy ttl --max-age 3600

# Nuclear: wipe entire cache
rapid-cache --evict-all
# Cache rebuild will occur on next execution
```

### Restore Previous State from Backup
```bash
# Daily cache snapshots in ~/.cache/rapid/backups/
# List available backups
ls ~/.cache/rapid/backups/

# Restore specific backup
rapid-restore-cache --backup ~/.cache/rapid/backups/20240115_0200.tar.gz

# After restore, re-run failed tasks only
rapid-exec --task $TASK_NAME --targets failed.txt --resume
```

### SSH Host Rollback (if command modified remote state)
```bash
# Pre-execution backup (before running rapid-exec)
for host in $(cat hosts.txt); do
  ssh $host "pg_dump -Fc mydb > /backups/mydb-$(date +%s).dump" &
done
wait

# After failure detection:
# rapid-exec detects non-zero exit, automatically triggers:
rapid-rollback-ssh --hosts hosts.txt --backup-pattern "/backups/mydb-*.dump"
```

### Resource Quota Restoration
```bash
# If rapid-exec modified system limits:
# e.g., ulimit -n 65536 or sysctl net.core.somaxconn

# Restore from stored baseline
rapid-restore-limits --baseline /etc/security/limits.d/rapid-baseline.conf

# Or manual:
sysctl -p /etc/sysctl.conf
ulimit -n 1024  # Default
```

### CI/CD Pipeline Rollback
```bash
# If rapid-exec triggered downstream deployments
# Check execution metadata:
cat results/$TASK_NAME/metadata.json | jq .deployment_id

# Use deployment ID to rollback via deployment system
deployment-cli rollback $(jq -r .deployment_id results/*/metadata.json)
```

### Verification post-rollback
```bash
# 1. Confirm no rapid-exec children remain
pkill -f "rapid-run-single" && echo "Cleaned"

# 2. Verify cache consistency
rapid-cache --verify

# 3. Check disk space reclaimed
du -sh ~/.cache/rapid/

# 4. Test with single target
rapid-exec --task $TASK_NAME --targets <test_target> --max-parallel 1
```

## Troubleshooting

### High memory usage despite --max-parallel
- Check: `ps aux | grep rapid-run-single | wc -l` (should equal max-parallel)
- Likely cause: targets have unbounded memory growth (node --max-old-space-size not set)
- Fix: Set `RAPID_MEMORY_LIMIT_MB` in env or `--memory-limit` flag

### SSH connection failures
- Verify hosts.txt format: `user@host:port` (port optional)
- Test individual: `ssh -i ~/.ssh/id_rsa admin@host -p 22 echo ok`
- Use `--ssh-options "-o ConnectTimeout=10"` to fail fast

### Cache misses despite no code changes
- Rapid-exec uses content hash of target directory (git HEAD + file mtimes)
- If git repo, check `git status` - uncommitted changes invalidate cache
- Force cache reuse: `rapid-exec --ignore-changes` (dangerous, use only for identical clones)

### AI suggestions seem suboptimal
- Check training data: `ls ~/.cache/rapid/ai-metrics/`
- Rebuild: `rapid-ai-train --historical-logs rapid.log.*.gz`
- Disable AI: `--ai-optimize false` (falls back to conservative defaults)

### Silent failures in parallel execution
- Always check `rapid.log` for individual exit codes
- Use `--strict-mode` to fail entire batch if any target fails
- Enable verbose: `RAPID_LOG_LEVEL=debug rapid-exec ...`
```