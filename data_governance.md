# Copilot Chat Conversation Export: Data governance in Databricks: A mentorship

**User:** @ahmed-mansoor_SAGCP
**Thread URL:** https://github.com/copilot/c/1a965f2a-b343-409f-82d6-3214cfce0b84

## @ahmed-mansoor_SAGCP

Data governance in databrick. How do you achieve it what is the purpose in all necessary aspect to gain exposure for work and interview - different use cases
How will you implement row level and column level  access or somethig
coach me as a new bie turn into expert with practical exposure based on real world engineering projects using small scenarios
Consider you are my mentor with 20 years of data engineering and practical teaching expertise

## Copilot

# Data Governance in Databricks: Complete Mentorship Guide

> 🎓 **Your Mentor Speaking:** I've spent 20 years building data platforms across finance, healthcare, retail, and tech. Let me take you from zero to interview-ready with real-world scenarios. Buckle up — this is going to be thorough.

---

## 🗺️ The Big Picture: What IS Data Governance?

Think of Data Governance as **the rule book for your data city**:

| Analogy | Data Governance Equivalent |
|---|---|
| City laws & regulations | Data policies & compliance rules |
| Police enforcing laws | Access control & auditing |
| Zoning laws | Data classification & sensitivity |
| Building permits | Data lineage & approval workflows |
| Census bureau | Data cataloging & discovery |
| Property records | Metadata management |

---

## 🏗️ Databricks Unity Catalog: The Governance Engine

> **Mentor Note:** Before Unity Catalog (UC) existed, governance in Databricks was a fragmented mess — table ACLs here, IAM there, Hive metastore everywhere. Unity Catalog changed everything in 2022. Every modern interview WILL ask about this.

### The 3-Level Hierarchy

```
Metastore (One per region — your governance root)
    └── Catalog (like a database server)
            └── Schema / Database (like a folder)
                    └── Table / View / Volume / Function
```

### Real-World Analogy:
```
Metastore     = Your company's entire data estate
Catalog       = "finance_catalog", "hr_catalog", "marketing_catalog"
Schema        = "transactions", "payroll", "campaigns"
Table         = "credit_card_txn", "employee_salary", "ad_spend"
```

---

## 🎯 The 6 Pillars of Data Governance in Databricks

```
┌─────────────────────────────────────────────────────────────┐
│                  DATA GOVERNANCE PILLARS                     │
├──────────────┬──────────────┬──────────────┬────────────────┤
│  1. Access   │  2. Data     │  3. Data     │  4. Audit &    │
│  Control     │  Discovery   │  Quality     │  Lineage       │
│  (Who sees   │  (Find data) │  (Trust      │  (Track data   │
│   what?)     │              │   data)      │   movement)    │
├──────────────┴──────────────┼──────────────┴────────────────┤
│      5. Data Classification  │   6. Compliance & Privacy    │
│      (Tag sensitive data)    │   (GDPR, HIPAA, PCI-DSS)    │
└──────────────────────────────┴──────────────────────────────┘
```

---

## 🔐 PILLAR 1: Access Control (The Core Interview Topic)

### 1A. Role-Based Access Control (RBAC)

```sql
-- SCENARIO: FinTech Company - You're the Data Engineer setting up governance

-- Step 1: Create Catalog per domain
CREATE CATALOG IF NOT EXISTS finance_catalog
  COMMENT 'All financial data - PCI-DSS compliant';

CREATE CATALOG IF NOT EXISTS hr_catalog
  COMMENT 'Human Resources data - GDPR sensitive';

-- Step 2: Create Schemas
CREATE SCHEMA IF NOT EXISTS finance_catalog.transactions;
CREATE SCHEMA IF NOT EXISTS finance_catalog.reporting;

-- Step 3: Grant Catalog-level access to groups
-- (Always use GROUPS, never individual users — best practice!)
GRANT USE CATALOG ON CATALOG finance_catalog TO `finance_analysts`;
GRANT USE SCHEMA ON SCHEMA finance_catalog.transactions TO `finance_analysts`;

-- Step 4: Grant Table-level access
GRANT SELECT ON TABLE finance_catalog.transactions.credit_card_txn 
  TO `finance_analysts`;

-- Step 5: Data Engineers get full access
GRANT ALL PRIVILEGES ON CATALOG finance_catalog TO `data_engineers`;
```

