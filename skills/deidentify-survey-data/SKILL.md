---
name: deidentify-survey-data
description: Use when preparing a de-identified working copy of a survey or similar dataset (household, individual, farm, firm or facility survey, listing, panel, or administrative records) so it can leave restricted storage and be used in AI-assisted research workflows or shared within a research team. Trigger whenever the user mentions PII, personal identifiers, de-identification, anonymization, GPS or phone numbers in data, enumerator or respondent names, stripping value labels, or making data safe to use with Claude or other AI tools, even if they do not use the word de-identify. Claude never sees the identified data: it writes scripts the researcher runs. Software- and country-agnostic; complete, tested Stata recipes, partial R and Python skeletons.
---

# Survey data de-identification

## Purpose

Produce a de-identified copy of a dataset that can leave restricted storage and be used in AI-assisted research workflows.

Many institutions, ethics boards and data agreements do not allow personally identifiable information (PII) to be shared with AI tools. This skill removes what identifies a respondent so that the resulting working file can be analysed with Claude's help. It must also preserve the dataset's analytical value, and that matters as much as the first goal: the output is a working file the researcher will run their actual analysis on. Anything dropped out of caution rather than necessity is pure cost.

**Out of scope:** public release, repository deposit, and replication packages. Those need disclosure control on quasi-identifiers, a check of consent scope and data agreements, and documentation. If the researcher asks for that, say it is a separate task and stop at the working file.

**No guarantee; a human must check.** The procedure reduces risk; it does not certify that the output is anonymous. Automated rules miss names and other identifiers, especially in free text. The researcher remains responsible for reviewing the output and for complying with their institution's rules, their ethics approval, and their data agreements. **Before any file produced with this skill is shared beyond the research team, and always before public release, a person must check it for PII**; public release additionally needs the disclosure-control review described above. Open the first response with this text, verbatim, before anything else, and repeat it at the end of the final verification report:

> **Before we start:** this skill reduces the risk of exposing personal data but does not guarantee the output is anonymous; automated rules miss identifiers, especially names in free text. A person must check the output for PII before it is shared beyond your research team, and always before any public release. You remain responsible for complying with your institution's rules, ethics approval and data agreements.

In the verification report, replace "Before we start:" with "Reminder:".

## The setup

The identified data live in restricted storage (**Y**): a secure server, an encrypted drive, or an access-controlled folder. The project folder Claude works in (**X**) contains everything else. **Claude has no access to Y and never will.** It cannot read the data, inspect a sample, or check its own output against the source.

The workflow is two scripts, both crossing the boundary in the same direction:

1. `extract_metadata` reads data in Y and writes a metadata log to X.
2. `deidentify` reads data in Y and writes the de-identified files to X.

Claude writes both. The researcher runs both, in Y. Claude reads what lands in X.

Consequences throughout:

- **Never ask to see the data.** The metadata log is not a workaround; it is the only channel that exists.
- **Both scripts take two paths pointing in opposite directions.** Parameterize them at the top and confirm which is which. Writing output back into Y defeats the purpose; reading from X finds nothing.
- **Claude cannot test the scripts on the real data.** They run unattended against data Claude has never seen. They must fail loudly, and Claude tests them on synthetic data built from the metadata log before handing them over (both in Step 5).
- **Intermediate files stay in Y.** Temp files created during de-identification hold identified data.

## Data classification

Use the researcher's institutional classification if they have one; otherwise this common three-tier scheme:

- **Public**: published outputs. No restrictions.
- **Internal**: unpublished research, de-identified working files, metadata logs. Share within the team; do not release.
- **Restricted**: anything that identifies, locates, or contacts a specific person, household, firm, or facility, alone or in combination with other data. Lives in Y. **Must not be entered into or accessed by Claude.**

Fieldwork staff names (enumerators, supervisors, facilitators) sit at the boundary. They are not respondent data and their presence is usually not a breach. But they have no analytical value in the output and stripping them is free. Treat them as remove-by-default; staff *codes* without their name labels can stay (enumerator fixed effects).

