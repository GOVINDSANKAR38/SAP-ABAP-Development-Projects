# 05 — Control Statements

## 📖 Introduction

Control statements decide which branch of logic gets executed. This chapter covers classical `IF`/`CASE` alongside the modern **functional operators** `COND` and `SWITCH`, which let you assign a value conditionally in a single expression.

## 🔀 CASE

```abap
CASE ls_order-order_type.
  WHEN 'ZSTD'.
  WHEN 'ZRET'.
  WHEN OTHERS.
ENDCASE.
```

## 🚦 CHECK

`CHECK <cond>` **leaves the entire current processing block** when the condition is **false**. Inside a `LOOP` it skips to the next iteration; inside a method, `FORM` or event block it exits that block entirely.

```abap
" Inside a loop: skip rows that already have an error
LOOP AT lt_items INTO DATA(ls_item).
  CHECK NOT line_exists( lt_return[ item = ls_item-posnr type = 'E' ] ).

  process_item( ls_item ).
ENDLOOP.
```

> ⚠️ **`CHECK` is not a general-purpose guard, and its polarity catches people out.** Because it exits when the condition is *false*, a statement such as `CHECK sy-subrc <> 0.` continues only on the **error** path and abandons the block on success — usually the exact opposite of the author's intent. Use `IF` when you want to branch, and keep `CHECK` for genuinely skipping the current loop pass.

## 🌿 IF / ELSE and the `COND` Operator

Classic `IF`/`ELSE` still has its place, but for **assigning a value** based on a condition, the functional `COND #( )` operator is more concise and avoids intermediate variables:

```abap
DATA lv_begin_date TYPE d VALUE '20200505'.
DATA lv_end_date   TYPE d VALUE '20200515'.

" State the result type explicitly when the branches are literals
DATA(lv_status) = COND char10( WHEN sy-datum < lv_begin_date THEN 'EARLY'
                               WHEN sy-datum > lv_end_date   THEN 'LATE'
                               ELSE                               'OK' ).

" '#' is fine when the type is clearly derivable from the operands
DATA(lv_flag) = COND #( WHEN lv_warehouse = lc_warehouse
                        THEN lc_indicator
                        ELSE space ).

DATA(lv_in_range) = COND abap_bool( WHEN lv_confirmation_no BETWEEN 50 AND 100
                                    THEN abap_true
                                    ELSE abap_false ).

DATA(lv_plant_code) = COND char3( WHEN is_defect->plant = '1000' THEN 'ONE'
                                  WHEN is_defect->plant = '1100' THEN 'TWO'
                                  WHEN is_defect->plant = '1200' THEN 'THR'
                                  ELSE                                '' ).
```

> ⚠️ **Know what `#` resolves to.** When the operand type cannot be derived from the position — as in an inline declaration — the ABAP documentation states that the type is taken from the operand after the **first `THEN`**. So `DATA(lv_status) = COND #( ... THEN 'EARLY' ... ELSE 'OK' )` compiles, but `lv_status` becomes `c LENGTH 5`, and a longer branch added later would be silently truncated. Write the type explicitly whenever the branches are literals.

## 🔁 SWITCH

`SWITCH` is the functional equivalent of `CASE`, useful for assigning a value based on a single variable's content:

```abap
DATA(lv_status) = SWITCH char10( sy-msgty
                                 WHEN 'S' THEN 'SUCCESS'
                                 WHEN 'W' THEN 'OK'
                                 WHEN 'E' THEN 'ERROR'
                                 ELSE          'UNKNOWN' ).
```

## 📊 COND vs. SWITCH vs. IF/CASE

| Construct | Best For | Returns a Value? |
|---|---|---|
| `IF` / `ELSEIF` / `ELSE` | Multiple, unrelated conditions; executing statements | ❌ No |
| `CASE` | Branching on **one** variable's exact value; executing statements | ❌ No |
| `COND #( )` | Assigning a value based on one or more conditions | ✅ Yes |
| `SWITCH #( )` | Assigning a value based on **one** variable's value | ✅ Yes |


