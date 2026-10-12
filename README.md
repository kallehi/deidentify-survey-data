# deidentify-survey-data

A Claude skill for producing a de-identified working copy of survey data
(household, individual, farm, firm or facility surveys, listings, panels,
and similar administrative records) **without Claude ever seeing the
identified data**.

Many institutions, ethics boards and data agreements do not allow
personally identifiable information (PII) to be shared with AI tools. This
skill lets a researcher use Claude to plan and script the de-identification
while the identified data stay in restricted storage.

## The central idea: two folders

- **Restricted folder.** Holds the identified data. **Claude has no access
  to it.** You run Claude's scripts there.
- **Project folder.** Where Claude works. It receives only what the
  scripts write: a metadata log (structure, no values) and, at the end,
  the de-identified files.

The skill instructs Claude never to read the restricted folder, but an
instruction is not an access control. **Enforce the separation yourself.**
From strongest to weakest:

1. **Keep the identified data where Claude Code cannot reach it**: a
   different machine, a remote server, or an encrypted drive that is not
   mounted while Claude Code runs. You run the scripts there and copy their
   output into the project folder.
2. **Use Claude Code's sandbox** (macOS, Linux and WSL2 only; on native
   Windows, Claude Code runs shell commands unsandboxed). Turn it on with
   the `/sandbox` command and block the folder for all shell commands and
   their child processes:

   ```json
   {
     "sandbox": {
       "filesystem": {
         "denyRead": ["/path/to/restricted"],
         "denyWrite": ["/path/to/restricted"]
       }
     }
   }
   ```

   The sandbox covers shell commands only, so add the deny rules in 3 as
   well.
3. **Add deny rules for Claude's file tools** to your user settings
   (`~/.claude/settings.json`). They block Claude's built-in Read, Edit and
   Write tools on the folder, Grep and Glob searches inside it (best
   effort), and common file commands that name the path (`cat`, `head`,
   `sed`, redirections):

   ```json
   {
     "permissions": {
       "deny": [
         "Read(//path/to/restricted/**)",
         "Edit(//path/to/restricted/**)"
       ]
     }
   }
   ```

   The leading `//` marks an absolute path. On Windows, write
   `C:\Data\Restricted` as `//c/Data/Restricted/**`.

   **These rules are not a security boundary on their own.** They do not
   stop a script or command that opens files without naming the path in
   the command, and shell-command deny rules only match command text.
   Without the sandbox (for example on native Windows), rely on 1, or on
   operating-system file permissions that keep the account running Claude
   Code out of the folder.