**If Restricted values appear in anything Claude is given**, such as a pasted log or a file in X, stop processing that content. Do not repeat the values back. Say what appears to be there and ask whether it was cleared or whether a redacted version exists. Continue once the researcher confirms. The one designed exception is the free-text screen in Step 6; its limits are stated there.

---

## Step 0: Intake questionnaire

The first response is the verbatim disclaimer from the Purpose section plus the first round of a short questionnaire, and nothing else. **Administer it with the `AskUserQuestion` tool**, not as prose. All questions in one call are shown together, so a question can only be skipped on the strength of an earlier round. The tool takes at most four questions per call. Give each item three or four concrete options plus a "Don't know" option that names the fallback from the table below; the tool adds "Other" for free text automatically. Mark the default with "(Recommended)" where the table gives one. Fall back to a numbered list in prose only if the tool is unavailable.

Skip any item whose answer is already evident from the request or the project folder (a `.dta` mentioned, a country in the project name, a form file in the folder). Before the first round, look in the project folder for a `CLAUDE.md` or an existing `deidentify_intake.md`; if an intake note exists, confirm it instead of re-asking. Do not start writing scripts until the answers are in.

Read the answers literally. Option text plus any note the researcher typed is the answer; a note may qualify an option, request a change, or ask you not to proceed.

**Round 1** (none of these depends on the form):

1. **Software.** Which package are the data in: Stata, R, Python, SPSS, SAS, other? Determines which script recipe to use.
2. **The CAPI form.** Is the deployed electronic form available (SurveyCTO or ODK XLSForm, Survey Solutions `document.json` or Designer export, CSPro dictionary)? If yes, ask for its location.
3. **Country.** Which country or countries? Selects the entry in `references/countries.md`.
4. **Output shape.** One de-identified file per source file, or split by questionnaire module? If split, how are modules identified: variable prefixes, section-marker variables, or a list the researcher provides?

**Between rounds:** if the form is available, read its question structure now, following "Reading the CAPI form" below, and use it to drop the round-2 items it answers.

**Round 2** (skip what the form or the folder settles; often only items 7 and 8 remain; item 7 needs the country from round 1):

5. **Platform.** Which platform collected the data (SurveyCTO, ODK or KoBoToolbox, Survey Solutions, CSPro, paper, other)? Selects the entry in `references/platform-metadata.md`. Skip if the form shows it.
6. **File structure.** One file, or several (modules, rosters, waves)? If several, how do they link (ID variables)? Determines whether the scripts loop over files. Skip if the form's rosters and the researcher's description settle it.
7. **Place names.** Down to which administrative level may place **names** stay in the working file? Codes are kept at every level regardless. Build the options from the country's name-disclosure ladder (Step 3; country is known from round 1): one option per cut-off from the **keep** levels through the **ask** levels, plus "No place names at all" and "Don't know". Mark the ladder's default as "(Recommended)". For Ethiopia: "Region and zone only (Recommended)", "Down to woreda", "No place names at all". Do not offer levels on the **remove** rung; if the researcher asks for them through "Other", explain that they are PII, record the request and the warning in the flag list, and leave the decision to the researcher. Without a country entry, draft the ladder from the population rule first and build the options from the draft. The analysis or clustering level is not asked: record it in the flag list if the researcher mentions it or the design shows it.
8. **Linked files.** Is there a listing, sampling frame, or partner file that shares the household or individual ID and is or will be shared outside the team? Is this one wave of a panel whose IDs must stay consistent with another wave?

**Round 3** (one decision about free text, asked together):

9. **Free-text languages.** Which languages and scripts appear in open-ended fields? Are any fields untranslated?
10. **Free-text review mode.** Who reviews the free text for names the rules missed? Present the two options with their trade-offs (table below) and let the researcher choose. If item 9 reports untranslated non-Latin script, say in option A that the name rules are weaker on such text and that translating it first is recommended.