### 1B. ⭐ ROW-LEVEL SECURITY (RLS) — Most Asked Interview Topic

> **Mentor Note:** This is THE question in every senior data engineering interview. Master this cold.

#### Scenario: Regional Sales Data
> A sales manager in **North America** should ONLY see North America sales. A manager in **Europe** sees only Europe data. The VP sees everything.

```sql
-- STEP 1: Create the base table
CREATE TABLE finance_catalog.sales.global_sales (
  sale_id        BIGINT,
  sale_amount    DECIMAL(10,2),
  region         STRING,        -- 'NA', 'EU', 'APAC'
  sales_rep_id   STRING,
  sale_date      DATE,
  customer_id    STRING
);

-- STEP 2: Create a user-region mapping table
-- (This is your security mapping table — critical pattern!)
CREATE TABLE finance_catalog.security.user_region_access (
  user_email     STRING,
  allowed_region STRING
);

-- Insert access rules
INSERT INTO finance_catalog.security.user_region_access VALUES
  ('john.smith@company.com',  'NA'),
  ('anna.mueller@company.com', 'EU'),
  ('raj.patel@company.com',   'APAC'),
  ('vp.sales@company.com',    'NA'),    -- VP gets all regions
  ('vp.sales@company.com',    'EU'),    -- by having multiple rows
  ('vp.sales@company.com',    'APAC');

-- STEP 3: Create a ROW FILTER FUNCTION (Unity Catalog feature)
CREATE OR REPLACE FUNCTION finance_catalog.security.region_row_filter(region STRING)
RETURNS BOOLEAN
RETURN 
  -- Admin/VP bypass: check if user is in admin group
  IS_ACCOUNT_GROUP_MEMBER('data_admins')
  OR
  -- Check if current user has access to this region
  EXISTS (
    SELECT 1 
    FROM finance_catalog.security.user_region_access
    WHERE user_email = CURRENT_USER()
      AND allowed_region = region
  );

-- STEP 4: Apply the Row Filter to the table
ALTER TABLE finance_catalog.sales.global_sales
  SET ROW FILTER finance_catalog.security.region_row_filter ON (region);

-- TESTING: When John Smith runs this, he ONLY sees NA rows
SELECT * FROM finance_catalog.sales.global_sales;
-- Result: Only rows where region = 'NA'

-- When VP runs it, they see everything
-- Result: All rows
```

#### Alternative Pattern: Current User Direct Filter
```sql
-- Simpler pattern when table has user_id column
-- e.g., Employees seeing only their own performance data

CREATE OR REPLACE FUNCTION hr_catalog.security.employee_self_filter(
  employee_email STRING
)
RETURNS BOOLEAN
RETURN 
  IS_ACCOUNT_GROUP_MEMBER('hr_managers')  -- HR sees all
  OR employee_email = CURRENT_USER();      -- Employee sees only their own

ALTER TABLE hr_catalog.hr.performance_reviews
  SET ROW FILTER hr_catalog.security.employee_self_filter ON (employee_email);
```

### 1C. ⭐ COLUMN-LEVEL SECURITY — Second Most Asked

> **Mentor Note:** Think of Column Masking as a bouncer who shows you a blurred version of sensitive data unless you're on the VIP list.

#### Scenario: Healthcare System — Mask PII for Non-Medical Staff

