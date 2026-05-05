# Code Creation Document — Recron / RPCM DMS (AL11)

| Field | Value |
|--------|--------|
| **Purpose** | Single document with **paste-ready ABAP** for `ZLOG_FTP_DMS_RECR` and `Z_LOG_FTP_DMS_RECR` |
| **Functional spec** | `FS_ZLOG_FTP_DMS_RECR.md` (v1.1) |
| **Technical reference** | `Tech_Spec_ZLOG_FTP_DMS_RECR.md` |
| **ABAP rules** | `ABAP Rules - 02-04-2026/` (NetWeaver 7.31) |
| **Transaction** | `ZSL_DMS_RECR` (SE93 → report `ZLOG_FTP_DMS_RECR`) |

---

## Instructions for the ABAP developer

1. **Function group:** Create (or use) a function group; activate **function module** `Z_LOG_FTP_DMS_RECR` using **§2** source (SE37 → create module → paste include / main source per your FG layout).  
2. **Report:** Create program `ZLOG_FTP_DMS_RECR` as **executable report**; paste **§1** source in the main program (single include is acceptable).  
3. **Text elements:** Maintain report texts per **§3**; maintain FM texts **text-001** / **text-002** per **§4**.  
4. **SE93:** Create `ZSL_DMS_RECR` as report transaction for `ZLOG_FTP_DMS_RECR`.  
5. **Roles:** Grant `S_TCODE` / `TCD` = `ZSL_DMS_RECR`.  
6. **SCI:** This FM still contains **`SELECT *`** on `ZSLINVSTAT` and **`SELECT SINGLE *`** on `ZDMSAPP` (inherited pattern). Replace with **named-field** `SELECT` + matching local types per `03-database.mdc` before production hardening.

**Activation order:** Function group + FM → Report → SE93.

---

## §1 — Report `ZLOG_FTP_DMS_RECR` (complete source, one include)

**Design:** Local class `lcl_report` holds all logic (no `FORM`/`PERFORM`). Selection screen remains global. Matches `FS_ZLOG_FTP_DMS_RECR.md` and `17-reports-structure.mdc`.