| Free-text review mode | How it works | For | Against |
|---|---|---|---|
| **A. Claude screens redacted text** (default) | After each run, the script writes the distinct values of every free-text variable from the *redacted* output to X, detached from IDs and row order. Claude reviews them and proposes patterns. In Y, the script also writes a suspicious-token report the researcher reads. | Raw text never reaches Claude; what Claude sees has already passed the name and phone rules. Fast iteration. | Names the rules missed do reach Claude, detached from IDs. |
| **B. Researcher screens, Claude never reads text** | The script writes the suspicious-token report and the distinct redacted values to Y only. The researcher reads them and reports problems in words ("a person's name in the comments field") or as patterns. | Strongest boundary. | Slower; depends on the researcher's time and attention; Claude cannot judge borderline strings. |

Do not offer a mode in which raw, unredacted free text is extracted to X: the raw "other, specify" and comment fields are where the names are, and detaching them from IDs does not make them non-identifying in small samples (Step 6).

**Not asked at intake: dates.** Default to keeping exact daily dates (interview date, date of birth) and dropping time of day. State the default in the flag list, where the researcher can ask for coarsening at Step 4.

Say up front, in the text preceding the first round, that "don't know" is an acceptable answer to everything except item 1. Record it and apply the fallback:

| Item | If the researcher doesn't know |
|---|---|
| 2. CAPI form | Proceed from the metadata log alone. |
| 3. Country | Needed for Step 6 patterns. If the country has no entry in `references/countries.md`, draft one from the template at Step 3 and ask the researcher to check the phone and ID formats before they are used. For multi-country data, run the checks per country. |
| 4. Output shape | One output file per source file. Splitting can be added later without redoing anything. |
| 5. Platform | The metadata log reveals it: `key` and `deviceid` mean SurveyCTO, `interview__id` means Survey Solutions, `_uuid` or `meta/instanceID` means ODK or KoBo. Identify it at Step 3 and say which entry you used. |
| 6. File structure | The metadata log settles it. Ask for the folder in Y; the extraction script loops over every file in it. |
| 7. Place names | The ladder's default: keep names on the **keep** rung, remove the rest. Without a country entry, draft the ladder from the population rule in Step 3 and confirm it with the researcher at Step 4. |
| 8. Linked files | Assume the IDs may be linkable. Keep them, flag that this was not confirmed, and ask again at Step 4. |
| 9. Free-text languages | The metadata log shows which fields are free text; the free-text screen reveals scripts and languages. Ask for translation only if non-Latin text appears. |
| 10. Review mode | Mode A. |

Write the answers, including any "don't know" and the fallback applied, to `<X>/deidentify_intake.md`. Like the flag list, this note is the durable record; the conversation will not be.

If the researcher answers only round 1 and wants to move on, proceed with the extraction script and ask rounds 2 and 3 before Step 3.

### Reading the CAPI form

The form is the most useful document after the metadata log. Question types identify free text, GPS, media, and list questions before any data are touched; a single-select question linked to a preloaded name list (a respondent picked from a roster of names) shows up as such in the form and nowhere else. It also tells Claude what each code means.

But form files can contain PII themselves. **Read only the question structure; never the answer lists or attached files.**

| Read | Do not read |
|---|---|
| Question type, variable name, question text, relevance and constraint expressions, group and roster structure | Answer-option lists that hold places, groups, people, or IDs: SurveyCTO and ODK `choices` sheet entries for such lists, `pulldata()` and preload CSVs, server datasets, ODK external itemsets; Survey Solutions `Categories/` and `Lookup Tables/`; CSPro lookup and value-set files for geography or staff |

In practice: parse the form programmatically and print only the read-side fields (a JSON or XLSX parser that skips the option lists), rather than opening the file. If the form's answer lists hold a code-to-village or ID-to-name crosswalk, that is exactly what the de-identified file withholds. If the researcher is unsure whether the form is clean, have the extraction script parse it inside Y and write the question-structure summary to X.

## Step 1: Write the metadata extraction script

Provide a complete runnable script for that package, as one block the researcher can save and run. Not prose describing what to do. Full scripts for Stata, R, and Python are in `references/metadata-extraction.md`; read it now and adapt the matching one. When the data are several files, use the multi-file version (Stata; for R or Python, wrap the single-file script in the same loop): it loops over every data file in the Y folder and its subfolders, writes one log per file plus an index, and skips platform system files that must not leave Y (Survey Solutions `interview__actions`, `interview__comments` and similar).

