# Registry Update Coalescing — Planning Doc

> **Status:** Planning  
> **Version:** 0.1  
> **Date:** 2026-07-07  
> **Authors:** Satvik Shetty

---

## Why this doc exists

`registry-first-promotion.md` notes a future concern under "Rapid Lifecycle Edits": rapid successive `UpdateModelPackage` calls can produce multiple EventBridge events and fan out into multiple concurrent promotion runs. This document works through every realistic scenario in detail and proposes an architecture that coalesces related changes into as few promotion triggers as possible — without losing any governance event.

Nothing in this doc changes the promotion contract (`ModelApprovalStatus` + `ModelLifeCycle.Stage/StageStatus` as the source of truth). It changes only how the *orchestration layer* reacts to clusters of related events.

---

## The root problem in one sentence

SageMaker emits **one EventBridge event per `UpdateModelPackage` API call**. There is no "batch update" API. If five model versions in the same family are all promoted to staging in a single operation, five events fire and the current (per-model) design would trigger five separate promotion pipeline runs — most of which overwrite each other's manifest or race on the same RegisterModel step.

---

## Scenario taxonomy

### Dimension 1 — How many model packages are touched?

| Cardinality | Example | Desired pipeline runs |
|---|---|---|
| **1 model, 1 change** | Promote release model to staging | 1 |
| **N models, same family, same target** | Promote release + 2 shadows for family `fraud` to staging | 1 |
| **N models, different families, same target** | ML Lead promotes `fraud` and `risk` families to prod in the same session | 1 per family (2 total, independent) |
| **N models, same family, different targets** | Shadow promoted to staging; release promoted to prod simultaneously (unusual but possible) | 1 per target (2 total, independent) |

### Dimension 2 — What is the intent?

| Intent | How to detect | Should trigger promotion? |
|---|---|---|
| **Intentional promotion** | `Stage ∈ {Staging, Production}` + `Approved` + `StageStatus ∈ {Release, Shadow}` | Yes |
| **Rapid correction** | Same model ARN: Stage=Staging (t=0) → Stage=Development (t=5s) | Latest event wins; no promotion |
| **Metadata-only edit** | Only `StageDescription` changed (no Stage/StageStatus change) | No |
| **Administrative rejection** | `ModelApprovalStatus=Rejected` or `StageStatus=Rejected` | Yes, triggers de-promotion |
| **Bulk supersession** | Multiple models set to `StageStatus=Superseded` | No — these are writes by RegisterModel only; do not re-trigger promotion |

### Dimension 3 — Timing patterns

```
Pattern A — Atomic (single call)
  T+0:   1 UpdateModelPackage (Stage + StageStatus in one call)
  → 1 event                                        ← trivially handled today

Pattern B — Near-simultaneous burst (same operator, same session)
  T+0:   UpdateModelPackage(release model)
  T+2s:  UpdateModelPackage(shadow_1)
  T+4s:  UpdateModelPackage(shadow_2)
  → 3 events within ~5 seconds, all intent=promote, family=fraud, target=Staging

Pattern C — Correction
  T+0:   UpdateModelPackage(model_A, Stage=Staging)   ← mistake
  T+3s:  UpdateModelPackage(model_A, Stage=Development) ← correction
  → 2 events, same model ARN, last event is the true intent

Pattern D — Slow wave (different operators / UI sessions, same family)
  T+0:    MLE promotes release model
  T+3min: ML Lead adds a shadow
  → 2 events 3 minutes apart — may or may not want to coalesce

Pattern E — Cross-family concurrent promotion
  T+0:  UpdateModelPackage(fraud release → Staging)
  T+1s: UpdateModelPackage(risk release → Staging)
  → 2 events, different families — these are independent, parallelism is fine

Pattern F — In-flight conflict
  CopyModels is running for family=fraud
  T+0:  New shadow approved for family=fraud → Staging
  → Should queue, not collide
```

---

## The fundamental design shift: per-model → per-family manifest

The current design writes one manifest per model-package version (one file per promotion event). The target design writes **one manifest per `{family}:{target_env}` pair** representing the **desired state** of the entire family in that environment — not a delta.

### Why desired-state matters

If you promote a release + 2 shadows as three separate events, and the orchestrator writes a per-model manifest:

```
Event 1 → writes fraud_staging.json = { "model": "txn_scorer", "stage_status": "Release" }
          → CodePipeline run A starts
Event 2 → writes fraud_staging.json = { "model": "txn_scorer_v2", "stage_status": "Shadow" }
          → CodePipeline run B starts (overwriting run A's source or racing on RegisterModel)
Event 3 → writes fraud_staging.json = { "model": "txn_scorer_v3", "stage_status": "Shadow" }
          → CodePipeline run C starts
```

