# 02 — Data Types

## 📖 Introduction

ABAP is a strongly typed language. Before you can store a value, you must declare its **type** — either an elementary type (`c`, `n`, `i`, `p`, `string`, ...) or a structured type (`TYPES ... BEGIN OF`). This chapter covers structures and common string-cleanup functions used together with data declarations.

## 🧱 Elementary Data Types Cheat Sheet

| Type | Description | Example |
|---|---|---|
| `c` | Fixed-length character | `DATA lv_char TYPE c LENGTH 10.` |
| `n` | Numeric text (digits only, leading zeros) | `DATA lv_num TYPE n LENGTH 4.` |
| `i` / `int4` | Integer | `DATA lv_int TYPE i.` |
| `p` | Packed decimal | `DATA lv_amt TYPE p LENGTH 8 DECIMALS 2.` |
| `string` | Variable-length character string | `DATA lv_str TYPE string.` |
| `d` / `datum` | Date (`YYYYMMDD`) | `DATA lv_date TYPE d.` |
| `t` / `tims` | Time (`HHMMSS`) | `DATA lv_time TYPE t.` |
| `xstring` | Variable-length byte string (binary) | `DATA lv_bin TYPE xstring.` |

## 🧩 Structures

A **structure** groups related fields together, similar to a `struct` in C or a record in other languages.

```abap
TYPES: BEGIN OF ty_viqmel,
         notif_no    TYPE char10,
         notif_type  TYPE char2,
         description TYPE char40,
       END OF ty_viqmel.

DATA ls_viqmel TYPE ty_viqmel.
```


## 🧹 Cleaning Up String Data

Two very common statements when working with character data read from the database or user input:

```abap
" Condense: Delete Space
CONDENSE lv_data.

" Shift: Delete Beginning Zeros
SHIFT lv_data LEFT DELETING LEADING '0'.
```

| Statement | Purpose |
|---|---|
| `CONDENSE` | Removes leading blanks and collapses each run of internal blanks to a single blank. With `NO-GAPS`, removes **all** blanks |
| `SHIFT ... LEFT DELETING LEADING '0'` | Strips leading zeros — useful before comparing numeric-looking strings |

## 🔁 Type Conversions

ABAP performs a lot of *implicit* conversions, but it's important to know how to convert **explicitly**, especially between internal keys and their "human readable" (ALPHA) form.

```abap
" ALPHA IN: Add leading zeros (internal format)
DATA lv_vbeln TYPE char10.
lv_vbeln = |{ is_data-vbeln ALPHA = IN }|.

" ALPHA OUT: Remove leading zeros (external/display format)
lv_vbeln = |{ is_data-vbeln ALPHA = OUT }|.

" Explicit type conversion with CONV
DATA(lv_data)  = CONV int4( ls_data-value ).
DATA(ls_data)  = CORRESPONDING zsm_t_data( ls_xdata ).
```

Conversion exits (`CONVERSION_EXIT_*`) are the classical, function-module–based way of doing the same thing and are still widely used with material numbers, dates, and other domain-specific fields — see [09-Modularization](../09-Modularization/README.md#-conversion-exits).

