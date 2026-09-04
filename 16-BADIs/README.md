# 16 — BADIs (Business Add-Ins)

## 📖 Introduction

> **Lifecycle:** `CURRENT / RECOMMENDED` as an enhancement technique. BAdIs are the preferred way to extend standard SAP behaviour on-premise, and released BAdIs remain the supported extension point under ABAP Cloud. The **classic** BAdI mechanism (SE18/SE19 adapter classes) is `CLASSIC BUT STILL RELEVANT` — you will maintain plenty of it.=

A **BAdI** is SAP's object-oriented enhancement technique for injecting custom logic into standard SAP processes without modifying standard code. Unlike classic exits, BAdIs are interface-based and can support multiple, filter-dependent implementations.

## 🧩 Implementing a BAdI Interface Method

This example implements `IF_EX_ME_PROCESS_PO_CUST~PROCESS_ITEM`, a well-known Purchasing BAdI used to default and validate purchase order item data:

```abap
METHOD if_ex_me_process_po_cust~process_item.
  CONSTANTS lc_doc_type_standard   TYPE esart VALUE 'NB'.
  CONSTANTS lc_overdelivery_tol    TYPE uebto VALUE '8.0'.

  DATA lo_header   TYPE REF TO if_purchase_order_mm.
  DATA ls_header   TYPE mepoheader.
  DATA ls_item     TYPE mepoitem.
  DATA ls_previous TYPE mepoitem.

  lo_header = im_item->get_header( ).
  ls_header = lo_header->get_data( ).
  ls_item   = im_item->get_data( ).

  " On a brand-new item there is no previous version yet
  TRY.
      im_item->get_previous_data( IMPORTING ex_data = ls_previous ).
    CATCH cx_sy_itab_line_not_found.
      CLEAR ls_previous.
  ENDTRY.

  IF ls_header-bsart = lc_doc_type_standard AND ls_previous IS INITIAL.
    ls_item-uebto = lc_overdelivery_tol.
    ls_item-webre = abap_true.

    " WRITE THE DATA BACK. get_data( ) returns a COPY - changing ls_item
    " alone has no effect on the document.
    im_item->set_data( ls_item ).
  ENDIF.
ENDMETHOD.
```

=

=

## 🧠 How This BAdI Implementation Works

1. `im_item->get_header( )` retrieves the PO header object, from which header-level data (`ls_header`, e.g. `bsart` — purchasing document type) can be read.
2. `im_item->get_data( )` retrieves the current item's data.
3. `im_item->get_previous_data( )` retrieves the item's data **before** the current change — wrapped in `TRY...CATCH cx_sy_itab_line_not_found` because on a brand-new item, there is no "previous" version yet.
4. The business logic then conditionally defaults fields (`uebto` — over-delivery tolerance, `webre` — GR-based invoice verification flag) only for new items (`ls_previous IS INITIAL`) of document type `'NB'` (standard PO).

## 🧱 BAdI Concepts Cheat Sheet

Two things are often conflated here, so keep them apart. **Which mechanism** a BAdI uses (classic vs. new) and **how many implementations** it permits (single-use vs. multiple-use) are independent properties.

**Mechanism — classic vs. new:**

| Concept | Description | Lifecycle |
|---|---|---|
| **Classic BAdI** | The original mechanism (SE18/SE19), based on generated adapter classes. Called via `CL_EXITHANDLER=>GET_INSTANCE`. | `CLASSIC BUT STILL RELEVANT` — widely present in existing systems |
| **New BAdI** | Part of the Enhancement Framework, defined inside an **enhancement spot** and called with the `GET BADI` / `CALL BADI` statements. Kernel-supported, faster, and supports fallback classes. | `CURRENT / RECOMMENDED` for on-premise enhancement |

**Cardinality and filtering — applies to either mechanism:**

| Concept | Description |
|---|---|
| **Single-use** | At most one implementation may be active at a time |
| **Multiple-use** | Several implementations may be active simultaneously; the order is not guaranteed |
| **Filter-dependent** | Implementations are activated only for specific filter values (e.g. per company code or document type) |
| **Fallback class** | (New BAdIs) A default implementation used when no customer implementation is active |

**Objects involved:**

| Concept | Description |
|---|---|
| **BAdI Definition** | Declares the interface, the cardinality and any filters (SE18, or an enhancement spot in SE80) |
| **BAdI Implementation** | Your class implementing that interface (SE19) |

```abap
" Calling a NEW BAdI from your own code
DATA lo_badi TYPE REF TO zsm_badi_pricing.

TRY.
    GET BADI lo_badi
      FILTERS doc_type = ls_header-bsart.

    CALL BADI lo_badi->adjust_price
      EXPORTING is_item  = ls_item
      CHANGING  cv_price = lv_price.

  CATCH cx_badi_not_implemented.
    " no active implementation - continue with the standard price
ENDTRY.
```


## 🖥️ Related Transaction Codes

| T-Code | Purpose |
|---|---|
| SE18 | BAdI Builder — define a BAdI / display an enhancement spot |
| SE19 | BAdI Builder — create and manage a BAdI implementation |
| SE80 | Enhancement spots and enhancement implementations |
| SPRO | Customizing activation for some BAdIs |