Three runs, each seeing only one model, and they race on the RegisterModel step. Even with `executionMode=QUEUED`, runs B and C will re-run CopyModels for the entire family when only the manifest changed — wasteful and fragile.

With a **desired-state family manifest**:

```
Event 1 → accumulate: fraud/Staging → {release: txn_scorer}
Event 2 → accumulate: fraud/Staging → {release: txn_scorer, shadow[1]: txn_scorer_v2}
Event 3 → accumulate: fraud/Staging → {release: txn_scorer, shadow[1]: txn_scorer_v2, shadow[2]: txn_scorer_v3}

<after coalescing window closes>
→ writes ONE manifest with all three models
→ ONE CodePipeline run: CopyModels (all 3 artifacts) → Approval gate → RegisterModel (all 3)
```

### Family manifest shape

```json
{
  "schema_version": "2",
  "family": "fraud",
  "target_env": "stage",
  "coalesced_at": "2026-07-07T14:32:10Z",
  "release": {
    "model": "txn_scorer",
    "version": "20260416_2253",
    "model_package_arn": "arn:aws:sagemaker:us-east-1:111122223333:model-package/fraud-txn-scorer/42",
    "source_gold": "s3://bcnc-gold-mlops-hub-dev-us-east-1/fraud/txn_scorer/versions/20260416_2253/model.tar.gz"
  },
  "shadows": [
    {
      "slot": 1,
      "model": "txn_scorer_v2",
      "version": "20260501_1000",
      "model_package_arn": "arn:aws:...",
      "source_gold": "s3://..."
    }
  ]
}
```

A single CodePipeline run reads this manifest and:
1. `CopyModels` — copies every artifact listed (both release and shadows)
2. `ManualApproval` (prod only)
3. `RegisterModel` — registers all models, marks prior versions `Superseded`

The S3 manifest key is `{env}/manifest/{family}.json` (one object per family per env). Because the key is fixed per family, CodePipeline re-triggers whenever the object content changes — which is exactly right.

---

## Coalescing architecture

The goal is a **quiet-period debounce**: collect all events for a given `{family}:{target_env}` within a window, then emit exactly one manifest write after the window closes.

### Option A — SQS standard queue + Lambda batching window (simplest)

```
SageMaker Registry Change
        ↓
EventBridge Rule (ModelLifeCycle or ModelApprovalStatus changed)
        ↓
SQS Standard Queue
  - visibility timeout: 90s
  - Lambda trigger: batchSize=100, MaximumBatchingWindowInSeconds=30–60
        ↓
Coalescing Lambda
  1. Deduplicate: for each model_package_arn, keep only the latest event
  2. Filter: discard metadata-only events (no Stage/StageStatus/ApprovalStatus change)
  3. Group: by {family}:{target_env}
  4. For each group:
       a. Query dev registry for all currently Approved + eligible models in the family
          (do NOT rely solely on the event batch — the registry is the ground truth)
       b. Build family manifest from registry state
       c. Conditionally write manifest to S3 (skip if content unchanged)
  5. De-promotion path: handle Rejected events separately (immediate, no batching needed)
```

| | |
|--|--|
| ✅ | Native SQS feature, no extra services |
| ✅ | 30–60s window coalesces Pattern B (same-session burst) |
| ✅ | "Latest event wins" per ARN handles Pattern C (corrections) |
| ✅ | Query-from-registry step ensures manifest is always complete even if events were missed |
| ⚠️ | 30–60s minimum latency added to every promotion |
| ⚠️ | Pattern D (slow wave, 3+ min apart) gets two separate pipeline runs — likely acceptable |
| ⚠️ | SQS standard doesn't guarantee ordering within the window; Lambda dedup by ARN handles this |

**Best for:** typical promotion workflows where ≤ 60s latency is fine and promotions within a family happen within a short UI session.

---

### Option B — DynamoDB desired-state accumulator + EventBridge Scheduler debounce (precise)

```
EventBridge Rule
        ↓
Accumulator Lambda (fast, ~50ms)
  1. Writes/updates a record to DynamoDB:
       PK:  {family}#{target_env}
       SK:  {model_package_arn}
       attrs: {stage_status, version, source_gold, updated_at, ttl: now+10min}
  2. Upserts a one-shot schedule in EventBridge Scheduler:
       name: flush-{family}-{target_env}
       schedule: rate(1 minute)  [or at(now+60s) if Scheduler supports sub-minute]
       target: FlushLambda
       (if schedule already exists: delete + recreate to reset the timer)
        ↓  (60 seconds after last update to this family+env)
FlushLambda
  1. Reads all pending records for {family}#{target_env} from DynamoDB
  2. Cross-references with live registry (source of truth)
  3. Builds family manifest
  4. Writes manifest to S3 using conditional PutObject (ETag check to prevent double-write)
  5. Deletes the DynamoDB records and the schedule
```

