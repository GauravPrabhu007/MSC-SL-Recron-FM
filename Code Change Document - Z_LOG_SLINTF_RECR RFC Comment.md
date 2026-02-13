# Code Change Document – Comment RFC-Related Code in Z_LOG_SLINTF_RECR

## Document Control

| Field | Value |
|-------|--------|
| **Document Title** | Comment RFC-related code in FM Z_LOG_SLINTF_RECR |
| **Object Type** | Function Module |
| **Object Name** | Z_LOG_SLINTF_RECR |
| **Document Date** | 13.02.2025 |
| **Change Type** | Code comment (disable RFC logic) |

---

## 1. Summary

Comment out the code block in **Z_LOG_SLINTF_RECR** that performs **RFC (Remote Function Call)** operations. This FM is the **Recron-specific** variant of the Shipping Line Interface; RFC calls to other systems (e.g. RIL) are not applicable in the Recron context and are therefore being disabled by commenting.

---

## 2. Rationale

- **Z_LOG_SLINTF_RECR** was copied from **Z_LOG_SLINTF** and contains only Recron-specific logic (RPCM); RIL-specific logic was removed.
- The commented block uses **RFC destination** `lv_rfcdest` (from ZLOG_EXEC_VAR, name `SLV_INV_RDEST_GST`) to:
  1. Call **Z_SCM_CHECK_GST** on a remote system to validate RIL GSTIN.
  2. If GSTIN exists, call **Z_LOG_SLINTF** on the remote system and return (delegate processing).
- In the **Recron** scenario, processing must stay in the current system; remote RFC calls are not in scope. Commenting this block ensures Recron FM runs locally only and avoids unnecessary or incorrect RFC usage.

---

## 3. Scope of Change

### 3.1 Location

- **Program:** Z_LOG_SLINTF_RECR (source: `FM - Z_LOG_SLINTF_RECR.txt`)
- **Section:** After the first read of header and the `j_1bbranch` SELECT for GSTIN
- **Approximate lines:** 461–482 (in the referenced source file)

### 3.2 RFC Calls Affected

| # | Function Module | Purpose |
|---|-----------------|---------|
| 1 | **Z_SCM_CHECK_GST** | Check if RIL GST number exists on remote system (DESTINATION lv_rfcdest) |
| 2 | **Z_LOG_SLINTF** | Call original Shipping Line Interface on remote system (DESTINATION lv_rfcdest) |

### 3.3 Code Block to Be Commented

The entire **IF sy-subrc <> 0 AND lv_rfcdest IS NOT INITIAL** block (including nested IF and both CALL FUNCTIONs) is to be commented with leading `*`.

**Current code (BEFORE):**

```abap
      IF sy-subrc <> 0 AND lv_rfcdest IS NOT INITIAL.
        IF lv_rfcdest+0(3) NE lv_sysid.

          REFRESH lt_return.
          CALL FUNCTION 'Z_SCM_CHECK_GST' DESTINATION lv_rfcdest
          EXPORTING
             im_gstin       = lw_zslintfh-rilgstno
           IMPORTING
            ex_gstin_exist  = lv_gstin_exists
            ex_return       = lt_return.

          IF lv_gstin_exists IS NOT INITIAL AND lv_gstin_exists EQ 'Y'.
            CALL FUNCTION 'Z_LOG_SLINTF' DESTINATION lv_rfcdest
            TABLES
              i_slintfh       =  i_slintfh
              i_slintf        =  i_slintf
              i_return        =  i_return.
              IF sy-subrc = 0.
                RETURN.
              ENDIF.
          ENDIF.
        ENDIF.
      ENDIF.
```

**Code after change (AFTER – commented):**

```abap
*     " BOC - RFC calls commented - Not applicable in Recron FM
*      IF sy-subrc <> 0 AND lv_rfcdest IS NOT INITIAL.
*        IF lv_rfcdest+0(3) NE lv_sysid.
*
*          REFRESH lt_return.
*          CALL FUNCTION 'Z_SCM_CHECK_GST' DESTINATION lv_rfcdest
*          EXPORTING
*             im_gstin       = lw_zslintfh-rilgstno
*           IMPORTING
*            ex_gstin_exist  = lv_gstin_exists
*            ex_return       = lt_return.
*
*          IF lv_gstin_exists IS NOT INITIAL AND lv_gstin_exists EQ 'Y'.
*            CALL FUNCTION 'Z_LOG_SLINTF' DESTINATION lv_rfcdest
*            TABLES
*              i_slintfh       =  i_slintfh
*              i_slintf        =  i_slintf
*              i_return        =  i_return.
*              IF sy-subrc = 0.
*                RETURN.
*              ENDIF.
*          ENDIF.
*        ENDIF.
*      ENDIF.
*     " EOC - RFC calls commented - Not applicable in Recron FM
```

---

## 4. Impact

| Area | Impact |
|------|--------|
| **Recron (RPCM) usage** | No change in intended behaviour; Recron processing continues locally without RFC. |
| **RIL / remote system** | No remote GST check or delegation to Z_LOG_SLINTF from this FM (by design for Recron). |
| **Variables (lv_rfcdest, lv_gstin_exists, etc.)** | May still be populated earlier in the FM but are no longer used in this block; can be left as-is or cleaned up in a later change. |
| **ZLOG_EXEC_VAR (SLV_INV_RDEST_GST)** | Configuration may remain; it will simply not be used by this block in Z_LOG_SLINTF_RECR. |

---

## 5. Testing Recommendations

- Execute **Z_LOG_SLINTF_RECR** with Recron-relevant test data (header + item tables) and confirm:
  - No RFC is triggered (e.g. no call to Z_SCM_CHECK_GST or Z_LOG_SLINTF via RFC).
  - Recron validations and updates (e.g. Z_SCE_FIORI_INVSL_UPDATE_REC, ZSLINVSTAT, ZSLINVCHRG, etc.) behave as before.
- Verify that when RIL GSTIN is not in `j_1bbranch`, the FM continues with local Recron logic instead of calling the remote system.

---

## 6. Approval / Transport (Optional)

| Role | Name | Date | Signature |
|------|------|------|-----------|
| Developer | | | |
| Reviewer | | | |
| Transport Request | | | |

---

## 7. References

- FM documentation: **FM - Z_LOG_SLINTF_RECR.md**
- Source code: **FM - Z_LOG_SLINTF_RECR.txt**
- Note in FM: *"THIS FM COPIED FROM FM: Z_LOG_SLINTF – In Z_LOG_SLINTF_RECR, RIL-specific validation logic is removed and only Recron-specific logic (RPCM) is kept."*

---

*End of Code Change Document*
