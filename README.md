
Final Methodology (Fully Updated)

This version includes all logic actually executed in the script.


---

Methodology

Objective

The purpose of this analysis is to identify Designated School Officials (DSOs) and Principal Designated School Officials (PDSOs) affiliated with a target list of schools, and to determine whether their names match a provided list of last names. In addition, the analysis determines vetted windows for these officials (initial and last vetted dates) using adjudication-related statuses, event codes, and school-level event activity, following a structured fallback hierarchy.


---

Data Sources

1. Oracle SEVIS database (SEVDBA schema)

SEV_SCHOOL — master school table.

SEV_SCHOOL_EVENT and SEV_SCHOOL_EVENT_DETAIL — historical event tables containing adjudication-related actions used to derive vetted windows.

SEV_USER_ROLE — assignments of users to schools and their roles (DSO/PDSO).

SEV_DOS_USER_ADJ — adjudication history table providing authoritative names for DSOs/PDSOs.

SEV_USER — backup source for user names when adjudication names are not available.

SEV_LOGIN — provides USER_NAME for additional matching and display fallback.


2. Reference Inputs (constructed as inline CTEs)

Target Schools: list of school names supplied by the audit team.

Target Last Names: list of last names to match against DSO/PDSO names.



---

Processing Steps

1. Input Setup

Two inline SQL tables (CTEs) are created directly from R inputs:

target_schools: all school names being audited.

last_names: last names supplied for matching.
Apostrophes are escaped for SQL safety.



---

2. School Selection

The school_ids CTE filters SEV_SCHOOL to only the schools in target_schools.

This is an exact-match join on SCHOOL_NAME.



---

3. Event Status and Code Windows

The script defines adjudication-relevant statuses and vetting codes used to infer vetted dates:

Vetting Statuses (APP_STATUS_CODE):

Used to indicate various stages of adjudication and vetting:

READYADJ – case prepared and ready for adjudication (post-vetting prechecks).

UNDREVBYADJ – case under active review by an adjudicator.

APPROVED – adjudication/vetting is complete and approved.


Vetting Event Codes (EVENT_CODE):

READYADJ

REINSADJD


These codes similarly indicate adjudication or re-adjudication processing.

Using these definitions, evt_school computes the following for each school:

Status-based windows: earliest and latest events where APP_STATUS_CODE is one of the three vetting statuses.

Event-code windows: earliest and latest events where EVENT_CODE indicates vetting.

Any-event windows: earliest and latest SEVIS event timestamps as a final fallback.


These create school-level vetted windows used when user-level vetted windows cannot be established.


---

4. Roster Construction

roster_raw selects all DSO/PDSO assignments (SEV_USER_ROLE) for the target schools.

roster deduplicates by:

preferring PDSO over DSO,

keeping the most recent timestamp per user per school.



This results in one official per school per user.


---

5. Name Resolution

Three data sources provide names:

1. adj_name (SEV_DOS_USER_ADJ): latest adjudicated name (authoritative).


2. u_name (SEV_USER): fallback name.


3. name_pick: joins roster to these two sources, preferring adjudicated names, then SEV_USER names.



This ensures the best available identity information for each DSO/PDSO.


---

6. User-Level Vetted Windows

Two vetted windows are computed per (school, user):

a) user_evt_status

Captures earliest and latest vetted dates using vetting statuses (APPROVED, READYADJ, UNDREVBYADJ).
Events are linked to users via:

SEV_SCHOOL_EVENT_DETAIL.CREATED_BY, or

SEV_SCHOOL_EVENT.CREATED_BY (fallback).


b) user_evt_code

Same logic, but uses vetting event codes (READYADJ, REINSADJD).

If available, these provide the most precise user-level vetted windows.


---

7. Login Reference

login_name brings in USER_NAME for display and for additional matching if last names do not produce matches.



---

8. Last-Name Matching

The script performs flexible matching of DSO/PDSO names against the provided last-name list:

1. normalize names by lowercasing and removing punctuation (REGEXP_REPLACE).


2. substring match using Oracle INSTR.


3. For users matching multiple last names, best_match selects the best candidate by:

preferring PDSO over DSO,

earlier substring position in the normalized name,

lexical ordering of last name.




This ensures one definitive last-name match per user.


---

9. Final Selection and Vetted Date Calculation

For each matched DSO/PDSO, the script outputs:

school name

full name (LNAME, FNAME; fallback to USER_NAME)

role (DSO/PDSO)

matched last name

initial and last vetted dates, computed using a five-tier fallback hierarchy:


Vetted Window Priority Chain

1. User-level vetted dates by status (user_evt_status)


2. User-level vetted dates by event code (user_evt_code)


3. School-level status window (evt_school.min/max_status_date)


4. School-level code window (evt_school.min/max_code_date)


5. School-level any-event window (evt_school.min/max_any_date)



This ensures a vetted window is produced even if some levels are missing.


---

10. Validation and Export

The script produces:

Main Output

A timestamped CSV (dso_vetting_YYYYMMDD_HHMMSS.csv) containing:

one row per matched DSO/PDSO

full school/identity details

vetted windows derived from the fallback chain


Validation SQL

A separate query checks completeness and matching accuracy by:

testing both last-name and username match sources,

recording match detail,

evaluating coverage for all school × last-name combinations,

identifying missing matches.


This produces:

1. validation_match_detail_*


2. validation_coverage_*


3. validation_missing_*



These ensure auditability and confirm no school/last-name combinations were unintentionally excluded.


---

Outputs

1. Main Report (dso_vetting_*.CSV)

One row per matched DSO/PDSO

School

Full name

Role

Matched last name

Initial vetted date

Last vetted date


2. Validation Reports

Match detail: all matches by source

Coverage: completeness of matching

Missing: unmatched school × last-name pairs


These files provide full transparency for auditors.
