---
name: sa-architecture-review
description: >
  Performs a subject area architecture review for a silver-layer banking data model.
  Use this skill when asked to evaluate domain scoping, identity strategy, SCD strategy,
  cross-entity relationships, or storage choices for a subject area.
allowed-tools: shell
---

# Subject Area Architecture Review

When asked to review the architecture of a subject area, follow these steps:

1. Read `sa/subject-areas.md` to get the canonical definition (schema, priority, description) of the subject area.
2. Read all files in `guidelines/` to load normative rules.
3. Read `sa/<subject_area>/input/<sa>_physical_structures.csv` to understand the physical implementation.
4. Read `bank_logical_model.xlsx` and filter to the assessed subject area.

Then evaluate:

## Domain Scoping & BIAN Alignment
- Is the subject area correctly scoped per `sa/subject-areas.md`?
- Does it overlap or duplicate another subject area?
- Does the entity set align with BIAN v11 service domain boundaries?
- Compare against IFW subject area design (Party vs Customer vs PartyRole, Account vs Ledger, Product vs ProductOffering).

## Identity Strategy
- Every entity must have `<entity>_sk` (deterministic MD5 surrogate) and `<entity>_bk` (business key).
- Verify consistency across all entities.
- Check that surrogate keys use `MD5(UPPER(TRIM(bk_columns)))`.

## SCD Strategy
- SCD-2 is the default. SCD-1 requires documented steward approval.
- Verify mandatory SCD columns: `_is_current`, `_valid_from`, `_valid_to`.

## Cross-Entity Relationships
- Identify missing foreign keys, undocumented join keys, or broken cardinality.
- Every FK should reference a valid parent entity's SK or BK.

## ETL Mapping & Lineage
- Check source-to-target mapping completeness.
- Flag unmapped entities or attributes.

## Storage & Physical Infrastructure
- Delta format required.
- Partition keys should align with query patterns.
- Auto-optimize (auto-compaction, optimized writes) should be enabled.

Output findings as a table:

| Gap | Category | Priority | Confidence |
|---|---|---|---|
| Description | Architecture / Identity / SCD / Storage / Lineage | P0/P1/P2 | 0.0–1.0 |
