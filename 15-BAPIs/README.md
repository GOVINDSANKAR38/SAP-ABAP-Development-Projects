# 15 — BAPIs

## 📖 Introduction

A **BAPI** (Business Application Programming Interface) is a standardized, RFC-enabled function module that provides a stable API for a SAP Business Object (e.g., creating a sales order, updating a material). Unlike direct table updates, BAPIs enforce business logic, validations, and consistency checks — always prefer them over direct table manipulation when one is available.

## 📨 The `BAPIRET2` Return Structure

Every well-behaved BAPI returns messages in a table of type `BAPIRET2`, which you should always check before committing.

```abap
" BAPI Return Message
DATA(lt_return) = VALUE bapiret2_t( ).
```

| Field | Meaning |
|---|---|
| `type` | Message type: `S` (success), `I` (info), `W` (warning), `E` (error), `A` (abort), `X` (exception) |
| `id` | Message class |
| `number` | Message number |
| `message` | Fully formatted message text |

## ✅ Commit & Rollback

```abap
DATA(lv_has_error) = xsdbool(    line_exists( lt_return[ type = 'E' ] )
                              OR line_exists( lt_return[ type = 'A' ] )
                              OR line_exists( lt_return[ type = 'X' ] ) ).

IF lv_has_error = abap_false.
  CALL FUNCTION 'BAPI_TRANSACTION_COMMIT'
    EXPORTING wait = abap_true.       " see the note on WAIT below
ELSE.
  CALL FUNCTION 'BAPI_TRANSACTION_ROLLBACK'.
ENDIF.
```


## 🧭 Typical BAPI Call Pattern

```abap
CALL FUNCTION 'BAPI_SALESORDER_CREATEFROMDAT2'
  EXPORTING  order_header_in = ls_header
  IMPORTING  salesdocument   = lv_vbeln
  TABLES     order_items_in  = lt_items
             return          = lt_return.

DATA(lv_has_error) = xsdbool(    line_exists( lt_return[ type = 'E' ] )
                              OR line_exists( lt_return[ type = 'A' ] )
                              OR line_exists( lt_return[ type = 'X' ] ) ).

IF lv_has_error = abap_false.
  CALL FUNCTION 'BAPI_TRANSACTION_COMMIT'
    EXPORTING wait = abap_true.
ELSE.
  CALL FUNCTION 'BAPI_TRANSACTION_ROLLBACK'.
ENDIF.
```