```sql
-- Base table with sensitive columns
CREATE TABLE health_catalog.patients.patient_records (
  patient_id      STRING,
  full_name       STRING,      -- PII
  date_of_birth   DATE,        -- PII  
  ssn             STRING,      -- HIGHLY SENSITIVE
  diagnosis_code  STRING,      -- PHI (Protected Health Info)
  treatment_plan  STRING,      -- PHI
  billing_amount  DECIMAL(10,2),
  doctor_id       STRING
);

-- COLUMN MASK for SSN
-- Only compliance officers and doctors see real SSN
CREATE OR REPLACE FUNCTION health_catalog.security.mask_ssn(ssn STRING)
RETURNS STRING
RETURN 
  CASE 
    WHEN IS_ACCOUNT_GROUP_MEMBER('medical_staff') 
      OR IS_ACCOUNT_GROUP_MEMBER('compliance_officers')
    THEN ssn                            -- See real value
    ELSE CONCAT('XXX-XX-', RIGHT(ssn, 4))  -- See masked: XXX-XX-1234
  END;

-- COLUMN MASK for full name
CREATE OR REPLACE FUNCTION health_catalog.security.mask_patient_name(
  full_name STRING
)
RETURNS STRING
RETURN
  CASE
    WHEN IS_ACCOUNT_GROUP_MEMBER('medical_staff') THEN full_name
    WHEN IS_ACCOUNT_GROUP_MEMBER('billing_team') 
    THEN CONCAT(LEFT(full_name, 1), '*** ', SPLIT_PART(full_name,' ',2))
    ELSE '*** REDACTED ***'
  END;

-- COLUMN MASK for date of birth (show only year for researchers)
CREATE OR REPLACE FUNCTION health_catalog.security.mask_dob(dob DATE)
RETURNS DATE
RETURN
  CASE
    WHEN IS_ACCOUNT_GROUP_MEMBER('medical_staff') THEN dob
    WHEN IS_ACCOUNT_GROUP_MEMBER('researchers')   
    THEN DATE(CONCAT(YEAR(dob), '-01-01'))  -- Only show year
    ELSE NULL                               -- Hide completely
  END;

-- Apply all masks to the table
ALTER TABLE health_catalog.patients.patient_records
  ALTER COLUMN ssn 
    SET MASK health_catalog.security.mask_ssn;

ALTER TABLE health_catalog.patients.patient_records
  ALTER COLUMN full_name 
    SET MASK health_catalog.security.mask_patient_name;

ALTER TABLE health_catalog.patients.patient_records
  ALTER COLUMN date_of_birth 
    SET MASK health_catalog.security.mask_dob;
```

#### What different users see:

```
┌─────────────────┬───────────────┬─────────────┬──────────────────┐
│ User Role       │ full_name     │ ssn         │ date_of_birth    │
├─────────────────┼───────────────┼─────────────┼──────────────────┤
│ Doctor          │ John Smith    │ 123-45-6789 │ 1985-03-15       │
│ Billing Team    │ J*** Smith    │ XXX-XX-6789 │ NULL             │
│ Researcher      │ *** REDACTED  │ XXX-XX-6789 │ 1985-01-01       │
│ Other Staff     │ *** REDACTED  │ XXX-XX-6789 │ NULL             │
└─────────────────┴───────────────┴─────────────┴──────────────────┘
```

---

## 🔍 PILLAR 2: Data Discovery & Cataloging

```python
# Using Databricks SDK to tag and document tables programmatically

from databricks.sdk import WorkspaceClient
from databricks.sdk.service.catalog import (
    TableInfo, ColumnInfo, SecurableType
)

w = WorkspaceClient()

# Add tags to classify sensitive tables
w.tables.update(
    full_name="health_catalog.patients.patient_records",
    # Business metadata tags
)

# Better: Use SQL for tagging (simpler for interviews)
```

