chooseCRANmirror()

##install.packages("C:/Users/levshitsi/Downloads/rJava_1.0-11.zip", repos = NULL, type = "win.binary")
##install.packages("RJDBC", repos = "https://cloud.r-project.org")
library(rJava)
library(DBI)
library(RJDBC)
library(stringr)

## 0) Connection (same as yours)
drv <- JDBC("oracle.jdbc.OracleDriver",
            classPath = "C:/oracle/instantclient_23_6/ojdbc8.jar")
con <- dbConnect(drv, "jdbc:oracle:oci:@sevrpprd_ssl", "ilevsh9709", "ntkp_!0822xhMBHil")

## 1) Inputs (same as yours)
target_schools <- c(
  "DG Zenith Academy Inc. DBA: ACES Learning Center",
  "Aston International Academy, Inc.",
  "Austin Presbyterian Theological Seminary",
  "Computer Systems Institute",
  "Delta Aeronautics, Inc DBA Miloud Aviation, LLC",
  "Galveston College",
  "HOFT Institute",
  "High Expectations LLC",
  "IVY Christian College",
  "Pilot's Choice Aviation, Inc.",
  "Shepherd School of Language",
  "The University of Texas Medical Branch, Galveston",
  "Virginia University of Integrative Medicine",
  "Virginia University of Science & Technology",
  "Collin college",
  "Collin County Community College District",
  "COLLIN COUNTY COMMUNITY COLLEGE",
  "boston international academy",
  "Boston International Academy",
  "Texas A & M University",
  "Texas A&M University",
  "Texas A&M"
)

last_names <- c(
  "Li","Qi","Quan","Nerove","Mural","Herrera","Irizarry","Zdancewicz","Loo","Medina",
  "McClellen","Mcclellen","Lowder","Horuk","Chansed","Dimitrov","Dzedzels","Ernst","Jobity","Kantor",
  "Rojchanapradist","Rungruengthitikul","Turek","Zou","Miloud","Grogg","Markum","Branum",
  "Wiste","Martin","Roark","Leonard","Snyder","Malik","Bourassa","Roberts","Malik","Yoo","McClellan",
  "Yun  Kim","Min  Kim","Sun  Yoo","Jenkins","Kim","Soderstrom","Kim","Brinlee","milo",
  "Kuttuva","Morgan","Egeland","Dayton","Kimbell","Lee","Spiller","Hilgemeier","Rosales",
  "Ferraro","Roberts","Duncan","Shaffer","Mintz","Fan","Jentoft-Moreno","Heine","Clement",
  "Li","Rydl","Flores","Hale","Bowen","Sanders","Boeh","Ma","Ong","Ma","Sun","Jentoft-Moreno","Jentoft", "Moreno", "Min", "Kim Min"
)

## 2) Inline CTEs for Oracle (kept as before)
cte_targets <- paste(
  sprintf("SELECT '%s' AS SCHOOL_NAME FROM dual", gsub("'", "''", target_schools)),
  collapse = "\nUNION ALL\n"
)

cte_lastnames <- paste(
  sprintf("SELECT '%s' AS LN FROM dual", gsub("'", "''", last_names)),
  collapse = "\nUNION ALL\n"
)

