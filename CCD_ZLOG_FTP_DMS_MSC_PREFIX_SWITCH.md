# Code Change Document (CCD)
## Prefix-Controlled PDF Search in Existing FM `Z_LOG_FTP_DMS`

| Field | Value |
|---|---|
| Document type | Code Change Document |
| Functional reference | `FS_ZLOG_FTP_DMS_MSC_PREFIX_SWITCH.md` |
| Existing objects | Report `ZLOG_FTP_DMS`, FM `Z_LOG_FTP_DMS` |
| New control parameter | `ZLOG_EXEC_VAR-NAME = ZSCE_MSC_RECRPVEND_PDF` |
| ABAP rules baseline | `ABAP Rules - 02-04-2026/` |
| Version | 1.0 |

---

## 1. Change summary

Enhance FM `Z_LOG_FTP_DMS` to support optional prefixed file names for configured vendor+BUKRS combinations:

- `RE_<TR_BILLNO>.PDF/.pdf`
- `RP_<TR_BILLNO>.PDF/.pdf`

Behavior is gated by active config in `ZLOG_EXEC_VAR`.  
If no active vendor config exists, FM runs fully as-is.

---

## 2. Scope of code changes

### 2.1 Objects changed

| Object type | Object name | Change type |
|---|---|---|
| Function Module | `Z_LOG_FTP_DMS` | Enhancement |

### 2.2 Objects not changed

| Object | Reason |
|---|---|
| `ZLOG_FTP_DMS` report | Existing call pattern remains valid |
| `ZSL_DMS` transaction | No entry-point change needed |
| Existing DMS/OCR/update blocks | Out of scope; no intended behavior change |

---

## 3. Configuration design

`ZLOG_EXEC_VAR` must hold active rows:

| NAME | LIFNR | BUKRS | RFCDEST | ACTIVE |
|---|---|---|---|---|
| ZSCE_MSC_RECRPVEND_PDF | 3807152 | 5155 | RE_ | X |
| ZSCE_MSC_RECRPVEND_PDF | 3807152 | 5184 | RP_ | X |

Usage:

- `NAME`: switch key
- `LIFNR`: vendor gate
- `BUKRS`: entity identification
- `RFCDEST`: prefix token
- `ACTIVE`: enable/disable

---

## 4. Technical logic to implement

### 4.1 New constants/types/data (inside FM)

Add:

- Constant `lc_name_pdf_pref TYPE rvari_vnam VALUE 'ZSCE_MSC_RECRPVEND_PDF'`.
- Local type for switch rows (`name`, `lifnr`, `bukrs`, `rfcdest`, `active`).
- Internal tables/work areas:
  - `lt_exec_pdf_pref`, `lw_exec_pdf_pref`
  - `lt_bl_bukrs`, `lw_bl_bukrs` (if not already available in this FM)
  - `lv_prefix`, `lv_prefix_enabled`, `lv_pref_file_opened`

### 4.2 Prefetch switch rows by vendor (outside invoice loop)

Read all active rows for current vendor and switch name in one SELECT:

- `WHERE name = lc_name_pdf_pref`
- `AND lifnr = i_lifnr`
- `AND active = abap_true`

If no rows: set `lv_prefix_enabled = space` and continue with existing flow.

### 4.3 Derive BUKRS map only when prefix switch enabled

If `lt_exec_pdf_pref` is not initial, derive `BL_NO -> BUKRS` map from:

`LIKP (BOLNR)` -> `LIPS (VBELN)` -> `T001W (WERKS)`

Use set-based read and binary search pattern.

### 4.4 Invoice-level decision logic

Inside `LOOP AT lt_slinvstat ASSIGNING <lfs_slinvstat>`:

1. Default `lv_pref_file_opened = space`.
2. If prefix switch enabled:
   - find `BUKRS` from map by `BL_NO`,
   - read matching switch row by `LIFNR + BUKRS`,
   - if found, set `lv_prefix = rfcdest` and attempt prefixed filename open first.
3. If prefixed attempt fails (or no BUKRS match), execute existing filename logic unchanged.
4. Continue current DMS processing after file open success.

### 4.5 File open sequence (final)

Recommended sequence for switched vendor rows:

1. prefixed filename (existing 4 case combinations)
2. existing non-prefixed filename logic (current code block)

For non-switched vendors:

1. existing non-prefixed filename logic only

---

## 5. ABAP rules compliance mapping

### 5.1 `03-database.mdc`

- No `SELECT *` for newly added queries (`ZLOG_EXEC_VAR`, `LIKP`, `LIPS`, `T001W`).
- Avoid `SELECT` in invoice loop for derivation logic.
- Use `FOR ALL ENTRIES` only with non-initial driver.
- `SORT` before `READ ... BINARY SEARCH`.

### 5.2 `02-naming.mdc`

Use prefixes:

- `lv_` variable, `lt_` table, `lw_` work area
- `lty_` local type
- `<lfs_...>` field-symbol
- `lc_` constants