```sql
-- TAG tables for data classification
ALTER TABLE health_catalog.patients.patient_records
  SET TAGS ('sensitivity' = 'HIGH', 
            'regulation'  = 'HIPAA', 
            'pii'         = 'true',
            'owner'       = 'data-governance-team',
            'retention_days' = '2555');  -- 7 years HIPAA retention

-- TAG specific columns
ALTER TABLE health_catalog.patients.patient_records
  ALTER COLUMN ssn 
    SET TAGS ('pii' = 'true', 'pci' = 'false', 'classification' = 'restricted');

ALTER TABLE health_catalog.patients.patient_records
  ALTER COLUMN diagnosis_code 
    SET TAGS ('phi' = 'true', 'hipaa' = 'true');

-- Search for all HIPAA tables in your catalog
SELECT table_catalog, table_schema, table_name, tag_name, tag_value
FROM system.information_schema.table_tags
WHERE tag_name = 'regulation' AND tag_value = 'HIPAA';
```

---

## 📊 PILLAR 3: Data Lineage (Track Where Data Flows)

> **Mentor Note:** Lineage answers: *"If I change table X, what downstream reports break?"* This is GOLD in incident management.

```python
# Databricks automatically tracks lineage for:
# - SQL queries
# - Delta Lake operations  
# - Notebooks
# - DLT (Delta Live Tables) pipelines

# View lineage programmatically
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# Get upstream lineage (what feeds into my table?)
lineage = w.lineage_tracking.table_lineage(
    table_name="finance_catalog.reporting.monthly_revenue"
)

# This shows you:
# monthly_revenue <-- daily_sales <-- raw_transactions <-- source_system
```

```sql
-- Query system tables for lineage
SELECT 
  source_table_full_name,
  target_table_full_name,
  event_time,
  entity_type
FROM system.access.table_lineage
WHERE target_table_full_name = 'finance_catalog.reporting.monthly_revenue'
ORDER BY event_time DESC;
```

### Visual Lineage Flow:
```
Source System (Kafka/S3)
        │
        ▼
[Bronze] raw_transactions          ← Autoloader ingests
        │
        ▼  
[Silver] cleaned_transactions      ← Transformations applied
        │
        ├──────────────────────────┐
        ▼                          ▼
[Gold] daily_sales          [Gold] fraud_detection
        │
        ▼
[Gold] monthly_revenue             ← Reports & Dashboards
        │
        ▼
Power BI / Tableau Dashboard       ← End consumers
```

---

## 📋 PILLAR 4: Audit Logging — "Who Did What, When?"

```sql
-- Unity Catalog System Tables — Automatic audit logs!

-- 1. Who accessed sensitive patient data last 24 hours?
SELECT 
  event_time,
  user_identity.email       AS user_email,
  action_name,
  request_params.full_name  AS table_accessed,
  response.status_code      AS status
FROM system.access.audit
WHERE 
  event_time >= CURRENT_TIMESTAMP() - INTERVAL 1 DAY
  AND request_params.full_name LIKE 'health_catalog.patients%'
  AND action_name IN ('SELECT', 'commandRun')
ORDER BY event_time DESC;

-- 2. Alert: Someone accessed data outside business hours
SELECT 
  user_identity.email,
  event_time,
  action_name,
  request_params.full_name
FROM system.access.audit
WHERE 
  HOUR(event_time) NOT BETWEEN 8 AND 18   -- Outside 8am-6pm
  AND DAYOFWEEK(event_time) NOT IN (1, 7)  -- Weekdays only
  AND action_name = 'SELECT'
  AND request_params.full_name LIKE '%patient%';

-- 3. Permission change audit — who granted access?
SELECT 
  event_time,
  user_identity.email       AS granted_by,
  action_name,
  request_params
FROM system.access.audit
WHERE action_name IN ('grantPermission', 'revokePermission')
  AND DATE(event_time) = CURRENT_DATE()
ORDER BY event_time DESC;
```

---

## 🏷️ PILLAR 5: Data Classification Framework

