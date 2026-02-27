# Code Change Document – Z_LOG_SLINTF_RECR Duplicate Invoice Check

## 1. Summary

| Item | Details |
|------|--------|
| **Change ID / Reference** | *(e.g. CD:8085337 or your ticket number)* |
| **Module** | Recron – Shipping Line Invoice Interface |
| **Main Object** | Function Module **Z_LOG_SLINTF_RECR** |
| **Related Object** | Function Module **Z_SCE_DUPL_INV_CHECK_REC** (unchanged; call point and parameters changed) |
| **Change Type** | Enhancement – Duplicate invoice check moved to start of FM; BUKRS derived from BL_NO via LIKP → LIPS → T001K |
| **Date** | 27.02.2026 |

**Summary in one sentence:**  
Duplicate invoice check (Z_SCE_DUPL_INV_CHECK_REC) is now executed at the start of Z_LOG_SLINTF_RECR using header data (I_SLINTFH), with BUKRS derived from BL_NO via LIKP (BOLNR) → LIPS (VBELN) → T001K (BWKEY), and the previous duplicate check inside the lt_fc1 block has been removed.

---

## 2. Objective / Business Reason

- Run duplicate invoice validation **as early as possible** so invalid (duplicate) invoices are rejected before further processing.
- Use **consistent BUKRS derivation** from the Bill of Lading (I_SLINTFH-BL_NO) via standard delivery/plant/company code tables (LIKP → LIPS → T001K) instead of relying on settlement data (lt_fc1) that may not yet be available.
- Simplify control flow by having **one** duplicate check at the start instead of a conditional check later in the FM.

---

## 3. Affected Objects

| Object Type | Object Name | Action |
|-------------|-------------|--------|
| Function Module | Z_LOG_SLINTF_RECR | Modified |
| Function Module | Z_SCE_DUPL_INV_CHECK_REC | No change (only call position and parameters) |
| Tables used for BUKRS derivation | LIKP, LIPS, T001K | Read-only (SELECT only) |

---

## 4. Detailed Change Description

### 4.1 New Data Declarations (Z_LOG_SLINTF_RECR)

- **Location:** With existing variables (e.g. after `lv_duplicate`).
- **Added variables:**
  - `lv_vbeln_del` (TYPE vbeln_vl) – Delivery document from LIKP.
  - `lv_werks` (TYPE werks_d) – Plant from LIPS.
  - `lv_bukrs_dupl` (TYPE bukrs) – Company code from T001K for the duplicate check call.

### 4.2 New Logic at Start of Z_LOG_SLINTF_RECR

- **Insert position:** Immediately after the first header is read from the input table (e.g. after the “brilo_005” block that fills `lw_slerr`) and before existing processing (e.g. before `CLEAR lv_rfcdest` and the rest of the FM).
- **Conditions:** Execute only when the first header was read successfully (`sy-subrc = 0`) and `lw_zslintfh-bl_no` is not initial.
- **Steps:**
  1. **BUKRS derivation**
     - **LIKP:** `SELECT vbeln FROM likp ... WHERE bolnr = lw_zslintfh-bl_no` (UP TO 1 ROWS) → `lv_vbeln_del`.
     - **LIPS:** `SELECT werks FROM lips ... WHERE vbeln = lv_vbeln_del` (UP TO 1 ROWS) → `lv_werks`.
     - **T001K:** `SELECT bukrs FROM t001k ... WHERE bwkey = lv_werks` (UP TO 1 ROWS) → `lv_bukrs_dupl`.
  2. **Duplicate check call**
     - Call **Z_SCE_DUPL_INV_CHECK_REC** with:
       - **IM_INV_NO** = `lw_zslintfh-tr_billno` (from I_SLINTFH-TR_BILLNO)
       - **IM_LIFNR** = `lw_zslintfh-lifnr` (from I_SLINTFH-LIFNR)
       - **IM_INV_DT** = `lw_zslintfh-tr_bill_dt` (from I_SLINTFH-TR_BILL_DT)
       - **IM_BUKRS** = `lv_bukrs_dupl` (derived as above)
     - **EX_DUPLICATE** → `lv_duplicate`.
  3. **If duplicate**
     - Set `lw_ret-type = gc_e`, `lw_ret-number = '005'`, `lw_ret-message = TEXT-003`.
     - Append `lw_ret` to `i_return`.
     - **RETURN** from the FM (no further processing).