## 3) SQL:
## - Uses SEV_LOGIN for USER_NAME (fallback for display only)
## - Builds school-level and user-level vetted windows (status/code/any)
## - Matches against provided last_names using adjudication LNAME, then SEV_USER.LNAME
## - Keeps PDSO over DSO for duplicate user/school combos
sql <- sprintf("
WITH
target_schools AS (
  %s
),
last_names AS (
  %s
),

/* Vetted-by-code and status sets (tune if needed) */
vet_event_codes(code) AS (
  SELECT 'READYADJ' FROM dual UNION ALL
  SELECT 'REINSADJD' FROM dual
),
evt_statuses(code) AS (
  SELECT 'APPROVED' FROM dual UNION ALL
  SELECT 'READYADJ' FROM dual UNION ALL
  SELECT 'UNDREVBYADJ' FROM dual
),

/* School IDs limited to input schools */
school_ids AS (
  SELECT s.SCHOOL_ID, s.SCHOOL_NAME
  FROM SEVDBA.SEV_SCHOOL s
  JOIN target_schools t
    ON s.SCHOOL_NAME = t.SCHOOL_NAME
),

/* School-level windows (status/code/any) for fallback */
evt_school AS (
  SELECT
    s.SCHOOL_ID,
    MIN(CASE WHEN UPPER(e.APP_STATUS_CODE) IN (SELECT code FROM evt_statuses)
             THEN NVL(e.EVENT_DATE, e.CREATION_DATE) END) AS min_status_date,
    MAX(CASE WHEN UPPER(e.APP_STATUS_CODE) IN (SELECT code FROM evt_statuses)
             THEN NVL(e.EVENT_DATE, e.CREATION_DATE) END) AS max_status_date,
    MIN(CASE WHEN e.EVENT_CODE IN (SELECT code FROM vet_event_codes)
             THEN NVL(e.EVENT_DATE, e.CREATION_DATE) END) AS min_code_date,
    MAX(CASE WHEN e.EVENT_CODE IN (SELECT code FROM vet_event_codes)
             THEN NVL(e.EVENT_DATE, e.CREATION_DATE) END) AS max_code_date,
    MIN(NVL(e.EVENT_DATE, e.CREATION_DATE)) AS min_any_date,
    MAX(NVL(e.EVENT_DATE, e.CREATION_DATE)) AS max_any_date
  FROM school_ids s
  JOIN SEVDBA.SEV_SCHOOL_EVENT e ON e.SCHOOL_ID = s.SCHOOL_ID
  GROUP BY s.SCHOOL_ID
),

/* Raw roster: DSO/PDSO for target schools */
roster_raw AS (
  SELECT
    ur.SCHOOL_ID,
    si.SCHOOL_NAME,
    ur.USER_ID,
    UPPER(ur.ROLE_CODE) AS ROLE_CODE,
    NVL(ur.LAST_UPDATE_DATE, ur.CREATION_DATE) AS ROLE_TS
  FROM SEVDBA.SEV_USER_ROLE ur
  JOIN school_ids si ON si.SCHOOL_ID = ur.SCHOOL_ID
  WHERE UPPER(ur.ROLE_CODE) IN ('DSO','PDSO')
),

/* Dedup: prefer PDSO, most recent row */
roster AS (
  SELECT *
  FROM (
    SELECT rr.*,
           ROW_NUMBER() OVER (
             PARTITION BY rr.SCHOOL_ID, rr.USER_ID
             ORDER BY CASE rr.ROLE_CODE WHEN 'PDSO' THEN 1 WHEN 'DSO' THEN 2 ELSE 99 END,
                      rr.ROLE_TS DESC NULLS LAST
           ) AS rn
    FROM roster_raw rr
  )
  WHERE rn = 1
),

/* Names from adjudication as authoritative, latest by timestamp */
adj_name AS (
  SELECT
    COALESCE(a.USER_ID, a.ADJ_USER_ID) AS USER_ID,
    MAX(a.FNAME) KEEP (DENSE_RANK LAST ORDER BY NVL(a.LAST_UPDATE_DATE,a.CREATION_DATE)) AS FNAME,
    MAX(a.LNAME) KEEP (DENSE_RANK LAST ORDER BY NVL(a.LAST_UPDATE_DATE,a.CREATION_DATE)) AS LNAME
  FROM SEVDBA.SEV_DOS_USER_ADJ a
  GROUP BY COALESCE(a.USER_ID, a.ADJ_USER_ID)
),

/* Backup names from SEV_USER when adjudication names are missing */
u_name AS (
  SELECT u.USER_ID, u.FNAME, u.LNAME
  FROM SEVDBA.SEV_USER u
),

/* User display name parts: prefer adj_name, else SEV_USER */
name_pick AS (
  SELECT
    r.SCHOOL_ID,
    r.SCHOOL_NAME,
    r.USER_ID,
    r.ROLE_CODE,
    COALESCE(n.LNAME, u.LNAME) AS LNAME,
    COALESCE(n.FNAME, u.FNAME) AS FNAME
  FROM roster r
  LEFT JOIN adj_name n ON n.USER_ID = r.USER_ID
  LEFT JOIN u_name  u  ON u.USER_ID = r.USER_ID
),

/* USER-LEVEL vetted windows (preferred): by status */
user_evt_status AS (
  SELECT
    e.SCHOOL_ID,
    ur.USER_ID,
    MIN(e.CREATION_DATE)                          AS user_initial_vetted_date,
    MAX(NVL(e.LAST_UPDATE_DATE, e.CREATION_DATE)) AS user_last_vetted_date
  FROM school_ids s
  JOIN SEVDBA.SEV_SCHOOL_EVENT e              ON e.SCHOOL_ID = s.SCHOOL_ID
  LEFT JOIN SEVDBA.SEV_SCHOOL_EVENT_DETAIL ed ON ed.EVENT_ID = e.EVENT_ID
  JOIN SEVDBA.SEV_USER_ROLE ur
       ON ur.SCHOOL_ID = e.SCHOOL_ID
      AND ur.USER_ID   = NVL(ed.CREATED_BY, e.CREATED_BY)
      AND UPPER(ur.ROLE_CODE) IN ('DSO','PDSO')
  WHERE UPPER(e.APP_STATUS_CODE) IN (SELECT code FROM evt_statuses)
  GROUP BY e.SCHOOL_ID, ur.USER_ID
),

/* USER-LEVEL vetted windows (fallback): by event code */
user_evt_code AS (
  SELECT
    e.SCHOOL_ID,
    ur.USER_ID,
    MIN(e.CREATION_DATE)                          AS user_initial_vetted_date_by_code,
    MAX(NVL(e.LAST_UPDATE_DATE, e.CREATION_DATE)) AS user_last_vetted_date_by_code
  FROM school_ids s
  JOIN SEVDBA.SEV_SCHOOL_EVENT e              ON e.SCHOOL_ID = s.SCHOOL_ID
  LEFT JOIN SEVDBA.SEV_SCHOOL_EVENT_DETAIL ed ON ed.EVENT_ID = e.EVENT_ID
  JOIN SEVDBA.SEV_USER_ROLE ur
       ON ur.SCHOOL_ID = e.SCHOOL_ID
      AND ur.USER_ID   = NVL(ed.CREATED_BY, e.CREATED_BY)
      AND UPPER(ur.ROLE_CODE) IN ('DSO','PDSO')
  WHERE e.EVENT_CODE IN (SELECT code FROM vet_event_codes)
  GROUP BY e.SCHOOL_ID, ur.USER_ID
),

/* Bring in login for display fallback only (may be null) */
login_name AS (
  SELECT l.USER_ID, l.USER_NAME
  FROM SEVDBA.SEV_LOGIN l
),

/* ==== LAST-NAME MATCHING (based on provided list) ==== */
/* normalize and require a positive substring match in LNAME */
ln_matches AS (
  SELECT
    np.SCHOOL_ID,
    np.SCHOOL_NAME,
    np.USER_ID,
    np.ROLE_CODE,
    np.FNAME,
    np.LNAME,
    ln.LN AS MATCHED_LAST_NAME_INPUT,
    INSTR(
      REGEXP_REPLACE(LOWER(NVL(np.LNAME, '')), '[^[:alpha:]]', ''),
      REGEXP_REPLACE(LOWER(ln.LN),             '[^[:alpha:]]', '')
    ) AS MATCH_POS
  FROM name_pick np
  CROSS JOIN last_names ln
  WHERE INSTR(
          REGEXP_REPLACE(LOWER(NVL(np.LNAME, '')), '[^[:alpha:]]', ''),
          REGEXP_REPLACE(LOWER(ln.LN),             '[^[:alpha:]]', '')
        ) > 0
),

/* If a user matches multiple input last names, pick best: PDSO, earliest pos, then LNAME */
best_match AS (
  SELECT *
  FROM (
    SELECT lm.*,
           ROW_NUMBER() OVER (
             PARTITION BY lm.SCHOOL_ID, lm.USER_ID
             ORDER BY CASE lm.ROLE_CODE WHEN 'PDSO' THEN 1 ELSE 2 END,
                      lm.MATCH_POS,
                      lm.LNAME
           ) AS rn
    FROM ln_matches lm
  )
  WHERE rn = 1
)

SELECT
  bm.SCHOOL_NAME,
  COALESCE(
    CASE
      WHEN bm.LNAME IS NOT NULL AND bm.FNAME IS NOT NULL THEN bm.LNAME || ', ' || bm.FNAME
      WHEN bm.LNAME IS NOT NULL THEN bm.LNAME
      WHEN bm.FNAME IS NOT NULL THEN bm.FNAME
      ELSE NULL
    END,
    l.USER_NAME,
    '(no name)'
  ) AS FULL_NAME,
  bm.ROLE_CODE,
  bm.MATCHED_LAST_NAME_INPUT,
  bm.MATCH_POS AS MATCH_POSITION_IN_LNAME,

  /* Vetted dates with robust fallback chain */
  COALESCE(ues.user_initial_vetted_date,
           uec.user_initial_vetted_date_by_code,
           es.min_status_date,
           es.min_code_date,
           es.min_any_date) AS INITIAL_VETTED_DT,

  COALESCE(ues.user_last_vetted_date,
           uec.user_last_vetted_date_by_code,
           es.max_status_date,
           es.max_code_date,
           es.max_any_date) AS LAST_VETTED_DT

FROM best_match bm
LEFT JOIN login_name    l   ON l.USER_ID    = bm.USER_ID
LEFT JOIN evt_school    es  ON es.SCHOOL_ID = bm.SCHOOL_ID
LEFT JOIN user_evt_status ues
       ON ues.SCHOOL_ID = bm.SCHOOL_ID AND ues.USER_ID = bm.USER_ID
LEFT JOIN user_evt_code   uec
       ON uec.SCHOOL_ID = bm.SCHOOL_ID AND uec.USER_ID = bm.USER_ID
ORDER BY bm.SCHOOL_NAME, FULL_NAME
", cte_targets, cte_lastnames)

## 4) Execute
final_report <- dbGetQuery(con, sql)

## 5) Quick validation (lightweight; safe if connection is flaky)
message(sprintf("Rows returned: %s", nrow(final_report)))
na_init  <- sum(is.na(final_report$INITIAL_VETTED_DT))
na_last  <- sum(is.na(final_report$LAST_VETTED_DT))
message(sprintf("Rows with NA INITIAL_VETTED_DT: %s", na_init))
message(sprintf("Rows with NA LAST_VETTED_DT: %s", na_last))

## Show a quick sample (including any with missing dates) to help diagnose
if (nrow(final_report) > 0) {
  print(utils::head(final_report[order(is.na(final_report$INITIAL_VETTED_DT),
                                       is.na(final_report$LAST_VETTED_DT),
                                       final_report$SCHOOL_NAME), ], 25))
}

## ================================
## 6) VALIDATION & EXPORTS (bolt-on)
## ================================

## --- Remove the two columns from the final export only ---
final_report$MATCHED_LAST_NAME_INPUT <- NULL
final_report$MATCH_POSITION_IN_LNAME <- NULL

## 6a) Export main report (timestamped)
## Force the requested file name: dso_vetting_20250926_152252.CSV
timestamp <- "20250926_152252"
out_dir   <- getwd()
main_csv  <- file.path(out_dir, sprintf("dso_vetting_%s.CSV", timestamp))

# Format dates for Excel-friendliness
df_out <- final_report
date_cols <- intersect(c("INITIAL_VETTED_DT","LAST_VETTED_DT"), names(df_out))
for (cc in date_cols) {
  if (inherits(df_out[[cc]], "POSIXt")) {
    df_out[[cc]] <- format(df_out[[cc]], "%Y-%m-%d %H:%M:%S")
  } else if (inherits(df_out[[cc]], "Date")) {
    df_out[[cc]] <- format(df_out[[cc]], "%Y-%m-%d")
  }
}
write.csv(df_out, main_csv, row.names = FALSE, na = "")
message("Main CSV written: ", main_csv)

## 6b) Build validation SQL using the same inputs (NO changes to main SQL)
validation_sql <- sprintf("
WITH
target_schools AS ( %s ),
last_names     AS ( %s ),

/* School scope */
school_ids AS (
  SELECT s.SCHOOL_ID, s.SCHOOL_NAME
  FROM SEVDBA.SEV_SCHOOL s
  JOIN target_schools t ON t.SCHOOL_NAME = s.SCHOOL_NAME
),

/* Roster (DSO/PDSO), dedup prefer PDSO, most recent */
roster_raw AS (
  SELECT ur.SCHOOL_ID, si.SCHOOL_NAME, ur.USER_ID, UPPER(ur.ROLE_CODE) AS ROLE_CODE,
         NVL(ur.LAST_UPDATE_DATE, ur.CREATION_DATE) AS ROLE_TS
  FROM SEVDBA.SEV_USER_ROLE ur
  JOIN school_ids si ON si.SCHOOL_ID = ur.SCHOOL_ID
  WHERE UPPER(ur.ROLE_CODE) IN ('DSO','PDSO')
),
roster AS (
  SELECT * FROM (
    SELECT r.*,
           ROW_NUMBER() OVER (
             PARTITION BY r.SCHOOL_ID, r.USER_ID
             ORDER BY CASE r.ROLE_CODE WHEN 'PDSO' THEN 1 ELSE 2 END,
                      r.ROLE_TS DESC NULLS LAST
           ) rn
    FROM roster_raw r
  ) WHERE rn = 1
),

/* Names: prefer adjudication; fallback to SEV_USER */
adj_name AS (
  SELECT COALESCE(a.USER_ID, a.ADJ_USER_ID) AS USER_ID,
         MAX(a.FNAME) KEEP (DENSE_RANK LAST ORDER BY NVL(a.LAST_UPDATE_DATE,a.CREATION_DATE)) AS FNAME,
         MAX(a.LNAME) KEEP (DENSE_RANK LAST ORDER BY NVL(a.LAST_UPDATE_DATE,a.CREATION_DATE)) AS LNAME
  FROM SEVDBA.SEV_DOS_USER_ADJ a
  GROUP BY COALESCE(a.USER_ID, a.ADJ_USER_ID)
),
u_name AS (
  SELECT u.USER_ID, u.FNAME, u.LNAME
  FROM SEVDBA.SEV_USER u
),
name_pick AS (
  SELECT r.SCHOOL_ID, r.SCHOOL_NAME, r.USER_ID, r.ROLE_CODE,
         COALESCE(n.LNAME, u.LNAME) AS LNAME,
         COALESCE(n.FNAME, u.FNAME) AS FNAME
  FROM roster r
  LEFT JOIN adj_name n ON n.USER_ID = r.USER_ID
  LEFT JOIN u_name  u  ON u.USER_ID = r.USER_ID
),

/* USER_NAME from SEV_LOGIN for username matching */
login_name AS (
  SELECT l.USER_ID, l.USER_NAME FROM SEVDBA.SEV_LOGIN l
),

/********** LNAME matches **********/
lname_matches AS (
  SELECT np.SCHOOL_ID, np.SCHOOL_NAME, np.USER_ID, np.ROLE_CODE,
         np.FNAME, np.LNAME,
         NULL AS USER_NAME,
         ln.LN AS LAST_NAME_INPUT,
         INSTR(
           REGEXP_REPLACE(LOWER(NVL(np.LNAME,'')), '[^[:alpha:]]', ''),
           REGEXP_REPLACE(LOWER(ln.LN),            '[^[:alpha:]]', '')
         ) AS MATCH_POS,
         'LNAME' AS MATCH_SOURCE
  FROM name_pick np
  CROSS JOIN last_names ln
  WHERE INSTR(
          REGEXP_REPLACE(LOWER(NVL(np.LNAME,'')), '[^[:alpha:]]', ''),
          REGEXP_REPLACE(LOWER(ln.LN),            '[^[:alpha:]]', '')
        ) > 0
),

/********** USERNAME matches (SEV_LOGIN) **********/
uname_matches AS (
  SELECT np.SCHOOL_ID, np.SCHOOL_NAME, np.USER_ID, np.ROLE_CODE,
         np.FNAME, np.LNAME,
         l.USER_NAME,
         ln.LN AS LAST_NAME_INPUT,
         INSTR(
           REGEXP_REPLACE(LOWER(NVL(l.USER_NAME,'')), '[^[:alpha:]]', ''),
           REGEXP_REPLACE(LOWER(ln.LN),               '[^[:alpha:]]', '')
         ) AS MATCH_POS,
         'USERNAME' AS MATCH_SOURCE
  FROM name_pick np
  JOIN login_name l ON l.USER_ID = np.USER_ID
  CROSS JOIN last_names ln
  WHERE INSTR(
          REGEXP_REPLACE(LOWER(NVL(l.USER_NAME,'')), '[^[:alpha:]]', ''),
          REGEXP_REPLACE(LOWER(ln.LN),               '[^[:alpha:]]', '')
        ) > 0
),

matches_union AS (
  SELECT * FROM lname_matches
  UNION ALL
  SELECT * FROM uname_matches
),

/* Best per (school,user,last,source) → prefer PDSO, earliest pos, then LNAME */
best_per_source AS (
  SELECT * FROM (
    SELECT mu.*,
           ROW_NUMBER() OVER (
             PARTITION BY mu.SCHOOL_ID, mu.USER_ID, mu.LAST_NAME_INPUT, mu.MATCH_SOURCE
             ORDER BY CASE mu.ROLE_CODE WHEN 'PDSO' THEN 1 ELSE 2 END,
                      mu.MATCH_POS,
                      NVL(mu.LNAME,'~')
           ) rn
    FROM matches_union mu
  ) WHERE rn = 1
),

/* All school × last_name pairs */
school_last_pairs AS (
  SELECT si.SCHOOL_ID, si.SCHOOL_NAME, ln.LN AS LAST_NAME_INPUT
  FROM school_ids si CROSS JOIN last_names ln
),

/* Coverage flags per pair (LNAME and/or USERNAME) */
coverage AS (
  SELECT
    slp.SCHOOL_ID, slp.SCHOOL_NAME, slp.LAST_NAME_INPUT,
    CASE WHEN EXISTS (
           SELECT 1 FROM best_per_source bps
           WHERE bps.SCHOOL_ID = slp.SCHOOL_ID
             AND bps.LAST_NAME_INPUT = slp.LAST_NAME_INPUT
             AND bps.MATCH_SOURCE = 'LNAME'
         ) THEN 1 ELSE 0 END AS has_lname_match,
    CASE WHEN EXISTS (
           SELECT 1 FROM best_per_source bps
           WHERE bps.SCHOOL_ID = slp.SCHOOL_ID
             AND bps.LAST_NAME_INPUT = slp.LAST_NAME_INPUT
             AND bps.MATCH_SOURCE = 'USERNAME'
         ) THEN 1 ELSE 0 END AS has_username_match
  FROM school_last_pairs slp
)

SELECT
  'MATCH_DETAIL' AS ROWSET,
  bps.SCHOOL_NAME,
  bps.LAST_NAME_INPUT,
  bps.MATCH_SOURCE,
  bps.USER_ID,
  bps.ROLE_CODE,
  bps.FNAME,
  bps.LNAME,
  bps.USER_NAME,
  bps.MATCH_POS
FROM best_per_source bps

UNION ALL

SELECT
  'COVERAGE' AS ROWSET,
  c.SCHOOL_NAME,
  c.LAST_NAME_INPUT,
  CAST(NULL AS VARCHAR2(30)) AS MATCH_SOURCE,
  CAST(NULL AS NUMBER) AS USER_ID,
  CAST(NULL AS VARCHAR2(10)) AS ROLE_CODE,
  CAST(NULL AS VARCHAR2(200)) AS FNAME,
  CAST(NULL AS VARCHAR2(200)) AS LNAME,
  CAST(NULL AS VARCHAR2(200)) AS USER_NAME,
  (c.has_lname_match + 10*c.has_username_match) AS MATCH_POS
FROM coverage c

UNION ALL

SELECT
  'MISSING' AS ROWSET,
  c.SCHOOL_NAME,
  c.LAST_NAME_INPUT,
  CAST(NULL AS VARCHAR2(30)) AS MATCH_SOURCE,
  CAST(NULL AS NUMBER) AS USER_ID,
  CAST(NULL AS VARCHAR2(10)) AS ROLE_CODE,
  CAST(NULL AS VARCHAR2(200)) AS FNAME,
  CAST(NULL AS VARCHAR2(200)) AS LNAME,
  CAST(NULL AS VARCHAR2(200)) AS USER_NAME,
  CAST(NULL AS NUMBER) AS MATCH_POS
FROM coverage c
WHERE c.has_lname_match = 0 AND c.has_username_match = 0

ORDER BY 1, 2, 3
", cte_targets, cte_lastnames)

validation_df <- dbGetQuery(con, validation_sql)

## 6c) Split and export validation tables
val_match_detail <- subset(validation_df, ROWSET == "MATCH_DETAIL")
val_coverage     <- subset(validation_df, ROWSET == "COVERAGE")
val_missing      <- subset(validation_df, ROWSET == "MISSING")

# Decode coverage flags into separate columns (1 = yes, 0 = no)
if (nrow(val_coverage)) {
  val_coverage$HAS_LNAME    <- ifelse(val_coverage$MATCH_POS %% 10 == 1, 1, 0)
  val_coverage$HAS_USERNAME <- ifelse(val_coverage$MATCH_POS %/% 10 == 1, 1, 0)
  val_coverage$MATCH_POS <- NULL
}

# Export CSVs
md_csv <- file.path(out_dir, sprintf("validation_match_detail_%s.csv", timestamp))
cv_csv <- file.path(out_dir, sprintf("validation_coverage_%s.csv", timestamp))
ms_csv <- file.path(out_dir, sprintf("validation_missing_%s.csv", timestamp))

write.csv(val_match_detail, md_csv, row.names = FALSE, na = "")
write.csv(val_coverage,     cv_csv, row.names = FALSE, na = "")
write.csv(val_missing,      ms_csv, row.names = FALSE, na = "")

message("Validation CSVs written:\n  ",
        md_csv, "\n  ",
        cv_csv, "\n  ",
        ms_csv)

## (Optional) Disconnect
# try(dbDisconnect(con), silent = TRUE)


explain step by step how this data was retrived and what logic was used. check against methodology below:

Methodology

Objective

The purpose of this analysis is to identify Designated School Officials (DSOs) and Principal Designated School Officials (PDSOs) affiliated with a target list of schools, and to determine whether their names match against a provided list of last names. In addition, the analysis establishes vetted windows for these officials (initial and last vetted dates) using adjudication events, school events, and login information.

Data Sources

1. Oracle SEVIS database (SEVDBA schema)
•	SEV_SCHOOL — master school table
•	SEV_SCHOOL_EVENT and SEV_SCHOOL_EVENT_DETAIL — event history tables, used to derive vetted status windows
•	SEV_USER_ROLE — assignment of users to schools with roles (DSO/PDSO)
•	SEV_DOS_USER_ADJ — adjudication table for authoritative user names
•	SEV_USER — backup source of user names
•	SEV_LOGIN — login table for username matching

2. Reference inputs (script inline CTEs)
•	Target Schools: list of school names provided by the audit team.
•	Target Last Names: list of last names to be searched against adjudication/user login names.

Processing Steps

1. Input Setup
Two inline tables (CTEs) are created directly in SQL from the R vectors:
•	target_schools holds the provided school names.
•	last_names holds the provided last names.
Apostrophes are escaped to ensure proper parsing.

2. School Selection
•	school_ids filters SEVIS schools (SEV_SCHOOL) down to those matching the input list.
•	Joins are exact string matches on SCHOOL_NAME.

3. Event Status and Code Windows
•	Event codes (READYADJ, REINSADJD) and statuses (APPROVED, READYADJ, UNDREVBYADJ) are defined as reference sets.
•	evt_school aggregates all events for each target school:
o	Computes min/max dates where events had valid statuses.
o	Computes min/max dates where events had specific vetting codes.
o	Computes min/max dates for all events.
•	These serve as fallback vetted windows when user-level details are missing.

4. Roster Construction
•	roster_raw: pulls all user-role records (DSO, PDSO) for the target schools.
•	roster: deduplicates by:
o	Preferring PDSO over DSO.
o	Keeping the most recent record if multiple remain.

5. Name Resolution
•	adj_name: authoritative names from SEV_DOS_USER_ADJ, using the latest updated name per user.
•	u_name: backup names from SEV_USER.
•	name_pick: combines roster with names, preferring adjudication (adj_name), falling back to SEV_USER if missing.

6. User-Level Vetted Windows
•	Two separate windows are built:
o	user_evt_status: earliest/latest vetted dates from events with valid statuses.
o	user_evt_code: earliest/latest vetted dates from events with specific vetting codes.
•	Both link events to users through event details (SEV_SCHOOL_EVENT_DETAIL) or event creators.

7. Login Reference
•	login_name: provides USER_NAME for display and as an alternate matching field when last names do not match.

8. Last-Name Matching
•	ln_matches: compares normalized (lowercased, non-alphabetic stripped) LNAME against each input last name.
o	Uses substring matching via Oracle INSTR.
o	Allows for flexible detection even if names include spaces or punctuation.
•	If multiple last names match a user, best_match picks the best candidate using:
o	Prefer PDSO role.
o	Earliest substring position.
o	Lexical order of last name.

9. Final Selection
•	For each match, the query outputs:
o	School name, user’s full display name (last + first, fallback to login name).
o	Role (DSO/PDSO).
o	The input last name that matched.
o	Initial and last vetted dates, resolved with a robust fallback chain:
1. User-level vetted dates by status (user_evt_status)
2. User-level vetted dates by code (user_evt_code)
3. School-level status windows (evt_school)
4. School-level code windows (evt_school)
5. Any event windows (evt_school)

10. Validation and Export
•	The script exports the final dataset to a timestamped CSV (dso_vetting_YYYYMMDD_HHMMSS.csv).
•	A separate validation SQL is run to:
o	Test both last-name and login-based matches.
o	Check coverage of all school × last-name pairs.
o	Flag missing cases where no match is found.

•	Validation exports three files:
o	validation_match_detail (all positive matches).
o	validation_coverage (coverage matrix by school × last name).
o	validation_missing (school × last names with no matches).

Outputs

1. Main Report (dso_vetting_*.CSV)
o	One row per matched official.
o	Includes school, full name, role, matched last name, initial and last vetted dates

2. Validation Reports
Match detail, coverage, and missing reports provide QA and auditability of the match process.