```
┌─────────────────────────────────────────────────────────────┐
│                CLASSIFICATION TIERS                          │
├─────────────┬──────────────┬─────────────────┬─────────────┤
│  PUBLIC     │  INTERNAL    │  CONFIDENTIAL   │  RESTRICTED │
│             │              │                 │             │
│ Open data   │ Employee     │ Customer PII    │ SSN, PHI    │
│ Marketing   │  metrics     │ Financial data  │ Passwords   │
│ Docs        │ Dept KPIs    │ Contracts       │ Crypto keys │
├─────────────┼──────────────┼─────────────────┼─────────────┤
│ Anyone      │ All staff    │ Need-to-know    │ Explicit    │
│             │              │                 �� approval    │
└─────────────┴──────────────┴─────────────────┴─────────────┘
```

```sql
-- Implement classification via tags + Row/Column filters
-- Create a classification policy table
CREATE TABLE governance_catalog.policies.data_classification (
  catalog_name        STRING,
  schema_name         STRING,
  table_name          STRING,
  column_name         STRING,
  classification_tier STRING,  -- PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED
  regulation          STRING,  -- GDPR, HIPAA, PCI-DSS, SOX
  pii_flag            BOOLEAN,
  retention_days      INT,
  owner_team          STRING,
  last_reviewed_date  DATE
);
```

---

## ⚖️ PILLAR 6: Compliance Use Cases

### GDPR — Right to Erasure (The "Forget Me" Request)

```python
# REAL SCENARIO: User submits GDPR deletion request
# You must delete/anonymize their data across ALL tables

from pyspark.sql import SparkSession
from pyspark.sql.functions import lit, sha2, concat
from delta.tables import DeltaTable

spark = SparkSession.builder.getOrCreate()

def process_gdpr_deletion(customer_id: str, request_id: str):
    """
    GDPR Article 17: Right to Erasure
    Anonymize customer data across all relevant tables
    """
    
    tables_with_pii = [
        "finance_catalog.customers.customer_profile",
        "finance_catalog.transactions.credit_card_txn",
        "marketing_catalog.campaigns.email_list"
    ]
    
    for table_path in tables_with_pii:
        delta_table = DeltaTable.forName(spark, table_path)
        
        # Instead of DELETE (which breaks audit trails),
        # ANONYMIZE by hashing the identifier
        delta_table.update(
            condition=f"customer_id = '{customer_id}'",
            set={
                "email":      lit(f"anonymized_{request_id}@deleted.com"),
                "full_name":  lit("GDPR_DELETED"),
                "phone":      lit(None),
                "address":    lit(None),
                "ip_address": lit(None)
            }
        )
        
        print(f"✅ Anonymized {customer_id} in {table_path}")
    
    # Log the deletion request for compliance proof
    spark.sql(f"""
        INSERT INTO governance_catalog.gdpr.deletion_log VALUES (
          '{request_id}',
          '{customer_id}', 
          CURRENT_TIMESTAMP(),
          'COMPLETED',
          CURRENT_USER()
        )
    """)

# Execute
process_gdpr_deletion("CUST_12345", "GDPR_REQ_2026_001")
```

---

## 🏭 REAL-WORLD PROJECT: End-to-End Governance Setup

### Scenario: Retail Bank Building a Data Lakehouse

```
REQUIREMENT:
- 500 data analysts across 5 countries
- Regulators can audit but not modify
- Tellers see only their branch customers
- Risk team sees all transactions but masked SSN
- C-Suite sees aggregated KPIs only
```

