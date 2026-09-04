# 03 — Variables

## 📖 Introduction

This chapter is a reference for the most common variable, constant, and object declarations you will write in almost every ABAP program — from simple scalar variables to class instances, internal tables, and pointers.

## 🧮 Declaring Variables

```abap
" BAPI Return Message
DATA(lt_return) = VALUE bapiret2_t( ).

" Boolean
DATA(rv_result) = xsdbool( sy-subrc = 0 ).

" Constant (TYPE, not LIKE, when referring to a Dictionary field)
CONSTANTS lc_notif_no TYPE bapi2080_nothdre-notif_no VALUE '000000000001'.

" Calling a static method with a returning value.
" In an operand position (right-hand side of an assignment) you may only
" supply INPUT parameters - see the note below.
DATA(lv_surname) = zsm_cl_test=>get_surname( iv_name = 'USER01' ).

" Clear
CLEAR lv_surname.

" Classic (explicit type) declarations
DATA lv_character  TYPE c LENGTH 120          VALUE 'S'.
DATA lv_numeric_id TYPE n LENGTH 10           VALUE '0000001907'.
DATA lv_integer    TYPE i                     VALUE 1994.
DATA lv_mimetype   TYPE w3conttype            VALUE 'application/pdf'.
DATA lv_number     TYPE p LENGTH 8 DECIMALS 2 VALUE '17.75'.
DATA lv_string     TYPE string                VALUE 'Example text'.

DATA(lt_returns) = VALUE bapiret2_tab( ).
DATA(lt_data)    = VALUE zsm_tt_value( ( ls_data ) ).
```

> ⚠️ **A functional method call in an operand position takes input parameters only.** `DATA(x) = cl=>meth( ... )` cannot carry `IMPORTING` or `CHANGING` parameters — for those, use a standalone call:
> ```abap
> zsm_cl_test=>get_surname( EXPORTING iv_name    = 'USER01'
>                           IMPORTING ev_surname = DATA(lv_result) ).
> ```
> Also note that a literal such as `'X'` can never be bound to a `CHANGING` parameter — a changing parameter is written back to, so it needs a variable.

> 💡 **`TYPE` rather than `LIKE`** when referring to a Dictionary field. `LIKE` referring to Dictionary objects is an obsolete form; `LIKE` referring to another *data object* in the same program is still valid.

> ⚠️ **Every variable may be declared once per scope.** Each block in this guide is meant to be independently pasteable, so watch for names you have already declared when combining snippets.

## 🧵 `FORM` / `PERFORM` (Classical Subroutines)


`FORM`/`PERFORM` with `TABLES`/`USING` parameters is still found in many long-lived programs:

```abap
DATA lt_header    LIKE TABLE OF bapi_order_header1    WITH HEADER LINE.
DATA lt_operation LIKE TABLE OF bapi_order_operation1 WITH HEADER LINE.
DATA lt_component LIKE TABLE OF bapi_order_component  WITH HEADER LINE.
DATA lv_order_qty TYPE int4.

PERFORM get_component TABLES lt_header
                             lt_operation
                             lt_component.
PERFORM use_data USING lv_order_qty.

FORM get_component TABLES pt_header    STRUCTURE bapi_order_header1
                          pt_operation STRUCTURE bapi_order_operation1
                          pt_component STRUCTURE bapi_order_component.
ENDFORM.

" Always TYPE a USING parameter - an untyped one accepts anything and
" defers every error to runtime.
FORM use_data USING pv_data TYPE int4.
ENDFORM.
```

## 📞 Calling Methods, Function Modules & Includes

```abap
" Calling a STATIC METHOD that returns a value.
" Note: CALL FUNCTION is for function modules only - a class method is never
" called with CALL FUNCTION.
DATA(lo_sender) = cl_cam_address_bcs=>create_internet_address(
                      i_address_string = CONV #( gv_sender_email ) ).

" Calling a FUNCTION MODULE
CALL FUNCTION 'CONVERSION_EXIT_ALPHA_INPUT'
  EXPORTING input  = lv_external
  IMPORTING output = lv_internal.

" Field Symbol
LOOP AT lt_data ASSIGNING FIELD-SYMBOL(<ls_data>).
ENDLOOP.

" Include
INCLUDE zsm_test_top.
INCLUDE zsm_test_frm.
```

> ⚠️ `CALL FUNCTION` expects a **function module name** (a character value, usually a literal or a variable). Writing `CALL FUNCTION cl_some_class=>some_method` is a syntax error. Verify a class method's real signature in SE24 before calling it 

## ✍️ Output & Pointers

```abap
START-OF-SELECTION.
  " Optional value (returns the initial value instead of raising if not found)
  DATA(lv_key) = VALUE #( lt_data[ name = 'Key' ]-value OPTIONAL ).

  " Output
  WRITE 'Hello'.
  WRITE / 'Hello'.
  WRITE: 'Hello', 'World'.

  " Quantity with its unit of measure
  DATA lv_qty_out TYPE c LENGTH 20.
  WRITE lv_kwmeng TO lv_qty_out UNIT lv_vrkme.

  " Pointer (data reference)
  DATA(lv_value) = '12345'.
  DATA(lr_ref)   = REF #( lv_value ).

  WRITE lr_ref->*.
```

## 📊 Declaration Styles at a Glance

| Style | Example | When to Use |
|---|---|---|
| Explicit `DATA` with `TYPE` | `DATA lv_x TYPE i.` | When the type must be visible/explicit, or declared before first use (top of routine) |
| Inline declaration | `DATA(lv_x) = 5.` | Modern ABAP (7.40 generation onward), when the type can be inferred; keeps declarations close to usage |
| `CONSTANTS` | `CONSTANTS lc_x TYPE i VALUE 5.` | Fixed values that never change during runtime |
| `FIELD-SYMBOL(<fs>)` | inline in `ASSIGN`/`LOOP` | Accessing data without copying it (performance) |

> ⚠️ **An inline declaration cannot carry a `TYPE` addition.** `DATA(lv_x) TYPE i.` is a syntax error — the whole point of `DATA(...)` is that the type comes from the assignment. If you need to state the type, use the classic form: `DATA lv_x TYPE i.`