| | |
|--|--|
| ✅ | Precise debounce — timer resets on every new event for the same family+env |
| ✅ | Pattern D (slow wave) handled: timer resets as long as activity continues |
| ✅ | DynamoDB gives a persistent "in-flight" view; recoverable if Lambda crashes mid-flight |
| ✅ | No minimum latency for promotion — flush fires 60s after the *last* event, not the *first* |
| ⚠️ | EventBridge Scheduler minimum rate: 1 minute (cannot set 30s) |
| ⚠️ | Delete + recreate of Scheduler rule to reset debounce timer adds latency and complexity |
| ⚠️ | More moving parts: DynamoDB table + Scheduler + two Lambdas |

**Best for:** scenarios where the promotion window may span minutes (e.g., ML Lead reviewing and approving models one-by-one across a 5-minute session) and you want all of them to coalesce into one pipeline run.

---

### Option C — EventBridge Pipes + SQS FIFO + Lambda batch (structured ordering)

```
EventBridge Rule
        ↓
EventBridge Pipe
  - enrichment step: Lambda adds {family, target_env} to the event
  - filter: drop metadata-only events early
        ↓
SQS FIFO Queue
  - MessageGroupId: {family}#{target_env}     ← one ordered lane per family×env
  - MessageDeduplicationId: {model_arn}#{stage}#{stage_status}#{epoch_5min}
    (5-minute SQS FIFO deduplication window — identical re-triggers are no-ops)
        ↓
Lambda (SQS trigger, batchSize=10, MaximumBatchingWindowInSeconds=30)
  - Same coalescing logic as Option A
  - BUT: FIFO ensures only one concurrent Lambda invocation per MessageGroupId
    → no concurrent processing of the same family, eliminating RegisterModel races
```

| | |
|--|--|
| ✅ | SQS FIFO MessageGroupId guarantees serial processing per family×env |
| ✅ | 5-minute FIFO dedup window handles rapid re-fires of the exact same event |
| ✅ | EventBridge Pipes handles enrichment and early filtering cleanly |
| ✅ | Pattern F (in-flight conflict): FIFO queues the new event until the current batch is processed |
| ⚠️ | SQS FIFO throughput: 300 msg/s (or 3000 with high-throughput mode) — fine for registry updates |
| ⚠️ | EventBridge Pipes adds a small cost per event |
| ⚠️ | FIFO MessageGroupId means all events for one family×env are serialized — Lambda cannot parallelize within a family |

**Best for:** teams that want the strongest correctness guarantees and clear per-family ordering, and are comfortable with slightly more infrastructure.

---

### Option D — Step Functions Standard Workflow (full orchestration)

```
EventBridge Rule → Step Functions (StartExecution)
  executionName: {family}-{target_env}-{YYYYMMDD_HH}   ← hour-level dedup

If an execution is already running for this name:
  → Step Functions returns ExecutionAlreadyExists → ignored (idempotent)

Step Functions workflow:
  State 1: Wait (X seconds — configurable debounce)
  State 2: QueryRegistry (Lambda — get current family state)
  State 3: BuildManifest (Lambda — build family manifest)
  State 4: WriteManifest + TriggerPipeline
  State 5: WaitForPipeline (optional — poll CodePipeline status)
```

| | |
|--|--|
| ✅ | `executionName` deduplication is first-class — native no-op on re-fire within the same hour |
| ✅ | Visual execution graph in the console aids debugging |
| ✅ | Native retry/error handling per state |
| ⚠️ | Hour-level dedup by execution name is coarse — promoting the same family twice in one hour requires a different name strategy |
| ⚠️ | Step Functions Standard Workflow: $0.025 per 1,000 state transitions (not free) |
| ⚠️ | "Wait" state in Step Functions burns execution time against the 1-year max |

**Best for:** teams that want a visual orchestration console and can tolerate the naming complexity.

---

## Recommendation

**Start with Option A** (SQS standard + Lambda batching window) with the three enhancements below. Upgrade to **Option B or C** if the slow-wave pattern (multi-minute family promotion sessions) becomes common.

The most important change — regardless of debounce mechanism — is:

> **Switch from per-model manifests to per-family desired-state manifests and make the Coalescing Lambda query the registry (not just the event) to build them.**

This alone fixes the main race condition. The SQS batch window is the minimum viable coalescing layer on top.

### Three enhancements to add regardless of option chosen

**1. Query-from-registry, not query-from-event**

The Coalescing Lambda should not trust that "the batch I received is the complete picture." It should:
1. Extract the `{family, target_env}` from each event
2. Call `ListModelPackages` for the family and filter to `Approved + Stage ∈ {Staging, Production} + StageStatus ∈ {Release, Shadow}` — i.e., re-derive the desired state from the registry itself
3. Build the manifest from the registry query, not the event payload