### 5.3 `13-sy-subrc.mdc`

Mandatory immediate checks after:

- `SELECT`
- `READ TABLE`
- `CALL FUNCTION`
- `OPEN DATASET`
- `SUBMIT`

### 5.4 `17-reports-structure.mdc`

No report structural change in this CCD. Existing report remains unchanged.

### 5.5 `12-documentation.mdc`

- Maintain change-history block in FM header.
- Add concise comments for new decision points (prefix switch and fallback).

---

## 6. Exact code-level implementation guide (ABAP copy/paste)

This section provides concrete insertion points and ABAP blocks for direct implementation in FM `Z_LOG_FTP_DMS`.

### 6.1 Block A - Add new type definition (in existing `TYPES` area)

Insert this after current local `TYPES` declarations:

```abap
         BEGIN OF lty_exec_pdf_pref,
           name    TYPE zlog_exec_var-name,
           lifnr   TYPE zlog_exec_var-lifnr,
           bukrs   TYPE zlog_exec_var-bukrs,
           rfcdest TYPE zlog_exec_var-rfcdest,
           active  TYPE zlog_exec_var-active,
         END OF lty_exec_pdf_pref.
```

### 6.2 Block B - Add new data declarations (in existing `DATA` area)

Insert in local data declarations:

```abap
  DATA: lt_exec_pdf_pref   TYPE TABLE OF lty_exec_pdf_pref,
        lw_exec_pdf_pref   TYPE lty_exec_pdf_pref,
        lv_prefix_enabled  TYPE abap_bool,
        lv_prefix          TYPE zlog_exec_var-rfcdest,
        lv_pref_file_opened TYPE abap_bool,
        lv_bukrs           TYPE bukrs.
```

If not already present in this FM, also add BL->BUKRS map structures/tables:

```abap
  TYPES: BEGIN OF lty_bl_bukrs,
           bl_no TYPE zslinvstat-bl_no,
           bukrs TYPE bukrs,
         END OF lty_bl_bukrs.

  DATA: lt_bl_bukrs TYPE TABLE OF lty_bl_bukrs,
        lw_bl_bukrs TYPE lty_bl_bukrs.
```

### 6.3 Block C - Add constant (in existing `CONSTANTS` block)

Insert in constants:

```abap
             lc_name_pdf_pref TYPE rvari_vnam VALUE 'ZSCE_MSC_RECRPVEND_PDF'.
```

### 6.4 Block D - Prefetch switch rows (insert before main invoice loop)

Insert this before `LOOP AT lt_slinvstat ASSIGNING <lfs_slinvstat>.`

```abap
  CLEAR: lt_exec_pdf_pref, lv_prefix_enabled.

  SELECT name
         lifnr
         bukrs
         rfcdest
         active
    FROM zlog_exec_var CLIENT SPECIFIED
    INTO TABLE lt_exec_pdf_pref
    WHERE mandt  = sy-mandt
      AND name   = lc_name_pdf_pref
      AND lifnr  = i_lifnr
      AND active = abap_true.
  IF sy-subrc = 0 AND lt_exec_pdf_pref IS NOT INITIAL.
    lv_prefix_enabled = abap_true.
    SORT lt_exec_pdf_pref BY lifnr bukrs.
  ELSE.
    CLEAR lv_prefix_enabled.
  ENDIF.
```

### 6.5 Block E - Build BL->BUKRS map only for switched vendor (insert after Block D)

Insert this only once before invoice loop:

```abap
  IF lv_prefix_enabled = abap_true.
    " Reuse existing project logic pattern from RECR FM:
    " BL_NO -> LIKP-BOLNR -> LIPS-VBELN -> T001W-WERKS -> BUKRS
    " Build lt_bl_bukrs as sorted table by BL_NO.
    " IMPORTANT: use set-based SELECTs and avoid SELECT in invoice loop.
  ENDIF.
```

Note: ABAP team should reuse the already documented BL->BUKRS block from `Z_LOG_FTP_DMS_RECR` to keep behavior consistent.

### 6.6 Block F - Prefix decision logic (insert at start of each invoice iteration)

Insert near beginning of current invoice loop, before current `OPEN DATASET` block:

```abap
    CLEAR: lv_pref_file_opened, lv_prefix, lv_bukrs, lw_exec_pdf_pref.

    IF lv_prefix_enabled = abap_true.
      READ TABLE lt_bl_bukrs INTO lw_bl_bukrs
           WITH KEY bl_no = <lfs_slinvstat>-bl_no
           BINARY SEARCH.
      IF sy-subrc = 0.
        lv_bukrs = lw_bl_bukrs-bukrs.

        READ TABLE lt_exec_pdf_pref INTO lw_exec_pdf_pref
             WITH KEY lifnr = i_lifnr
                      bukrs = lv_bukrs
             BINARY SEARCH.
        IF sy-subrc = 0.
          lv_prefix = lw_exec_pdf_pref-rfcdest.
        ENDIF.
      ENDIF.
    ENDIF.
```