The header parameterizes both paths:

```
* Folder holding the identified data: this script READS from here
local confidential "<path to Y>"

* Project folder: this script WRITES the log here
local project "<path to X>"
```

The script must produce:

- Variable names, types, storage widths
- Variable labels and any attached notes or question text
- A summary line per value label or factor (entries, code range, attached variables); label **text** only for small labels not attached to geographic or name-type variables, and **never for those holding staff names**
- For each text variable: non-missing count, distinct-value count, maximum length
- For candidate ID variables: whether each is unique alone, whether they are jointly unique, missing count, and digit or character length range. No values.

Things to flag when handing it over:

**Do not dump value-label text wholesale.** Enumerator and supervisor names are commonly stored as value labels on numeric variables, so a full label dump reproduces the field roster. A geography label dump reproduces the village code-to-name mapping that the de-identification exists to withhold. A household-ID or respondent label may be a full ID-to-name list. The script drops staff-name labels first (auto-detected from variable names and labels, plus a manual list) and prints label text only for small labels not attached to geographic or name-type variables. Everything else appears as a summary line.

**Print no data values, text or numeric.** Counts carry the triage signal for text: a field with 40 non-missing and 38 distinct values needs redaction; one with 2 non-missing and 1 distinct probably doesn't. Do not summarize numeric variables either: the minimum and maximum of a phone-number or ID variable are real values. Lengths and digit counts are fine.

**Save the script in X; don't run it interactively.** It gets re-run when the questionnaire changes or a later wave arrives, and it documents what was inspected.

## Step 2: Log placement

The log is **Internal**: structure and question text, no respondent values, but a detailed description of a restricted file.

- Written directly to a `metadata/` folder in X by the extraction script.
- **Keep it out of version control.** Add the metadata folder to `.gitignore`.
- **Name it so vintage is obvious**: `metadata_<dataset>_<YYYYMMDD>.log`. A stale log silently describing a superseded file is a real failure mode on multi-wave projects.

Once it lands, confirm staff-name labels were excluded and no place-name or person-name label text was printed. Read every label whose text *was* printed (small labels can still be recoded "other, specify" answers). If names came through, say so, do not quote them, and ask for a regenerated log before reading further.

## Step 3: Identify candidate variables

Large surveys can exceed several thousand variables. Don't attempt one pass. Work in sections, by file, questionnaire module, section-marker variable, or name prefix, and write a flag list per section. **The flag list is the durable artifact**; the log will fall out of context on a large file.

**Direct identifiers.** Names of any person (respondent, head, spouse, children, staff, referrals), phone numbers, national or digital ID numbers, addresses, coordinates, email, social media handles, account numbers, file paths for photos and audio.

**Names hidden in structure.** Check for these explicitly; none of them looks like a name field in a variable list:

- A single-select question whose value label is a list of respondent names keyed by household ID (the respondent was picked from a preloaded roster). The variable is numeric and its codes equal the IDs; the label is a full ID-to-name crosswalk.
- List or text-list questions that export as one string variable per slot (`member__0` to `member__14`, `plot__0` ...), holding names of household members, plots (often named after people or places), or groups. Unused slots carry platform placeholders such as `##N/A##`, which inflate the non-missing counts.
- Roster titles: in a separate roster file, the roster's source list often reappears as a string variable.
- Business, group, and facility names (processor, cooperative, savings group, school). Private businesses are often named after their owners; group names can identify a village. Default: encode to anonymous codes without labels, so "sold to the same buyer" survives.

**Collection platform metadata.** Device identifiers, SIM and subscriber IDs, device phone numbers, submission UUIDs and instance names, server links to the interview, staff logins, audit and paradata references, IP addresses, timestamps finer than daily. Field names vary by platform; `references/platform-metadata.md` lists them for SurveyCTO, ODK and KoBoToolbox, Survey Solutions, CSPro, and Qualtrics. Read it when the platform is known.