This means: if events were missed (Lambda error, SQS hiccup), the next triggered run still produces a correct manifest. It also means a correction (Pattern C) that happens *after* the batch window opens will always produce the correct manifest, because the registry state will already reflect the correction.

**2. Content-addressed manifest write**

Before writing the manifest to S3, hash its content. If the hash matches the current S3 object's ETag, skip the write (no-op). This prevents a CodePipeline re-trigger when events fire but the desired family state hasn't actually changed (e.g., someone adds a `StageDescription` note while a promotion is already in flight).

```python
new_manifest_json = json.dumps(manifest, sort_keys=True)
current_etag = s3.head_object(Bucket=..., Key=...).get("ETag", "")
new_etag = f'"{hashlib.md5(new_manifest_json.encode()).hexdigest()}"'
if current_etag == new_etag:
    logger.info("Manifest unchanged, skipping write")
    return
s3.put_object(Bucket=..., Key=..., Body=new_manifest_json, ContentType="application/json")
```

**3. De-promotion handling — micro-debounce, not zero-debounce**

Rejection events should be processed faster than a normal promotion (no waiting 30–60s) but **not instantaneously**. "Immediate" is too aggressive — a user who accidentally sets `ModelApprovalStatus=Rejected` and corrects it 3 seconds later would still trigger a full de-promotion run if the event reaches the orchestrator before the correction. A **5–10 second micro-debounce** is the right default: fast enough that genuine rejections feel responsive, long enough to absorb obvious accidents.

Route rejection events to a **separate SQS queue** from promotion events with `MaximumBatchingWindowInSeconds=5` (or 10). This queue does not need the 30–60s promotion window — it just needs enough headroom to absorb a rapid revert. See "De-promotion and the Accidental Rejection" below for full analysis.

---

## Scenario walkthrough with the recommended architecture (Option A + enhancements)

### Pattern B — Release + 2 shadows, same family, 5-second burst

```
T+0:  Event: txn_scorer (Release → Staging)     → SQS
T+2s: Event: txn_scorer_v2 (Shadow → Staging)   → SQS
T+4s: Event: txn_scorer_v3 (Shadow → Staging)   → SQS
T+34s: Batch window closes, Lambda receives all 3

Lambda:
  dedup by ARN → 3 unique models, all latest event = promote
  group by family: fraud × Staging

  QueryRegistry(fraud, Staging):
    → txn_scorer: Approved, Stage=Staging, StageStatus=Release   ✅
    → txn_scorer_v2: Approved, Stage=Staging, StageStatus=Shadow  ✅
    → txn_scorer_v3: Approved, Stage=Staging, StageStatus=Shadow  ✅

  BuildManifest → fraud_staging.json (all 3 models)
  ContentHash → different from current → Write
  CodePipeline triggers ONCE
  CopyModels copies 3 artifacts, RegisterModel registers all 3
```

**Result: 1 pipeline run, all 3 models promoted atomically.**

---

### Pattern C — Correction (wrong model promoted, immediately reversed)

```
T+0:  Event: model_X (Stage=Staging)   → SQS  ← mistake
T+3s: Event: model_X (Stage=Development) → SQS  ← correction
T+30s: Batch window closes, Lambda receives both

Lambda:
  dedup by ARN: model_X has 2 events → keep latest → Stage=Development
  group by family: fraud × Staging
  model_X has Stage=Development → excluded from promotion query

  QueryRegistry(fraud, Staging):
    → model_X: Stage=Development → excluded
    → (only models that are currently Stage=Staging + Approved)

  If result is empty or unchanged from current manifest → ContentHash match → no write
  CodePipeline does NOT trigger
```

**Result: 0 pipeline runs, correction is absorbed.**

---

### Pattern D — Slow wave (MLE promotes release at T=0, then adds a shadow 3 minutes later)

With **Option A (SQS 30s window)**:
```
T+0:   txn_scorer (Release → Staging) → batch closes at T+30s → 1 pipeline run (release only)
T+3min: txn_scorer_v2 (Shadow → Staging) → batch closes at T+3m30s → 1 pipeline run (release + shadow)
```
Two runs. Run 2 is idempotent for the release (RegisterModel is a no-op for already-registered versions).

With **Option B (DynamoDB + 60s debounce reset)**:
```
T+0:    Accumulate: txn_scorer → timer set for T+60s
T+3min: Accumulate: txn_scorer_v2 → timer reset for T+3m60s
T+4min: FlushLambda fires → manifest with both models → 1 pipeline run
```
One run. But 4 minutes of total delay from first event.

**Trade-off:** Option B coalesces perfectly but adds latency. For most teams, two idempotent pipeline runs is acceptable; the 4-minute delay is not. Option A is the better default here.

