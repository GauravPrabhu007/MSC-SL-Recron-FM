# Code Change Document
## Remove SAC Code Error Logic from FM `Z_LOG_SLINTF_RECR`

---

| Field            | Details                                              |
|------------------|------------------------------------------------------|
| **Document Date**    | 06-March-2026                                    |
| **Function Module**  | `Z_LOG_SLINTF_RECR`                              |
| **Source File**      | FM - Z_LOG_SLINTF_RECR - 06.03.2026.txt          |
| **Prepared For**     | ABAP Development Team                            |
| **Change Type**      | Removal of inapplicable error logic (comment out)|

---

## 1. Background and Objective

The error **"SACCODE does not match with one mapped to charges claimed"** (UPDFLAG = `'S'`, message number `'014'`) is **not applicable for Recron**. This error was originally designed for the standard Shipping Line FM (`Z_LOG_SLINTF`) and was carried over into the Recron FM (`Z_LOG_SLINTF_RECR`) when it was cloned.

There are **two distinct scenarios** in which UPDFLAG `'S'` is written for a Recron invoice — both must be suppressed:

| Scenario | Root Cause | Tables Affected |
|----------|-----------|-----------------|
| **A** | Charge code from invoice not found in `ZSLCHRGCODE`. When this mapping fails, both `'M'` and `'S'` are written as a pair to `lt_slerr` → **ZSLINVERR**. The `'M'` is correct and must remain; the companion `'S'` is not applicable for Recron and must be removed. | ZSLINVERR |
| **B** | SAC code in the invoice does not match the SAC code in `ZLOG_CHRG_CODE` (via `ZSL_GST_SAC_CHECK` switch). A standalone `'S'` is written to `lt_slerr` → **ZSLINVERR**. Entirely inapplicable for Recron. | ZSLINVERR |
| **C** | When the first message in `i_return` has number `'014'`, UPDFLAG `'S'` is set on `lw_invstat` → **ZSLINVSTAT**. Not applicable for Recron. | ZSLINVSTAT |

---

## 2. Code Changes Required

---

### Change 1 — Remove companion `'S'` written alongside `'M'` for mapping error

**Lines: 776–782**
**Section:** Charge code mapping check loop (`BOA - brilo_005`)

**Context:** When a charge code (`kschl`) from the invoice is not found in `ZSLCHRGCODE`, the existing code writes **both** `'M'` (mapping missing) and `'S'` (SAC code invalid) to `lt_slerr`. The `'M'` entry is correct and must remain. The `'S'` companion is not applicable for Recron and must be commented out.

**Current Code (lines 764–790):**

```abap
******BOA - brilo_005
  CLEAR:lw_zslintf, lw_slchrgcode.
  lv_counter  = 1.
  LOOP AT lt_zslintf INTO lw_zslintf.
    READ TABLE lt_slchrgcode INTO  lw_slchrgcode WITH KEY zchrg = lw_zslintf-kschl BINARY SEARCH.
    IF sy-subrc <> 0.
      lw_slerr-counter = lv_counter .
      lw_slerr-chrg           = lw_zslintf-kschl.
      lw_slerr-updflag        = 'M'.
      APPEND lw_slerr TO lt_slerr.
      CLEAR: lw_slerr-chrg.

      lv_counter  = lv_counter + 1.          "<--- COMMENT OUT FROM HERE
      lw_slerr-counter = lv_counter .
      lw_slerr-chrg           = lw_zslintf-kschl.
      lw_slerr-updflag        = 'S'.
      APPEND lw_slerr TO lt_slerr.
      CLEAR: lw_slerr-chrg.
      lv_counter  = lv_counter - 1.          "<--- TO HERE (lines 776–782)
    ENDIF.
  ENDLOOP.

  IF lw_slerr-updflag EQ 'S'. "above error not there.   "<--- COMMENT OUT FROM HERE
    lv_counter  = lv_counter + 1.
  ENDIF.                                                  "<--- TO HERE (lines 786–788)
  CLEAR: lw_slerr-updflag.
******EOA - brilo_005
```

