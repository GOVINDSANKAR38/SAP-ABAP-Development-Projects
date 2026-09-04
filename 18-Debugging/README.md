# 18 — Debugging, Messages, Logging & Exceptions

## 📖 Introduction

Robust ABAP programs communicate clearly with users (`MESSAGE`), handle errors gracefully (`TRY`/`CATCH`), and keep a trace of what happened (application log / custom log tables). This chapter also covers a few small but useful debugging/system-check utilities.

## 💬 MESSAGE Statement Variants

```abap
DATA et_return TYPE bapiret2_t.
DATA lv_message TYPE bapi_msg.

" Build a BAPIRET2-style message from the current sy-msg* fields
MESSAGE ID sy-msgid TYPE sy-msgty NUMBER sy-msgno
        WITH sy-msgv1 sy-msgv2 sy-msgv3 sy-msgv4
        INTO lv_message.

et_return = VALUE #( ( type = 'E' id = 'ZSM_MSG' number = '001' message = lv_message ) ).

" Simple ad-hoc messages
MESSAGE 'An error occurred' TYPE 'E'.
MESSAGE 'The process completed successfully' TYPE 'I' DISPLAY LIKE 'S'.
MESSAGE TEXT-001 TYPE 'W'.
MESSAGE i001(zsm_msg).

" Capturing a message into a variable instead of displaying it
MESSAGE e002(zsm_msg) INTO lv_message.
MESSAGE e003(zsm_msg) WITH lv_value1 lv_value2 INTO lv_message.  " &1 &2 placeholders

" Building a return message directly with a string template
APPEND VALUE #( type = 'E' message = |Error occurred: { lv_text }| ) TO et_return.

" Merging one return table into another
APPEND LINES OF lt_return TO et_return.
```

### Acting on a Return Table

```abap
" Inside a loop over the documents being processed
LOOP AT lt_documents INTO DATA(ls_document).
  process_document( EXPORTING is_document = ls_document
                    IMPORTING et_return   = DATA(lt_return) ).

  DATA(lv_has_error) = xsdbool(    line_exists( lt_return[ type = 'E' ] )
                                OR line_exists( lt_return[ type = 'A' ] )
                                OR line_exists( lt_return[ type = 'X' ] ) ).

  APPEND LINES OF lt_return TO et_return.

  IF lv_has_error = abap_true.
    CONTINUE.               " skip this document, keep processing the rest
  ENDIF.

  APPEND ls_document TO lt_processed.
ENDLOOP.
```

