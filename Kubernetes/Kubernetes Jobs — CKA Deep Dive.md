---
created: 2026-06-21T16:07:35+02:00
modified: 2026-06-21T16:08:22+02:00
---

# Kubernetes Jobs — CKA Deep Dive

---

## What a Job Is (and Isn't)

A **Job** is a controller that runs Pods to **completion**, unlike a Deployment which keeps Pods running indefinitely. The Job's contract with the cluster is: *"run this work until it succeeds N times, then stop."*

The key distinction: a Pod managed by a Job that exits with code `0` is considered **successful and done**. A Pod managed by a Deployment that exits with code `0` is **restarted**, because it shouldn't have stopped.

---

## Core Spec Fields — The Ones That Actually Matter

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processor
spec:
  completions: 5          # Total successful completions required
  parallelism: 2          # Pods running concurrently at any time
  backoffLimit: 4         # Max retries before Job is marked Failed
  activeDeadlineSeconds: 300  # Hard wall-clock timeout for the entire Job
  ttlSecondsAfterFinished: 120 # Auto-delete Job (and its Pods) after N seconds
  completionMode: Indexed      # "NonIndexed" (default) or "Indexed"
  template:
    spec:
      restartPolicy: Never  # Only "Never" or "OnFailure" are valid
      containers:
      - name: worker
        image: busybox
        command: ["sh", "-c", "echo done"]
```

### The `restartPolicy` Rule — A Common Trap

`Always` is **forbidden** in a Job Pod template. The exam will test this. The choice between `Never` and `OnFailure` has significant behavioral consequences:

| `restartPolicy` | On container failure | Effect |
|---|---|---|
| `Never` | Kubelet creates a **new Pod** | Old failed Pods accumulate; each counts toward `backoffLimit` |
| `OnFailure` | Kubelet **restarts the container in-place** | Same Pod, new container; Pod count stays low |

`Never` is safer for debugging (failed Pods remain for `kubectl logs`). `OnFailure` is more resource-efficient but you lose failure artifacts.

---

## Completion and Parallelism — The Mental Model

Think of it as a token bucket: the Job needs `completions` tokens. Pods that exit `0` deposit a token. Pods that fail do not. `parallelism` caps how many Pods can be pursuing tokens simultaneously.

**Example:** `completions: 6, parallelism: 3`
- At any time, up to 3 Pods run.
- As each Pod succeeds, a new one starts (up to the parallelism cap).
- When 6 successful exits are recorded, the Job is complete.

**Special cases:**
- `completions` unset + `parallelism: 1` → run until **one** success (the default single-run Job).
- `completions` unset + `parallelism: N` → **work queue** mode: Job completes when *any* Pod exits `0`. All other Pods are then terminated. This is the pattern for workers pulling from a queue where any worker finishing signals "the queue is empty."

---

## Indexed Jobs (`completionMode: Indexed`)

Each Pod gets a unique index `0` to `completions-1`, injected as:
- **Env var:** `JOB_COMPLETION_INDEX`
- **Annotation:** `batch.kubernetes.io/job-completion-index`

This is critical for **statically partitioned work** — sharding a dataset, processing specific files, etc. Each Pod knows its identity and processes its own slice. Unlike work-queue mode, you don't need an external queue system.

```bash
# Pod name pattern for Indexed Jobs:
<job-name>-<index>-<random-suffix>
```

---

## `backoffLimit` — The Retry Budget

`backoffLimit` (default: `6`) is the number of **Pod failures** the Job tolerates before giving up. Once exhausted, the Job phase transitions to `Failed` and no new Pods are created.

**Critical subtlety:** The backoff delay between retries is **exponential** — 10s, 20s, 40s… capped at 6 minutes. This matters for `activeDeadlineSeconds` interactions: a Job can time out while waiting for a backoff interval, even though it hasn't consumed all retries.

**Failure counting edge case:** With `restartPolicy: Never`, every failed Pod increments the counter. With `restartPolicy: OnFailure`, container restarts within a Pod do **not** individually increment `backoffLimit` — only Pod-level failures do.

---

## `activeDeadlineSeconds` vs. `backoffLimit` — The Interaction

These are **two independent termination conditions**. Whichever fires first wins.

- `backoffLimit` counts **failure events**.
- `activeDeadlineSeconds` counts **wall-clock seconds** from the Job's start time.

If `activeDeadlineSeconds` fires, the Job is marked `Failed` with reason `DeadlineExceeded` *regardless* of remaining `backoffLimit`. This is a frequent source of confusion on the exam: a Job that "should have had retries left" but was killed by the deadline.

---

## `ttlSecondsAfterFinished` — Automatic Cleanup

Without this, finished Jobs (and their Pods) accumulate forever. Setting this field enables the **TTL controller** to garbage-collect the Job N seconds after it reaches `Complete` or `Failed`.

**Exam note:** This is a real operational concern. Cluster-wide, unmanaged Job accumulation causes etcd bloat and slows `kubectl get pods` across namespaces. In production, this or a CronJob-level `successfulJobsHistoryLimit`/`failedJobsHistoryLimit` is essential.

---

## CronJob → Job Relationship

A **CronJob** is a Job factory. Every time the schedule fires, it creates a new Job object, which then creates Pods. Key fields:

```yaml
spec:
  schedule: "*/5 * * * *"
  concurrencyPolicy: Forbid     # Allow | Forbid | Replace
  startingDeadlineSeconds: 60   # If missed by this many seconds, skip
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:  # <-- This is a full Job spec
      ...
