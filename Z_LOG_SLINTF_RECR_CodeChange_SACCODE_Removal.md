# Code Change Document: Remove SACCODE Error (UPDFLAG 'S') from Recron FM Z_LOG_SLINTF_RECR

**Document for:** ABAP Team  
**Function Module:** Z_LOG_SLINTF_RECR  
**Source reference:** FM - Z_LOG_SLINTF_RECR.txt / FM - Z_LOG_SLINTF_RECR.md  

**Objective:** The error *"SACCODE does not match with one mapped to charges claimed"* (UPDFLAG = 'S' in table ZSLINVERR) is **not applicable for Recron**. This document describes the code changes required to remove this error from the Recron FM so that it is never written to ZSLINVERR or ZSLINVSTAT.

---

## 1. Background: Where UPDFLAG 'S' (SACCODE Error) Is Set

| Location | Description |
|----------|-------------|
| **A** | **ZSL_GST_MULTI_SAC_CHECK path:** After `Z_LOG_CLAIM_SL_VALIDAT`, a LOOP runs on `lt_claim_sl_charg`. Where `zindicator` is initial (SAC invalid), the FM appends to `lt_slerr` with `updflag = 'S'`. These entries are later written to **ZSLINVERR**. |
| **B** | **ZSL_GST_SAC_CHECK path:** In the single SAC-check branch, when charge/SAC from the interface does not match `lt_chrg_code`, the FM appends to `lt_slerr` with `updflag = 'S'` → **ZSLINVERR**. |
| **C** | **ZSLINVSTAT update:** When the first error in `i_return` has number `'014'`, the FM sets `lw_invstat-zupdflag = 'S'` before modifying **ZSLINVSTAT**. (In current RECR, 014 is not appended to `i_return` in the SAC block; this block is for consistency.) |

**Important:** The block that sets `updflag = 'M'` and `updflag = 'S'` for **condition mapping** (charge not found in ZSLCHRGCODE) around lines 803–816 must **not** be changed. That 'S' is used there in conjunction with 'M' for mapping errors, not for SACCODE.

---

## 2. Code Changes Required

### Change 1: Do Not Write to ZSLINVERR When Multi-SAC Check Fails (ZSL_GST_MULTI_SAC_CHECK)

**Approx. lines:** 1010–1025  

**Current logic:** After `Z_LOG_CLAIM_SL_VALIDAT`, a LOOP runs on `lt_claim_sl_charg`. Where `zindicator` is initial, the code appends to `lt_slerr` with `updflag = 'S'`, which is later inserted into ZSLINVERR.

**Required change:** For Recron, do **not** append to `lt_slerr` for this SAC failure. Remove or comment out the block that fills `lw_slerr` and appends it.

**Option A – Comment out the append block (recommended for traceability):**

```abap
          SORT lt_slchrgcode_old BY zchrg_code.
          LOOP AT lt_claim_sl_charg INTO lw_claim_sl_charg.
            IF lw_claim_sl_charg-zindicator IS INITIAL.
              "BOA - brilo_005 : 09.07.2018
              "BOA - Recron: SACCODE error (UPDFLAG 'S') not applicable - do not write to ZSLINVERR
*              CLEAR: lw_slchrgcode_old.
*              READ TABLE lt_slchrgcode_old INTO lw_slchrgcode_old WITH KEY
*              zchrg_code = lw_claim_sl_charg-zchrg_code BINARY SEARCH.
*              IF sy-subrc EQ 0.
*                lw_slerr-counter        = lv_counter.
*                lw_slerr-chrg           = lw_slchrgcode_old-zchrg.
*                lw_slerr-updflag        = 'S'.
*                APPEND lw_slerr TO lt_slerr.
*                CLEAR: lw_slerr-chrg.
*              ENDIF.
              "EOA - Recron: SACCODE error not written to ZSLINVERR
            ENDIF.
          ENDLOOP.
```

**Option B – Delete the inner block:**  
Remove the lines from `CLEAR: lw_slchrgcode_old.` through the inner `ENDIF.` (so the LOOP and `IF zindicator IS INITIAL` remain but no append to `lt_slerr` occurs).

