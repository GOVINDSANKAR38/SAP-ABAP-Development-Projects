# 09 — Modularization

## 📖 Introduction

Modularization means splitting logic into reusable, testable units. ABAP offers several mechanisms: **function modules** (RFC-callable, package-based), **`FORM`/`PERFORM`** (classical subroutines, , **macros** (`DEFINE`/`END-OF-DEFINITION`, textual/preprocessor-like), and — the modern, recommended approach — **classes and methods** 

## 🧩 Calling a Function Module

```abap
" A function module is called by NAME. Always handle its exceptions.
CALL FUNCTION 'CONVERSION_EXIT_ALPHA_INPUT'
  EXPORTING input  = lv_external_value
  IMPORTING output = lv_internal_value.
```


### 🔁 Conversion Exits

Conversion exits (function modules named `CONVERSION_EXIT_<NAME>_INPUT/OUTPUT`) convert between the internal storage format and the human-readable display format of special data elements (material numbers, dates, WBS elements, units, etc.).

```abap
" Convert Date To String
DATA lv_tarih  TYPE datum.
DATA lv_string TYPE string.

CALL FUNCTION 'CONVERSION_EXIT_PDATE_OUTPUT'
  EXPORTING input  = lv_tarih
  IMPORTING output = lv_string.

" Convert Material Number (INPUT = display -> internal, OUTPUT = internal -> display)
CALL FUNCTION 'CONVERSION_EXIT_MATN1_INPUT'
  EXPORTING  input        = ls_data-material
  IMPORTING  output       = ls_data-material
  EXCEPTIONS length_error = 1
             OTHERS       = 2.

CALL FUNCTION 'CONVERSION_EXIT_MATN1_OUTPUT'
  EXPORTING  input        = ls_data-material
  IMPORTING  output       = ls_data-material
  EXCEPTIONS length_error = 1
             OTHERS       = 2.

" Convert internal characteristic to characteristic name (ATINN -> ATNAM)
CALL FUNCTION 'CONVERSION_EXIT_ATINN_OUTPUT'
  EXPORTING input  = <measurement_document>-internal_characteristic
  IMPORTING output = <measurement_document>-internal_characteristic_text.

" Convert unit of measure to its display text
CALL FUNCTION 'CONVERSION_EXIT_CUNIT_OUTPUT'
  EXPORTING input    = ls_data-meins
            language = sy-langu
  IMPORTING output   = ls_data-meins.

" WBS element. The two formats are:
"   EXTERNAL / display  : the readable coded form, e.g. 'P-1000.01.01' (PRPS-POSID)
"   INTERNAL            : the numeric key, e.g. 00001223              (PRPS-PSPNR)
" INPUT  converts external -> internal
" OUTPUT converts internal -> external
DATA lv_posid TYPE prps-posid.   " external / display format
DATA lv_pspnr TYPE prps-pspnr.   " internal numeric key

CALL FUNCTION 'CONVERSION_EXIT_ABPSP_INPUT'
  EXPORTING  input     = lv_posid
  IMPORTING  output    = lv_pspnr
  EXCEPTIONS not_found = 1
             OTHERS    = 2.

IF sy-subrc <> 0.
  " WBS element not found - handle as a business error
ENDIF.

CALL FUNCTION 'CONVERSION_EXIT_ABPSP_OUTPUT'
  EXPORTING input  = lv_pspnr
  IMPORTING output = lv_posid.

" Generic ALPHA conversion exit (used for most numeric keys, e.g. document numbers)
CALL FUNCTION 'CONVERSION_EXIT_ALPHA_INPUT'
  EXPORTING input  = ls_data-data
  IMPORTING output = ls_data-data.
```



### 📐 Unit Conversions

```abap
DATA lv_material      TYPE matnr.
DATA lv_weight_kg     TYPE brgew_ap.
DATA lv_volume_m3     TYPE ntgew_ap.
DATA lv_unit_kg       TYPE meins VALUE 'KG'.
DATA lv_unit_m3       TYPE meins VALUE 'M3'.

" Convert a weight in KG into a volume in M3 for the given material
CALL FUNCTION 'MATERIAL_UNIT_CONVERSION'
  EXPORTING  input                = lv_weight_kg
             kzmeinh              = abap_true
             matnr                = lv_material
             meinh                = lv_unit_m3
             meins                = lv_unit_kg
  IMPORTING  output               = lv_volume_m3
  EXCEPTIONS conversion_not_found = 1
             input_invalid        = 2
             material_not_found   = 3
             meinh_not_found      = 4
             meins_missing        = 5
             no_meinh             = 6
             output_invalid       = 7
             overflow             = 8
             OTHERS               = 9.

" IF, not CHECK. CHECK leaves the whole processing block when its condition is
" FALSE - so "CHECK sy-subrc <> 0." would abort on SUCCESS, the exact opposite
" of what is intended here.
IF sy-subrc <> 0.
  MESSAGE e001(zsm_msg) WITH lv_material lv_unit_kg lv_unit_m3.
ENDIF.

" Generic material unit conversion (any unit -> any unit for a given material)
CALL FUNCTION 'MD_CONVERT_MATERIAL_UNIT'
  EXPORTING  i_matnr              = p_matnr
             i_in_me              = p_meins
             i_out_me             = 'PAL'
             i_menge              = p_menge
  IMPORTING  e_menge              = lv_menge
  EXCEPTIONS error_in_application = 1
             error                = 2
             OTHERS               = 3.
```

### 🧰 Other Useful Standard Function Modules

```abap
" Extract a file extension from a MIME type (application/pdf -> pdf).
" Size the receiving field for the LONGEST extension you expect - a
" CHAR1 target would silently truncate 'pdf' to 'p'.
DATA lv_extension TYPE string.
DATA lv_mime_type TYPE w3conttype.

CALL FUNCTION 'SDOK_FILE_NAME_EXTENSION_GET'
  EXPORTING mimetype  = lv_mime_type
  IMPORTING extension = lv_extension.

" Validate/convert a time value
CALL FUNCTION 'CONVERT_TIME_INPUT'
  EXPORTING  input                     = ls_data-value
             plausibility_check        = 'X'
  IMPORTING  output                    = ls_data-value
  EXCEPTIONS plausibility_check_failed = 1
             wrong_format_in_input     = 2
             OTHERS                    = 3.

" Get the last date of a given month (YYYYMM -> DD.MM.YYYY)
DATA lv_last_date_of_month TYPE sy-datum.
DATA lv_year_month         TYPE jva_prod_month.

CALL FUNCTION 'JVA_LAST_DATE_OF_MONTH'
  EXPORTING year_month         = lv_year_month
  IMPORTING last_date_of_month = lv_last_date_of_month.

" Get personnel number from user ID (HR)
DATA lv_personel_no TYPE persno.

CALL FUNCTION 'RP_GET_PERNR_FROM_USERID'
  EXPORTING  begda     = sy-datum
             endda     = sy-datum
             usrid     = sy-uname
             usrty     = '0001'
  IMPORTING  usr_pernr = lv_personel_no
  EXCEPTIONS retcd     = 1
             OTHERS    = 2.

" Get user address and lock status
DATA ls_address   TYPE bapiaddr3.
DATA ls_is_locked TYPE bapislockd.
DATA lt_return    TYPE TABLE OF bapiret2.
DATA lv_locked    TYPE xfeld.
DATA lv_username  TYPE bapibname-bapibname.

CALL FUNCTION 'BAPI_USER_GET_DETAIL'
  EXPORTING username = lv_username
  IMPORTING address  = ls_address
            islocked = ls_is_locked
  TABLES    return   = lt_return.

IF NOT line_exists( lt_return[ type = 'E' ] ).
  IF ls_is_locked-glob_lock = 'L' OR ls_is_locked-local_lock = 'L' OR ls_is_locked-no_user_pw = 'L' OR ls_is_locked-wrng_logon = 'L'.
    lv_locked = abap_true.
  ENDIF.
ENDIF.

" Progress indicator for long-running batch jobs
" (use a text symbol so the message can be translated)
CALL FUNCTION 'SAPGUI_PROGRESS_INDICATOR'
  EXPORTING percentage = 10
            text       = |{ TEXT-p01 } { lv_step }/{ lv_total }|.
```

### 📡 Calling a Function Module via RFC Destination

> **Lifecycle:** `CLASSIC BUT STILL RELEVANT`. RFC remains the backbone of system-to-system integration in on-premise landscapes. Under ABAP Cloud, outbound calls go through released APIs and communication scenarios instead.

```abap
CONSTANTS lc_rfc_name TYPE tfdir-funcname VALUE 'ZSM_F_TEST'.

" The destination is maintained in SM59 and should come from Customizing,
" not be hardcoded in the program.
DATA(lv_destination) = get_destination( ).   " TYPE rfcdest

" A remote call can fail for reasons a local call cannot. ALWAYS handle
" system_failure and communication_failure - otherwise an unreachable or
" misconfigured destination produces a short dump.
CALL FUNCTION lc_rfc_name DESTINATION lv_destination
  EXPORTING  iv_uname              = lv_uname
  IMPORTING  ev_is_admin           = lv_admin
  EXCEPTIONS system_failure        = 1 MESSAGE DATA(lv_system_msg)
             communication_failure = 2 MESSAGE DATA(lv_comm_msg)
             OTHERS                = 3.

CASE sy-subrc.
  WHEN 0.
    " success
  WHEN 1.
    MESSAGE lv_system_msg TYPE 'E'.
  WHEN 2.
    MESSAGE lv_comm_msg TYPE 'E'.
  WHEN OTHERS.
    MESSAGE 'Remote call failed' TYPE 'E'.
ENDCASE.
```



## 🧵 Macros (`DEFINE` / `END-OF-DEFINITION`)

> **Lifecycle:** `LEGACY / HISTORICAL REFERENCE`. Macros are obsolete for new code and are not available in ABAP Cloud. They are kept here because they appear constantly in existing programs — particularly in ALV and BAPI-filling code — and reading them is a real skill.

Macros perform a **textual substitution** before compilation. There is no type checking of parameters, no signature, and no line-by-line debugging: the debugger steps over the whole macro as one statement. Prefer a small private method for anything beyond trivial local repetition.

```abap
" Simple macro
DEFINE printer.
  WRITE :/ 'Hello', &1, &2.
END-OF-DEFINITION.

WRITE / 'Before Using Macro'.
printer 'ABAP' 'Macros'.

" Macro used to fill a BAPI structure + its "X" (changed-flag) companion structure together
DATA ls_header_in  LIKE bapisdhd1.  " SD Document Header
DATA ls_header_inx LIKE bapisdhd1x. " SD Document Header Checkbox

DEFINE gx.
  &1-&2 = &3.
  &1x-&2 = abap_true.
END-OF-DEFINITION.

gx ls_header_in doc_type  'ZSTD'.
gx ls_header_in sales_org '1000'.

WRITE: ls_header_in-doc_type, ls_header_inx-doc_type.

" Macro that both replaces a character and condenses the result
DEFINE conv_char.
  REPLACE ALL OCCURRENCES OF &1 IN &2 WITH &3.
  CONDENSE &2.
END-OF-DEFINITION.

conv_char '-' <fs_data>-value ' '.
```


## 📞 Calling Other Programs — SUBMIT & Screen Chaining

```abap
" Call a report, letting the user see/adjust its selection screen
SUBMIT zsm_r_test VIA SELECTION-SCREEN AND RETURN.

" Call a report, passing selection-screen parameters directly (no screen shown)
SUBMIT zsm_r_test
       AND RETURN
       WITH p_bukrs = ls_data-bukrs
       WITH p_gjahr = ls_data-gjahr
       WITH p_belnr = ls_data-belnr
       WITH rb_fat  = abap_true.

" Call a screen at a specific position on the current window
CALL SCREEN 0200 STARTING AT 50 10.

" Read another program's global internal table by name (advanced / debugging technique)
ASSIGN ('(SAPMV54A)XVBUV[]') TO FIELD-SYMBOL(<fs_xvbuv>).

IF <fs_xvbuv> IS ASSIGNED.
  DATA(lt_xvbuv) = <fs_xvbuv>.
ENDIF.
```
