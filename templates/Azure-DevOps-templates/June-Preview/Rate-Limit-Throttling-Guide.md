# Azure DevOps Import Rate Limit and Throttling Guide

This guide documents the importer settings that control request pacing and explains how to tune them based on observed progress logs.

Location:
- Script launcher: Run.ps1
- Import client logic: bpc_ado_import/ado.py
- Progress line rendering: bpc_ado_import/cli.py

## 1) Settings That Affect Throttling

Set these as process environment variables (Run.ps1 already sets recommended values):

| Variable | Current Default | Purpose | Increase When | Decrease When |
|---|---:|---|---|---|
| BPC_ADO_IMPORT_PARALLEL_WORKERS | 1 (Run.ps1), importer default is 4 | Concurrent create workers. Higher workers increase burst pressure. | You have no throttling and want more throughput. | You see 429s or long idle waits. |
| BPC_ADO_IMPORT_RETRY_DELAY_SECONDS | 120 (Run.ps1), wrapper default is 30 | Backoff base for transient failures. | Repeated retries happen too quickly. | Retries recover fast and waits are too long. |
| BPC_ADO_IMPORT_SOFT_THROTTLE_THRESHOLD | 0.35 | Soft throttle trigger when remaining/limit drops below threshold. | Soft throttle starts too late. | Throughput is too low with no stalls. |
| BPC_ADO_IMPORT_SOFT_THROTTLE_MAX_DELAY_SECONDS | 3.0 | Maximum proactive sleep added before requests. | Still hitting hard stalls. | Throughput is unnecessarily reduced. |
| BPC_ADO_IMPORT_SOFT_THROTTLE_INITIAL_LIMIT_PERCENT | 0.80 | New trigger: soft throttle also starts when current limit drops below this fraction of initial limit (per resource bucket). | Limit denominator drops quickly before stalls. | Limit denominator is stable and pacing is too conservative. |
| BPC_ADO_IMPORT_HEARTBEAT_SECONDS | 60 | Progress heartbeat interval. Observability only, does not change API rate. | You want more frequent status updates. | Log noise is too high. |
| BPC_ADO_IMPORT_MAX_RETRIES | 8 (wrapper default) | Retry attempts for transient failures. | Intermittent errors recover eventually. | Failures are persistent and should fail fast. |

## 2) How the Current Throttling Strategy Works

The importer uses three layers:

1. Hard header-based gate (highest priority)
- Honors Retry-After, X-RateLimit-Delay, and reset-based wait when remaining is zero.
- Produces progress fragments such as:
  - retry-after 60.0s
  - next request in 247.2s

2. Proactive soft throttle (remaining ratio)
- Computes pressure from remaining/limit.
- Adds pre-request delay up to BPC_ADO_IMPORT_SOFT_THROTTLE_MAX_DELAY_SECONDS when remaining ratio is below BPC_ADO_IMPORT_SOFT_THROTTLE_THRESHOLD.

3. Proactive soft throttle (initial-limit drop)
- Tracks initial X-RateLimit-Limit per resource (for example TFS/Short, TFS/Long).
- If current limit falls below BPC_ADO_IMPORT_SOFT_THROTTLE_INITIAL_LIMIT_PERCENT of that initial limit, soft throttling engages even when remaining ratio alone would not trigger early enough.

Effective delay = max(ratio-triggered delay, initial-limit-triggered delay).

## 3) Reading Progress Log Examples

### Example A: Hard stall (old behavior pattern)

```text
[rate-limit: remaining 0/187, header delay 0.0s, retry-after 60.0s, next request in 278.0s, resource TFS/Short]
```

Interpretation:
- Bucket exhausted (remaining 0).
- A hard wait is now active (next request in ...).
- Import appears idle until this countdown reaches zero.

### Example B: Soft throttle active

```text
[rate-limit: remaining 37/202, limit 202/206 init (98%), soft delay 0.53s (<35% rem or <80% init limit), resource TFS/Short]
```

Interpretation:
- Remaining ratio is low, so the importer slows before exhausting the bucket.
- Initial limit has not degraded much (98% of initial), so this is mostly ratio-driven pacing.

### Example C: Initial-limit degradation warning

```text
[rate-limit: remaining 41/152, limit 152/190 init (80%), soft delay ... (<35% rem or <80% init limit), resource TFS/Long]
```

Interpretation:
- Denominator fell from 190 to 152 in the same resource bucket.
- This is a pre-stall warning pattern seen in practice.
- The initial-limit trigger is designed to start slowing earlier in this scenario.

## 4) Observed Behavior and Tuning Hints

Observed in this project:
- TFS/Short and TFS/Long buckets are both used.
- Stalls were caused by sudden drops to remaining 0 with large next request in values.
- One notable pattern before a stall: remaining looked safe in absolute terms, but the denominator dropped quickly and then hit zero in the next minute.

Recommended baseline (current):
- BPC_ADO_IMPORT_PARALLEL_WORKERS=1
- BPC_ADO_IMPORT_SOFT_THROTTLE_THRESHOLD=0.35
- BPC_ADO_IMPORT_SOFT_THROTTLE_MAX_DELAY_SECONDS=3.0
- BPC_ADO_IMPORT_SOFT_THROTTLE_INITIAL_LIMIT_PERCENT=0.80

If hard stalls still appear:
1. Raise initial limit trigger first:
- 0.80 -> 0.85
2. Then raise max delay if needed:
- 3.0 -> 4.0
3. Keep workers at 1 while stabilizing.

If no stalls for a long run and throughput is too low:
1. Lower max delay first:
- 3.0 -> 2.5
2. Then consider lowering threshold slightly:
- 0.35 -> 0.30
3. Only then consider raising workers.

## 5) Practical Tuning Workflow

1. Run for 15-30 minutes.
2. Count hard-throttle lines (containing next request in).
3. Check whether most soft-delay lines are from:
- low remaining ratio, or
- initial limit degradation.
4. Adjust one setting at a time.
5. Re-run and compare:
- hard-stall count
- average created/min
- time spent at 0.0/min

## 6) Where to Change Values

In Run.ps1, update these lines:

```powershell
Set-EnvValue -Name "BPC_ADO_IMPORT_PARALLEL_WORKERS" -Value "1"
Set-EnvValue -Name "BPC_ADO_IMPORT_RETRY_DELAY_SECONDS" -Value "120"
Set-EnvValue -Name "BPC_ADO_IMPORT_SOFT_THROTTLE_THRESHOLD" -Value "0.35"
Set-EnvValue -Name "BPC_ADO_IMPORT_SOFT_THROTTLE_MAX_DELAY_SECONDS" -Value "3.0"
Set-EnvValue -Name "BPC_ADO_IMPORT_SOFT_THROTTLE_INITIAL_LIMIT_PERCENT" -Value "0.80"
```

Then run setup_wizard.py phase 5 again in the same PowerShell session.
