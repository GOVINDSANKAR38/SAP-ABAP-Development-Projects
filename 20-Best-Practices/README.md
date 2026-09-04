# 20 — Best Practices & Clean ABAP

## 📖 Introduction

This closing chapter distills recurring best practices, naming conventions, and clean-code guidance referenced throughout the previous 19 chapters — a quick checklist to review before submitting code for review/transport.

## 🏷️ Naming Conventions (Classic Enterprise ABAP)

> 📝 **On prefixes.** This guide uses the classic prefix notation (`lv_`, `lt_`, `ls_`, `lo_`, …) because it is near-universal in existing SAP code and you need to read it fluently. Modern style guides — including SAP's own Clean ABAP — recommend *against* encoding the type in the name, preferring semantic names such as `order_items` over `lt_order_items`. Both positions are defensible; what matters is that a codebase is consistent. Know the convention your team uses, and know that these two schools exist.

| Prefix | Meaning | Example |
|---|---|---|
| `lv_` | Local variable | `lv_count` |
| `gv_` | Global variable | `gv_flag` |
| `ls_` | Local structure | `ls_data` |
| `gs_` | Global structure | `gs_layout` |
| `lt_` / `gt_` | Local/global internal table | `lt_mara` |
| `lr_` | Range table | `lr_matnr` |
| `lo_` / `go_` | Local/global object reference | `lo_alv` |
| `is_` / `it_` / `iv_` | Importing structure/table/value parameter | `iv_matnr` |
| `es_` / `et_` / `ev_` | Exporting structure/table/value parameter | `et_return` |
| `cs_` / `ct_` / `cv_` | Changing structure/table/value parameter | `ct_data` |
| `rs_` / `rt_` / `rv_` | Returning structure/table/value | `rv_result` |
| `lc_` / `gc_` | Constant | `lc_number` |
| `<fs_...>` | Field symbol | `<fs_data>` |

## ✅ Code Review Checklist

**Correctness & data access**
- [ ] No hardcoded literals for business-relevant values — use constants or Customizing tables.
- [ ] `SELECT` statements avoid `SELECT *`; only needed fields are read.
- [ ] `SELECT SINGLE` is qualified by the full key.
- [ ] No `SELECT` inside a `LOOP`.
- [ ] `FOR ALL ENTRIES` driver tables are checked for `IS NOT INITIAL` and de-duplicated; the implicit `DISTINCT` has been considered.
- [ ] Restrictions on the optional side of an outer join are in the `ON` clause, not `WHERE`.
- [ ] No explicit client (`mandt`) predicates.

**Transactions & safety**
- [ ] The transaction boundary is owned by the top-level caller; reusable units do not commit.
- [ ] `AND WAIT` is used only where synchronous completion is genuinely required.
- [ ] BAPI return tables are fully checked (`E`, `A`, `X`) before `BAPI_TRANSACTION_COMMIT`.
- [ ] No direct DML against SAP standard application tables.
- [ ] Mass `UPDATE`/`DELETE` statements are key-qualified; `sy-dbcnt` is checked.
- [ ] Locking is considered wherever concurrent changes are possible.

**Security**
- [ ] Dynamic table/field names come from validated sources.
- [ ] Generic table access is authorization-checked, not just existence-checked.
- [ ] `CALL TRANSACTION` states its authorization intent explicitly.
- [ ] RFC entry points authorization-check the caller.

**Structure & error handling**
- [ ] Methods have a single, clear responsibility (no "god methods").
- [ ] Exceptions are caught **specifically**, ordered most-specific first; any `cx_root` catch is at a boundary and logs or re-raises.
- [ ] `CHECK` is not used where `IF` is meant.
- [ ] Field symbols and references are checked with `IS ASSIGNED`/`IS BOUND`, and `sy-subrc` is checked after `ASSIGN`.
- [ ] Table expressions are guarded (`line_exists`, `OPTIONAL`, `TRY`, or `ASSIGN`).
- [ ] No macros for logic beyond trivial repetition — prefer methods.
- [ ] Comments explain **why**, not **what** (the code already shows "what").
- [ ] Classic technologies used deliberately, not by habit — see [21-Classic-vs-Modern-ABAP](../21-Classic-vs-Modern-ABAP/README.md).

## 🧹 General Code Style

- Prefer **modern ABAP** (`VALUE`, `COND`, `SWITCH`, `REDUCE`, `FOR`, inline `DATA(...)`) over classical statements (`MOVE`, `CONCATENATE`, `APPEND ... TO` in a loop) when it improves readability — but understand classical syntax too, since most production systems still contain plenty of it.
- Keep global state to a minimum; prefer method parameters and class attributes over global `DATA` declarations.
- Structure programs with a clear separation of concerns: **selection screen** → **data retrieval** → **business logic/transformation** → **presentation** (ALV/output).
- Favor small, named, single-purpose methods over long procedural blocks — improves testability and readability.

