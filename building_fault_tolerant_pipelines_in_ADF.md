# Copilot Chat Conversation Export: Building fault-tolerant pipelines in ADF

**User:** @ahmed-mansoor_SAGCP
**Thread URL:** https://github.com/copilot/c/243c0062-f279-473f-b19d-1def0d083bf3

## @ahmed-mansoor_SAGCP

How do you build pipelines in ADF based on fault tolerance
coach me as a new bie turn into expert with practical exposure based on real world engineering projects using small scenarios
Consider you are my mentor with 20 years of data engineering and practical teaching expertise
don't go in shot go layer by layer

## Copilot

Great question, Ahmed! The support search returned GitHub-specific results (not ADF), so I'm drawing on my **20 years of data engineering mentorship knowledge** to coach you properly. Let's build this layer by layer — the right way.

---

# 🎓 ADF Fault-Tolerant Pipelines — Mentor's Masterclass

> **Philosophy before tools**: *A fault-tolerant pipeline doesn't just run — it SURVIVES failures, LEARNS from them, and RECOVERS gracefully. That's production-grade engineering.*

---

## 🗺️ Your Learning Roadmap (Layer by Layer)

```
Layer 1 → What is Fault Tolerance? (Mental Model)
Layer 2 → ADF Building Blocks You Must Know
Layer 3 → Your First Fault-Tolerant Pipeline (Scenario)
Layer 4 → Retry Policies & Timeouts
Layer 5 → Error Handling with If Condition + Failure Paths
Layer 6 → Logging Failures (Real World Pattern)
Layer 7 → Idempotency — The Secret Weapon
Layer 8 → Alerting & Monitoring
Layer 9 → Real-World Architecture Pattern
Layer 10 → Production Checklist
```

We start **Layer 1 today** and go deeper each step.

---

## 🧱 LAYER 1 — The Mental Model: What is Fault Tolerance in ADF?

### 🤔 Think of it like THIS (Mentor's Analogy)

> Imagine you're a courier delivery driver. Your job is to deliver 10,000 packages a day.
> - **Without fault tolerance**: If 1 package has the wrong address → you stop, go home, deliver nothing.
> - **With fault tolerance**: You log the bad address, skip it, deliver the other 9,999, then retry the bad one later or alert someone.

**That's exactly what a fault-tolerant ADF pipeline does.**

---

### 🏗️ The 5 Pillars of Fault Tolerance in ADF

```
┌─────────────────────────────────────────────────────┐
│           FAULT TOLERANCE = 5 PILLARS               │
├──────────────┬──────────────────────────────────────┤
│  Pillar 1    │  RETRY          → Try again auto     │
│  Pillar 2    │  SKIP & LOG     → Skip bad rows/files│
│  Pillar 3    │  BRANCHING      → Handle success/fail│
│  Pillar 4    │  IDEMPOTENCY    → Safe to re-run     │
│  Pillar 5    │  ALERTING       → Know when it breaks│
└──────────────┴──────────────────────────────────────┘
```

---

## 🧱 LAYER 2 — ADF Building Blocks You Must Master First

Before building fault-tolerant pipelines, know your tools:

| ADF Concept | What it Does | Fault Tolerance Role |
|---|---|---|
| **Pipeline** | Container of activities | Orchestrates the flow |
| **Activity** | A single unit of work (Copy, Lookup, etc.) | Where failures happen |
| **Retry Policy** | Auto-retry on failure | Handles transient errors |
| **Dependency Condition** | Success/Failure/Completion/Skipped | Controls flow on failure |
| **If Condition Activity** | Branch: true/false path | Handle different outcomes |
| **Set Variable** | Store values | Track state/errors |
| **Web Activity** | Call REST APIs | Send alerts |
| **Stored Procedure** | Run SQL | Log errors to DB |
| **Fault Tolerance (Copy)** | Skip bad rows | Data-level resilience |

---

## 🧱 LAYER 3 — Your First Real-World Scenario 🏭

### 📦 Scenario: "Daily Sales File Ingestion"

