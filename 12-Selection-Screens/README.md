# 12 — Selection Screens, Screens & Popups

## 📖 Introduction

Selection screens gather report input from the user (`PARAMETERS`, `SELECT-OPTIONS`). Dynpros (`SCREEN`) provide fully custom, interactive screens (tabstrips, subscreens, PBO/PAI modules). This chapter also covers popup windows and screen field control.

> **Lifecycle:** `CLASSIC BUT STILL RELEVANT`. Selection screens and dynpros are the standard on-premise report and dialog UI and remain in active use. They are **not** part of the ABAP Cloud development model, where the UI layer is Fiori/UI5 over OData. 

## 🖥️ Selection Screen Elements

```abap
" TABLES is obsolete for general use, but is still REQUIRED to reference
" dynpro/selection-screen structures such as SSCRFIELDS, and to declare the
" work areas that SELECT-OPTIONS ... FOR refers to.
TABLES: mara, vbak, mseg, tadir, sscrfields.

DATA lt_values TYPE vrm_values.

SELECTION-SCREEN BEGIN OF BLOCK b1 WITH FRAME TITLE TEXT-001.
  PARAMETERS p_matnr TYPE mara-matnr OBLIGATORY.
  SELECT-OPTIONS s_vbeln FOR vbak-vbeln.

  SELECTION-SCREEN SKIP.

  PARAMETERS p_value AS CHECKBOX.
  SELECT-OPTIONS s_devcl FOR tadir-devclass OBLIGATORY DEFAULT 'Z*' OPTION CP SIGN I.

  SELECTION-SCREEN SKIP.

  " Radio button group. Group names are identifiers (max 4 characters),
  " not numeric literals - 'g1', not '00'.
  PARAMETERS p_rad1 RADIOBUTTON GROUP g1 USER-COMMAND rad DEFAULT 'X'.
  PARAMETERS p_rad2 RADIOBUTTON GROUP g1.
  PARAMETERS p_rad3 RADIOBUTTON GROUP g1.

  SELECTION-SCREEN SKIP.

  " MODIF ID groups fields so LOOP AT SCREEN can address them together.
  " The name must match what you compare against screen-group1 later.
  PARAMETERS p_count TYPE zsm_t_data-print_count AS LISTBOX VISIBLE LENGTH 5
                     MODIF ID gr2 DEFAULT 1 OBLIGATORY.
  SELECT-OPTIONS s_mjahr FOR mseg-mjahr MODIF ID gr2.
  SELECT-OPTIONS s_lgort FOR mseg-lgort MODIF ID gr2.

  SELECTION-SCREEN SKIP.

  PARAMETERS p_id     TYPE zsm_d_id MATCHCODE OBJECT zsm_sh_id.
  PARAMETERS p_mail   AS CHECKBOX.
  PARAMETERS p_number TYPE i AS LISTBOX VISIBLE LENGTH 7 DEFAULT 3 OBLIGATORY.
  SELECT-OPTIONS s_date FOR zsm_t_doc-erdat DEFAULT sy-datum OBLIGATORY.

  SELECTION-SCREEN SKIP.

  " Elements arranged on one line, with comments beside them
  SELECTION-SCREEN BEGIN OF LINE.
    PARAMETERS p_optn1 RADIOBUTTON GROUP g2 USER-COMMAND radio DEFAULT 'X'.
    SELECTION-SCREEN COMMENT (10) TEXT-002.
    PARAMETERS p_optn2 RADIOBUTTON GROUP g2.
    SELECTION-SCREEN COMMENT (10) TEXT-003.
    PARAMETERS p_optn3 RADIOBUTTON GROUP g2.
    SELECTION-SCREEN COMMENT (10) TEXT-004.
  SELECTION-SCREEN END OF LINE.

  SELECTION-SCREEN BEGIN OF LINE.
    PARAMETERS p_cb AS CHECKBOX MODIF ID gr3.
    SELECTION-SCREEN COMMENT (12) TEXT-005 MODIF ID gr3.
  SELECTION-SCREEN END OF LINE.

SELECTION-SCREEN END OF BLOCK b1.

SELECTION-SCREEN FUNCTION KEY 1.
SELECTION-SCREEN FUNCTION KEY 2.
```