```python
# governance_setup.py — Production-grade setup script

class BankDataGovernance:
    
    def __init__(self, spark):
        self.spark = spark
    
    def setup_catalog_hierarchy(self):
        """Step 1: Create domain-isolated catalogs"""
        catalogs = {
            "retail_banking":   "Retail customer accounts and transactions",
            "risk_management":  "Fraud and risk models - restricted",
            "regulatory":       "Regulator read-only access",
            "data_products":    "Curated, business-ready datasets"
        }
        
        for catalog, comment in catalogs.items():
            self.spark.sql(f"""
                CREATE CATALOG IF NOT EXISTS {catalog}
                COMMENT '{comment}'
            """)
    
    def setup_branch_row_security(self):
        """Step 2: Tellers see only their branch"""
        
        # Create user-branch mapping
        self.spark.sql("""
            CREATE TABLE IF NOT EXISTS 
            retail_banking.security.teller_branch_map (
              teller_email  STRING,
              branch_id     STRING,
              branch_name   STRING,
              country       STRING
            )
        """)
        
        # Row filter function
        self.spark.sql("""
            CREATE OR REPLACE FUNCTION 
            retail_banking.security.branch_row_filter(branch_id STRING)
            RETURNS BOOLEAN
            RETURN
              IS_ACCOUNT_GROUP_MEMBER('bank_managers')    -- Managers see all
              OR IS_ACCOUNT_GROUP_MEMBER('risk_officers') -- Risk sees all
              OR EXISTS (
                SELECT 1 
                FROM retail_banking.security.teller_branch_map
                WHERE teller_email = CURRENT_USER()
                  AND branch_id = branch_id
              )
        """)
        
        # Apply to customer accounts table
        self.spark.sql("""
            ALTER TABLE retail_banking.accounts.customer_accounts
              SET ROW FILTER 
              retail_banking.security.branch_row_filter ON (branch_id)
        """)
    
    def setup_ssn_masking(self):
        """Step 3: Mask SSN based on role"""
        
        self.spark.sql("""
            CREATE OR REPLACE FUNCTION
            retail_banking.security.mask_ssn(ssn STRING)
            RETURNS STRING
            RETURN CASE
              WHEN IS_ACCOUNT_GROUP_MEMBER('compliance_officers') THEN ssn
              WHEN IS_ACCOUNT_GROUP_MEMBER('risk_officers')
              THEN CONCAT('XXX-XX-', RIGHT(ssn, 4))
              ELSE '***-**-****'
            END
        """)
        
        self.spark.sql("""
            ALTER TABLE retail_banking.accounts.customer_accounts
              ALTER COLUMN ssn
                SET MASK retail_banking.security.mask_ssn
        """)
    
    def grant_regulatory_access(self):
        """Step 4: Read-only auditor access"""
        
        self.spark.sql("""
            -- Regulators get SELECT only — no INSERT/UPDATE/DELETE
            GRANT USE CATALOG ON CATALOG retail_banking 
              TO `regulatory_auditors`;
            GRANT USE SCHEMA ON ALL SCHEMAS IN CATALOG retail_banking
              TO `regulatory_auditors`;
            GRANT SELECT ON ALL TABLES IN CATALOG retail_banking
              TO `regulatory_auditors`;
        """)

# Run setup
governance = BankDataGovernance(spark)
governance.setup_catalog_hierarchy()
governance.setup_branch_row_security()
governance.setup_ssn_masking()
governance.grant_regulatory_access()
```

---

## 💡 Dynamic Views (Legacy Pattern — Still Asked in Interviews)

> **Mentor Note:** Before Unity Catalog Row Filters, Dynamic Views were THE solution. Many companies still use them. Know both!

```sql
-- LEGACY PATTERN: Dynamic View for Row-Level Security
CREATE OR REPLACE VIEW finance_catalog.sales.v_sales_regional AS
SELECT *
FROM finance_catalog.sales.global_sales
WHERE 
  -- If user is in admin group, show everything
  IS_MEMBER('data_admins')
  OR
  -- Otherwise filter by their region mapping
  region IN (
    SELECT allowed_region 
    FROM finance_catalog.security.user_region_access
    WHERE user_email = CURRENT_USER()
  );

-- Grant access to the VIEW (not the base table!)
GRANT SELECT ON VIEW finance_catalog.sales.v_sales_regional 
  TO `finance_analysts`;

-- REVOKE access to base table for analysts
REVOKE SELECT ON TABLE finance_catalog.sales.global_sales
  FROM `finance_analysts`;
```

---

## 🎤 Interview Questions + Model Answers

