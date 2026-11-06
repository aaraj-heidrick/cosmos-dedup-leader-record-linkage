# Reuse Guide

This guide explains how the supplied deduplication, senior-leadership filtering, and Leader Record linkage artifacts fit together.

## A. Reusing the existing senior-leadership population

If the target environment can access:

```text
PC_FIVETRAN_DB.DA_WORKSPACE.V_COSMOS_SENIOR_LEADERSHIP_CONSUMPTION
```

use that view directly as the Cosmos input to the V18 linkage notebook.

The V18 notebook expects at minimum the posting fields used by its code, including `CANONICAL_JOB_ID`, titles, company/employer identifiers, RCID, posting dates, country fields, role-tier fields, and job-description context.

## B. Reusing `consumption_filter_reuse.sql`

`sql/consumption_filter_reuse.sql` is a read-only reproduction of the final selection/filter layer. It expects two upstream canonical inputs:

```text
PC_FIVETRAN_DB.DA_WORKSPACE.COSMOS_C_SUITE_CLEAN_CANONICAL_OUTPUT
PC_FIVETRAN_DB.DA_WORKSPACE.COSMOS_SVP_PRESIDENT_CLEAN_CANONICAL_OUTPUT
```

When porting it, replace those two input references and preserve the required contracts, especially:

- `CANONICAL_JOB_ID`
- `ROLE_TIER`
- raw and translated title fields
- the match-source fields used to identify how the title was selected
- employer / RCID fields required downstream

The SQL applies exclusion rules, retains `KEEP_FLAG = TRUE`, and selects one survivor per `CANONICAL_JOB_ID` by role tier.

It does **not** recreate the upstream raw-posting deduplication/canonicalization process.

## C. Reusing the Q1 dedup notebook

`notebooks/cosmos_dedup_q1_2025.ipynb` is the most complete supplied raw-posting dedup reference implementation. Its current parameters are Q1 2025 and should be treated as configuration rather than permanent production scope.

Before reusing it for another period or population:

1. Change the date parameters and output naming.
2. Confirm the source schema still supplies the referenced fields.
3. Decide how null `ULTIMATE_PARENT_RCID` should behave.
4. Confirm whether records with blank/empty cleaned job descriptions should be outside the dedup path or preserved separately.
5. Review the 10-day / 30-day date guardrails and 0.90 / 0.75 containment thresholds with the business owner.
6. Decide whether the 150-row / 5,000-pair diagnostic caps are appropriate for the target warehouse.
7. Keep Cortex translation disabled unless there is explicit approval to enable it.
8. Run all row-count QA, lineage, review-bucket, and spot-check outputs.

## D. Connecting dedup to senior-leadership consumption

The supplied artifacts document an integration boundary rather than a single one-click chain.

The dedup notebook produces a general canonical posting output. The senior-leadership consumption SQL expects already prepared canonical C-suite and qualified VP/President inputs. DE therefore needs to define one of these approaches:

- preserve the existing upstream senior-role canonical tables and use the dedup notebook only as reference logic; or
- adapt the dedup output into equivalent C-suite and VP/President input contracts before applying `consumption_filter_reuse.sql`.

Do not silently assume the Q1 canonical table is schema-compatible with the two consumption inputs.

## E. Reusing V18 linkage

The V18 notebook is designed to run after the senior-leadership consumption input exists.

Primary source contracts:

```text
Cosmos: PC_FIVETRAN_DB.DA_WORKSPACE.V_COSMOS_SENIOR_LEADERSHIP_CONSUMPTION
Leader: PC_FIVETRAN_DB.DA_WORKSPACE.LEADER_RECORD
```

Primary output:

```text
PC_FIVETRAN_DB.DA_WORKSPACE.LNK_V18_T3_FINAL
```

Run the notebook from top to bottom in one Snowflake notebook/session unless DE intentionally refactors session variables, helper objects, and UDF creation into an orchestrated pipeline.

The V18 assertion cell must pass before promotion. Diagnostic samples should also be reviewed for January 1 dates, word matches, ties, executive-office/support contexts, and unexpected employer aliases.
