# Data-collection platform metadata

Read this at Step 3 once the collection platform is known. Each platform exports a set of system fields alongside the questionnaire variables. Most carry no analytical value and several are identifiers. Field names below are the defaults in each platform's export; form authors can rename some, so treat the list as a starting point and confirm against the metadata log.

Disposition codes: **drop** (identifier or no analytical value), **coarsen** (keep at daily resolution or as a round indicator), **keep** (harmless and sometimes useful), **decide** (depends on the study).

## Contents

- [SurveyCTO](#surveycto)
- [ODK and KoBoToolbox](#odk-and-kobotoolbox)
- [Survey Solutions](#survey-solutions)
- [CSPro](#cspro)
- [Qualtrics and other web survey tools](#qualtrics-and-other-web-survey-tools)
- [Generic CAPI patterns](#generic-capi-patterns)
- [Form files that may hold PII](#form-files-that-may-hold-pii)
- [Paradata and audit files](#paradata-and-audit-files)

---

## SurveyCTO

Stata export via the SurveyCTO Stata template or the desktop app.

| Field | What it is | Disposition |
|---|---|---|
| `deviceid` | Tablet or phone hardware ID | drop |
| `subscriberid` | SIM subscriber (IMSI) | drop |
| `simid` | SIM serial | drop |
| `devicephonenum` | Phone number of the device | drop |
| `username` | Enumerator login | drop |
| `key` | Submission UUID (`uuid:...`) | drop, unless needed to link media or audit files; then treat as an identifier and keep only in Y |
| `instancename` | Form-author-defined submission label, often built from respondent name or ID | drop |
| `formdef_version` | Form version number | keep or drop; useful for tracking questionnaire changes, harmless |
| `caseid` | Case-management ID, often the household ID | keep if it is the analysis ID; check whether it encodes geography |
| `submissiondate` | Server receipt timestamp | coarsen to date, or drop |
| `starttime`, `endtime` | Interview timestamps | coarsen to date; keep `duration` instead |
| `duration` | Interview length in seconds | keep |
| `text_audit`, `audio_audit`, `audio_audit_*` | Paths to audit files | drop |
| `comments` | Enumerator free-text comments | redact or drop |
| `review_status`, `review_comments`, `review_corrections`, `review_quality` | Back-office review fields | drop; comments may contain names |
| `<name>latitude`, `<name>longitude`, `<name>altitude`, `<name>accuracy` | GPS capture split into four numeric fields | drop below analysis level; the accuracy field alone is harmless |
| `<name>` for `image`, `audio`, `video`, `file`, `signature` types | Media file names, often containing the UUID or a respondent identifier | drop |
| `sensor_statistics`, `sensor_stream` fields | Light, movement, sound levels | drop |

Structural patterns:

- `select_multiple` questions expand to one dummy per choice: `var_1`, `var_2`, ..., with negative choice codes written as `var__99` (double underscore). Not PII.
- Repeat groups export with `_1`, `_2`, ... suffixes on the variable name, or as separate long-format files linked by `key` and `setofrepeat`. If the roster is in a separate file, `key` becomes the merge key and must be retained until the merge is done in Y, then dropped.
- `calculate` fields have whatever name the form author gave them; common prefixes are `calc_`, `c_`, `check_`. Inspect the label.
- `note` fields export as empty variables. Safe to drop.

---

## ODK and KoBoToolbox

Exports from ODK Central, KoBoToolbox, or legacy ODK Aggregate. Column names use `/` for groups in raw exports (`meta/instanceID`) and `_` or `-` in some templates.

| Field | What it is | Disposition |
|---|---|---|
| `start`, `end`, `today` | Timestamps and date | coarsen to date |
| `deviceid`, `subscriberid`, `simserial`, `phonenumber` | Device and SIM identifiers | drop |
| `username`, `email` | Enumerator account | drop |
| `meta/instanceID`, `_uuid`, `instanceID` | Submission UUID | drop, unless needed to link attachments; then Y only |
| `meta/instanceName`, `_instanceName` | Form-author-defined label | drop |
| `meta/deprecatedID` | Previous UUID after edits | drop |
| `formhub/uuid`, `__version__`, `_version_` | Form identifiers | keep or drop; harmless |
| `_id` | Server-side integer submission ID | drop, or keep as a technical key if nothing else links repeats |
| `_submission_time`, `_submitted_by`, `_status`, `_validation_status`, `_notes`, `_tags` | Server metadata | drop; `_notes` and `_validation_status` may contain reviewer text |
| `_index`, `_parent_index`, `_parent_table_name` | Keys linking repeat-group tables | keep until merged in Y, then drop |
| `_geolocation` | Latitude and longitude as a list | drop |
| `<name>_latitude`, `<name>_longitude`, `<name>_altitude`, `<name>_precision` | Geopoint split into columns | drop below analysis level |
| `_attachments` | Media metadata (filenames, URLs) | drop |
| `audit` | Reference to the audit log CSV | drop |

Structural patterns: `select_multiple` expands to `var/choice` or `var_choice` dummies. Repeat groups export as separate sheets or CSVs linked by `_index` and `_parent_index`.

---

## Survey Solutions

Exports from the World Bank Survey Solutions server, in Stata, SPSS, or tab-delimited format.

| Field | What it is | Disposition |
|---|---|---|
| `interview__id` | Interview GUID | drop, unless needed to link roster files; then Y only |
| `interview__key` | Human-readable interview key (`12-34-56-78`) | decide; often written on paper cover sheets, and it is the key to the identified record on the server. Often the only link between roster files and the main file (see below) |
| `assignment__id` | Assignment number | drop |
| `interview__status`, `has__errors` | Workflow state | drop |
| `sssys_irnd` | Random number generated per interview | keep or drop; harmless |
| `responsible` | Interviewer or supervisor login | drop |
| `interview__duration` | Length | keep |
| `<name>__Latitude`, `<name>__Longitude`, `<name>__Accuracy`, `<name>__Altitude`, `<name>__Timestamp` | GPS question split into columns | drop below analysis level |
| `<roster>__id` | Roster item index | keep; structural, not PII |

Separate files in the export: `interview__actions` (who did what when, with `responsible__name` and `role`), `interview__comments` (free text by field, with the responsible's login), `interview__errors`, `interview__diagnostics`, and the paradata archive. None of these should leave Y.

Structural patterns: multi-select questions export as `var__1`, `var__2`, ...; list questions as `var__0`, `var__1`, ...; rosters as separate files linked by `interview__id` / `interview__key` and `<roster>__id`.

Traps seen in practice:

- **Rosters carry no household ID**, only the interview keys. To drop or replace the keys, build a crosswalk in Y from the main file (interview key to household ID, plus an interview number if the household ID is not unique, e.g. revisits or replacements) and merge it onto every roster file before the keys are touched.
- **Text-list questions** (a list of household members, plots, enterprises) export as `var__0` ... `var__N` string slots in the main file and reappear as the roster title variable in the roster file. Unused slots hold `##N/A##`. All slots are name fields.
- **Single-select questions linked to a preloaded list** (pick the respondent's name from the assignment's household list) export as a numeric code whose value label is the name list, often keyed by household ID. Treat the label as a full ID-to-name crosswalk.
- **Rosters with one row per interview and blank keys.** A roster file can hold one row per interview, with the roster ID and `interview__key` blank on rows that have no roster data (seen in a third-party cleaned export: 26 filled rows out of 2,178). A merge on `interview__key` then fails for most rows. Use the metadata log's ID block to find the key that is complete in every file (here `interview__id`), merge on it, assert that the other key agrees where present, and fill it from the main file. Allow missing roster IDs in that file's uniqueness check.
- **Exports cleaned by a third party** (survey firm, evaluator) may add login and server-link variables (`responsible`, `sprvsr`, `enum_*`, `intlink`, `inthist`), enumerator-name labels on helper variables (`temp_enum`), and secondary files with encoded copies of the ID.

---

## CSPro

CSPro has no fixed system-field set; the dictionary defines everything. Look for:

- Operator or interviewer ID items, usually in the questionnaire cover section, often with a lookup to a staff roster.
- Timestamp items added by the application (`START_TIME`, `END_TIME`, or similar).
- Case notes and field notes, exported separately or as a notes item. Free text; redact or drop.
- The case ID items, which in census-style applications concatenate geography codes. Check whether the analysis ID encodes location.
- GPS items if the application captured them.
- Partial-save and verification flags. Safe to drop.

Ask the researcher for the dictionary listing rather than guessing names.

---

## Qualtrics and other web survey tools

Qualtrics CSV or SPSS export. Other web tools (SurveyMonkey, LimeSurvey, Google Forms) follow the same pattern with different names.

| Field | What it is | Disposition |
|---|---|---|
| `IPAddress` | Respondent IP | drop |
| `LocationLatitude`, `LocationLongitude` | IP-derived coordinates | drop |
| `RecipientFirstName`, `RecipientLastName`, `RecipientEmail`, `ExternalReference` | Panel or contact-list fields | drop; `ExternalReference` may be the study ID, in which case keep it and check what else it links to |
| `ResponseId` | Response UUID | drop unless it is the only key |
| `StartDate`, `EndDate`, `RecordedDate` | Timestamps | coarsen to date |
| `Duration (in seconds)` | Length | keep |
| `Status`, `Progress`, `Finished`, `DistributionChannel`, `UserLanguage` | Workflow | keep or drop; harmless |
| Embedded-data fields | Whatever the survey designer passed through the link (often name, email, phone) | inspect each one |

---

## Generic CAPI patterns

Regardless of platform, the following appear in most exports and are safe to drop for an internal file:

- Helper and calculated variables: `calc_*`, `c_*`, `chk_*`, `check_*`, `flag_*`, `err_*`.
- Display-only and skip-logic markers: `note_*`, `n_*`, `intro_*`, `sec*_start`, `sec*_end`, `mod*_marker`.
- Section timing: `dur_*`, `time_*`, `*_start`, `*_end`.
- Consent readback text fields (the consent *response* is analytically relevant and stays).
- Duplicate-check variables comparing to a preload (`pl_name`, `preload_*`), which often contain the preloaded respondent name. Drop.

Keep: consent indicator, interview outcome or result code, language of interview, survey round or wave, replacement or attrition flags, and `duration`.

---

## Form files that may hold PII

Read the form's question structure (SKILL.md Step 0, "Reading the CAPI form"), but not these, which often hold place, group, staff or respondent names, or ID-to-name crosswalks:

| Platform | Files or parts to skip |
|---|---|
| SurveyCTO | `choices` sheet lists for geography, groups, staff and respondents; attached preload CSVs (`pulldata()`, `search()`); server datasets used for case management |
| ODK, KoBoToolbox | `choices` sheet lists as above; `external_choices` sheet and `itemsets.csv`; attached CSVs used by `pulldata()` or `select_one_from_file` |
| Survey Solutions | `Categories/` (reusable option lists) and `Lookup Tables/` in the Designer export; option lists inside `document.json` for linked or geography questions |
| CSPro | Lookup files, external dictionaries, and value sets for geography or staff |

When in doubt, have a script parse the form in Y and write only question types, variable names, question text and relevance conditions to X.

---

## Paradata and audit files

Text audits, audio audits, sensor streams, Survey Solutions paradata, and ODK audit logs record when each field was answered, sometimes with GPS traces and audio. They are valuable for data-quality research but are a separate confidential dataset. Do not move them to X as part of de-identification. If a data-quality analysis needs them, treat that as its own de-identification task with its own metadata log.