---

### Pattern E — Two families, concurrent (correct behaviour: parallel, independent)

```
T+0:  Event: fraud/txn_scorer (→ Staging)
T+1s: Event: risk/risk_scorer (→ Staging)

Lambda batch (T+30s):
  group 1: fraud × Staging → QueryRegistry(fraud) → write fraud_staging.json
  group 2: risk × Staging  → QueryRegistry(risk)  → write risk_staging.json

Both manifest writes are independent keys → two CodePipeline runs fire concurrently
```

**Result: 2 independent pipeline runs. No interference.**

---

### Pattern F — New shadow arrives while CopyModels is in flight

```
CopyModels running for fraud/Staging (txn_scorer Release)
T+0: Event: txn_scorer_v2 (Shadow → Staging) → SQS

After batch window:
  Lambda writes fraud_staging.json = {release: txn_scorer, shadow[1]: txn_scorer_v2}
  S3 key changes → CodePipeline triggers a NEW run

CodePipeline executionMode=QUEUED:
  New run queues behind the running one
  Running run (release only) completes → RegisterModel for release
  Queued run starts → CopyModels for both (idempotent for release), RegisterModel for shadow
```

**Result: 2 runs, correctly ordered, idempotent. Shadow is registered after release.**

The important configuration: **CodePipeline `executionMode=QUEUED`** (not `PARALLEL` or `SUPERSEDED`) so in-flight runs complete before the next one starts.

---

## De-promotion and the accidental rejection

### The problem

The natural instinct is to make rejection as fast as possible — it's a safety gate. But "immediate" creates a trap: every accidental click on the wrong menu item in Studio can trigger a cascade that, in the prod case, requires a new manual approval gate pass to recover. The accidental rejection is much more likely than a genuine malicious or erroneous promotion that needs to be stopped in under a second.

### The two types of rejection are not symmetric

| Rejection type | Scope | Typical motivation | Reversibility concern |
|---|---|---|---|
| `ModelApprovalStatus=Rejected` | **Global** — propagates to all envs | Compliance issue, fundamental model failure | A revert re-promotes to every env that had it; prod requires a new approval gate pass |
| `StageStatus=Rejected` (on a specific `Stage` value) | **Per-env** — removes from one env only | Operational rollback, performance regression in one env | A revert only affects one env; still requires a new approval gate for prod |

Both types need to be fast, but neither needs to be instantaneous. Inference pipelines run on a schedule — the next scheduled run is unlikely to start in the 5–10 seconds between an accidental rejection and its correction.

### The quick-revert race: what actually happens

With zero debounce (current assumption):

```
T+0s:   Operator sets ModelApprovalStatus=Rejected (accident)
          → event arrives at orchestrator immediately
          → orchestrator writes de-promotion manifest to S3
          → CodePipeline Source stage detects change, execution starts
T+3s:   Operator catches mistake, sets ModelApprovalStatus=Approved + Stage=Production
          → two new events fire (approval change + lifecycle change)
          → these go through the 30s promotion batch window
T+33s:  Coalescing Lambda processes the correction events
          → re-writes promotion manifest
          → new CodePipeline run queued (behind the de-promotion run still running)
T+??:   De-promotion run completes → RegisterModel sets prod version to Rejected
T+??:   Promotion run starts → CopyModels (no-op, artifact still there) → ManualApproval gate
          → ML Lead must re-approve before RegisterModel runs again
```

**The production outcome:** a 3-second accident results in a new approval gate that may take hours to resolve, during which the previously live prod model has `ModelApprovalStatus=Rejected` in the prod registry — inference finds no eligible model.

### Staged approach by rejection type

#### `StageStatus=Rejected` (per-env, less severe)

A 5–10s micro-debounce SQS window absorbs most accidents. The correction fires within seconds; both events land in the same batch; "latest event wins" — if the corrected state is not `Rejected`, no de-promotion runs.

```
T+0s:  StageStatus=Rejected (accident)   → micro-debounce SQS (5s window)
T+3s:  StageStatus=Release (correction)  → same SQS batch
T+5s:  Batch window closes
         Lambda: dedup by ARN, latest event = StageStatus=Release (not Rejected)
         → no de-promotion manifest written
         → CodePipeline does NOT trigger
```

**Result: correction absorbed, zero pipeline runs.**

If the rejection was genuine (no correction follows within 5s):

```
T+0s:  StageStatus=Rejected  → batch window
T+5s:  Window closes, no correction received
         Lambda: latest event = Rejected
         → directly invoke stage RegisterModel Lambda (cross-account assume-role)
           sets Rejected in stage registry
           ✗ no manifest written, ✗ no CodePipeline triggered
```