### 6.7 Block G - Prefixed filename attempt (insert immediately before existing as-is `OPEN DATASET` logic)

Add prefixed file attempt only when `lv_prefix` is available:

```abap
    IF lv_prefix IS NOT INITIAL.
      CLEAR p_file.
      TRANSLATE lw_value2 TO UPPER CASE.
      CONCATENATE lw_value1 lw_value2 '/' lv_prefix <lfs_slinvstat>-tr_billno text-001
             INTO p_file.
      CONDENSE p_file.
      OPEN DATASET p_file FOR INPUT IN BINARY MODE.
      IF sy-subrc = 0.
        lv_pref_file_opened = abap_true.
      ELSE.
        CLEAR p_file.
        TRANSLATE lw_value2 TO LOWER CASE.
        CONCATENATE lw_value1 lw_value2 '/' lv_prefix <lfs_slinvstat>-tr_billno text-002
               INTO p_file.
        CONDENSE p_file.
        OPEN DATASET p_file FOR INPUT IN BINARY MODE.
        IF sy-subrc = 0.
          lv_pref_file_opened = abap_true.
        ELSE.
          CLEAR p_file.
          TRANSLATE lw_value2 TO UPPER CASE.
          CONCATENATE lw_value1 lw_value2 '/' lv_prefix <lfs_slinvstat>-tr_billno text-002
                 INTO p_file.
          CONDENSE p_file.
          OPEN DATASET p_file FOR INPUT IN BINARY MODE.
          IF sy-subrc = 0.
            lv_pref_file_opened = abap_true.
          ELSE.
            CLEAR p_file.
            TRANSLATE lw_value2 TO LOWER CASE.
            CONCATENATE lw_value1 lw_value2 '/' lv_prefix <lfs_slinvstat>-tr_billno text-001
                   INTO p_file.
            CONDENSE p_file.
            OPEN DATASET p_file FOR INPUT IN BINARY MODE.
            IF sy-subrc = 0.
              lv_pref_file_opened = abap_true.
            ENDIF.
          ENDIF.
        ENDIF.
      ENDIF.
    ENDIF.
```

### 6.8 Block H - Fallback trigger (wrap existing as-is open block)

Wrap the current non-prefixed `OPEN DATASET` multi-attempt block:

```abap
    IF lv_pref_file_opened IS INITIAL.
      " existing as-is file open logic (unchanged)
    ENDIF.
```

Important: keep the existing block unchanged inside this wrapper so current RIL behavior remains intact.

### 6.9 Block I - No changes after file-open success

All existing downstream steps remain unchanged:

- binary read loop
- DMS create/check-in
- OCR handling
- `ZSLINVSTAT` update
- optional `ZSCM_SFTP_ARCH_DMS` submit

### 6.10 Developer self-check (must pass before transport)

- [ ] Vendor not configured -> existing code path only
- [ ] Configured 5155/RE_ works
- [ ] Configured 5184/RP_ works
- [ ] Prefix missing -> fallback non-prefix works
- [ ] `SY-SUBRC` checks added immediately after each new `SELECT`, `READ TABLE`, `OPEN DATASET`
- [ ] No `SELECT *` in newly added code
- [ ] Existing behavior unchanged for RIL vendors not in switch table

---

## 7. Regression safety controls

1. Vendor gate:
   - no active vendor row -> existing behavior.
2. BUKRS gate:
   - no matching `LIFNR + BUKRS` row -> existing behavior.
3. Prefix open fallback:
   - prefixed file missing -> existing behavior.

This ensures minimum impact on RIL and non-configured vendors.

---

## 8. Unit/System test checklist for ABAP + QA

| # | Test case | Expected |
|---|---|---|
| 1 | Vendor not in `ZLOG_EXEC_VAR` switch | Existing code path only |
| 2 | Vendor+BUKRS in switch with `RE_` | Prefixed file read |
| 3 | Vendor+BUKRS in switch with `RP_` | Prefixed file read |
| 4 | Prefix missing, non-prefix exists | Fallback success |
| 5 | Prefix configured, BUKRS derive fail | Existing code path |
| 6 | Both paths fail | Existing skip behavior |
| 7 | Existing RIL regression run | No functional change |

Quality gates (per `08-testing.mdc`):

- SCI: 0 errors/warnings
- SLIN: 0 errors/warnings
- no dead code / breakpoints
- `SY-SUBRC` checks present for all required statements

---

## 9. Transport and deployment notes

- Single transport can include FM enhancement and text updates.
- No DDIC table change required.
- Config rows in `ZLOG_EXEC_VAR` must be maintained before go-live.
- Rollback approach: deactivate config rows (`ACTIVE` blank) to return to pure as-is behavior.

---

## 10. Sign-off

| Role | Name | Sign | Date |
|---|---|---|---|
| Functional lead |  |  |  |
| ABAP developer |  |  |  |
| ABAP reviewer |  |  |  |
| QA lead |  |  |  |

