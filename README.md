# Cosmos Senior Leadership Posting Pipelines

This repository contains the technical handoff for two related Snowflake/Snowpark workflows:

1. **Cosmos Dedup** — deduplicates raw Cosmos job-posting records into canonical postings using parent-company/title candidate grouping, date guardrails, job-description similarity, lineage, and QA.
2. **Cosmos → Leader Record Linkup (V18)** — links deduplicated senior-leadership Cosmos postings to the most likely matching Leader Record work-experience record using employer RCID, title evidence, date windows, role safeguards, and deterministic candidate ranking.

The repository is intended as a **Data Engineering handoff / reference implementation**. Productionization decisions and validation steps should be completed in the target Snowflake environment before deployment.

---

## Repository Structure

```text
.
├── README.md
│
├── cosmos_and_leader_record_linkup/
│   ├── consumption_filter_reuse.sql
│   ├── Cosmos_Leader_Linkage_Final.ipynb
│   └── Cosmos_Leader_Record_Final.docx
│
└── cosmos_dedup/
    ├── Final_version_Comos_dedup_notebook.ipynb
    └── Final_version_Cosmos_dedup_doc.docx
```

---

# 1. Cosmos → Leader Record Linkup

## Purpose

The V18 matching pipeline links deduplicated senior-leadership postings from Cosmos to the most likely matching role in Leader Record.

The pipeline:

- starts from one canonical Cosmos posting per `CANONICAL_JOB_ID`;
- matches employers using numeric `Cosmos.RCID = Leader Record.REVELIO_RCID`;
- normalizes and compares raw and translated posting titles with `SOURCE_ROLE_TITLE`;
- applies seniority, C-suite, support-role, lexical, regional, and title-context safeguards;
- evaluates role-start timing around the posting month;
- assigns the first qualifying V18 rule (`R1`–`R8`);
- ranks eligible people using rule, match tier, title similarity, country match, date distance, and `PERSON_ID`;
- retains unmatched postings as `NO_CANDIDATE`;
- preserves candidate counts, match status, review flags, and matching evidence for QA.

### Main Snowflake inputs

```text
PC_FIVETRAN_DB.DA_WORKSPACE.V_COSMOS_SENIOR_LEADERSHIP_CONSUMPTION
PC_FIVETRAN_DB.DA_WORKSPACE.LEADER_RECORD
```

### Main output

```text
PC_FIVETRAN_DB.DA_WORKSPACE.LNK_V18_T3_FINAL
```

### Files

**`Cosmos_Leader_Linkage_Final.ipynb`**  
Executable Snowflake notebook containing the V18 matching logic, UDFs, title normalization, candidate creation, ranking, validation assertions, and diagnostic queries.

**`Cosmos_Leader_Record_Final.docx`**  
Technical design and notebook guide covering the matching rules, title tiers, date logic, January 1 handling, tie-breakers, validation logic, and cell-by-cell implementation.

**`consumption_filter_reuse.sql`**  
Read-only reproduction of the senior-leadership consumption filter. It applies exclusion rules and retains one eligible row per `CANONICAL_JOB_ID`.

> **Reuse note:** the SQL file contains an older comment referring to `REUSE_GUIDE.md`. For this repository, the reuse guidance is consolidated in this README; a separate reuse guide is not required.

## Consumption Filter Reuse

The SQL assumes these existing canonical senior-leadership inputs:

```text
PC_FIVETRAN_DB.DA_WORKSPACE.COSMOS_C_SUITE_CLEAN_CANONICAL_OUTPUT
PC_FIVETRAN_DB.DA_WORKSPACE.COSMOS_SVP_PRESIDENT_CLEAN_CANONICAL_OUTPUT
```

When porting the SQL, preserve the expected contracts for:

- `CANONICAL_JOB_ID`
- `TITLE_RAW`
- `TITLE_TRANSLATED`
- match-source fields
- `ROLE_TIER`
- employer identifiers

The SQL reproduces the **final selection/filter layer only**. It does **not** rebuild the upstream raw-posting canonicalization process.

---

# 2. Cosmos Dedup

## Purpose

The dedup pipeline creates canonical job-posting records from the raw Cosmos/Revelio posting source.

### Source

```text
H_EDW.HS_REVELIO_EDW_V.V_POSTINGS_UNIFIED_INDIVIDUAL
```

The supplied notebook is configured for the Q1 2025 reference scope:

```text
POST_DATE >= '2025-01-01'
POST_DATE <  '2025-04-01'
```

## Core Logic

The pipeline:

1. creates a candidate key using `ULTIMATE_PARENT_RCID` plus the best available title;
2. cleans job descriptions and creates exact-description hashes;
3. builds rolling candidate clusters using a 10-day adjacent posting gap;
4. scores job-description similarity inside candidate contexts;
5. processes one- and two-row clusters in SQL;
6. processes 3+ row clusters using Snowpark/Python active-group logic;
7. applies a 10-day active-group edge and a 30-day maximum merge-group span;
8. auto-merges exact JD matches or token-containment matches at `>= 0.90`;
9. routes borderline similarity (`>= 0.75` and `< 0.90`) to review;
10. creates a lineage-preserving canonical output and QA/review tables.

A **candidate cluster is only a search boundary**. It is not the same as the final merge group. One candidate cluster can produce multiple final merge groups.

### Files

**`Final_version_Comos_dedup_notebook.ipynb`**  
Executable Snowflake/Snowpark notebook containing the dedup implementation, active-group assignment, canonical output creation, QA, lineage, review samples, and optional translation logic.

**`Final_version_Cosmos_dedup_doc.docx`**  
Architecture and handoff document explaining parameters, candidate clustering, pair scoring, active-group processing, canonicalization, QA, edge cases, and production considerations.

---

# Relationship Between the Two Workflows

The two folders are related but should not be interpreted as a single one-click pipeline.

```text
Raw Cosmos postings
        │
        ▼
Posting deduplication / canonicalization
        │
        ▼
Senior-leadership selection and consumption layer
        │
        ▼
V_COSMOS_SENIOR_LEADERSHIP_CONSUMPTION
        │
        ▼
Cosmos → Leader Record V18 matching
        │
        ▼
LNK_V18_T3_FINAL
```

The current V18 notebook reads from the existing:

```text
V_COSMOS_SENIOR_LEADERSHIP_CONSUMPTION
```

The Q1 Cosmos dedup notebook is a reference implementation of the posting-dedup architecture. DE should explicitly define how a production dedup output feeds the senior-leadership canonical/consumption layer rather than assuming the two notebooks are directly connected as-is.

---

# Execution and Validation

## Cosmos Dedup

Run the notebook from top to bottom in Snowflake/Snowpark.

Before productionization, validate:

- treatment of rows with null `ULTIMATE_PARENT_RCID`;
- handling of postings with blank/empty cleaned job descriptions;
- date scope and refresh/hold-window design;
- large-cluster performance and Snowflake cost;
- pair-scoring limits and skipped-diagnostic volume;
- review-bucket movement;
- false-positive / false-negative merge behavior;
- row-count reconciliation and lineage completeness.

The current notebook uses execution-stability limits for large pair-scoring contexts. These are engineering guardrails rather than business deduplication rules.

Optional JD translation is disabled by default and should only be enabled after business/cost approval.

## Cosmos → Leader Record V18

Run the notebook from top to bottom in the target Snowflake environment.

The automated validation/assertion cell is the main execution gate. Confirm that all integrity checks pass before promotion.

Review diagnostics for:

- `NO_CANDIDATE`;
- `SINGLE_CANDIDATE`;
- `BEST_OF_MULTIPLE`;
- `TIED_MULTIPLE`;
- word-containment matches;
- pre-posting role starts;
- January 1 role-start cases;
- executive-office / support-role conflicts;
- unexpected employer aliases;
- tied top candidates.

The linkage pipeline identifies the **most plausible work-experience match** based on available employer, title, and timing evidence. It should not be interpreted as independent proof that the selected person was hired into the posting.

---

# Recommended Handoff Sequence

For Data Engineering review:

1. Read `cosmos_dedup/Final_version_Cosmos_dedup_doc.docx`.
2. Review and execute `cosmos_dedup/Final_version_Comos_dedup_notebook.ipynb`.
3. Review `cosmos_and_leader_record_linkup/consumption_filter_reuse.sql` and confirm the senior-leadership input contracts.
4. Read `cosmos_and_leader_record_linkup/Cosmos_Leader_Record_Final.docx`.
5. Execute `cosmos_and_leader_record_linkup/Cosmos_Leader_Linkage_Final.ipynb`.
6. Confirm the V18 automated assertions and diagnostic summaries.
7. Define production orchestration, scheduling, object ownership, monitoring, and the interface between canonical Cosmos output and the senior-leadership consumption layer.

---

# Handoff Status

These materials provide:

- working reference notebooks;
- reusable senior-leadership filtering logic;
- technical design documentation;
- QA and validation logic;
- lineage and review outputs;
- known production considerations.

They should be treated as a **reference implementation ready for DE review and productionization**, rather than as a fully deployed production pipeline.
