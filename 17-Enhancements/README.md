# 17 — Enhancements

## 📖 Introduction

Beyond BAdIs ([16-BADIs](../16-BADIs/README.md)), SAP provides several other **enhancement techniques** to add custom logic to standard programs without modifying them directly. This chapter is a conceptual overview to complement the BAdI chapter, since enhancements are a closely related and frequently confused topic.

## 🧭 Enhancement Techniques Overview

| Technique | Era | Modifies Standard Code? | Multiple Implementations? | Typical Use | Lifecycle |
|---|---|---|---|---|---|
| **User Exit** (`USEREXIT_*` form routines) | Oldest — classic SD | ⚠️ Effectively yes — you edit a delivered include | ❌ No | Classic SD enhancements in includes such as `MV45AFZZ` | `LEGACY / HISTORICAL REFERENCE` |
| **Customer Exit / Function Exit** (`CALL CUSTOMER-FUNCTION`) | Classic (SMOD/CMOD) | ❌ No — a pre-planned hook | ❌ No (one CMOD project per enhancement) | Function, menu and screen exits in older modules | `LEGACY / HISTORICAL REFERENCE` |
| **BAdI** (Business Add-In) | From SAP R/3 4.6 onward | ❌ No | ✅ Yes (multiple-use and/or filter-dependent) | The standard object-oriented extension point | `CURRENT / RECOMMENDED` |
| **Enhancement Point / Section** (Enhancement Framework) | NetWeaver 7.0 / ECC 6.0 onward | ❌ No — inserted at explicit or implicit positions | ✅ Yes | Inserting code inside standard logic | `CLASSIC BUT STILL RELEVANT` |
| **Explicit Enhancement Spot** | NetWeaver 7.0 / ECC 6.0 onward | ❌ No | ✅ Yes | Extension positions SAP designed deliberately | `CLASSIC BUT STILL RELEVANT` |
| **Modification (access key)** | Classic | ✅ Yes — direct change to an SAP object | N/A | Last resort only | `LEGACY / HISTORICAL REFERENCE` |

> 📝 **NEEDS OFFICIAL VERIFICATION** for the exact release in which each technique was introduced. The eras above are expressed deliberately loosely; check SAP Help for your target release before quoting a version number.

## 🔧 User Exits vs. Customer Exits

These two are constantly confused, including in job interviews. They are different mechanisms.

**User exits** (classic SD) are empty `FORM` routines that SAP delivers inside modification-enabled includes such as `MV45AFZZ`. You write your code directly into the delivered include:

```abap
" In include MV45AFZZ (delivered by SAP, intended to be edited)
FORM userexit_save_document_prepare.
  " your validation / defaulting logic
ENDFORM.
```

Because you are editing an SAP object, these are registered as modifications in some landscapes and show up in `SPAU` during an upgrade.

**Customer exits** (also called function exits) are a genuine hook mechanism: SAP calls `CALL CUSTOMER-FUNCTION 'nnn'` from standard code, and you implement the corresponding function module — activated through a project in `CMOD` that references an SAP enhancement in `SMOD`. You never edit standard code:

```abap
" Inside standard SAP code (not modified by you):
CALL CUSTOMER-FUNCTION '001'.

" You implement the generated function module EXIT_SAPMV45A_001, whose
" coding lives in a customer include such as ZXVVAU01, and activate the
" containing enhancement through a CMOD project.
```

## 🧵 Enhancement Points & Implicit Enhancements

Implicit enhancement points/options exist automatically at the start/end of nearly every `FORM`, `METHOD`, and `PROGRAM` in the SAP system, viewable directly in the ABAP Editor via **Edit → Enhancement Operations → Show Implicit Enhancement Options**.

```abap
FORM standard_form.
  " ENHANCEMENT-POINT ep_standard_form_01 SPOTS es_standard_form.
  " your custom coding can be inserted here via an enhancement implementation
ENDFORM.
```

## 🆚 Choosing Between Them

- **User exit** (`USEREXIT_*`): oldest, procedural, one implementation, and you edit a delivered include — so it carries modification-like upgrade cost.
- **Customer exit** (`CALL CUSTOMER-FUNCTION` + SMOD/CMOD): a real hook, but one active project per enhancement, and procedural.
- **BAdI**: object-oriented, can support multiple filter-dependent implementations. The standard answer for a planned extension point.
- **Enhancement point/spot**: lets you insert code at explicit positions SAP designed, or at *implicit* positions that exist almost everywhere. Powerful, and correspondingly easy to abuse.

## ☁️ Under ABAP Cloud

Most of this chapter describes on-premise techniques. In the ABAP Cloud development model the picture narrows sharply: extension happens through **released** extension points — released BAdIs, released APIs and the defined extensibility options — not through modifications, implicit enhancements, or edits to delivered includes. That is the practical meaning of "keep the core clean". The techniques above remain correct and necessary for the on-premise systems that run today; they simply are not the path for a cloud-model extension. See [21-Classic-vs-Modern-ABAP](../21-Classic-vs-Modern-ABAP/README.md).


## 🖥️ Related Transaction Codes

| T-Code | Purpose |
|---|---|
| SMOD | Display/manage SAP enhancements (customer exits) |
| CMOD | Create the project that activates a customer exit |
| SE18 / SE19 | Define / implement a BAdI |
| SE80 | Enhancement spots and enhancement implementations |
| SPAU / SPDD | Adjust modifications (repository / Dictionary) during an upgrade |