| Element | Purpose |
|---|---|
| `PARAMETERS` | Single-value input field |
| `SELECT-OPTIONS` | Range input (low/high, multiple values) — generates a range table (see [06-Loops](../06-Loops/README.md#-range-tables-ranges--select-options)) |
| `RADIOBUTTON GROUP` | Mutually exclusive options |
| `AS CHECKBOX` | Boolean toggle |
| `AS LISTBOX` | Dropdown (values set at runtime via `VRM_SET_VALUES`) |
| `MODIF ID` | Groups fields so their screen attributes (visible/enabled) can be changed together in `LOOP AT SCREEN` |
| `MATCHCODE OBJECT` | Attaches an F4 search help to a parameter |

## 🎛️ Selection-Screen Events & Dynamic Behavior

```abap
" Runs ONCE, before the selection screen is built.
" Good for: default values, listbox contents, toolbar texts.
" NOT for screen modification - see AT SELECTION-SCREEN OUTPUT below.
INITIALIZATION.
  PERFORM init.

" Runs on EVERY screen display (the selection screen's PBO).
" This is the ONLY place where LOOP AT SCREEN / MODIFY SCREEN takes effect.
AT SELECTION-SCREEN OUTPUT.
  PERFORM modify_screen.

" F4 help for a specific parameter
AT SELECTION-SCREEN ON VALUE-REQUEST FOR p_id.
  PERFORM value_request_for_id.

" Field-level validation
AT SELECTION-SCREEN ON p_matnr.
  PERFORM check_material.

" Toolbar function keys
AT SELECTION-SCREEN.
  CASE sscrfields-ucomm.
    WHEN 'FC01'.
    WHEN 'FC02'.
  ENDCASE.

START-OF-SELECTION.
  PERFORM get_data.

FORM init.
  " Populate a listbox's values at runtime
  lt_values = VALUE #( ( key = '1' text = 'One' )
                       ( key = '2' text = 'Two' )
                       ( key = '3' text = 'Three' ) ).

  CALL FUNCTION 'VRM_SET_VALUES'
    EXPORTING id     = 'P_NUMBER'
              values = lt_values.

  " Custom function-key icons/text on the selection screen toolbar
  sscrfields-functxt_01 = VALUE smp_dyntxt( icon_id   = '@J2@'
                                            text      = TEXT-u01
                                            icon_text = TEXT-u01
                                            quickinfo = TEXT-u01 ).
  sscrfields-functxt_02 = VALUE smp_dyntxt( icon_id   = '@0O@'
                                            text      = TEXT-u02
                                            icon_text = TEXT-u02
                                            quickinfo = TEXT-u02 ).

  " Pre-fill a SELECT-OPTIONS default from the current user's settings
  SELECT SINGLE lgort
    FROM zsm_t_user_default
    WHERE username = @sy-uname
    INTO @DATA(lv_lgort).

  IF sy-subrc = 0.
    s_lgort = VALUE #( ( sign = 'I' option = 'EQ' low = lv_lgort ) ).
  ENDIF.
ENDFORM.

FORM modify_screen.
  " Hide/disable a group of fields dynamically.
  " The compared value must match the MODIF ID used in the declarations
  " ('GR2' here, uppercase - screen-group1 is a character field).
  LOOP AT SCREEN INTO DATA(ls_screen).
    IF ls_screen-group1 = 'GR2'.
      ls_screen-active    = 0.
      ls_screen-invisible = 1.
      MODIFY SCREEN FROM ls_screen.
    ENDIF.
  ENDLOOP.
ENDFORM.
```


## 🪟 Custom Screens (Dynpros)

A dynpro has two processing blocks: **PBO** (Process Before Output, runs before the screen is displayed) and **PAI** (Process After Input, runs after the user acts).
two listings below are shown separately for exactly that reason.

**Flow logic — maintained in SE51, not in the ABAP source:**

```abap
* --- Screen 0100, Flow Logic tab ---
PROCESS BEFORE OUTPUT.
  MODULE status_0100.
  CALL SUBSCREEN sub1 INCLUDING sy-repid '0101'.

PROCESS AFTER INPUT.
  MODULE user_command_0100.
  CALL SUBSCREEN sub1.
```

**ABAP source — the program that owns the screen:**

```abap
REPORT zsm_test.

CONTROLS go_tab TYPE TABSTRIP.

DATA: gt_values TYPE vrm_values,
      gv_flag   TYPE xfeld,
      gv_id     TYPE vrm_id,
      gv_value  TYPE i.

START-OF-SELECTION.
  CALL SCREEN 0100.

MODULE status_0100 OUTPUT.
  SET PF-STATUS 'STATUS_0100'.
  SET TITLEBAR 'TITLE_0100'.

  PERFORM set_dropdown_sh.
  PERFORM set_screen_fields.
ENDMODULE.

MODULE user_command_0100 INPUT.
  " Function codes beginning with '&' are reserved for the system -
  " use plain names for your own commands.
  CASE sy-ucomm.
    WHEN 'BACK' OR 'EXIT' OR 'CANC'.
      LEAVE TO SCREEN 0.
    WHEN 'DISABLE' OR 'ENABLE'.
      gv_flag = COND xfeld( WHEN sy-ucomm = 'DISABLE' THEN abap_false
                                                      ELSE abap_true ).
    WHEN 'TAB1'.
      go_tab-activetab = 'TAB1'.
    WHEN 'TAB2'.
      go_tab-activetab = 'TAB2'.
    WHEN OTHERS.
  ENDCASE.
ENDMODULE.

FORM set_dropdown_sh.
  gv_id = 'GV_VALUE'.
  gt_values = VALUE vrm_values( ( key = '1' text = 'A' )
                                ( key = '2' text = 'B' )
                                ( key = '3' text = 'C' ) ).

  CALL FUNCTION 'VRM_SET_VALUES'
    EXPORTING
      id     = gv_id
      values = gt_values.
ENDFORM.
```

### Screen Field Modules (PBO/PAI at Field Level)

**Flow logic (SE51):**

```abap
* --- Screen 0102, Flow Logic tab ---
PROCESS BEFORE OUTPUT.
  MODULE status_0102.

PROCESS AFTER INPUT.
  FIELD qmel-zzcarrier MODULE check_carrier.
  FIELD qmel-zzcarrier MODULE get_carrier_text ON INPUT.
  MODULE user_command_0102.
```

**ABAP source:**

```abap
MODULE status_0102 OUTPUT.
  " Make the custom fields display-only in the display transactions.
  " NOTE: 'x IN ( a, b )' is ABAP SQL syntax and is NOT valid in an ABAP IF.
  " Use an OR chain, or a range table for longer lists.
  LOOP AT SCREEN INTO DATA(ls_screen).
    IF ( ls_screen-name = 'QMEL-ZZPROD_CODE' OR
         ls_screen-name = 'QMEL-ZZCARRIER' )
   AND ( sy-tcode = 'QM03' OR sy-tcode = 'IW23' ).
      ls_screen-input = 0.
      MODIFY SCREEN FROM ls_screen.
    ENDIF.
  ENDLOOP.
ENDMODULE.
```

> ```abap
> DATA(lr_display_tcodes) = VALUE rseloption( sign = 'I' option = 'EQ'
>                                             ( low = 'QM03' ) ( low = 'IW23' ) ).
> IF sy-tcode IN lr_display_tcodes.
> ```

## 🔔 Popups

```abap
" A stand-alone selection screen displayed as a modal popup window.
" Declare it once, then call it where you need it.
SELECTION-SCREEN BEGIN OF SCREEN 601 AS WINDOW.
  SELECT-OPTIONS s_datum FOR zsm_t_doc-erdat DEFAULT sy-datum OBLIGATORY.
  SELECT-OPTIONS s_dtype FOR zsm_t_doc-doc_type.
  SELECTION-SCREEN SKIP.
  SELECT-OPTIONS s_uname FOR zsm_t_doc-ernam.
SELECTION-SCREEN END OF SCREEN 601.

START-OF-SELECTION.
  CALL SELECTION-SCREEN 601 STARTING AT 40 8.
  IF sy-subrc <> 0.
    RETURN.                        " user cancelled the popup
  ENDIF.

" Simple informational popup
CALL FUNCTION 'POPUP_TO_DISPLAY_TEXT'
  EXPORTING textline1 = 'Processing finished'.
```