**Required Code After Change (lines 764–790):**

```abap
******BOA - brilo_005
  CLEAR:lw_zslintf, lw_slchrgcode.
  lv_counter  = 1.
  LOOP AT lt_zslintf INTO lw_zslintf.
    READ TABLE lt_slchrgcode INTO  lw_slchrgcode WITH KEY zchrg = lw_zslintf-kschl BINARY SEARCH.
    IF sy-subrc <> 0.
      lw_slerr-counter = lv_counter .
      lw_slerr-chrg           = lw_zslintf-kschl.
      lw_slerr-updflag        = 'M'.
      APPEND lw_slerr TO lt_slerr.
      CLEAR: lw_slerr-chrg.

*     "BOC - Recron: SAC Code error (updflag 'S') not applicable - do not write to ZSLINVERR
*      lv_counter  = lv_counter + 1.
*      lw_slerr-counter = lv_counter .
*      lw_slerr-chrg           = lw_zslintf-kschl.
*      lw_slerr-updflag        = 'S'.
*      APPEND lw_slerr TO lt_slerr.
*      CLEAR: lw_slerr-chrg.
*      lv_counter  = lv_counter - 1.
*     "EOC - Recron: SAC Code error not written to ZSLINVERR
    ENDIF.
  ENDLOOP.

* "BOC - Recron: Counter adjustment for 'S' not required - SAC Code error not applicable
*  IF lw_slerr-updflag EQ 'S'. "above error not there.
*    lv_counter  = lv_counter + 1.
*  ENDIF.
* "EOC - Recron
  CLEAR: lw_slerr-updflag.
******EOA - brilo_005
```

**Lines commented out:** 776–782 (companion `'S'` entry and counter logic) and 786–788 (post-loop counter adjustment)

**Why the counter lines (776 and 782) must also be commented:** The counter was temporarily incremented by 1 to give the `'S'` entry a unique counter slot, then decremented back. Once the `'S'` entry is removed, the temporary increment and decrement serve no purpose and must also be commented to avoid counter corruption.

**Why line 786–788 must also be commented:** The `IF lw_slerr-updflag EQ 'S'` check after the loop was specifically written to detect that the last processed charge had an `'S'` error and to advance the counter accordingly. Once `'S'` is never written, this condition will never be true. Commenting it out keeps the code logically consistent.

---

### Change 2 — Remove standalone SAC code validation `'S'` (ZSL_GST_SAC_CHECK path)

**Lines: 1019–1031**
**Section:** PEOL_033 SAC validation — `ZSL_GST_SAC_CHECK` branch

**Context:** When the switch `ZSL_GST_SAC_CHECK` is active in `ZLOG_STLMNTVR`, the FM compares the SAC code from the invoice interface (`lt_claim_sl_charg`) against the SAC code in the charge code master (`ZLOG_CHRG_CODE`). If they do not match, a standalone `'S'` is written to `lt_slerr` → ZSLINVERR. This validation is entirely inapplicable for Recron.

**Current Code (lines 1014–1036):**

```abap
            IF lw_saccode_validation_ind = abap_true.
              LOOP AT lt_chrg_code INTO lw_chrg_code.
                READ TABLE lt_claim_sl_charg INTO lw_claim_sl_charg
                WITH KEY zchrg_code = lw_chrg_code-zchrg_code
                        saccode = lw_chrg_code-saccode.
                IF sy-subrc <> 0.           "<--- COMMENT OUT FROM HERE
                  "BOA - brilo_005 : 09.07.2018
                  CLEAR: lw_slchrgcode_old.
                  READ TABLE lt_slchrgcode_old INTO lw_slchrgcode_old WITH KEY
                  zchrg_code = lw_claim_sl_charg-zchrg_code BINARY SEARCH.
                  IF sy-subrc = 0.
                    lw_slerr-counter        = lv_counter.
                    lw_slerr-chrg           = lw_slchrgcode_old-zchrg.
                    lw_slerr-updflag        = 'S'.
                    APPEND lw_slerr TO lt_slerr.
                    CLEAR: lw_slerr-chrg.
                  ENDIF.
                ENDIF.                      "<--- TO HERE (lines 1019–1031)
              ENDLOOP.
              CLEAR lw_chrg_code.
            ENDIF.
          ENDIF.
        ENDIF.
```