**Free text: use structure, not keywords.** A text variable whose label or note contains a specify-prompt ("Other, specify", "Please specify") is almost certainly free text and a redaction target. Wider storage means more room for a name. Enumerator comment fields and open-ended attitude questions are the same category. Media capture fields are also text-typed but hold file paths; a label containing a capture instruction ("PHOTOGRAPH", "RECORD") identifies them regardless of name.

**Geography: codes stay at every level; names follow a disclosure ladder.**

*Codes.* Keep the numeric codes of every administrative level. A code without its name does not locate anyone on its own, and codes below the analysis level are often needed later (clustering, spillover analysis, merging in secondary data).

*Names.* Whether a level's place **names** may stay depends on the level, and the levels differ by country. Use the name-disclosure ladder in the country's entry in `references/countries.md`:

- **keep**: large units whose names identify no one (in Ethiopia: region, zone);
- **ask**: borderline units; settled by the researcher's answer to intake item 7, or at Step 4 if it was not answered (in Ethiopia: woreda);
- **remove**: small units, whose names are PII (in Ethiopia: kebele, got or village). Never keep these names, and never print their label text in the metadata log.

*Building the ladder for any country.* Rank levels by the typical population of one unit, the logic statistical agencies use for geographic detail in microdata (US health-data rules allow a geographic code only for areas above 20,000 people; US Census public-use files identify no area below 100,000):

| Typical population of one unit | Names |
|---|---|
| about 200,000 or more | keep |
| about 20,000 to 200,000 | ask |
| below about 20,000 | remove |

Overrides, whatever the size: **remove** names of units named after a person (headman villages); **remove** names at a level where the survey covers most households (a village census); **ask** when the study covers only a handful of units at a level, since their names then identify the study sites, which matters if the study topic is sensitive.

In Ethiopia this reproduces a researcher's own judgment: zone (a million or more people) keep, woreda (about 100,000) ask, kebele (a few thousand) remove. If the country has no entry, estimate typical unit sizes from general knowledge, draft the ladder, and have the researcher confirm it before any name is kept. Record the ladder and the answers in the flag list.

Names come in two storage patterns, often in the same file:

- Numeric code with a place-name label: for **remove** levels, strip the label and keep the code; for **keep** levels, the label may stay.
- Text variable holding the place name: for **remove** levels, encode to a numeric code and strip the resulting label; for **keep** levels, the string may stay.

**Group and facility names take the rung of the place they identify.** Names of development or farmer groups, cooperatives, savings groups, schools, health posts and similar usually identify a kebele or village. Remove them whenever that place's name would be removed, even though they are not administrative units. (Their codes stay, like all codes.)

Drop a geographic variable outright only if it is a coordinate, a free-text location description, or an address. GPS altitude alone does not locate a household and is often analytically useful (agriculture, health); offer to keep it rounded. Place names of **remove** levels also turn up in free text (comments, "other, specify"); the free-text screen looks for them.

Once labels are stripped, the code-to-name lookup exists only in Y; note in the removal log where the researcher can re-attach names if needed.

**Identifiers: keep them.** Stability between the identified and working versions is a feature: analysis code written on one runs on the other, and panel waves link. Check rather than assume:

- Does the ID encode information? Composite IDs embedding geography and listing order (`07-14-032` = district, sub-district, household) carry location even after geographic variables are stripped.
- Does the ID appear in an identifiable file that is or will be shared outside the team (a listing with coordinates, a sampling frame, a partner's records)? Then it is a join key back to identified data.
- Is a variable that is *named* like the ID actually the ID? A secondary file may hold an encoded version: numeric codes 1..N whose value label is the ID. The metadata log's label summary (entries, code range) gives it away.
- How do roster files link to the main file? On some platforms (Survey Solutions) rosters carry only the platform interview key, not the household ID. Then the script builds a crosswalk in Y from the main file and attaches the household ID (and, if the household ID is not unique, an interview number) to every output file. Whether to also keep the platform key is the researcher's decision.

If none applies, keep the IDs and generate no crosswalk. Re-mapping is a decision for the researcher, never a default.

**Quasi-identifiers.** Exact dates of birth, exact interview dates, household or firm size, landholding, asset and livestock counts, occupation, sector. Individually non-identifying, jointly sometimes not. For a working file these are **kept**; the re-identification threat they address is an outside analyst with auxiliary data, which is not the situation here. If the researcher wants dates coarsened: date of birth to age or birth year; interview date to month-year or survey round.

**Platform artifacts.** Calculated and helper variables, display and skip-logic markers, section duration fields, note-display fields. Multiple-response questions expand into dummy sets with numeric suffixes. Not PII; know the patterns so they don't confuse the scan.

## Step 4: Present for approval

For each flagged variable report: name, label, type, why flagged, recommendation (drop, recode, encode, coarsen, redact, or keep). Note whether dropping breaks existing analysis code. Put the open decisions (linking IDs, business names, altitude, date precision if any date variable is flagged, place names on the **ask** rung if intake item 7 was not answered, anything the intake left as "don't know") to the researcher with `AskUserQuestion`.

Bias toward keeping. The output is a working file; over-stripping destroys its purpose. Drop what must go, keep what is arguably useful, and say which is which.

**Do not generate destructive code before the researcher has approved the list.** Record the decisions in the flag list.

## Step 5: Write the de-identification script

The header parameterizes both paths, same pattern as Step 1: reads Y, writes X, temp files in Y. Full skeletons for Stata, R, and Python are in `references/code-conventions.md`. Read it before writing the script.

**Coverage by language.** The complete procedure (name harvest from labels and list variables, ID crosswalk, guarded redaction, leak assertions, byte scrub and byte scan, free-text review outputs) exists and has been tested in Stata and Mata only. The R and Python skeletons cover the basic stages (checks, label harvest, drop and strip, redaction, assertions, save, export) and have not been executed. For R or Python, port the missing pieces from the Stata version, say so to the researcher, and test on synthetic data before handing over.

Stages: setup, paths, and checks; harvest names from labels and name variables (into memory only); build an ID crosswalk if needed; drop and transform; redact; assertions; label cleanup and save; **byte-level scrub and scan of the saved file**; free-text review outputs; plain-text export for verification.

If the source is several files, or the intake asked for module output, the save stage writes one file per file or module, each carrying the ID variables, and checks uniqueness of the IDs per file.

### Fail loudly

The drop list was built from a log that may be stale relative to the data in Y, and the script runs unattended. Silent failure is the dangerous case: a `drop` that quietly skips a mistyped variable produces a file that looks clean and isn't. Verify every listed variable exists before changing anything; verify the input file and output directory exist; assert the observation count, ID uniqueness, and the count of variables matched by any wildcard. Stop with a non-zero return code on any mismatch.

### Redacting names without handling names

Where names are enumerated in value labels, rosters, or list questions, harvest them programmatically inside the script and loop the replacement. Names stay in Y, out of the conversation and out of the script source. Redact full names and name bigrams everywhere; redact single name words only with guards (capitalised, rare), since many given names and surnames are ordinary words. `references/code-conventions.md` has the procedure and the reasons. Three additions:

- **Name connectors.** After a redacted name, also redact the word that follows a patronymic or honorific connector (`Abba`, `A/`, `bin`, `son of`; per country in `references/countries.md`). Full-name redaction otherwise leaves the second element of "X Abba Y" behind.
- **Whitelist from the form.** Words in the form's non-personal answer lists (crops, trees, livestock, units, materials) are never redacted as single words. Those lists are safe to read, and local plant and crop names often coincide with personal names.
- **Known limits.** Names typed in lower case and names carrying a grammatical suffix (agglutinative languages) defeat the single-word guards. Say so to the researcher; the free-text screen catches them.

**Use replace-all functions.** Stata's `regexr()`, R's `sub()`, and Python's `str.replace(..., 1)` replace only the first match and produce partially redacted text that passes a spot check.

### Binary files can hold leftover strings

A statistical package can write uninitialised memory into the unused bytes of fixed-width fields. Stata does: the slack after the terminator in variable names, formats, value-label names, variable labels, and `str#` cells can contain fragments of strings that were in memory earlier in the session, including names from another file. They are invisible in the package and visible to anyone reading the bytes. Re-saving in a fresh session does not clear them. The script must zero those bytes after saving, then scan the raw bytes of each file for harvested names before copying it to X. The Stata routine is in `references/code-conventions.md`. Whether R and Python writers zero their padding has not been established: run the byte-level scan regardless.

### Multilingual and non-Latin free text

Free text may be in a local language, a non-Latin script, or inconsistent transliteration. Names in these forms survive a Latin-script replacement pass. Flag any field containing letters outside the Latin script (ICU and PCRE regex `[^\p{Latin}\p{Common}\p{Inherited}]`; Python needs the `regex` module for `\p{}` classes) and recommend translating it before the final screen. Transliterated names often have several spellings, and single-spelling replacement will miss some.

### Test on synthetic data before handing over

Claude cannot run the script on the real data, but it can run it on synthetic data built from the metadata log, and should whenever the researcher's software is available locally. Generate files with the same names, variables, types, and label names, a few dozen rows, and planted fakes: names in labels and list variables, the same names in free text in several spellings and cases, phone numbers in each written format (including at the end of a sentence), a duplicated ID, a one-value text variable. Run the extraction and de-identification scripts against them in a scratch folder, then grep both the plain-text exports and the raw bytes of the binary output for every planted fake. Fix and re-run until clean. Tell the researcher what was tested and what was not. In one real application this found three bugs before the first real run, including the leftover-bytes leak.

### Changing a script after hand-over

Before editing a script the researcher has been given, ask whether it is running. Stata reads a do-file as it executes, so an edit during a run can change what executes partway through; on synced or Windows folders the file may also be locked. Make the change after the run finishes or is stopped, then ask for a fresh run from the top.

### Cloud-synced folders

If X or Y is a synced folder (Dropbox, OneDrive, Google Drive), repeated writes can hit file locks. Write intermediate files to a local temp directory inside Y, then copy the final output to X once.

## Step 6: Verify in X, and expect to iterate

Once the files land in X, Claude runs the checks directly and reports PASS or FAIL per check.

1. **Header scan** of all variable names for identifier-suggestive tokens: name, phone, mobile, tel, contact, gps, lat, lon, coord, altitude, accuracy, device, sim, subscriber, imei, ip, interviewer, supervisor, enumerator, username, email, address, audit, instance, uuid, key, caseid, photo, image, audio, signature. Match short tokens (`lat`, `lon`, `ip`, `tel`, `sim`, `key`) only as whole name parts (between underscores or at the ends), or `long_income` and `shortlong` become hits. Report matches with labels; expect some (kept keys, rounded altitude) and say why each is fine.
2. **Phone patterns**, country-specific, from `references/countries.md`.
3. **ID number patterns**, country-specific, often fixed-length numeric. Same source.
4. **Coordinate patterns**: decimals within the country's bounding box, plus coordinate-suggestive names.
5. **Byte scan** of each binary output file for the patterns above (a grep of the raw file, not the package's view of it). Map every hit to its file section and variable before reporting it: in fixed-width data, a match can run from the end of one field into the next (an interview key followed by a version number looks like a phone number). The snippet is in `references/code-conventions.md`.
6. **Free-text screen**, per the review mode chosen at intake (below).
7. **Geography check**: levels on the **remove** rung carry codes only (no place-name labels or strings), **ask** levels match the researcher's decision, and no unattached label definitions survive in the file. For every level whose names were kept, check that the label has the expected number of entries (as many as there are units at that level in the sample) and show its text to the researcher. Once the level is approved, printing it is allowed, and it catches a label that holds more than its level's names.

One check cannot run in X: **redaction completeness against harvested names**, since those names do not cross the boundary. The script runs it in Y as an assertion, on both the data and the saved file's bytes.

**Pattern checks need a plain-text form of the file.** Have the script write a delimited export to a `check/` folder for verification; delete it once verification passes.

### The free-text screen

The first pass will not catch everything. Specify and comment fields routinely contain names and phone numbers no ex-ante rule could anticipate: a respondent naming a neighbour, an enumerator recording a contact number. This is expected, not a failure. Two or three passes is normal on a large survey.

**Mode A (Claude screens redacted text).** The script writes, per free-text variable, the distinct non-missing values of the *redacted* text, sorted, with no ID, row position, or adjacent variable. Claude reads them and looks for surviving names, numbers, and places. In Y, the script also writes a suspicious-token report for the researcher: words following an honorific or kin term, harvested name words left unredacted by the rarity guard, capitalised words that do not start a value, and runs of six or more digits, each with the variable and the number of distinct values containing it. Recipes for both outputs (`txt_screen()`, `tok_report()`) are in `references/code-conventions.md`.

**Mode B (researcher screens).** The same two outputs go to Y only. The researcher reads them and tells Claude what to add, as a description or a pattern.

What to look for, in this order:

1. **Comments about names and households.** Enumerator comments on name corrections, spellings, nicknames, replacements, or household composition concentrate surviving names ("respondent name corrected to ...", "called ... by neighbours"). Search for name, correct, spelling, replace, substitute, wife, son, and their local-language equivalents, and read those values first.
2. **Names the rules cannot see**: lower case, with a suffix attached, after a connector, nicknames in quotation marks.
3. **Numbers**: phone numbers in unusual formats, ID numbers, and household IDs written next to names.
4. **Rare traits in prose.** A free-text answer can identify someone without a name: "visited by the prime minister", "exports his coffee", "the only household with a car", a named employer or office held. Report these to the researcher as quasi-identifiers and let them decide; in an internal file they are usually kept.
5. **Over-redaction**: `[NAME]` in lists of crops, trees or places means a harvested name coincides with an ordinary word. Add the word to the whitelist.

Limits on detachment, worth stating rather than glossing:

- **Small-n fields carry more linkage than the extraction implies.** A value appearing once, in a field with few responses, in a survey whose geography is known from the study design, is weaker than it looks. Common first names are nothing; an unusual full name in a small sample is not.
- **Some fields are self-linking.** "Name of the group chairperson" or "who did you borrow from" attaches a role to the name even without the ID. Handle these by pattern or by dropping, not by reading values.

When feeding findings back into the script:

- **Prefer patterns to literals.** Phone numbers, ID numbers, and dates have structure. A regex rule keeps values out of the script source and generalizes to the next wave.
- **Use literals only where no pattern exists**, such as a one-off personal name with nothing to match on. Values found this way enter the conversation and the script; that is the argument for patterns wherever possible.
- **Ask where literals live.** Plain literals in the script are simplest, but they put real names in the project folder. The alternative stores a hash of each name (Mata `hash1()`, R `digest`, Python `hashlib`) and redacts words whose hash matches, at the cost of a few lines and a negligible risk of a collision redacting an unrelated word. Let the researcher choose.
- **Add an assertion for every literal**, so a later wave or a re-run cannot silently lose it.

Then re-run from Step 5 against the source in Y. **Do not edit the output files in X.** Every change goes through the script, so the transformation stays reproducible and the next wave inherits the fix. Iterate until the screen comes back clean.

## Output

- Intake note with the questionnaire answers, in X
- Metadata extraction script (written in X, run in Y)
- Flag list with disposition, rationale, and the researcher's decisions
- De-identification script (written in X, run in Y), tested on synthetic data
- De-identified files in X
- Verification report, PASS or FAIL per check, across all iterations
- Removal log: what was dropped, redacted, encoded, coarsened, and why

**Never leaves Y:** the identified source, intermediate files, harvested names, any crosswalk, the suspicious-token report.

## Reference files

Read these when the step calls for them, not all at once.

| File | Read when |
|---|---|
| `references/metadata-extraction.md` | Step 1. Extraction scripts for Stata (single and multi-file), R, Python. |
| `references/platform-metadata.md` | Steps 0 and 3. System fields per platform, and which form files may hold PII. |
| `references/code-conventions.md` | Steps 5 and 6. Script skeletons, name harvest and redaction, byte scrub, free-text review outputs, synthetic tests, verification snippets, lessons from field use. |
| `references/countries.md` | Steps 3 and 6. Template for admin hierarchy, phone and ID formats, coordinate bounds, scripts and naming conventions, with two worked examples. |