```abap
*&---------------------------------------------------------------------*
*& Report  ZLOG_FTP_DMS_RECR
*&---------------------------------------------------------------------*
*& DMS creation for Recron / RPCM invoices from AL11 (PDF).
*& Calls Z_LOG_FTP_DMS_RECR. BUKRS derived in FM. T-code ZSL_DMS_RECR.
*&---------------------------------------------------------------------*
REPORT zlog_ftp_dms_recr LINE-SIZE 40.

DATA: gt_slinvstat TYPE STANDARD TABLE OF zslinvstat,
      gw_slinvstat TYPE zslinvstat,
      gw_trbillno  TYPE ztrstlmnt_m_a-tr_billno,
      gt_trbillno  TYPE zlog_trbillno_tt,
      gt_strblno   TYPE RANGE OF ztrbillno,
      gw_strblno   LIKE LINE OF gt_strblno.

SELECTION-SCREEN BEGIN OF BLOCK a WITH FRAME TITLE text-000.
PARAMETERS:     p_lifnr TYPE lifnr OBLIGATORY.
SELECT-OPTIONS: s_trblno FOR gw_trbillno NO INTERVALS.
SELECTION-SCREEN END OF BLOCK a.

*----------------------------------------------------------------------*
* Local class — report logic (ABAP Rules: OOP for new reports)
*----------------------------------------------------------------------*
CLASS lcl_report DEFINITION FINAL.
  PUBLIC SECTION.
    CLASS-METHODS:
      authority_check,
      validate_vendor,
      execute.
ENDCLASS.

CLASS lcl_report IMPLEMENTATION.

  METHOD authority_check.
    CONSTANTS lc_tcode TYPE char20 VALUE 'ZSL_DMS_RECR'.
    AUTHORITY-CHECK OBJECT 'S_TCODE'
         ID 'TCD' FIELD lc_tcode.
    IF sy-subrc <> 0.
      MESSAGE e726(zlog) WITH sy-uname.
    ENDIF.
  ENDMETHOD.

  METHOD validate_vendor.
    DATA: lv_lifnr TYPE lifnr.
    SELECT SINGLE lifnr
      FROM lfa1
      INTO lv_lifnr
      WHERE lifnr = p_lifnr.
    IF sy-subrc <> 0.
      MESSAGE text-003 TYPE 'E'.
    ENDIF.
  ENDMETHOD.

  METHOD execute.
    CLEAR: gt_trbillno, gt_slinvstat.

    IF s_trblno[] IS NOT INITIAL.
      LOOP AT s_trblno INTO gw_strblno.
        gw_trbillno = gw_strblno-low.
        APPEND gw_trbillno TO gt_trbillno.
        CLEAR gw_trbillno.
      ENDLOOP.
      SORT gt_trbillno.
      DELETE gt_trbillno WHERE tr_billno IS INITIAL.
      DELETE ADJACENT DUPLICATES FROM gt_trbillno COMPARING tr_billno.
    ENDIF.

    CALL FUNCTION 'Z_LOG_FTP_DMS_RECR'
      EXPORTING
        i_lifnr      = p_lifnr
        it_tr_billno = gt_trbillno[]
      TABLES
        e_zslinvstat = gt_slinvstat.

    IF sy-batch = 'X'.
      LOOP AT gt_slinvstat INTO gw_slinvstat.
        IF sy-tabix = 1.
          WRITE: sy-uline.
          WRITE: / sy-vline, 2 text-001, 22 sy-vline, 24 text-002, 40 sy-vline.
          WRITE: sy-uline.
        ENDIF.
        WRITE: / sy-vline, 2 gw_slinvstat-tr_billno, 22 sy-vline,
                 24 gw_slinvstat-tr_bill_dt, 40 sy-vline.
        WRITE: sy-uline.
      ENDLOOP.
    ELSE.
      IF gt_slinvstat IS NOT INITIAL.
        MESSAGE text-004 TYPE 'S'.
      ELSE.
        MESSAGE text-005 TYPE 'E'.
      ENDIF.
    ENDIF.
  ENDMETHOD.

ENDCLASS.

*----------------------------------------------------------------------*
* Report events
*----------------------------------------------------------------------*
INITIALIZATION.
  lcl_report=>authority_check( ).

AT SELECTION-SCREEN ON p_lifnr.
  lcl_report=>validate_vendor( ).

START-OF-SELECTION.
  lcl_report=>execute( ).
```

---

## §2 — Function module `Z_LOG_FTP_DMS_RECR` (complete source)

Paste into the function module’s main include (replace interface-generated `FUNCTION`/`ENDFUNCTION` wrapper if your system generated an empty shell — keep the **same** interface as below).

**Interface (SE37):**

- Import: `I_LIFNR` TYPE `LIFNR`  
- Import: `IT_TR_BILLNO` TYPE `ZLOG_TRBILLNO_TT` (optional)  
- Tables: `E_ZSLINVSTAT` LIKE `ZSLINVSTAT` (optional)

