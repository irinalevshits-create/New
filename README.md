
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