```

### `concurrencyPolicy` Behaviors

| Value | Behavior |
|---|---|
| `Allow` (default) | Multiple Jobs can run concurrently |
| `Forbid` | If the previous Job is still running, skip this trigger |
| `Replace` | Kill the running Job and start a new one |

### The Missed Schedule Problem

If the CronJob controller was down (e.g., control plane restart) and more than **100 schedules** were missed, Kubernetes refuses to backfill and logs `Cannot determine if job needs to be started`. This is a hard limit, not configurable. `startingDeadlineSeconds` helps by narrowing the window of "misses" the controller considers.

---

## Troubleshooting Cases

### Case 1: Job Never Completes, Pods Keep Restarting

**Symptoms:** `kubectl get pods` shows many `Error` or `OOMKilled` Pods; Job has not reached `Complete`.

**Diagnosis:**
```bash
kubectl describe job <name>        # Check Events, see failure reasons
kubectl get pods -l job-name=<name> --show-labels
kubectl logs <failed-pod-name>     # Only works if restartPolicy: Never
kubectl describe pod <failed-pod>  # Exit code, OOMKilled, etc.
```

**Likely causes:**
- Application bug → fix the container.
- OOMKilled → increase memory limits.
- Image pull error → `imagePullBackOff`; check image name and registry credentials.

---

### Case 2: Job Marked `Failed` with `BackoffLimitExceeded`

```bash
kubectl describe job <name>
# Conditions:
#   Type: Failed
#   Reason: BackoffLimitExceeded
```

**Check:** How many Pods were created, and why did they fail? Examine pod exit codes. Consider whether `backoffLimit` was set too low for a legitimately flaky workload, or whether the application truly has a bug.

**Fix path:** Delete the Job, fix the root cause, re-create with higher `backoffLimit` if appropriate:
```bash
kubectl delete job <name>
# edit the manifest
kubectl apply -f job.yaml
```

---

### Case 3: Job Marked `Failed` with `DeadlineExceeded`

```bash
kubectl describe job <name>
# Conditions:
#   Reason: DeadlineExceeded
```

The job ran longer than `activeDeadlineSeconds`. The work may be legitimate but slow.

**Fix:** Increase `activeDeadlineSeconds`, or investigate why the workload is slower than expected (resource contention, node issues, etc.).

---

### Case 4: Pods Are Stuck in `Pending`

```bash
kubectl describe pod <job-pod>
# Events: 0/3 nodes are available: 3 Insufficient memory.
```

**Diagnosis:** Resource requests exceed available cluster capacity, or node selectors/tolerations are too restrictive.

**Fix:** Lower resource requests, add nodes, or fix scheduling constraints.

---

### Case 5: Job Completes But Pods Are Gone and You Need Logs

If `ttlSecondsAfterFinished` was set aggressively short, Pods are deleted before you can inspect them.

**Prevention:** Don't set `ttlSecondsAfterFinished` too low in development. Or use a logging aggregator (Loki, Elasticsearch) so logs outlive Pods.

**Recovery (if Pods still exist):**
```bash
kubectl get pods -l job-name=<name> --field-selector=status.phase=Failed
kubectl logs <pod-name> --previous   # If restartPolicy: OnFailure, get previous container logs
```

---

### Case 6: CronJob Not Firing

**Checklist:**
1. Is the schedule valid? Use [crontab.guru](https://crontab.guru) to verify.
2. Check CronJob status: `kubectl describe cronjob <name>` — look for `Last Schedule` and Events.
3. Was `startingDeadlineSeconds` missed? Gaps in the controller (e.g., during upgrades) can cause skips.
4. Is `concurrencyPolicy: Forbid` blocking new Jobs because the previous one is still running?
5. Check the controller-manager logs: `kubectl logs -n kube-system -l component=kube-controller-manager`

---

## Key `kubectl` Commands for the Exam

```bash
# Create a Job imperatively (fastest in exam conditions)
kubectl create job test --image=busybox -- sh -c "echo hello"

# Create a CronJob imperatively
kubectl create cronjob cleanup --image=busybox --schedule="0 3 * * *" -- sh -c "rm -rf /tmp/*"

# Watch Job progress
kubectl get jobs -w
kubectl get pods -l job-name=<name> -w

# Describe for events and status
kubectl describe job <name>

# Manually trigger a CronJob immediately (create a Job from its template)
kubectl create job --from=cronjob/<name> manual-run-01

# Delete a Job and its Pods
kubectl delete job <name>             # Pods deleted via owner reference cascade
kubectl delete job <name> --cascade=orphan  # Leave Pods running

# Suspend a Job (stops creating new Pods; existing ones keep running)
kubectl patch job <name> -p '{"spec":{"suspend":true}}'
```

---

## CKA Exam Focus Areas — Summary Checklist

| Topic | What to Know |
|---|---|
| `restartPolicy` | Only `Never` or `OnFailure`; `Always` is invalid |
| `backoffLimit` | Default is 6; each failed Pod (with `Never`) counts |
| `activeDeadlineSeconds` | Wall-clock deadline; overrides `backoffLimit` |
| `completions` + `parallelism` | Token-bucket mental model; work-queue mode when `completions` unset |
| `completionMode: Indexed` | `JOB_COMPLETION_INDEX` env var; statically partitioned workloads |
| `ttlSecondsAfterFinished` | Garbage collection; not set by default |
| CronJob history limits | `successfulJobsHistoryLimit`, `failedJobsHistoryLimit` |
| CronJob `concurrencyPolicy` | `Allow` / `Forbid` / `Replace` semantics |
| Imperative commands | `kubectl create job` and `kubectl create job --from=cronjob/...` |
| Troubleshooting | Events on Job + Pod; exit codes; `backoffLimit` vs `DeadlineExceeded` |

The highest-leverage exam skill for Jobs is being fast with the imperative `kubectl create job` command and immediately reading `kubectl describe job` output fluently — most failures announce themselves clearly in the Events section.