5 seconds is not a meaningful delay for a genuine operational rollback.

#### `ModelApprovalStatus=Rejected` (global, most severe)

The same 5–10s micro-debounce applies, but the correction path is more painful because reverting a global rejection requires setting `ModelApprovalStatus=Approved` AND re-advancing `Stage=Production` (two separate `UpdateModelPackage` calls, two events) plus a new prod approval gate.

**Recommendation for global rejection:** apply the micro-debounce, AND add a **confirmation step** for prod-affecting rejections — not a full approval gate, but a 60-second hold with a DynamoDB `pending_rejection` record and alerts via **SES email + Teams webhook**. This gives the operator a visible window to cancel before any de-promotion action is taken.

```
T+0s:   ModelApprovalStatus=Rejected (accident)
          → micro-debounce queue (5s)
T+5s:   No correction in window → write "pending_rejection" to DynamoDB (TTL=75s)
          → SES email: "⚠️ Global rejection pending for {family}/{model}/{version}
             — will propagate to prod in 60s. Set Approved to cancel."
          → Teams webhook: same message to #mlops-alerts channel
          → start 60s countdown (EventBridge Scheduler one-shot at T+65s)
T+8s:   Operator sees Teams alert, sets ModelApprovalStatus=Approved
          → correction event arrives → DynamoDB record deleted
          → Scheduler one-shot cancelled (delete + no-op if already fired)
          → no de-promotion manifest written, no CodePipeline run
```

If no correction within 60 seconds:

```
T+65s:  Scheduled Lambda fires
          → DynamoDB record still present → proceed
          → stage: directly invoke stage RegisterModel Lambda (cross-account)
                   sets Rejected in stage registry — no manifest, no pipeline
          → prod:  write de-promote manifest → De-promotion CodePipeline
                   Source → ManualApproval gate (ML Lead) → RegisterModel sets Rejected
          → SES + Teams: "🔴 De-promotion in progress. Stage: done. Prod: awaiting gate."
```

This is the **two-phase de-promotion** pattern. The 60-second window is long enough to catch accidents; the SES + Teams alerts make the window actionable in real time. Genuine rejections still propagate within ~65 seconds for stage and as soon as the ML Lead approves for prod.

#### What if the correction arrives AFTER de-promotion has already run?

Worst case: operator doesn't notice for 2+ minutes, the de-promotion pipeline has completed.

1. **Stage recovery** (automated, no gate): Set `ModelApprovalStatus=Approved` + re-advance `Stage=Staging` in dev → new promotion event → stage pipeline runs immediately → CopyModels (idempotent, artifact still in stage gold) → RegisterModel sets back to `Approved + Active`.  
   _Duration: ~2–5 minutes._

2. **Prod recovery** (gated): Set `ModelApprovalStatus=Approved` + re-advance `Stage=Production` in dev → new promotion event → prod CodePipeline starts → CopyModels → **ManualApproval gate** → ML Lead approves → RegisterModel sets back to `Approved + Active`.  
   _Duration: hours, depending on ML Lead availability._

   This is expected and consistent: **anything that updates prod is gated**, whether it's a promotion or a de-promotion reversal. The painful recovery time is the reason the 60s confirmation window exists — to prevent this scenario in the first place.

### Summary of recommendations

| Rejection type | Debounce | Gate? | Mechanism | Alert | Recovery if not caught |
|---|---|---|---|---|---|
| `StageStatus=Rejected` (stage only) | 5s micro-debounce | ❌ No gate | Direct Lambda call to stage RegisterModel (cross-account) — no manifest, no CodePipeline | None needed | Re-set StageStatus + re-advance `Stage=Staging` → ~5 min automated |
| `StageStatus=Rejected` (prod) | 5s micro-debounce | ✅ ManualApproval gate | De-promotion CodePipeline: Source → ManualApproval → RegisterModel (no CopyModels) | SES email + Teams alert when gate is waiting | Re-set + re-advance `Stage=Production` → new prod gate |
| `ModelApprovalStatus=Rejected` (global) | 5s micro-debounce | ✅ 60s window + prod ManualApproval gate | Stage: direct Lambda. Prod: De-promotion CodePipeline after 60s window expires | SES email + Teams alert during 60s window and when prod gate is waiting | Re-approve + re-advance all affected stages → new prod gate (hours) |

**Rule: anything that writes to prod — promotion or de-promotion — requires a `ManualApproval` gate. Stage de-promotion does not, and never touches CodePipeline.**

### EventBridge rule split for rejection routing

Two separate EventBridge rules feed two SQS queues with different batch windows. The De-promotion Lambda distinguishes stage from prod based on the `Stage` field in the event and executes differently for each — **de-promotion never copies artifacts**, so neither path needs `CopyModels`.