**Required Code After Change (lines 1014–1036):**

```abap
            IF lw_saccode_validation_ind = abap_true.
              LOOP AT lt_chrg_code INTO lw_chrg_code.
                READ TABLE lt_claim_sl_charg INTO lw_claim_sl_charg
                WITH KEY zchrg_code = lw_chrg_code-zchrg_code
                        saccode = lw_chrg_code-saccode.
*               "BOC - Recron: SAC Code validation error (updflag 'S') not applicable
*                IF sy-subrc <> 0.
*                  "BOA - brilo_005 : 09.07.2018
*                  CLEAR: lw_slchrgcode_old.
*                  READ TABLE lt_slchrgcode_old INTO lw_slchrgcode_old WITH KEY
*                  zchrg_code = lw_claim_sl_charg-zchrg_code BINARY SEARCH.
*                  IF sy-subrc = 0.
*                    lw_slerr-counter        = lv_counter.
*                    lw_slerr-chrg           = lw_slchrgcode_old-zchrg.
*                    lw_slerr-updflag        = 'S'.
*                    APPEND lw_slerr TO lt_slerr.
*                    CLEAR: lw_slerr-chrg.
*                  ENDIF.
*                ENDIF.
*               "EOC - Recron: SAC Code validation error not written to ZSLINVERR
              ENDLOOP.
              CLEAR lw_chrg_code.
            ENDIF.
          ENDIF.
        ENDIF.
```

**Lines commented out:** 1019–1031

---

### Change 3 — Remove ZSLINVSTAT update for message `'014'`

**Lines: 1695–1698**
**Section:** ZSLINVSTAT error flag update block

**Context:** When `i_return` is not empty, the FM reads the first error message. If that message has number `'014'` (SACCODE does not match), it sets `lw_invstat-zupdflag = 'S'` and writes it to table **ZSLINVSTAT**. Since message `'014'` is not raised in the Recron FM, this block is redundant — but must be explicitly commented out to prevent any future risk.

**Current Code (lines 1695–1698):**

```abap
      IF lw_ret1-number = '014'.
**"    SACCODE does not match with one mapped to charges claimed
        lw_invstat-zupdflag = 'S'.
      ENDIF.
```

**Required Code After Change:**

```abap
*     "BOC - Recron: SAC Code error (msg 014) sets zupdflag 'S' in ZSLINVSTAT - not applicable
*      IF lw_ret1-number = '014'.
***"    SACCODE does not match with one mapped to charges claimed
*        lw_invstat-zupdflag = 'S'.
*      ENDIF.
*     "EOC - Recron
```

**Lines commented out:** 1695–1698

---

## 3. What Must NOT Be Changed

| Lines | Code | Reason |
|-------|------|--------|
| 770–774 | `lw_slerr-updflag = 'M'` block | Mapping error flag (`'M'`) is valid for Recron. Only the companion `'S'` is removed. |
| 1939–1941 | `IF lw_ret-number = '002' OR '003' OR '014'. CONTINUE.` | Safety guard in ZSLINVERR loop — prevents message 002/003/014 from creating ZSLINVERR rows. **Keep as-is.** |
| 2008–2010 | `IF lw_ret-number = '014'. CONTINUE. ENDIF.` | Second safety guard in ZSLINVERR update loop. **Keep as-is.** |
| 993–1003 | `ZSL_GST_MULTI_SAC_CHECK` path (sets `lw_tr_billdt`) | This block only sets the bill date for downstream SAC exchange logic and does not write `'S'` to ZSLINVERR in the Recron FM. No change needed. |
| 1015–1018 | LOOP/READ in `ZSL_GST_SAC_CHECK` branch | The LOOP and READ table statements themselves can remain; only the inner `IF sy-subrc <> 0` block (that writes `'S'`) is removed. |

