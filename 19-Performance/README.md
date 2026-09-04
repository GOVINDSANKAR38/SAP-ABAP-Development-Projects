# 19 — Performance & Memory

## 📖 Introduction

Performance problems in ABAP usually come from three sources: inefficient database access, inefficient internal table processing, or unnecessary memory usage. This chapter covers ABAP Memory (`EXPORT`/`IMPORT`/`SUBMIT ... AND RETURN`) and consolidates performance guidance referenced throughout this guide.

## 💾 ABAP Memory — Passing Data Between Programs

ABAP Memory (`MEMORY ID`) lets you pass data between an `EXPORT`ing and `SUBMIT`ted/called program within the same session/user context — useful for passing data to a called report without database round trips.

```abap
" Standard EXPORT/IMPORT via ABAP Memory
EXPORT lt_deliveries TO MEMORY ID 'ZSM_DELIVERIES'.
IMPORT lt_deliveries FROM MEMORY ID 'ZSM_DELIVERIES'.

" Calling another report and collecting its result:
" clear the ID, run the report (which EXPORTs to it), then IMPORT.
DATA lt_result TYPE zsm_tt_export.

FREE MEMORY ID 'ZSM_EXPORT'.

SUBMIT zsm_r_export
       WITH s_vkorg  IN it_vkorg
       WITH p_from   EQ lv_date_from
       WITH p_to     EQ lv_date_to
       WITH p_export EQ abap_true
       AND RETURN.

IMPORT lt_result FROM MEMORY ID 'ZSM_EXPORT'.
IF sy-subrc <> 0.
  " nothing was exported - handle explicitly rather than using stale data
ENDIF.
```

> **Scope:** ABAP Memory (`MEMORY ID`) belongs to the **current external session (mode)** and its internal session hierarchy — it survives across `SUBMIT` and `CALL TRANSACTION` within that mode, but it is not shared with other modes or other users. Always `FREE MEMORY ID '...'` before reusing an ID, or you will silently read the previous run's data. Do not confuse it with:
> - **SAP Memory** (`SET`/`GET PARAMETER ID`) — user-wide parameter values, shared across modes;
> - **Shared memory** (shared-memory-enabled classes) — the mechanism for genuinely cross-session data.
>
> For passing data within one call stack, prefer normal method parameters. Reserve ABAP Memory for genuinely decoupled program-to-program communication.
>
> **Lifecycle:** `CLASSIC BUT STILL RELEVANT` — legitimate for `SUBMIT`-based decoupling on-premise, restricted under ABAP Cloud.



## 🗄️ Database Performance

- Avoid `SELECT *`; select only the columns you need.
- Avoid `SELECT` inside a `LOOP` ("SELECT in a loop") — replace with a single bulk `SELECT ... FOR ALL ENTRIES` or a `JOIN`.
- Use `SELECT SINGLE @abap_true` (or `COUNT(*)` only when the exact count matters) for existence checks.
- Index custom (Z) tables on the fields most frequently used in `WHERE` clauses, in coordination with the Basis/DBA team.
- Use `ST05` (SQL trace) to verify the actual number of database round trips and rows fetched.
- Read large result sets in blocks with `SELECT ... PACKAGE SIZE n ... ENDSELECT` rather than pulling millions of rows into memory at once.
- Do the aggregation and filtering **in the database**, not in ABAP. `SUM`, `COUNT`, `GROUP BY`, `CASE` and joins in ABAP SQL move the work to where the data already is; reading everything and looping is the classic mistake.



## 🖥️ Related Transaction Codes

| T-Code | Purpose |
|---|---|
| ST05 | SQL Trace |
| SAT | ABAP Runtime Analysis (profiling; successor to `SE30`) |
| ST22 | Short dump analysis (e.g. `TSV_TNEW_PAGE_ALLOC_FAILED` for memory issues) |
| ST02 | Buffer/memory statistics |
| SM12 | Display and manage lock entries |

