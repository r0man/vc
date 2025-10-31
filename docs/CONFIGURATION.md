# Configuration Reference

This document contains all environment variable configuration options for VC.

---

## 🔍 Deduplication Configuration

VC uses AI-powered deduplication to prevent filing duplicate issues. This feature can be tuned via environment variables to balance between avoiding duplicates and avoiding false positives.

### Default Configuration (Performance Optimized)

The default settings are optimized for performance while maintaining accuracy:

- **Confidence threshold**: 0.85 (85%) - High confidence required to mark as duplicate
- **Lookback window**: 7 days - Only compare against issues from the past week
- **Max candidates**: 25 - Compare against up to 25 recent issues (reduced from 50 for speed)
- **Batch size**: 50 - Process 50 comparisons per AI call (increased from 10 for efficiency)
- **Within-batch dedup**: Enabled - Deduplicate within the same batch of discovered issues
- **Fail-open**: Enabled - File the issue if deduplication fails (prefer duplicates over lost work)
- **Include closed issues**: Disabled - Only compare against open issues
- **Min title length**: 10 characters - Skip dedup for very short titles
- **Max retries**: 2 - Retry AI calls twice on failure
- **Request timeout**: 30 seconds - Timeout for AI API calls

**Performance Impact** (vc-159):
With 3 discovered issues and default config:
- **Old** (BatchSize=10, MaxCandidates=50): ~15 AI calls, ~90 seconds
- **New** (BatchSize=50, MaxCandidates=25): ~3 AI calls, ~18 seconds
- **Result**: 80% reduction in API calls and deduplication time!

### Environment Variables

All deduplication settings can be customized via environment variables:

```bash
# Confidence threshold (0.0 to 1.0, default: 0.85)
# Higher = more conservative (fewer false positives, more false negatives)
# Lower = more aggressive (more false positives, fewer false negatives)
export VC_DEDUP_CONFIDENCE_THRESHOLD=0.85

# Lookback period in days (default: 7)
# How many days of recent issues to compare against
export VC_DEDUP_LOOKBACK_DAYS=7

# Maximum number of issues to compare against (default: 50)
# Limits AI API costs and processing time
export VC_DEDUP_MAX_CANDIDATES=50

# Batch size for AI calls (default: 10)
# Number of comparisons to send in a single AI API call
export VC_DEDUP_BATCH_SIZE=10

# Enable within-batch deduplication (default: true)
# If multiple discovered issues are duplicates of each other, only keep the first
export VC_DEDUP_WITHIN_BATCH=true

# Fail-open behavior (default: true)
# If true: file the issue anyway when deduplication fails
# If false: return error and block issue creation
export VC_DEDUP_FAIL_OPEN=true

# Include closed issues in comparison (default: false)
# Useful for preventing re-filing of recently closed issues
export VC_DEDUP_INCLUDE_CLOSED=false

# Minimum title length for deduplication (default: 10)
# Very short titles lack semantic meaning for comparison
export VC_DEDUP_MIN_TITLE_LENGTH=10

# Maximum retry attempts (default: 2)
# Number of times to retry AI API calls on failure
export VC_DEDUP_MAX_RETRIES=2

# Request timeout in seconds (default: 30)
# Timeout for individual AI API calls
export VC_DEDUP_TIMEOUT_SECS=30
```

### Tuning Guidelines

**To reduce false positives** (issues incorrectly marked as duplicates):
- Increase `VC_DEDUP_CONFIDENCE_THRESHOLD` to 0.90 or 0.95
- Decrease `VC_DEDUP_MAX_CANDIDATES` to compare against fewer issues
- Decrease `VC_DEDUP_LOOKBACK_DAYS` to only compare against very recent issues

**To reduce false negatives** (actual duplicates not caught):
- Decrease `VC_DEDUP_CONFIDENCE_THRESHOLD` to 0.75 or 0.80 (use with caution)
- Increase `VC_DEDUP_MAX_CANDIDATES` to compare against more issues
- Increase `VC_DEDUP_LOOKBACK_DAYS` to compare against older issues
- Enable `VC_DEDUP_INCLUDE_CLOSED=true` to catch recently closed duplicates

**To reduce costs**:
- Decrease `VC_DEDUP_MAX_CANDIDATES` to limit API calls
- Decrease `VC_DEDUP_LOOKBACK_DAYS` to narrow the search window
- Increase `VC_DEDUP_BATCH_SIZE` to make fewer API calls (up to 100)

**For debugging**:
- Set `VC_DEDUP_CONFIDENCE_THRESHOLD=1.0` to effectively disable deduplication
- Set `VC_DEDUP_MAX_CANDIDATES=0` to skip deduplication entirely
- Check logs for `[DEDUP]` messages showing comparison results

### Example Configurations

**Conservative Configuration** (for critical projects where missing work is worse than having duplicates):