```
Rule 1 — Promotion events:
  source: aws.sagemaker
  detail-type: SageMaker Model Package State Change
  detail.ModelLifeCycle.Stage: [Staging, Production]
  detail.ModelApprovalStatus: Approved
  → SQS Promotion Queue (MaximumBatchingWindowInSeconds=30)
  → Coalescing Lambda → family manifest → Promotion CodePipeline
      stage: Source → CopyModels → RegisterModel            (no gate)
      prod:  Source → CopyModels → ManualApproval → RegisterModel

Rule 2 — Rejection / de-promotion events:
  source: aws.sagemaker
  detail-type: SageMaker Model Package State Change
  detail.ModelApprovalStatus: Rejected          (OR)
  detail.ModelLifeCycle.StageStatus: Rejected
  → SQS Rejection Queue (MaximumBatchingWindowInSeconds=5)
  → De-promotion Lambda:
      if target = Stage only:
        → directly invoke stage RegisterModel Lambda (cross-account assume-role)
          sets ModelApprovalStatus=Rejected in stage registry
          ✗ no manifest, ✗ no CodePipeline
      if target includes Prod (or global ModelApprovalStatus=Rejected):
        → write "pending_rejection" to DynamoDB
        → SES SendEmail + Teams webhook POST → #mlops-alerts
        → schedule one-shot flush at T+60s (EventBridge Scheduler)
        → on flush:
            stage → directly invoke stage RegisterModel Lambda (cross-account)  ✗ no pipeline
            prod  → write de-promote manifest → De-promotion CodePipeline
                      Source → ManualApproval → RegisterModel   ✗ no CopyModels
```

**Why two different CodePipelines for promotion vs de-promotion?**

The promotion pipeline (`CopyModels → [gate] → RegisterModel`) and the de-promotion pipeline (`[gate] → RegisterModel`) have different step shapes. Reusing the same pipeline and branching on `action=demote` is possible but creates a shared pipeline that handles conflicting concerns. A separate, simpler de-promotion pipeline (`ManualApproval → RegisterModel`) is cleaner and easier to reason about. Both pipelines share the same `RegisterModel` Lambda.

**Alert implementation:**
- **SES**: Lambda calls `ses.send_email()` with a template; sender address and recipient list stored in SSM Parameter Store (`/{project}/notifications/ses-sender`, `/{project}/notifications/mlops-recipients`).
- **Teams**: Lambda POSTs an Adaptive Card payload to an incoming webhook URL stored in SSM (`/{project}/notifications/teams-webhook-url`). No SNS involved.

---

## Edge cases and mitigations

| Edge case | Risk | Mitigation |
|---|---|---|
| Model registered in dev but `StageDescription` only changed | Spurious trigger (no governance change) | EventBridge rule filter: `detail.UpdatedModelPackageFields` must contain `ModelApprovalStatus` or `ModelLifeCycle` — exclude `StageDescription`-only changes |
| Two operators promoting different models in same family concurrently (Pattern B across different people) | Both events land in the same batch window → coalesced correctly | ✅ Coalescing Lambda handles this correctly; last registry query sees both |
| Stale event replayed (SQS retry after Lambda failure) | Possibly re-triggers a promotion that already completed | Content-hash check: manifest unchanged → no-op |
| Family has no Approved + eligible models when FlushLambda fires (all corrections cancelled the promotions) | Manifest written with empty `models` list | Guard: if manifest has no release model, do not write and do not trigger pipeline |
| **Accidental rejection, corrected within seconds** | De-promotion fires before correction is processed; prod gate-recovery can take hours | 5s micro-debounce on rejection queue; for prod-affecting rejections, a 60s confirmation window with SES email + Teams alert before the de-promote manifest is written; see "De-promotion and the accidental rejection" |
| De-promotion (Rejected) event arrives mid-batch alongside a promotion event for the same ARN | Promotion event wins if it's later; rejection wins if it's later | Rejection events route to a separate queue and are processed as soon as their 5s window closes; they do not compete with the 30s promotion batch |
| Large family (many models) — CopyModels step duration grows | In-flight duration increases, queue depth grows | CopyModels step is parallelisable (loop over models with concurrent CodeBuild invocations or multiple build phases); RegisterModel loop is sequential but fast |
| Cross-region manifests | Stage and prod may be multi-region | Manifest key should include region: `{env}/{region}/manifest/{family}.json`; orchestrator writes to each region's manifest bucket separately |

---

## Implementation steps (phased)

### Phase 0 — Prerequisite (no new infra)
- Change the orchestrator to **query the registry** (not just use the event payload) to determine which models to include in the manifest — even if still writing per-model manifests today.
- Add **content-hash guard** before every manifest write.
- Do **not** route rejection events to a zero-debounce immediate path — leave them going through the orchestrator for now (fix races in Phase 1).