Whichever you choose, test it before starting: ask Claude to list the
restricted folder and to read a file in it. Both should be refused or
fail. Check the current
[permissions](https://code.claude.com/docs/en/permissions) and
[sandboxing](https://code.claude.com/docs/en/sandboxing) documentation, as
the details change.

Anything that lands in the project folder is sent to Anthropic when Claude
reads it: the metadata log, the redacted free text Claude screens if you
choose that review mode, and the de-identified files. Make sure your rules
allow that.

## Disclaimer

**Use at your own risk.** The skill is provided as is, without warranty of
any kind (see `LICENSE`). It reduces the risk of exposing PII; it does not
guarantee that the output is anonymous. Automated rules miss identifiers,
especially names in free-text fields.

- **A person must check the output for PII before it is shared beyond the
  research team, and always before public release.**
- The output is a *working file* for internal analysis. Public release,
  repository deposit and replication packages additionally need disclosure
  control on quasi-identifiers, a check of consent scope and data
  agreements, and documentation. The skill does not do that.
- You remain responsible for complying with your institution's rules, your
  ethics approval, and your data agreements, including whether sharing even
  de-identified data or metadata with an AI tool is allowed.

## How it works

Claude never reads the identified data. It writes two scripts that you run
where the data live, and it reads only what those scripts write to your
project folder.

1. **Intake.** Claude asks a short questionnaire in up to three rounds:
   software, whether the CAPI form is available, country and output
   layout; then (skipping whatever the form already answers) platform,
   file structure, how far down place names may stay (codes are always
   kept), linked files and panel waves; then
   languages in free text and who reviews it (Claude, on already redacted
   text, or you alone). Dates default to exact days without time of day;
   you can change that when approving the flag list.
2. **Metadata extraction.** You run a script that writes a log of the
   data's structure to the project folder: variable names, types, labels,
   value-label summaries, counts for text variables, and ID uniqueness. No
   data values. Labels that hold staff names, place names or respondent
   names are summarised, not printed.
3. **Flag list.** Claude proposes what to drop, strip, encode, coarsen,
   redact or keep, with reasons, and asks you to decide the open points.
   Nothing changes until you approve. The default is to keep anything not
   identifying: the output must remain useful for analysis.
4. **De-identification script.** Claude writes a script that fails loudly
   if the data do not match the plan, harvests names from labels and name
   variables in memory to redact free text, asserts that no harvested name
   or phone number survives, and, for Stata files, zeroes leftover bytes in
   the saved file and scans its raw bytes before it leaves restricted
   storage. Where Stata, R or Python is installed on the machine Claude
   runs on, Claude first tests the scripts on synthetic data built from
   the metadata, with planted fake names and phone numbers.
5. **Verification.** Claude checks the output in the project folder
   (variable names, phone, ID and coordinate patterns, raw bytes, geography
   labels, and a free-text screen) and reports pass or fail per check.
   Expect two or three passes on free text.

Stata has complete, tested scripts. R and Python have partial skeletons
(basic stages only, not executed): the name harvest from list variables,
ID crosswalk, byte scan and free-text review outputs exist only in Stata
and Mata so far, and Claude ports them when needed. SPSS and SAS are
covered by pointers.

## Requirements

- Claude Code (the intake uses its multiple-choice question tool).
- A way to run scripts where the identified data live (Stata, R or
  Python), and to let their output files reach the project folder.
- For the Stata routines: tested in StataNow 19, run with `version 17`
  set; earlier Stata releases not tested. The byte scrub handles `.dta`
  formats 118 and 119 (the default `save` formats of Stata 14 onward) and
  stops with an error on any other.

## Install

This repository is a Claude Code plugin and its own marketplace.

```
claude plugin marketplace add kallehi/deidentify-survey-data
claude plugin install deidentify-survey-data@deidentify-survey-data
```

The skill triggers when you ask Claude to de-identify or anonymize data,
remove PII, or make data safe to use with AI tools. To call it explicitly,
run `/deidentify-survey-data:deidentify-survey-data`.

To use the skill without the plugin system, copy
`skills/deidentify-survey-data/` into `~/.claude/skills/` (all projects) or
`<project>/.claude/skills/` (one project).

## Contents

```
skills/deidentify-survey-data/
  SKILL.md                         The procedure (Steps 0 to 6)
  references/
    metadata-extraction.md         Extraction scripts: Stata (single and multi-file), R, Python
    code-conventions.md            De-identification skeletons, name redaction, byte scrub,
                                   synthetic test harness, lessons from field use
    platform-metadata.md           System fields per CAPI platform; form files that may hold PII
    countries.md                   Country template, with Ethiopia and Malawi as worked examples
```

## Countries

Country conventions (administrative levels, phone and ID formats,
coordinate bounds, scripts, naming) matter for detection. `countries.md`
has a template and two worked examples. For other countries, Claude drafts
an entry and asks you to confirm the phone and ID formats before using
them; a wrong pattern matches nothing and passes silently. Contributions
of verified entries are welcome.

## Testing status

As of 2026-10, in StataNow 19:

- The multi-file metadata extraction script was tested on synthetic data
  and then run on a 16-file Survey Solutions export.
- The name-redaction and byte-scrub routines, and a full de-identification
  script built with them, were tested on synthetic data with planted names
  and phone numbers, then run on the same export. The first verification
  passed every check except the free-text screen, which found a few
  surviving names (lower-case, suffixed, after a name connector); the
  lessons are in the skill. The free-text review routines were tested on
  synthetic data only.
- An earlier version of the procedure was used on an 11,000-variable
  SurveyCTO export.

The R and Python code has not been executed; test it before relying on
it.

## Reporting problems

Open an issue on GitHub. **Never paste data values, metadata logs, form
option lists or script output from the restricted folder into an issue.**
Describe the problem, the step where it occurred, and the software and
platform involved.

## Licence

MIT. See `LICENSE`.