---

## 4. Summary of Changes

| # | Lines | Block Description | Table Affected | Action |
|---|-------|-------------------|---------------|--------|
| **1a** | 776–782 | Companion `'S'` entry and counter logic written alongside `'M'` when charge code not found in `ZSLCHRGCODE` | ZSLINVERR | **Comment out** |
| **1b** | 786–788 | Post-loop counter adjustment tied to the companion `'S'` above | — (counter variable) | **Comment out** |
| **2** | 1019–1031 | Standalone SAC code mismatch `'S'` written when invoice SAC code ≠ `ZLOG_CHRG_CODE` SAC code (`ZSL_GST_SAC_CHECK` path) | ZSLINVERR | **Comment out** |
| **3** | 1695–1698 | `ZSLINVSTAT-zupdflag = 'S'` set when first error in `i_return` is message `'014'` | ZSLINVSTAT | **Comment out** |

**Net effect:** After these changes, UPDFLAG `'S'` will **never** be written to **ZSLINVERR** or **ZSLINVSTAT** for Recron invoices — whether the underlying cause is a charge code mapping failure or a SAC code mismatch.

---

## 5. Impact Assessment

| Area | Impact |
|------|--------|
| **ZSLINVERR** | `'S'` rows will no longer be inserted for Recron invoices. `'M'` rows (condition mapping error) will continue to be written correctly. |
| **ZSLINVSTAT** | `zupdflag = 'S'` will no longer be set for message `'014'`. All other status flags (`'B'`, `'M'`, `'A'`, `'G'`, `'R'`, `'N'`, `'W'`, `'I'`, `'D'`, `'T'`, `'C'`) are unaffected. |
| **Invoice Processing** | No change to invoice acceptance or rejection logic. The changes only suppress the writing of an inapplicable error flag. |
| **Other FMs** | `Z_LOG_SLINTF` (the base/RIL FM) is **not affected** — changes are confined to `Z_LOG_SLINTF_RECR` only. |

---

## 6. Testing Recommendations

1. **Mapping error scenario:** Submit a Recron invoice where one or more charge codes are **not mapped** in `ZSLCHRGCODE`. Confirm:
   - `ZSLINVERR` contains `'M'` rows for the unmapped charges.
   - `ZSLINVERR` does **not** contain `'S'` rows alongside the `'M'` rows.

2. **SAC code mismatch scenario** *(if `ZSL_GST_SAC_CHECK` switch is active):* Submit a Recron invoice where the SAC code in the interface does not match `ZLOG_CHRG_CODE`. Confirm:
   - No `'S'` row is written to `ZSLINVERR`.
   - `ZSLINVSTAT` does not show `zupdflag = 'S'`.

3. **Valid invoice scenario:** Submit a Recron invoice with all charge codes correctly mapped and amounts within tolerance. Confirm:
   - Invoice processes successfully with `zupdflag = 'C'` in `ZSLINVSTAT`.
   - No unintended rows in `ZSLINVERR`.

4. **Other error scenarios:** Submit invoices that trigger other valid errors (`'B'` — BL not found, `'A'` — incorrect amount, `'G'`/`'R'` — GST errors). Confirm those errors are still captured correctly and the changes have no side effects.

---

*Document prepared for ABAP development team reference.*
*Source FM: `Z_LOG_SLINTF_RECR` | File: FM - Z_LOG_SLINTF_RECR - 06.03.2026.txt*