> This alone prevents the "manifest overwrite race" and makes the system safe even before a proper coalescing queue is added.

### Phase 1 — SQS coalescing layer + rejection micro-debounce
- Add **SQS Promotion Queue** between EventBridge and the orchestrator Lambda (`MaximumBatchingWindowInSeconds=30`).
- Add **SQS Rejection Queue** with `MaximumBatchingWindowInSeconds=5`, feeding a separate De-promotion Lambda.
- Upgrade orchestrator to group events by `{family}:{target_env}` and write **family-level manifests**.
- Add `executionMode=QUEUED` to the **Promotion CodePipeline** (unchanged shape: Source → CopyModels → [ManualApproval prod only] → RegisterModel).
- Add a separate **De-promotion CodePipeline** for prod only (shape: Source → ManualApproval → RegisterModel — no CopyModels). Reuses the same `RegisterModel` Lambda.
- Stage de-promotion is a **direct cross-account Lambda invocation** from the De-promotion Lambda — no manifest, no pipeline.
- For prod-affecting global rejections: add DynamoDB `pending_rejection` table, EventBridge Scheduler one-shot, SES `send_email` call, and Teams webhook POST. Store sender/recipient/webhook config in SSM Parameter Store.
- Upgrade `RegisterModel` Lambda to accept and loop over an array of models (not a single model) and to handle both `action=promote` (sets Active) and `action=demote` (sets Rejected).
- Upgrade `CopyModels` CodeBuild spec to loop over the `models[]` array in the manifest.

### Phase 2 — (If needed) Longer debounce for slow-wave pattern
- If Pattern D (multi-minute promotion sessions) becomes common: add DynamoDB accumulator + EventBridge Scheduler debounce (Option B).
- Keep the SQS layer as a pre-filter; DynamoDB as the accumulation store.

### Phase 3 — (If needed) Per-family ordering
- If concurrent processing within a family causes issues: add SQS FIFO with `MessageGroupId={family}:{target_env}` (Option C).
- Alternatively, add DynamoDB conditional write (optimistic lock) per `{family}:{target_env}` in the Coalescing Lambda.

---

## Service comparison summary

| Concern | SQS + batch window (A) | DynamoDB + Scheduler (B) | SQS FIFO + Pipes (C) | Step Functions (D) |
|---|:---:|:---:|:---:|:---:|
| Coalesces same-session burst (< 60s) | ✅ | ✅ | ✅ | ✅ |
| Coalesces slow wave (3–10 min) | ❌ (two runs, idempotent) | ✅ | ❌ (two runs) | ✅ (if window > 3 min) |
| Handles rapid corrections | ✅ latest-event-wins | ✅ latest-event-wins | ✅ FIFO dedup | ✅ |
| Per-family serial ordering | ❌ (need DynamoDB lock) | ✅ via single flush path | ✅ FIFO MessageGroupId | ✅ via execution name |
| No extra infrastructure | ✅ SQS only | ❌ DynamoDB + Scheduler | ❌ Pipes + FIFO | ❌ Step Functions |
| Rejection events bypass batching | manual routing | manual routing | manual routing | needs separate path |
| Content-hash no-op | add as enhancement | add as enhancement | add as enhancement | add as enhancement |
| Relative complexity | Low | Medium | Medium | High |

---

## Open questions

1. **What is the maximum acceptable latency from "ML Lead approves" to "CodePipeline starts"?** If 60 seconds is too long, the SQS batch window must be shorter (or omitted and replaced with a content-hash-only guard).

2. **Should the family manifest replace the current per-model manifest, or should both exist?** Recommendation: replace — the family manifest is strictly more powerful and the RegisterModel step already needs to handle arrays for the shadow-slot model.

3. **How is "slot assignment" for shadow models handled in the manifest?** The current design has `MAX_SHADOW_NODES = 3` with slot positions (1, 2, 3). The Coalescing Lambda needs a deterministic slot assignment rule (e.g., sorted by model version timestamp, oldest shadow = slot 1). This rule must be stable across re-runs.

4. **Is cross-region in scope now?** The cross-region replication design (`cross-region-replication.md`) may add additional manifest targets per event. The coalescing layer should be designed to write to N manifest buckets (one per region), not just one.

5. **Pattern D tolerance:** Is it acceptable that promoting a release model now and adding a shadow 3 minutes later results in two CodePipeline runs (both successful, second one idempotent for the release)? If yes, Option A is sufficient. If not, Option B is needed.

6. **Teams webhook provisioning**: Who owns the incoming webhook URL for #mlops-alerts, and how is it rotated / managed? The URL needs to live in SSM Parameter Store and be accessible from the dev account's De-promotion Lambda. Is there an existing Teams channel + webhook, or does one need to be created?
