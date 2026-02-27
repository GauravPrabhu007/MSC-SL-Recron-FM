# Code Change Document – Success Message in i_return (Recron Invoice Created)

## Document Control

| Field | Value |
|-------|--------|
| **Document Title** | Pass success message in i_return when Recron invoice is created in zsce_rec_invsl |
| **Object Type** | Function Module |
| **Object Name** | Z_LOG_SLINTF_RECR |
| **Document Date** | 27.02.2026 |
| **Change Type** | Enhancement (return message logic) |

---

## 1. Summary

At the **end of the main FM Z_LOG_SLINTF_RECR**, when the return table **i_return** is blank (i.e. no errors occurred), add a check to see if a Recron invoice was created in table **zsce_rec_invsl** for the same vendor and bill (LIFNR, TR_BILLNO, TR_BILL_DT from **I_SLINTFH**). Only **active** records are considered (where **del_ind** is blank or not equal to 'X'). If such an entry exists, append a **success message** to **i_return**: type **'S'** and message **"Invoice created successfully"**, so the caller receives explicit success feedback when the Recron invoice is created.

---

## 2. Rationale

- **Z_LOG_SLINTF_RECR** creates Recron invoices by updating tables **zsce_rec_invsl** and **zsce_rec_invchrg** via **Z_SCE_FIORI_INVSL_UPDATE_REC**. When processing completes without errors, **i_return** remains empty and the caller has no explicit confirmation that the invoice was written to the Recron tables.
- Aligning with return-logic patterns used in the base FM **Z_LOG_SLINTF**, the Recron variant should signal success when an active invoice exists in **zsce_rec_invsl**, so that callers can show a clear "Invoice created successfully" message and success is tied to actual data in **zsce_rec_invsl**.
- The **del_ind** filter ensures that only non-deleted (active) invoice records are considered; logically deleted rows (del_ind = 'X') must not trigger the success message.

---

## 3. Scope of Change

### 3.1 Location

- **Program:** Z_LOG_SLINTF_RECR
- **Placement:** At the **end of the FM**, after the block **"End of code - brilo_005"** and **before** **ENDFUNCTION**.
- **Reference source:** `FM - Z_LOG_SLINTF_RECR_27.02.2026.txt` (or current FM source).

### 3.2 Objects Referenced

| Object | Usage |
|--------|--------|
| **I_SLINTFH** | Input table; first row used for LIFNR, TR_BILLNO, TR_BILL_DT |
| **I_RETURN** | TABLES parameter; success message appended here |
| **zsce_rec_invsl** | Database table; existence check for created invoice (with del_ind filter) |
| **lw_zslintfh** | Work area for header (from I_SLINTFH) |
| **lw_invsl_data** | Work area (zsce_rec_invsl); used to receive SELECT result |
| **lw_ret** | BAPIRET2 work area for building the success message |

### 3.3 Condition for Success Message

- **i_return[]** is **initial** (no errors).
- **I_SLINTFH** has at least one row (read INDEX 1).
- **zsce_rec_invsl** contains at least one row for:
  - **mandt** = sy-mandt
  - **lifnr** = I_SLINTFH-LIFNR (from first row)
  - **tr_billno** = I_SLINTFH-TR_BILLNO
  - **tr_bill_dt** = I_SLINTFH-TR_BILL_DT
  - **del_ind** = blank **OR** del_ind <> 'X' (exclude logically deleted records).

---

## 4. Code Change

### 4.1 Insertion Point

Insert the following block **after**:

```
*******End of code - brilo_005
```

and **before**:

```
ENDFUNCTION.
```

### 4.2 New Code (to be added)

```abap
* If no errors, check if Recron invoice was created and pass success message
  IF i_return[] IS INITIAL.
    READ TABLE i_slintfh INTO lw_zslintfh INDEX 1.
    IF sy-subrc = 0.
      SELECT SINGLE lifnr
        FROM zsce_rec_invsl CLIENT SPECIFIED
        INTO lw_invsl_data-lifnr
        WHERE mandt     = sy-mandt
          AND lifnr     = lw_zslintfh-lifnr
          AND tr_billno = lw_zslintfh-tr_billno
          AND tr_bill_dt = lw_zslintfh-tr_bill_dt
          AND ( del_ind IS INITIAL OR del_ind <> 'X' ).
      IF sy-subrc = 0.
        CLEAR lw_ret.
        lw_ret-type    = 'S'.
        lw_ret-message = 'Invoice created successfully'.
        APPEND lw_ret TO i_return.
      ENDIF.
    ENDIF.
  ENDIF.
```

### 4.3 Behaviour Summary

| Step | Action |
|------|--------|
| 1 | If **i_return** is not initial, do nothing (errors already present). |
| 2 | Read first header from **i_slintfh** into **lw_zslintfh**. |
| 3 | **SELECT SINGLE** from **zsce_rec_invsl** for matching LIFNR, TR_BILLNO, TR_BILL_DT and active flag (del_ind IS INITIAL OR del_ind <> 'X'). |
| 4 | If a row is found, append one BAPIRET2 line to **i_return** with type 'S' and message 'Invoice created successfully'. |

---

## 5. Testing Recommendations

- **Success path:** Run the FM with valid input so that **Z_SCE_FIORI_INVSL_UPDATE_REC** creates rows in **zsce_rec_invsl** (and **zsce_rec_invchrg**) with del_ind not 'X'. Confirm **i_return** contains one entry with type 'S' and message 'Invoice created successfully'.
- **No invoice in zsce_rec_invsl:** Simulate a run where no errors occur but **zsce_rec_invsl** has no matching active row. Confirm **i_return** remains empty.
- **Error path:** Run with input that causes validation errors. Confirm **i_return** is filled with errors and that the new block does **not** add the success message.
- **Deleted invoice:** When the only matching row in **zsce_rec_invsl** has **del_ind = 'X'**, confirm the success message is **not** appended.

---

## 6. Dependencies / Notes

- No new data declarations are required; existing variables **lw_zslintfh**, **lw_invsl_data**, and **lw_ret** are used.
- If the message text **"Invoice created successfully"** is to be maintained as a text symbol or message class, replace the literal in **lw_ret-message** with the appropriate reference.
- Field name **del_ind** must match the actual field in table **zsce_rec_invsl** (adjust casing if required by the data dictionary).

---

## 7. Revision History

| Version | Date | Author | Description |
|---------|------|--------|-------------|
| 1.0 | 27.02.2026 | — | Initial code change document: success message in i_return when Recron invoice found in zsce_rec_invsl (del_ind not 'X'). |