> ⚠️ `CONTINUE` is only valid **inside** a loop (`LOOP`, `DO`, `WHILE`). Outside one, use `RETURN` to leave the current processing block, or restructure the condition. The transaction decision itself belongs to the caller — see [08 — SAP LUW](../08-Open-SQL/README.md#-sap-luw--transaction-ownership).

### 🔖 Message Types

| Type | Meaning | Typical Effect |
|---|---|---|
| `S` | Success | Shown in the status bar |
| `I` | Information | Shown as a popup (in dialog processing) |
| `W` | Warning | Shown as a popup, execution can usually continue |
| `E` | Error | Stops processing / requires user correction |
| `A` | Abort | Ends the current transaction |
| `X` | Exit / Exception | Short dump (used to signal a serious/unexpected error) |

### 📦 Reusable "Append Return Message" Helper

```abap
" A method is the modern form. Identifiers are alphanumeric plus underscore -
" characters such as '$' are not valid in an ABAP name.
CLASS lcl_message_collector DEFINITION.
  PUBLIC SECTION.
    METHODS add IMPORTING iv_type    TYPE bapiret2-type
                          iv_message TYPE bapi_msg
                CHANGING  ct_return  TYPE bapiret2_t.
ENDCLASS.

CLASS lcl_message_collector IMPLEMENTATION.
  METHOD add.
    APPEND VALUE #( type    = iv_type
                    message = iv_message ) TO ct_return.
  ENDMETHOD.
ENDCLASS.
```

### 🏷️ Message Classes

Message texts and numbers are maintained per **message class** (transaction `SE91`), so they can be translated and reused consistently:

```abap
" Message class ZSM_MSG, maintained in SE91. '&' marks a text placeholder.
REPORT zsm_report MESSAGE-ID zsm_msg.

MESSAGE i001.
```

### ⚙️ Propagating System Messages into a BAPIRET2 Table

A very common integration pattern: convert whatever the last statement's `sy-msg*` fields hold into a `BAPIRET2` row, so system and custom errors end up in one consistent return table.

```abap
CLASS lcl_message_collector DEFINITION.
  PUBLIC SECTION.
    METHODS add_system_message IMPORTING is_syst    TYPE syst
                                         iv_message TYPE bapi_msg OPTIONAL
                               CHANGING  ct_return  TYPE bapiret2_t.
ENDCLASS.

CLASS lcl_message_collector IMPLEMENTATION.
  METHOD add_system_message.
    DATA(ls_message) = VALUE bapiret2( id         = is_syst-msgid
                                       number     = is_syst-msgno
                                       type       = is_syst-msgty
                                       message_v1 = is_syst-msgv1
                                       message_v2 = is_syst-msgv2
                                       message_v3 = is_syst-msgv3
                                       message_v4 = is_syst-msgv4 ).

    ls_message-message = COND bapi_msg( WHEN iv_message IS NOT INITIAL
                                        THEN iv_message
                                        ELSE ls_message-message ).

    APPEND ls_message TO ct_return.
  ENDMETHOD.
ENDCLASS.

" Usage
DATA lt_messages TYPE bapiret2_t.

MESSAGE e007(zsm_msg) INTO DATA(lv_text).

NEW lcl_message_collector( )->add_system_message(
    EXPORTING is_syst    = syst
              iv_message = CONV #( lv_text )
    CHANGING  ct_return  = lt_messages ).
```

## 🧯 Exception Handling

ABAP has two error-handling mechanisms. **Classic exceptions** (`RAISE`, `EXCEPTIONS` in a `CALL FUNCTION`) are declared in the function module's interface in SE37 and mapped to `sy-subrc` at the call site — they are obsolete for new code but ubiquitous in existing code. **Class-based exceptions** (`RAISE EXCEPTION TYPE`, `TRY`/`CATCH`) carry a real object with a message, a cause chain and typed attributes, and are the current mechanism.

```abap
" Class-based exception handling.
" CATCH clauses are evaluated TOP-DOWN, so the MOST SPECIFIC class must come
" first. A superclass listed before its subclass makes the subclass CATCH
" unreachable - the compiler and the extended check both flag this.
TRY.
    lv_result = lv_dividend / lv_divisor.        " may raise CX_SY_ZERODIVIDE
    lv_number = CONV i( lv_character_input ).    " may raise CX_SY_CONVERSION_NO_NUMBER

  CATCH cx_sy_zerodivide INTO DATA(lx_zerodivide).
    " most specific first ...
    MESSAGE lx_zerodivide->get_text( ) TYPE 'E'.

  CATCH cx_sy_arithmetic_error INTO DATA(lx_arithmetic).
    " ... then its superclass
    MESSAGE lx_arithmetic->get_text( ) TYPE 'E'.

  CATCH cx_sy_conversion_no_number INTO DATA(lx_no_number).
    MESSAGE lx_no_number->get_text( ) TYPE 'E'.

  CATCH cx_sy_conversion_error INTO DATA(lx_conversion).
    " superclass of cx_sy_conversion_no_number - must come AFTER it
    MESSAGE lx_conversion->get_text( ) TYPE 'E'.
ENDTRY.
```

```abap
" Raising your own exception
IF lv_quantity <= 0.
  RAISE EXCEPTION TYPE zcx_invalid_quantity
    EXPORTING iv_quantity = lv_quantity.
ENDIF.
```

```abap
" Classic exceptions from a function module call (LEGACY, but everywhere).
" For a REMOTE call, always handle system_failure and communication_failure.
DATA lv_system_msg TYPE string.
DATA lv_comm_msg   TYPE string.

CALL FUNCTION 'ZSM_F_TEST' DESTINATION lv_destination
  EXPORTING  iv_organization_id    = iv_organization_id
  IMPORTING  et_person             = et_person
  EXCEPTIONS system_failure        = 1 MESSAGE lv_system_msg
             communication_failure = 2 MESSAGE lv_comm_msg
             OTHERS                = 3.

CASE sy-subrc.
  WHEN 0.
  WHEN 1.       MESSAGE lv_system_msg TYPE 'E'.
  WHEN 2.       MESSAGE lv_comm_msg   TYPE 'E'.
  WHEN OTHERS.  MESSAGE 'Remote call failed' TYPE 'E'.
ENDCASE.
```


> ```abap
> " Boundary handler - logs and re-raises, does not swallow
> TRY.
>     run_job( ).
>   CATCH cx_root INTO DATA(lx_unexpected).
>     log_failure( lx_unexpected ).
>     RAISE EXCEPTION lx_unexpected.
> ENDTRY.
> ```

## 📔 Simple Custom Logging Table

```abap
" TABLE ZSM_T_LOG
"   mandt    mandt
"   username uname
"   logdate  erdat
"   logtime  erzet

" Read
SELECT SINGLE username, logdate, logtime
  FROM zsm_t_log
  WHERE username = @sy-uname
  INTO @DATA(ls_log).

" Collect (the caller decides when to write and commit)
DATA lt_log TYPE TABLE OF zsm_t_log.

APPEND VALUE #( username = sy-uname
                logdate  = sy-datum
                logtime  = sy-uzeit ) TO lt_log.
```

>37 rather than copying a signature from any guide, including this one.

## 🛠️ System/Environment Checks — and Why to Avoid Them

```abap
" ⚠️ ANTI-PATTERN - do not branch business logic on the system ID or client.
CASE sy-sysid.
  WHEN 'DEV'.
    " ... development-only behaviour ...
  WHEN 'PRD'.
    " ... production-only behaviour ...
ENDCASE.
```


> ```abap
> SELECT SINGLE is_active
>   FROM zsm_t_feature
>   WHERE feature = 'EXTENDED_CHECK'
>   INTO @DATA(lv_feature_active).
>
> IF lv_feature_active = abap_true.
>   " ...
> ENDIF.
> ```


## 🖥️ Related Transaction Codes

| T-Code | Purpose |
|---|---|
| SE91 | Maintain message classes |
| SLG0 | Define application log objects/sub-objects |
| SLG1 | Display application log |
| SAAB | Maintain checkpoint groups (assertions, breakpoints, logging) |
| ST22 | Analyze short dumps |
| `/h` | OK-code (not a transaction) — activates the ABAP Debugger from any screen |
