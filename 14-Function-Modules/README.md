# 14 — Function Modules & Batch Input (BDC)

> **Lifecycle:** `CLASSIC BUT STILL RELEVANT`. BDC remains the practical fallback for mass loads into transactions that expose no API, and you will meet it in almost every long-lived SAP landscape. It is a last resort, not a first choice, and it is **not** part of the ABAP Cloud development model — see [21-Classic-vs-Modern-ABAP](../21-Classic-vs-Modern-ABAP/README.md).

## 📖 Introduction

This chapter focuses on **Batch Data Communication (BDC/Batch Input)** — simulating user input into a classic dynpro transaction programmatically. It's still widely used for mass data loads into transactions that don't have a BAPI. (General function module usage/calls are covered in [09-Modularization](../09-Modularization/README.md).)

## 📥 Batch Input — Simulating Screen Input

```abap
" TOP
DATA gt_bdctable TYPE TABLE OF bdcdata WITH EMPTY KEY.
DATA gt_messtab  TYPE TABLE OF bdcmsgcoll WITH EMPTY KEY.

" FORM
CLEAR gt_messtab.

PERFORM bdc_append
  USING 'SAPLMR1M'
        '6150'
        ''
        ''.
PERFORM bdc_append
  USING ''
        ''
        'BDC_OKCODE'
        '/00'.
PERFORM bdc_append
  USING ''
        ''
        'RBKP-BELNR'
        lv_belnr.
PERFORM bdc_append
  USING ''
        ''
        'RBKP-GJAHR'
        lv_gjahr.

PERFORM bdc_append
  USING 'SAPLMR1M'
        '6000'
        ''
        ''.
PERFORM bdc_append
  USING ''
        ''
        'BDC_OKCODE'
        '/EPPCH'.

" Make the authorization decision EXPLICIT.
" WITH AUTHORITY-CHECK raises CX_SY_AUTHORIZATION_ERROR when the user is not
" authorized for the called transaction.
TRY.
    CALL TRANSACTION 'MIR4' WITH AUTHORITY-CHECK
                            USING        gt_bdctable
                            UPDATE       'S'
                            MODE         'E'
                            MESSAGES INTO gt_messtab.

  CATCH cx_sy_authorization_error INTO DATA(lx_auth).
    MESSAGE lx_auth->get_text( ) TYPE 'E'.
ENDTRY.

" PERFORM
FORM bdc_append
  USING iv_program    TYPE bdcdata-program
        iv_dynpro     TYPE bdcdata-dynpro
        iv_fieldname  TYPE bdcdata-fnam
        iv_fieldvalue TYPE bdcdata-fval.

  DATA(ls_bdctable) = VALUE bdcdata(
      program  = COND #( WHEN iv_fieldname IS INITIAL THEN iv_program ELSE '' )
      dynpro   = COND #( WHEN iv_fieldname IS INITIAL THEN iv_dynpro  ELSE '' )
      dynbegin = COND #( WHEN iv_fieldname IS INITIAL THEN 'X'        ELSE '' )
      fnam     = iv_fieldname
      fval     = iv_fieldvalue ).

  APPEND ls_bdctable TO gt_bdctable.
ENDFORM.
```


### 📋 BDC Structure Reference

| Field | Purpose |
|---|---|
| `program` / `dynpro` | Identifies the screen (program name + screen number) — set only on the **first** entry of a screen block |
| `dynbegin` | `'X'` marks the start of a new screen block |
| `fnam` / `fval` | Field name and the value to enter into it — used for all subsequent entries within that screen block |

### 🕹️ `CALL TRANSACTION` Modes

| Mode | Behavior |
|---|---|
| `A` | All screens shown (foreground, for debugging BDC scripts) |
| `E` | Show screens only if an error occurs (semi-background) |
| `N` | No screens shown (fully background/silent) |