If BL_NO is initial or any of the three selects fails, BUKRS may remain initial; Z_SCE_DUPL_INV_CHECK_REC already handles initial BUKRS (e.g. by returning without setting duplicate).

### 4.3 Removal of Previous Duplicate Check (lt_fc1 Block)

- **Location:** Inside `IF lt_fc1 IS NOT INITIAL`, after `READ TABLE lt_fc1 INTO lw_fc1 INDEX 1` and `IF sy-subrc EQ 0`.
- **Removed:** Entire block that called Z_SCE_DUPL_INV_CHECK_REC with `lw_fc1-bukrs` and the IF/ELSE on `lv_duplicate` (append error vs. CLEAR lt_fc1_t).
- **Replaced with:** Single statement: `CLEAR lt_fc1_t[]` (with comment that duplicate check is done at start of FM), so downstream logic that relies on `lt_fc1_t` being cleared in the “non-duplicate” path remains correct.

---

## 5. Before vs After Behaviour

| Aspect | Before | After |
|--------|--------|--------|
| When duplicate check runs | Only when `lt_fc1` was not initial (i.e. when ZTRSTLMNT_M_A had data for the invoice). | At the start of the FM, whenever the first header is read and BL_NO is not initial. |
| BUKRS source | From first record in `lt_fc1` (ZTRSTLMNT_M_A), i.e. settlement data. | Derived from I_SLINTFH-BL_NO via LIKP (BOLNR) → LIPS (VBELN) → T001K (BWKEY). |
| Effect on duplicate invoice | Error 005 appended; processing continued until normal end. | Error 005 appended and FM exits immediately with RETURN. |
| Effect when no BL in LIKP / no plant / no T001K | N/A (duplicate check sometimes not executed). | Duplicate check still called with possibly initial BUKRS; Z_SCE_DUPL_INV_CHECK_REC behaviour for initial BUKRS unchanged. |

---

## 6. Parameters Passed to Z_SCE_DUPL_INV_CHECK_REC (After Change)

| Parameter | Source |
|-----------|--------|
| IM_INV_NO | I_SLINTFH-TR_BILLNO (via lw_zslintfh-tr_billno) |
| IM_LIFNR | I_SLINTFH-LIFNR (via lw_zslintfh-lifnr) |
| IM_INV_DT | I_SLINTFH-TR_BILL_DT (via lw_zslintfh-tr_bill_dt) |
| IM_BUKRS | BUKRS from T001K, where BWKEY = WERKS from LIPS (VBELN from LIKP where BOLNR = I_SLINTFH-BL_NO) |

---

## 7. Testing Recommendations

- **Duplicate in same FY:** Invoice (same LIFNR, TR_BILLNO, TR_BILL_DT) already present in ZTRSTLMNT_M_A or ZSCE_REC_INVSL for the derived BUKRS → expect error 005 and immediate return; no updates to ZSLINVSTAT / ZSCE tables for that invoice.
- **No duplicate:** Same invoice not present → no error 005; FM continues and behaves as before (including CLEAR lt_fc1_t in the lt_fc1 block).
- **BUKRS derivation:**  
  - BL_NO present in LIKP-BOLNR with valid delivery → LIPS has WERKS, T001K has BUKRS for that plant → duplicate check uses this BUKRS.  
  - BL_NO missing in LIKP or no WERKS/BUKRS → duplicate check called with initial BUKRS; confirm no regression (e.g. no duplicate set when it shouldn’t be).
- **BL_NO initial:** Duplicate check block is skipped (no call); rest of FM as before.
- **Regression:** Run existing scenarios (valid invoice, other errors like 001/002/004) and confirm behaviour unchanged except for duplicate handling as above.

---

## 8. Rollback

- Revert Z_LOG_SLINTF_RECR to the previous version: remove the new declarations, remove the new “duplicate check at start” block, and restore the original duplicate check and IF/ELSE (with CLEAR lt_fc1_t) inside the lt_fc1 block.

---

## 9. Document Control

| Version | Date | Author | Description |
|---------|------|--------|-------------|
| 1.0 | 27.02.2026 | *(Your name)* | Initial code change document – duplicate check at start, BUKRS via LIKP/LIPS/T001K |