### Q1: "What is Unity Catalog and why is it important?"
> **Answer:** Unity Catalog is Databricks' centralized governance layer that provides a single metastore for all data assets across workspaces. It's important because it enables fine-grained access control at catalog/schema/table/row/column levels, automated data lineage, audit logging via system tables, and supports multi-cloud governance — all from a single control plane instead of managing ACLs across multiple systems.

### Q2: "Explain Row-Level Security in Databricks"
> **Answer:** In Unity Catalog, RLS is implemented via **Row Filter Functions** — SQL functions that evaluate `TRUE/FALSE` per row based on the current user's identity or group membership. You attach them to tables with `ALTER TABLE ... SET ROW FILTER`. The filter is transparent to users — they query normally but only see authorized rows. Before UC, we used **Dynamic Views** to achieve the same.

### Q3: "How do you implement Column Masking?"
> **Answer:** Via **Column Mask Functions** in Unity Catalog. You create a SQL function that returns the masked or original value based on `CURRENT_USER()` or `IS_ACCOUNT_GROUP_MEMBER()`, then apply it with `ALTER TABLE ... ALTER COLUMN ... SET MASK`. Unlike views, masks are enforced at the storage layer — no way to bypass them.

### Q4: "How do you handle GDPR data deletion in a Delta Lake?"
> **Answer:** Instead of hard deletes (which break audit trails), we **anonymize in-place** using Delta Lake's UPDATE command, replacing PII fields with hashed/null values. We also use Delta Lake's `VACUUM` carefully to purge old file versions after retention period. We maintain a deletion log table for compliance proof. For streaming data, we implement tombstone records.

### Q5: "What are system tables in Unity Catalog?"
> **Answer:** System tables are auto-generated audit tables under `system.*` schema that capture: `system.access.audit` (all access events), `system.access.table_lineage` (data lineage), `system.billing.usage` (compute costs), `system.information_schema` (metadata). They enable governance reporting, anomaly detection, and cost attribution without any manual instrumentation.

---

## 🗺️ Your Learning Roadmap

```
WEEK 1-2: FOUNDATIONS
├── Set up Databricks Community Edition (free)
├── Create Unity Catalog metastore
├── Practice GRANT/REVOKE SQL commands
└── Build your first Row Filter

WEEK 3-4: INTERMEDIATE
├── Implement Column Masking on sample healthcare data
├── Query system.access.audit tables
├── Build a Dynamic View (legacy pattern)
└── Create a data classification tagging system

WEEK 5-6: ADVANCED
├── End-to-end GDPR deletion workflow
├── Delta Live Tables with governance
├── Cross-workspace data sharing (Delta Sharing)
└── Build anomaly detection on audit logs

WEEK 7-8: INTERVIEW READY
├── Mock architecture: Design governance for a bank
├── Mock architecture: Design governance for healthcare
├── Code walkthroughs without notes
└── System design: Multi-cloud governance strategy
```

---

## 🔑 Key Takeaways

| Concept | Tool | When to Use |
|---|---|---|
| Row-Level Security | UC Row Filter Functions | Restrict rows by user/region/dept |
| Column Masking | UC Column Mask Functions | Hide/mask PII, SSN, PHI |
| RBAC | GRANT/REVOKE on Groups | Control catalog/schema/table access |
| Data Discovery | Tags + System Tables | Find and classify sensitive data |
| Audit Logging | `system.access.audit` | Compliance, anomaly detection |
| Data Lineage | UC Lineage + System Tables | Impact analysis, root cause |
| GDPR Compliance | Delta Lake UPDATE + VACUUM | Right to erasure |
| Legacy Pattern | Dynamic Views | Older workspaces without UC |

> 🎓 **Final Mentor Advice:** The fastest way to become an expert is to **build a fake company's data platform** — pick "MedCorp" (healthcare) or "FinTrust" (bank), create all their catalogs, users, and policies, then try to break your own security. When you can attack AND defend your own system, you're ready for any interview.

Want me to deep-dive into any specific pillar, build a complete project end-to-end, or run mock interview questions with you? 🚀