**Business Problem:**
> Every morning at 6 AM, 50 CSV files land in Azure Blob Storage from different regional stores. You need to copy them to Azure SQL Database. Some files might be malformed. The business cannot afford the entire pipeline to fail because of 1 bad file.

### 🗂️ Without Fault Tolerance (Beginner Mistake)
```
[Get Files] → [For Each File] → [Copy Data]
                                    ↑
                             If 1 file fails → 
                             ENTIRE pipeline FAILS ❌
```

### ✅ With Fault Tolerance (What You'll Build)
```
[Get Files] → [For Each File] → [Copy Data]
                                   ├── SUCCESS → [Log Success] ✅
                                   └── FAILURE → [Log Error to SQL] 
                                                → [Send Alert Email]
                                                → [Continue next file] ✅
```

---

## 🔧 LAYER 4 — Retry Policies (Your First Line of Defense)

### 🎯 Concept
Most failures in data pipelines are **transient** — network blip, API throttling, DB momentarily busy. A retry policy handles these **automatically without human intervention**.

### ⚙️ Where to Set It in ADF
Every activity has a **Settings tab** → Retry section:

```
┌─────────────────────────────────┐
│  Activity Settings              │
│  ┌───────────────────────────┐  │
│  │ Retry:          3         │  │  ← Try 3 times
│  │ Retry interval: 30 sec    │  │  ← Wait 30s between retries
│  │ Timeout:        00:30:00  │  │  ← Give up after 30 mins
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

### 🏭 Real World Rules I Follow (Mentor's Cheat Sheet)

| Source Type | Retry Count | Interval | Why |
|---|---|---|---|
| REST API | 3 | 60s | Rate limiting recovers in ~1 min |
| Azure SQL | 2 | 30s | Transient connection drops |
| Blob Storage | 2 | 15s | Very rare, quick recovery |
| SAP / On-prem | 5 | 120s | Legacy systems are slow |
| Snowflake/Synapse | 3 | 45s | Query queue clears fast |

### ⚠️ Mentor Warning
> **Never set retry to 0** on production pipelines. That's a rookie mistake. Even a `retry: 1` with `30s interval` saves you from 80% of transient failures.

---

## 🧱 LAYER 5 — Dependency Conditions (The Traffic Lights of ADF)

This is where the **real magic** of fault tolerance lives.

### 🚦 The 4 Dependency Conditions

```
┌──────────────┬────────────────────────────────────────┐
│  Condition   │  Meaning                               │
├──────────────┼────────────────────────────────────────┤
│  ✅ Success  │  Run only if previous activity PASSED  │
│  ❌ Failure  │  Run only if previous activity FAILED  │
│  ⏭️ Skipped  │  Run if previous was SKIPPED           │
│  🔄 Complete │  Run REGARDLESS of success or failure  │
└──────────────┴────────────────────────────────────────┘
```

### 🏭 Practical Scenario — Adding Failure Branch

In your Sales File pipeline:

```
[Copy Data Activity]
        │
        ├──[On SUCCESS]──→ [Stored Procedure: Log_Success]
        │
        └──[On FAILURE]──→ [Stored Procedure: Log_Error]
                                      │
                                      └──→ [Web Activity: Send Teams Alert]