---

### Change 2: Do Not Write to ZSLINVERR When Single SAC Check Fails (ZSL_GST_SAC_CHECK)

**Approx. lines:** 1037–1054  

**Current logic:** In the ELSE branch (ZSL_GST_SAC_CHECK), a LOOP checks `lt_chrg_code` against `lt_claim_sl_charg`. When no match is found, the code appends to `lt_slerr` with `updflag = 'S'`, which is later written to ZSLINVERR.

**Required change:** For Recron, do **not** append to `lt_slerr` for this case.

**Option A – Comment out the append block:**

```abap
                IF sy-subrc <> 0.
                  "BOA - brilo_005 : 09.07.2018
                  "BOA - Recron: SACCODE error (UPDFLAG 'S') not applicable - do not write to ZSLINVERR
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
                  "EOA - Recron: SACCODE error not written to ZSLINVERR
                ENDIF.
```

**Option B – Delete the inner block:**  
Remove from `CLEAR: lw_slchrgcode_old.` through the inner `ENDIF.` (keeping the outer `IF sy-subrc <> 0` and its `ENDIF.`).

---

### Change 3: Do Not Set ZSLINVSTAT-zupdflag = 'S' for Message 014

**Approx. lines:** 1710–1713  

**Current logic:** If the first return message has number `'014'`, the FM sets `lw_invstat-zupdflag = 'S'`, which is then written to ZSLINVSTAT.

**Required change:** For Recron, do not set zupdflag = 'S' for message 014.

**Option A – Comment out:**

```abap
*      IF lw_ret1-number = '014'.
**"    SACCODE does not match with one mapped to charges claimed
*        lw_invstat-zupdflag = 'S'.
*      ENDIF.
```

**Option B – Delete:**  
Remove the entire IF block.

---

## 3. What Must Not Be Changed

- **Lines 803–816:** The block that sets `updflag = 'M'` and `updflag = 'S'` when the charge is **not** in ZSLCHRGCODE (condition mapping). That 'S' is for the mapping error scenario, not SACCODE; leave this logic unchanged.
- **Lines 1954, 2023:** The `IF lw_ret-number = '002' OR lw_ret-number = '003' OR lw_ret-number = '014'. CONTINUE. ENDIF.` — keep as is so that if 014 ever appears in `i_return`, no ZSLINVERR entry is created from it.
- **SAC validation logic itself** (e.g. call to `Z_LOG_CLAIM_SL_VALIDAT`, parameter checks): may remain; only the **writing** of UPDFLAG 'S' to ZSLINVERR and ZSLINVSTAT is to be removed for Recron.

---

## 4. Summary Table

| # | Location (approx.) | Table Affected | Action |
|---|--------------------|----------------|--------|
| 1 | 1011–1025 | ZSLINVERR | Do not append `lw_slerr` with `updflag = 'S'` when ZSL_GST_MULTI_SAC_CHECK fails (zindicator initial). |
| 2 | 1043–1053 | ZSLINVERR | Do not append `lw_slerr` with `updflag = 'S'` when ZSL_GST_SAC_CHECK fails (no match in lt_chrg_code). |
| 3 | 1710–1713 | ZSLINVSTAT | Do not set `zupdflag = 'S'` when `lw_ret1-number = '014'`. |

---

## 5. Testing Recommendations

- Run the Recron interface with data that would previously trigger SACCODE validation failure: confirm **no** new rows in **ZSLINVERR** with UPDFLAG = 'S' for that scenario.
- Confirm **ZSLINVSTAT** is never updated with zupdflag = 'S' for the SACCODE (014) case.
- Re-test a scenario that triggers the **condition mapping** error (charge not in ZSLCHRGCODE): confirm ZSLINVERR still receives the intended 'M'/'S' entries for that case.

---

*Document created for ABAP team reference. Base FM source: Z_LOG_SLINTF_RECR (Recron – Shipping Line Invoice Interface).*