```abap
FUNCTION z_log_ftp_dms_recr.
*"----------------------------------------------------------------------
*"*"Local Interface:
*"  IMPORTING
*"     VALUE(I_LIFNR) TYPE  LIFNR
*"     VALUE(IT_TR_BILLNO) TYPE  ZLOG_TRBILLNO_TT OPTIONAL
*"  TABLES
*"      E_ZSLINVSTAT STRUCTURE  ZSLINVSTAT OPTIONAL
*"----------------------------------------------------------------------
*-----------------------------------------------------------------------
* CHANGE HISTORY
*-----------------------------------------------------------------------
*SrNo| Date       | User ID  | Description               | Change Label
*-----------------------------------------------------------------------
* 1  | DD.MM.YYYY | <UserID> | New FM: Recron/RPCM DMS   | <CD Number>
*-----------------------------------------------------------------------

*-----------------------------------------------------------------------
* Local Type Definitions
*-----------------------------------------------------------------------
  TYPES: BEGIN OF lty_ts_raw_line,
           line TYPE orblk,
         END OF lty_ts_raw_line,

         BEGIN OF lty_zlog_stlmntvr,
           name   TYPE zlog_stlmntvr-name,
           active TYPE zlog_stlmntvr-active,
         END OF lty_zlog_stlmntvr,

         BEGIN OF lty_bl_bukrs,
           bl_no TYPE zbl_no,
           bukrs TYPE bukrs,
         END OF lty_bl_bukrs,

         BEGIN OF lty_likp,
           vbeln TYPE vbeln_vl,
           bolnr TYPE bolnr,
         END OF lty_likp,

         BEGIN OF lty_lips,
           vbeln TYPE vbeln_vl,
           werks TYPE werks_d,
         END OF lty_lips,

         BEGIN OF lty_t001w,
           werks TYPE werks_d,
           bukrs TYPE bukrs,
         END OF lty_t001w,

         BEGIN OF lty_exec_var,
           name TYPE rvari_vnam,
           numb TYPE tvarv_numb,
         END OF lty_exec_var.

*-----------------------------------------------------------------------
* Data Declarations
*-----------------------------------------------------------------------
  DATA: lt_slinvstat         TYPE TABLE OF zslinvstat,
        lw_slinvstat         LIKE LINE OF lt_slinvstat,
        lt_slinvstat_bl      TYPE TABLE OF zslinvstat,
        lw_slinvstat_bl      LIKE LINE OF lt_slinvstat_bl,
        v_excel_string(2000) TYPE c,
        p_file               LIKE v_excel_string,
        ln                   LIKE sy-tabix,
        lt_files2            TYPE cvapi_tbl_doc_files,
        lw_message           TYPE messages,
        lw_files2            TYPE cvapi_doc_file,
        ln_size              TYPE i.

  DATA: lw_zdmsapp              TYPE zdmsapp,
        lw_drat                 TYPE bapi_doc_drat,
        lt_drat                 TYPE TABLE OF bapi_doc_drat,
        lt_drad                 TYPE TABLE OF bapi_doc_drad,
        lt_classallocations     TYPE TABLE OF bapi_class_allocation,
        lt_characteristicvalues TYPE TABLE OF bapi_characteristic_values,
        lw_doc                  TYPE bapi_doc_draw2,
        lw_drao                 TYPE drao,
        lw_api_control          TYPE cvapi_api_control,
        lt_drao                 TYPE TABLE OF drao,
        lw_objectid             TYPE string,
        lw_draw                 TYPE draw,
        lw_doctype              TYPE bapi_doc_draw2-documenttype,
        lw_docnumber            TYPE bapi_doc_draw2-documentnumber,
        lw_docpart              TYPE bapi_doc_draw2-documenttype,
        lw_docversion           TYPE bapi_doc_draw2-documentversion,
        lw_return               TYPE bapiret2,
        lw_strippedname         TYPE draw-filep,
        lw_offset1              TYPE i.

  DATA: lt_tab            TYPE STANDARD TABLE OF lty_ts_raw_line,
        lw_bindata        LIKE LINE OF lt_tab,
        lt_tab2           TYPE z_orblk_tt,
        lw_tab            TYPE lty_ts_raw_line,
        lw_value1         TYPE zyttspara-value1,
        lw_value2         TYPE zyttspara-value2,
        lw_zlogocrinvoice TYPE zlogocrinvoice.

  DATA: lt_stlmntvr TYPE TABLE OF lty_zlog_stlmntvr,
        lw_failure  TYPE char1,
        lt_ocr_err  TYPE TABLE OF zsce_ocr_err,
        lw_ocr_err  TYPE zsce_ocr_err.

  DATA: lt_bl_bukrs TYPE TABLE OF lty_bl_bukrs,
        lw_bl_bukrs TYPE lty_bl_bukrs,
        lt_likp     TYPE TABLE OF lty_likp,
        lw_likp     TYPE lty_likp,
        lt_lips     TYPE TABLE OF lty_lips,
        lw_lips     TYPE lty_lips,
        lt_t001w    TYPE TABLE OF lty_t001w,
        lw_t001w    TYPE lty_t001w,
        lv_bukrs    TYPE bukrs.

  DATA: lw_exec_var TYPE lty_exec_var.

  CONSTANTS: lc_name_read     TYPE rvari_vnam VALUE 'Z_SCE_OCR_READ_ACTIVE',
             lc_name_check    TYPE rvari_vnam VALUE 'Z_SCE_OCR_CHECK_ACTIVE',
             lc_name_stat_upd TYPE rvari_vnam VALUE 'Z_SCE_OCR_STAT_UPD',
             gc_e             TYPE char1      VALUE 'E',
             gc_o             TYPE char1      VALUE 'O',
             lc_name          TYPE rvari_vnam VALUE 'ZSCM_SFTP_ARCH_DMS'.

  FIELD-SYMBOLS: <lfs_slinvstat> LIKE LINE OF lt_slinvstat.

*=======================================================================
* STEP 1: Fetch Pending Invoices from ZSLINVSTAT
*=======================================================================
  CLEAR lt_ocr_err.

  IF it_tr_billno IS INITIAL.
    SELECT *
      FROM zslinvstat
      INTO TABLE lt_slinvstat
      WHERE lifnr    = i_lifnr
        AND zupdflag IN ('C','O').
    IF sy-subrc EQ 0.
      SORT lt_slinvstat.
    ELSE.
      RETURN.
    ENDIF.

  ELSE.
    SORT it_tr_billno BY tr_billno.
    DELETE ADJACENT DUPLICATES FROM it_tr_billno COMPARING tr_billno.
    DELETE it_tr_billno WHERE tr_billno IS INITIAL.

    IF it_tr_billno IS NOT INITIAL.
      SELECT *
        FROM zslinvstat
        INTO TABLE lt_slinvstat
        FOR ALL ENTRIES IN it_tr_billno
        WHERE lifnr    = i_lifnr
          AND tr_billno = it_tr_billno-tr_billno.
      IF sy-subrc EQ 0.
        DELETE lt_slinvstat WHERE zupdflag NE 'C' AND zupdflag NE 'O'.
        SORT lt_slinvstat BY tr_billno tr_bill_dt DESCENDING.
        DELETE ADJACENT DUPLICATES FROM lt_slinvstat COMPARING tr_billno.
      ENDIF.
    ENDIF.

    IF lt_slinvstat IS INITIAL.
      RETURN.
    ENDIF.

  ENDIF.

*=======================================================================
* STEP 2: Fetch DMS Application Configuration from ZDMSAPP
*=======================================================================
  SELECT SINGLE * FROM zdmsapp INTO lw_zdmsapp
    WHERE applcode = 'ZVI'.
  IF sy-subrc NE 0.
    RETURN.
  ENDIF.

*=======================================================================
* STEP 3: Derive BUKRS per BL Number (bulk, before main loop)
*=======================================================================
  lt_slinvstat_bl[] = lt_slinvstat[].
  SORT   lt_slinvstat_bl BY bl_no.
  DELETE ADJACENT DUPLICATES FROM lt_slinvstat_bl COMPARING bl_no.
  DELETE lt_slinvstat_bl WHERE bl_no IS INITIAL.

  IF lt_slinvstat_bl IS NOT INITIAL.

    SELECT vbeln bolnr
      FROM likp
      INTO TABLE lt_likp
      FOR ALL ENTRIES IN lt_slinvstat_bl
      WHERE bolnr = lt_slinvstat_bl-bl_no.
    IF sy-subrc EQ 0.
      SORT lt_likp BY vbeln.
      DELETE ADJACENT DUPLICATES FROM lt_likp COMPARING vbeln.

      SELECT vbeln werks
        FROM lips
        INTO TABLE lt_lips
        FOR ALL ENTRIES IN lt_likp
        WHERE vbeln = lt_likp-vbeln.
      IF sy-subrc EQ 0.
        SORT lt_lips BY vbeln.

        SELECT werks bukrs
          FROM t001w
          INTO TABLE lt_t001w
          FOR ALL ENTRIES IN lt_lips
          WHERE werks = lt_lips-werks.
        IF sy-subrc EQ 0.
          SORT lt_t001w BY werks.
        ENDIF.
      ENDIF.
    ENDIF.

    SORT lt_likp BY bolnr.

    LOOP AT lt_slinvstat_bl INTO lw_slinvstat_bl.
      CLEAR: lw_bl_bukrs, lw_likp, lw_lips, lw_t001w.

      READ TABLE lt_likp INTO lw_likp
           WITH KEY bolnr = lw_slinvstat_bl-bl_no
           BINARY SEARCH.
      IF sy-subrc NE 0.
        CONTINUE.
      ENDIF.

      READ TABLE lt_lips INTO lw_lips
           WITH KEY vbeln = lw_likp-vbeln
           BINARY SEARCH.
      IF sy-subrc NE 0.
        CONTINUE.
      ENDIF.

      READ TABLE lt_t001w INTO lw_t001w
           WITH KEY werks = lw_lips-werks
           BINARY SEARCH.
      IF sy-subrc NE 0.
        CONTINUE.
      ENDIF.

      lw_bl_bukrs-bl_no = lw_slinvstat_bl-bl_no.
      lw_bl_bukrs-bukrs = lw_t001w-bukrs.
      APPEND lw_bl_bukrs TO lt_bl_bukrs.

    ENDLOOP.

    SORT lt_bl_bukrs BY bl_no.

  ENDIF.

*=======================================================================
* STEP 4: Main Processing Loop
*=======================================================================
  LOOP AT lt_slinvstat ASSIGNING <lfs_slinvstat>.

    CLEAR: p_file,        lw_doc,       lw_objectid,   lw_offset1,
           lw_doctype,    lw_docnumber, lw_docpart,    lw_strippedname,
           lw_docversion, lw_return,    lw_draw,       lw_message,
           lw_value1,     lw_value2,    lv_bukrs,      lw_bl_bukrs.

    REFRESH: lt_tab[], lt_drat, lt_drao, lt_files2.

    READ TABLE lt_bl_bukrs INTO lw_bl_bukrs
         WITH KEY bl_no = <lfs_slinvstat>-bl_no
         BINARY SEARCH.
    IF sy-subrc NE 0.
      CONTINUE.
    ENDIF.
    lv_bukrs = lw_bl_bukrs-bukrs.

    SELECT value1 value2 UP TO 1 ROWS
      FROM zyttspara
      INTO (lw_value1, lw_value2)
      WHERE param1      = 'Z_SL_PDF_INTF'
        AND param2      = i_lifnr
        AND param3      = lv_bukrs
        AND active_flag = abap_true.
    ENDSELECT.

    IF sy-subrc NE 0.
      CONTINUE.
    ENDIF.

    TRANSLATE lw_value1 TO LOWER CASE.

    TRANSLATE lw_value2 TO UPPER CASE.
    CONCATENATE lw_value1 lw_value2 '/' <lfs_slinvstat>-tr_billno
                text-001 INTO p_file.
    CONDENSE p_file.

    OPEN DATASET p_file FOR INPUT IN BINARY MODE.
    IF sy-subrc NE 0.
      CLEAR: p_file.
      TRANSLATE lw_value2 TO LOWER CASE.
      CONCATENATE lw_value1 lw_value2 '/' <lfs_slinvstat>-tr_billno
                  text-002 INTO p_file.
      CONDENSE p_file.
      OPEN DATASET p_file FOR INPUT IN BINARY MODE.

      IF sy-subrc NE 0.
        CLEAR: p_file.
        TRANSLATE lw_value2 TO UPPER CASE.
        CONCATENATE lw_value1 lw_value2 '/' <lfs_slinvstat>-tr_billno
                    text-002 INTO p_file.
        CONDENSE p_file.
        OPEN DATASET p_file FOR INPUT IN BINARY MODE.

        IF sy-subrc NE 0.
          CLEAR: p_file.
          TRANSLATE lw_value2 TO LOWER CASE.
          CONCATENATE lw_value1 lw_value2 '/' <lfs_slinvstat>-tr_billno
                      text-001 INTO p_file.
          CONDENSE p_file.
          OPEN DATASET p_file FOR INPUT IN BINARY MODE.

          IF sy-subrc NE 0.
            CONTINUE.
          ENDIF.
        ENDIF.
      ENDIF.
    ENDIF.

    CLEAR: ln_size, lt_tab[], lw_tab, lt_tab2.

    DO.
      CLEAR: ln, lw_tab.
      READ DATASET p_file INTO lw_tab LENGTH ln.
      IF sy-subrc <> 0.
        IF ln > 0.
          ln_size = ln_size + ln.
          APPEND lw_tab TO lt_tab.
          APPEND lw_tab-line TO lt_tab2.
        ENDIF.
        EXIT.
      ENDIF.
      ln_size = ln_size + ln.
      APPEND lw_tab TO lt_tab.
      APPEND lw_tab-line TO lt_tab2.
    ENDDO.
    CLOSE DATASET p_file.

    IF lt_tab IS NOT INITIAL.

      lw_doc-documenttype    = lw_zdmsapp-documenttype.
      lw_doc-documentversion = lw_zdmsapp-documentversion.
      lw_doc-documentpart    = lw_zdmsapp-documentpart.
      lw_doc-statusextern    = lw_zdmsapp-statusextern.
      lw_doc-laboratory      = '  '.
      lw_doc-statusintern    = 'CC'.

      CONCATENATE i_lifnr
                  <lfs_slinvstat>-tr_billno
                  <lfs_slinvstat>-tr_bill_dt+0(4)
                  INTO lw_objectid SEPARATED BY '/'.

      lw_drat-language    = sy-langu.
      lw_drat-description = lw_objectid.
      APPEND lw_drat TO lt_drat.
      CLEAR lw_drat.

      CALL FUNCTION 'BAPI_DOCUMENT_CREATE2'
        EXPORTING
          documentdata         = lw_doc
        IMPORTING
          documenttype         = lw_doctype
          documentnumber       = lw_docnumber
          documentpart         = lw_docpart
          documentversion      = lw_docversion
          return               = lw_return
        TABLES
          characteristicvalues = lt_characteristicvalues
          classallocations     = lt_classallocations
          documentdescriptions = lt_drat
          objectlinks          = lt_drad.

      IF lw_return-type NA 'EA'.

        CALL FUNCTION 'BAPI_TRANSACTION_COMMIT'
          EXPORTING
            wait = 'X'.

        lw_draw-dokar = lw_zdmsapp-documenttype.
        lw_draw-dokvr = lw_zdmsapp-documentversion.
        lw_draw-doktl = lw_zdmsapp-documentpart.
        lw_draw-dwnam = sy-uname.
        lw_draw-dokst = 'CC'.

        lw_api_control-tcode = 'CV01N'.

        LOOP AT lt_tab INTO lw_bindata.
          CLEAR lw_drao.
          lw_drao-orblk = lw_bindata-line.
          lw_drao-orln  = ln_size.
          lw_drao-dokar = lw_draw-dokar.
          lw_drao-doknr = lw_docnumber.
          lw_drao-dokvr = lw_draw-dokvr.
          lw_drao-doktl = lw_draw-doktl.
          lw_drao-appnr = '1'.
          APPEND lw_drao TO lt_drao.
          CLEAR lw_drao.
        ENDLOOP.

        TRANSLATE p_file TO LOWER CASE.
        FIND ALL OCCURRENCES OF '/' IN p_file MATCH OFFSET lw_offset1.
        IF lw_offset1 NE 0.
          lw_offset1      = lw_offset1 + 1.
          lw_strippedname = p_file+lw_offset1.
        ELSE.
          lw_strippedname = p_file.
        ENDIF.

        CALL FUNCTION 'CV120_DOC_GET_APPL'
          EXPORTING
            pf_file   = lw_strippedname
          IMPORTING
            pfx_dappl = lw_files2-dappl.

        lw_draw-filep = lw_strippedname.
        lw_draw-dappl = lw_files2-dappl.

        lw_files2-appnr       = '1'.
        lw_files2-filename    = lw_strippedname.
        lw_files2-updateflag  = 'I'.
        lw_files2-langu       = sy-langu.
        lw_files2-storage_cat = lw_zdmsapp-storagecategory.
        lw_files2-description = <lfs_slinvstat>-tr_billno.
        APPEND lw_files2 TO lt_files2.
        CLEAR lw_files2.

        CALL FUNCTION 'CVAPI_DOC_CHECKIN'
          EXPORTING
            pf_dokar           = lw_draw-dokar
            pf_doknr           = lw_docnumber
            pf_dokvr           = lw_draw-dokvr
            pf_doktl           = lw_draw-doktl
            ps_api_control     = lw_api_control
            pf_content_provide = 'TBL'
          IMPORTING
            psx_message        = lw_message
          TABLES
            pt_files_x         = lt_files2
            pt_content         = lt_drao.

        IF NOT lw_message-msg_type CA 'EA' AND
               lw_docnumber IS NOT INITIAL.

          CALL FUNCTION 'BAPI_TRANSACTION_COMMIT'
            EXPORTING
              wait = 'X'.

          <lfs_slinvstat>-zupdflag = 'U'.

          CLEAR lt_stlmntvr.
          SELECT name active INTO TABLE lt_stlmntvr
            FROM zlog_stlmntvr
            WHERE name IN (lc_name_read, lc_name_check, lc_name_stat_upd)
              AND lifnr  = i_lifnr
              AND active = abap_true.
          IF sy-subrc = 0.

            READ TABLE lt_stlmntvr TRANSPORTING NO FIELDS
                 WITH KEY name = lc_name_read.
            IF sy-subrc = 0.
              CLEAR: lw_failure, lw_zlogocrinvoice, lt_ocr_err.
              CALL FUNCTION 'Z_SCE_SL_OCR_INVOICE_READ'
                EXPORTING
                  im_lifnr         = i_lifnr
                  im_file          = lt_tab2
                IMPORTING
                  e_zlogocrinvoice = lw_zlogocrinvoice
                  e_read_failure   = lw_failure.

              IF lw_failure IS INITIAL.
                READ TABLE lt_stlmntvr TRANSPORTING NO FIELDS
                     WITH KEY name = lc_name_check.
                IF sy-subrc = 0.
                  CALL FUNCTION 'Z_SCE_SL_OCR_INVOICE_CHECK'
                    EXPORTING
                      im_lifnr          = i_lifnr
                      im_zlogocrinvoice = lw_zlogocrinvoice
                    IMPORTING
                      et_zsce_ocr_err   = lt_ocr_err.
                ENDIF.
              ELSE.
                CLEAR lw_ocr_err.
                lw_ocr_err-lifnr      = <lfs_slinvstat>-lifnr.
                lw_ocr_err-tr_bill_no = <lfs_slinvstat>-tr_billno.
                lw_ocr_err-tr_bill_dt = <lfs_slinvstat>-tr_bill_dt.
                lw_ocr_err-created_by = sy-uname.
                lw_ocr_err-created_on = sy-datum.
                lw_ocr_err-created_tm = sy-uzeit.
                lw_ocr_err-msg_type   = gc_e.
                lw_ocr_err-zocrstat   = gc_e.
                <lfs_slinvstat>-zupdflag = gc_o.
                MODIFY zsce_ocr_err FROM lw_ocr_err.
                IF sy-subrc = 0.
                  COMMIT WORK.
                ELSE.
                  ROLLBACK WORK.
                ENDIF.
              ENDIF.
            ENDIF.
          ENDIF.

          READ TABLE lt_stlmntvr TRANSPORTING NO FIELDS
               WITH KEY name = lc_name_stat_upd.
          IF sy-subrc = 0.
            READ TABLE lt_ocr_err TRANSPORTING NO FIELDS
                 WITH KEY msg_type = gc_e.
            IF sy-subrc = 0.
              <lfs_slinvstat>-zupdflag = gc_o.
            ENDIF.
          ENDIF.

        ELSE.
          CALL FUNCTION 'BAPI_TRANSACTION_ROLLBACK'.
        ENDIF.

      ELSE.
        CALL FUNCTION 'BAPI_TRANSACTION_ROLLBACK'.
      ENDIF.

    ENDIF.

  ENDLOOP.

*=======================================================================
* STEP 5: Final Bulk Update of ZSLINVSTAT
*=======================================================================
  DELETE lt_slinvstat WHERE zupdflag NE 'U' AND zupdflag NE 'O'.

  IF lt_slinvstat IS NOT INITIAL.

    LOOP AT lt_slinvstat INTO lw_slinvstat.
      CALL FUNCTION 'ENQUEUE_EZSLINVSTAT'
        EXPORTING
          lifnr          = lw_slinvstat-lifnr
          tr_billno      = lw_slinvstat-tr_billno
          tr_bill_dt     = lw_slinvstat-tr_bill_dt
        EXCEPTIONS
          foreign_lock   = 1
          system_failure = 2
          OTHERS         = 3.
    ENDLOOP.

    UPDATE zslinvstat FROM TABLE lt_slinvstat.

    LOOP AT lt_slinvstat INTO lw_slinvstat.
      CALL FUNCTION 'DEQUEUE_EZSLINVSTAT'
        EXPORTING
          lifnr      = lw_slinvstat-lifnr
          tr_billno  = lw_slinvstat-tr_billno
          tr_bill_dt = lw_slinvstat-tr_bill_dt.
    ENDLOOP.

*=======================================================================
* STEP 6: SFTP Archival / File Deletion (optional)
*=======================================================================
    SELECT name numb FROM zlog_exec_var CLIENT SPECIFIED
      INTO lw_exec_var UP TO 1 ROWS
      WHERE mandt  = sy-mandt
        AND name   = lc_name
        AND active = abap_true.
    ENDSELECT.
    IF sy-subrc EQ 0.
      SUBMIT zscm_sftp_file_del
        WITH p_lifnr = i_lifnr
        WITH s_inv   IN it_tr_billno.
      RETURN.
    ENDIF.

  ENDIF.

  e_zslinvstat[] = lt_slinvstat.

ENDFUNCTION.
```

---

## §3 — Report text elements (`ZLOG_FTP_DMS_RECR`)

| ID | Text (English example) |
|----|-------------------------|
| `000` | Selection Criteria |
| `001` | Bill Number |
| `002` | Bill Date |
| `003` | Vendor does not exist |
| `004` | DMS created successfully |
| `005` | No records processed |

---

## §4 — Function module texts (`Z_LOG_FTP_DMS_RECR`)

| ID | Text |
|----|------|
| `text-001` | `.PDF` |
| `text-002` | `.pdf` |

---

## §5 — Related documents

| Document | Path |
|----------|------|
| Functional Spec | `FS_ZLOG_FTP_DMS_RECR.md` |
| Technical Spec (full narrative + legacy notes) | `Tech_Spec_ZLOG_FTP_DMS_RECR.md` |
| Code change / compliance checklist | `CCD_ZLOG_FTP_DMS_RECR.md` |

---

## Document control

| Version | Date | Description |
|---------|------|-------------|
| 1.0 | — | Initial code creation package: OOP report + full FM body |
