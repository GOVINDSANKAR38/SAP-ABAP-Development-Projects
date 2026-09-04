# 10 — Objects & OOP

## 📖 Introduction

Object-oriented ABAP (`CLASS`/`METHODS`) is the recommended approach for all new development. This chapter covers class definition basics (visibility sections, static vs. instance members), inheritance, and working with classic OLE/legacy objects (`ole2_object`) as well as OData model objects.

## 🧱 Defining and Using a Class

```abap
" GLOBAL CLASS
DATA lv_sum    TYPE int4.
DATA lv_result TYPE int4.

START-OF-SELECTION.
  DATA(lo_class) = NEW zsm_cl_test( ).

  " Instance method with an IMPORTING (returned) parameter
  lo_class->sum_two_numbers( EXPORTING iv_first_number  = 10
                                       iv_second_number = 20
                             IMPORTING ev_sum           = lv_sum ).

  " Static method
  zsm_cl_test=>multiply_two_numbers( EXPORTING iv_first_number  = 10
                                               iv_second_number = 20
                                     IMPORTING ev_result        = lv_result ).

  " A method with a RETURNING parameter can be used directly in an expression,
  " and its result can be captured with an inline declaration
  DATA(lv_product) = zsm_cl_test=>multiply( iv_first_number  = 10
                                            iv_second_number = 20 ).
```


## 🔐 Visibility Sections (Encapsulation)

```abap
CLASS lcl_class DEFINITION.
  PUBLIC SECTION.
    DATA lv_public TYPE i.

    METHODS data_declaration.

  PROTECTED SECTION.
    DATA lv_protected TYPE i.

  PRIVATE SECTION.
    DATA lv_private TYPE i.
ENDCLASS.


CLASS lcl_class IMPLEMENTATION.
  METHOD data_declaration.
    lv_public = 1.
    lv_protected = 2.
    lv_private = 3.
  ENDMETHOD.
ENDCLASS.
```

| Section | Visible From |
|---|---|
| `PUBLIC SECTION` | Anywhere (the class's external interface) |
| `PROTECTED SECTION` | The class itself and its subclasses |
| `PRIVATE SECTION` | Only the class itself |

## 🧬 Inheritance

```abap
CLASS lcl_sub DEFINITION INHERITING FROM lcl_class.
  PUBLIC SECTION.
    " REDEFINITION overrides an inherited method; the parameters are inherited
    " and must not be repeated.
    METHODS data_declaration REDEFINITION.
ENDCLASS.

CLASS lcl_sub IMPLEMENTATION.
  METHOD data_declaration.
    super->data_declaration( ).   " call the inherited implementation first
    lv_protected = 20.            " PROTECTED members are visible here
  ENDMETHOD.
ENDCLASS.
```

`lcl_sub` inherits all `PUBLIC` and `PROTECTED` members of `lcl_class`. Use `INHERITING FROM` for "is-a" relationships; prefer composition (holding a reference to another object) for "has-a" relationships.

> 💡 Every `CLASS ... DEFINITION` needs a matching `CLASS ... IMPLEMENTATION` containing a `METHOD ... ENDMETHOD` block for each declared method — a declaration without an implementation does not activate.

## 🧭 Instance vs. Static Members

| | Instance (`METHODS`, `DATA`) | Static (`CLASS-METHODS`, `CLASS-DATA`) |
|---|---|---|
| Belongs to | A specific object instance | The class itself (shared) |
| Call syntax | `lo_object->method( )` | `zcl_class=>method( )` |
| Needs `NEW #( )`? | ✅ Yes | ❌ No |
| Typical use | Business object state & behavior | Utility/factory methods, singletons |

## 🧰 Legacy & Interop Objects (OLE, OData Model)

```abap
DATA lo_excel       TYPE ole2_object.
DATA lo_workbooks   TYPE ole2_object.
DATA lo_model       TYPE REF TO /iwbep/if_mgw_odata_model.
DATA lo_entity_type TYPE REF TO /iwbep/if_mgw_odata_entity_typ.

" 1. Create the OLE Automation object FIRST
CREATE OBJECT lo_excel 'EXCEL.APPLICATION'.
IF sy-subrc <> 0.
  MESSAGE 'Could not start Excel' TYPE 'S' DISPLAY LIKE 'E'.
  RETURN.
ENDIF.

" 2. Then call methods on it
CALL METHOD OF lo_excel 'Workbooks' = lo_workbooks.
CALL METHOD OF lo_workbooks 'Add'.

" 3. Always release OLE objects when finished
FREE OBJECT lo_excel.

" Guard an object reference BEFORE using it: return when it is NOT bound
IF lo_model IS NOT BOUND.
  RETURN.
ENDIF.

" Getting entity metadata from an OData model (SAP Gateway)
lo_entity_type = lo_model->get_entity_type( iv_entity_name = 'PurchaseOrder' ).
```