```

**The KEY insight**: The `ForEach` loop continues to the NEXT file even when one fails, because you handled the failure inside the loop — you didn't let it bubble up and kill the parent pipeline.

---

## 🧱 LAYER 6 — Logging Failures (Production-Grade Pattern)

### 🗄️ Create a Pipeline Log Table in SQL

```sql
-- Create this ONCE in your Azure SQL Database
CREATE TABLE pipeline_execution_log (
    log_id          INT IDENTITY(1,1) PRIMARY KEY,
    pipeline_name   VARCHAR(200),
    activity_name   VARCHAR(200),
    source_file     VARCHAR(500),
    status          VARCHAR(50),     -- 'SUCCESS' or 'FAILED'
    error_message   VARCHAR(MAX),
    rows_read       BIGINT,
    rows_written    BIGINT,
    execution_date  DATETIME DEFAULT GETDATE(),
    run_id          VARCHAR(200)     -- ADF Pipeline Run ID
);
```

### 🔧 In ADF — Log with Stored Procedure Activity

**On Success path:**
```json
{
  "storedProcedureName": "usp_log_pipeline",
  "storedProcedureParameters": {
    "pipeline_name":  "@pipeline().Pipeline",
    "activity_name":  "CopySalesFile",
    "source_file":    "@item().name",
    "status":         "SUCCESS",
    "rows_written":   "@activity('CopySalesFile').output.rowsCopied",
    "run_id":         "@pipeline().RunId"
  }
}
```

**On Failure path:**
```json
{
  "storedProcedureName": "usp_log_pipeline",
  "storedProcedureParameters": {
    "pipeline_name":  "@pipeline().Pipeline",
    "activity_name":  "CopySalesFile",
    "source_file":    "@item().name",
    "status":         "FAILED",
    "error_message":  "@activity('CopySalesFile').error.message",
    "run_id":         "@pipeline().RunId"
  }
}
```

### 💡 Mentor's Key ADF System Variables

```
@pipeline().Pipeline       → Pipeline name
@pipeline().RunId          → Unique run ID (gold for debugging)
@pipeline().TriggerTime    → When it was triggered
@activity('X').error.message → Error from activity X
@activity('X').output.rowsCopied → Rows written
@item().name               → Current file in ForEach
```

---

## 🧱 LAYER 7 — Idempotency (The Secret Weapon of Senior Engineers)

### 🤔 What is Idempotency?

> **"If I run this pipeline 10 times, the result is the same as running it once."**

This is **CRITICAL** for fault tolerance. If your pipeline fails halfway and you re-run it, you must NOT get duplicate data.

### 🏭 Real Scenario — The Duplicate Data Disaster
```
Day 1: Pipeline loads 100,000 sales rows ✅
Day 2: Pipeline fails at row 60,000 → You restart it
Day 2 (restart): Pipeline loads ALL 100,000 again → 60,000 DUPLICATES ❌
```

### ✅ The Fix — UPSERT Pattern or Staging Table

**Option A: Staging Table (Most Common in ADF)**
```
[Copy to STAGING table] → [Merge STAGING into FINAL table] → [Truncate STAGING]
         │                           │
    Full reload safe            Handles duplicates via MERGE
```

**Option B: Pre-Copy Script in Copy Activity**
```
In Copy Activity → Sink tab → Pre-copy script:
TRUNCATE TABLE staging_sales;
-- OR
DELETE FROM sales WHERE load_date = '@{formatDateTime(pipeline().TriggerTime,'yyyy-MM-dd')}'
```

This deletes today's data first, then reloads — making re-runs safe.

---

## 📊 Where We Are Now

```
✅ Layer 1 → Mental Model (DONE)
✅ Layer 2 → ADF Building Blocks (DONE)
✅ Layer 3 → First Fault-Tolerant Scenario (DONE)
✅ Layer 4 → Retry Policies (DONE)
✅ Layer 5 → Dependency Conditions (DONE)
✅ Layer 6 → Logging Failures (DONE)
✅ Layer 7 → Idempotency (DONE)
⏳ Layer 8 → Alerting & Monitoring (Next)
⏳ Layer 9 → Real-World Architecture Pattern (Next)
⏳ Layer 10 → Production Checklist (Final)
```

---

## 🎯 Your Homework Before Layer 8

Try this hands-on:

1. **Create** a simple ADF pipeline with a Copy Activity
2. **Add** a Retry policy (count: 2, interval: 30s)
3. **Add** a failure dependency branch with a simple Set Variable activity that captures `@activity('YourActivity').error.message`
4. **Intentionally break** your source connection and watch the retry + failure branch kick in

> 💬 **Tell me**: Did you understand Layers 1–7? Any part you want me to go deeper on? Or say **"ready for Layer 8"** and we go to Alerting & Monitoring! 🚀