```bash
export VC_DEDUP_CONFIDENCE_THRESHOLD=0.95  # Very high confidence required
export VC_DEDUP_FAIL_OPEN=true             # File on error
export VC_DEDUP_MAX_CANDIDATES=30          # Limited comparisons
```

**Aggressive Configuration** (for projects with lots of duplicate work being filed):

```bash
export VC_DEDUP_CONFIDENCE_THRESHOLD=0.75  # Lower threshold
export VC_DEDUP_LOOKBACK_DAYS=14           # Longer lookback
export VC_DEDUP_MAX_CANDIDATES=100         # More candidates
export VC_DEDUP_INCLUDE_CLOSED=true        # Include closed issues
```

### Configuration Validation

The executor validates all deduplication settings on startup. Invalid values (out of range, wrong type, etc.) will cause the executor to exit with a clear error message.

Validation checks:
- `VC_DEDUP_CONFIDENCE_THRESHOLD` must be between 0.0 and 1.0
- `VC_DEDUP_LOOKBACK_DAYS` must be between 1 and 90 days
- `VC_DEDUP_MAX_CANDIDATES` must be between 0 and 500
- `VC_DEDUP_BATCH_SIZE` must be between 1 and 100
- `VC_DEDUP_MIN_TITLE_LENGTH` must be between 0 and 500
- `VC_DEDUP_MAX_RETRIES` must be between 0 and 10
- `VC_DEDUP_TIMEOUT_SECS` must be between 1 and 300 seconds

See [docs/QUERIES.md](./QUERIES.md) for queries to monitor deduplication metrics.

---

## 🐛 Debug Environment Variables

**Debug Prompts:**
```bash
# Log full prompts sent to agents (useful for debugging agent behavior)
export VC_DEBUG_PROMPTS=1
```

**Debug Events:**
```bash
# Log JSON event parsing details (tool_use events from Amp --stream-json)
export VC_DEBUG_EVENTS=1
```

---

## 🔑 AI Supervision Configuration

**ANTHROPIC_API_KEY** (Required for AI supervision):
```bash
# Required for AI supervision (assessment and analysis)
export ANTHROPIC_API_KEY=your-key-here
```

Without this key, the executor will run without AI supervision (warnings will be logged).

AI supervision can be explicitly disabled via config: `EnableAISupervision: false`

---

## 🗄️ Event Retention Configuration (Future Work)

**Status:** Not yet implemented. Punted until database size becomes a real issue (vc-184, vc-198).

### Why Punted?

Following the lesson learned from deduplication metrics (vc-151), we're deferring event retention infrastructure until we have real production data showing it's needed. This avoids building observability for theoretical future problems.

### When to Implement

Implement event retention when:
- `.beads/vc.db` exceeds 100MB
- Query performance degrades noticeably
- Developers complain about database size
- Event table has >100k rows

Until then: **YAGNI** (You Aren't Gonna Need It).

### Proposed Configuration

When we do implement this, here's the plan:

**Retention Policy Tiers:**
- **Regular events** (progress, file_modified, etc.): 30 days
- **Critical events** (error, watchdog_alert): 180 days
- **Per-issue limit**: 1000 events max per issue
- **Global limit**: Configurable, default 50k events

**Proposed Environment Variables:**
```bash
# Event retention in days (default: 30)
export VC_EVENT_RETENTION_DAYS=30

# Critical event retention in days (default: 180)
export VC_EVENT_CRITICAL_RETENTION_DAYS=180

# Per-issue event limit (default: 1000, 0 = unlimited)
export VC_EVENT_PER_ISSUE_LIMIT=1000

# Global event limit (default: 50000, 0 = unlimited)
export VC_EVENT_GLOBAL_LIMIT=50000

# Cleanup frequency in hours (default: 24)
export VC_EVENT_CLEANUP_INTERVAL_HOURS=24

# Batch size for cleanup (default: 1000)
export VC_EVENT_CLEANUP_BATCH_SIZE=1000
```

**Cleanup Strategy:**
- Run as background goroutine in executor
- Execute every 24 hours (configurable)
- Transaction-based deletion in batches of 1000
- Log cleanup metrics (events deleted, time taken)

**CLI Command (Not Yet Implemented):**
```bash
# Manual cleanup trigger
vc cleanup events --dry-run  # Preview what would be deleted
vc cleanup events             # Execute cleanup
vc cleanup events --force     # Bypass safety checks
```

### Related Issues

- vc-183: Agent Events Retention and Cleanup [OPEN - Low Priority]
- vc-184: Design event retention policy [CLOSED - Design complete]
- vc-193 through vc-197: Implementation tasks [OPEN - Punted]
- vc-199: Tests for event retention [OPEN - Punted]

**Remember:** Build this when you need it, not before. Let real usage drive the requirements.

See [docs/QUERIES.md](./QUERIES.md) for event retention monitoring queries (for future use).
