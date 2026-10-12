# Metadata extraction scripts

Read this at Step 1. Pick the script for the researcher's software, fill nothing in yourself except the parts you know, and hand it over as one block. Every script:

- reads from the confidential folder (Y) and writes only to `<X>/metadata/`
- drops or skips staff-name value labels **before** anything is printed
- prints structure, variable labels, notes, per-text-variable counts, and a per-value-label summary; value-label text only for small non-geographic, non-name labels
- prints **no data values**, text or numeric (no `summarize`, `codebook`, `list`, `tab` of values)
- names the log `metadata_<dataset>_<YYYYMMDD>.log`
- stops with an error if the input file is missing

The researcher must fill in the paths, the dataset name, and the staff-label list. Tell them how to find the staff-label names: in Stata, `label dir` in Y lists label names; in R and Python, the variable names of enumerator and supervisor fields.

## Contents

- [Stata](#stata)
- [Stata, several files](#stata-several-files)
- [R](#r)
- [Python](#python)
- [SPSS and SAS](#spss-and-sas)
- [What to check when the log lands](#what-to-check-when-the-log-lands)

---

## Stata

```stata
*==============================================================================
* extract_metadata.do
* Reads the confidential dataset in the restricted folder (Y) and writes a
* metadata log — structure only, no data values — to the project folder (X).
* Run in the restricted environment. Re-run whenever the source data change.
*==============================================================================
version 16
clear all
set more off
set linesize 200

*--- Paths: this script READS from confidential and WRITES to project --------
local confidential "<path to Y>"          // restricted folder, read only
local project      "<path to X>"          // project folder, log lands here
local dataset      "<filename>.dta"

*--- Value labels that hold fieldwork staff names (FILL IN) -------------------
* Run `label dir` on the data to see label names. Enumerator, supervisor,
* team-leader and facilitator labels typically carry names. List them here so
* they are dropped BEFORE the log opens. Separate with spaces.
local staff_labels "<FILL IN, e.g. enumerator supervisor>"

*--- Checks -------------------------------------------------------------------
confirm file "`confidential'/`dataset'"
capture mkdir "`project'/metadata"

local stem  = subinstr("`dataset'", ".dta", "", .)
local today : di %tdCCYYNNDD daily("`c(current_date)'", "DMY")
local logfile "`project'/metadata/metadata_`stem'_`today'.log"

use "`confidential'/`dataset'", clear

* Drop staff-name labels before the log opens
foreach l of local staff_labels {
    capture label drop `l'
}

log using "`logfile'", replace text

di as text "Dataset: `dataset'"
di as text "Observations: " _N "   Variables: " c(k)
di as text "Extracted: `c(current_date)' `c(current_time)'"
di as text "Staff-name labels dropped before listing: `staff_labels'"

*--- Structure ----------------------------------------------------------------
di as text _n "==== describe ===="
describe, fullnames

di as text _n "==== notes (question text often lives here) ===="
notes

*--- Value labels (staff labels already removed) ------------------------------
* Label TEXT is printed only for small labels (<= `maxshow' entries) that
* are not attached to a geographic or name-type variable. Everything else
* gets a one-line summary: entries, code range, attached variables. This
* keeps place-name code-to-name mappings, and any other enumerated
* names, inside Y.
local maxshow 15
local geo_rx "region|province|state|zone|district|county|sub-?district|ward|village|settlement|community|cluster|\bea\b|enumeration|location|market|school|clinic|facility|group|cooperative|union|name|household id"
* add the country's level names from countries.md (e.g. woreda|kebele, or \bta\b|\bgvh\b)
di as text _n "==== value labels: summary (text shown only if entries<=`maxshow' and not geo/name) ===="
quietly label dir
local alllabels `r(names)'
local nlab : word count `alllabels'
forvalues i = 1/`nlab' {
    local att`i' ""
    local geo`i' 0
}
* one pass over variables builds the label -> attached-variables map
foreach v of varlist _all {
    local vl : value label `v'
    if "`vl'" == "" continue
    local i : list posof "`vl'" in alllabels
    if `i' == 0 continue
    local att`i' `att`i'' `v'
    local lab : variable label `v'
    local hay = lower(`"`v' `lab'"')
    if ustrregexm(`"`hay'"', "`geo_rx'") local geo`i' 1
}
forvalues i = 1/`nlab' {
    local l : word `i' of `alllabels'
    if ustrregexm(lower("`l'"), "`geo_rx'") local geo`i' 1
    quietly label list `l'
    local k = r(k)
    di as text %-24s "`l'" "  entries=" %6.0f `k' "  min=" %10.0f r(min) ///
        "  max=" %10.0f r(max) "  geo_or_name=`geo`i''  attached: `att`i''"
    if `k' <= `maxshow' & `geo`i'' == 0 {
        label list `l'
    }
}

*--- Text variables: counts only, never values (Mata, one pass) ---------------
* Define the helper BEFORE the call (a Mata block after `exit` is never read).
mata:
void _strcounts(string rowvector vars)
{
    real scalar i, n, d, ml
    string colvector x, nm
    for (i = 1; i <= cols(vars); i++) {
        x  = st_sdata(., vars[i])
        nm = select(x, x :!= "")
        n  = rows(nm)
        if (n > 0) {
            d  = rows(uniqrows(nm))
            ml = max(ustrlen(nm))
        }
        else {
            d  = 0
            ml = 0
        }
        printf("%-32s  type=%-8s  nonmissing=%8.0f  distinct=%8.0f  maxlen=%5.0f
",
               vars[i], st_vartype(vars[i]), n, d, ml)
    }
}
end
di as text _n "==== text variables: non-missing and distinct counts ===="
ds, has(type string)
local strvars `r(varlist)'
mata: _strcounts(tokens(st_local("strvars")))

log close
di as result "Metadata log written to: `logfile'"
```

Notes for Stata:

- `describe, fullnames` shows storage type and width (`str12`, `strL`), label, and attached value-label name. Width is the free-text signal.
- Value-label text is printed only for small labels that are not attached to a geographic or name-type variable; every other label gets a summary line (entries, code range, attached variables). A code-to-place-name mapping is the thing the de-identification exists to withhold, and in several countries village names are headman names, so a full `label list` would leak both. Step 3 flags geographic labels from the summary line and the variable label, not from the label text. If a specific small label's text is needed to decide a recode, ask the researcher to print that one label.
- `notes` prints dataset and variable notes. SurveyCTO and ODK imports often put question text there. If the file has characteristics instead, add `char list` but check the output does not contain data values.
- Wide CAPI exports can hold thousands of string variables (repeat groups expanded to `_1_7`-style suffixes). Per-variable `egen tag` and a per-label scan of all variables make the script run for a long time on such files; the Mata counter and the one-pass label map above run in seconds. Variables with zero non-missing values are exported placeholders; they still print, at no cost.

---

## Stata, several files

Use when Y holds several files (modules, rosters, a platform export). Loops over every `.dta` in Y and its first-level subfolders, writes one log per file plus an index, skips Survey Solutions system files, and adds an ID block: for candidate ID variables, whether each is unique alone and jointly, the missing count, and the digit or character length range, with no values. Tested in StataNow 19 (2026-10) on synthetic data and then on a 16-file Survey Solutions export.

```stata
*==============================================================================
* extract_metadata_multi.do
* Purpose : Write metadata logs (structure only, no data values) for every
*           .dta file in the restricted folder (Y) to the project folder (X).
* Inputs  : all *.dta in Y and its first-level subfolders
* Outputs : X/metadata/metadata_<tag>_<file>_<YYYYMMDD>.log  (one per file)
*           X/metadata/metadata_<tag>_index_<YYYYMMDD>.log  (file list)
* Run     : by the researcher, in Stata, with access to Y. Reads Y, writes X.
*           Prints no data values: no summarize/codebook/list/tab of values.
*==============================================================================
version 17
clear all
set more off
set linesize 200

*--- Paths: this script READS from confidential and WRITES to project --------
local confidential "<path to Y>"
local project      "<path to X>"
local tag          "<dataset tag, e.g. baseline>"

*--- Staff-name value labels: dropped BEFORE any log opens --------------------
* Manual list (from the form or the researcher): variables whose value labels
* are enumerator or supervisor names. Variables matching the regexes below are
* dropped too, in case they were renamed during cleaning.
local staff_vars   "<FILL IN, e.g. enumerator supervisor>"
local staff_vn_rx  "enum|superv|interviewer|responsible|team_?lead|facilitator|collector"
local staff_lab_rx "supervisor|choose your name|interviewer|enumerator name|name of (the )?enumerator|team leader|facilitator|data collector"

*--- Labels whose TEXT must never print (place and person names) --------------
* Summary line only (entries, code range, attached variables).
local name_vars "<FILL IN: geography, household-ID and respondent-name variables>"
local geo_rx    "region|province|state|zone|district|county|sub-?district|ward|village|settlement|community|cluster|\bea\b|enumeration|location|market|school|clinic|facility|group|cooperative|\bcoop|union|name|household id|hhid|hh_id"
* Add the country's own level names from countries.md, e.g. Ethiopia:
* woreda|wereda|kebele|qebele|\bgot\b    Malawi: \bta\b|\bgvh\b|traditional
local maxshow   15
* Set to 0 to print no value-label text at all (safest; use when neither the
* CAPI form nor a country entry was available to fill name_vars and geo_rx)
local show_text 1

* Survey Solutions system files that must not leave Y; not even described
local skip_rx "^interview__(actions|comments|errors|diagnostics)|^assignment__actions|paradata"

*--- Checks -------------------------------------------------------------------
mata: st_local("ok", strofreal(direxists(st_local("confidential"))))
if "`ok'" != "1" {
    di as error "Confidential folder not found: `confidential'"
    exit 601
}
mata: st_local("ok", strofreal(direxists(st_local("project"))))
if "`ok'" != "1" {
    di as error "Project folder not found: `project'"
    exit 601
}
capture mkdir "`project'/metadata"

local today : di %tdCCYYNNDD daily("`c(current_date)'", "DMY")

*--- Mata helper: per-string-variable counts, never values --------------------
mata:
void _strcounts(string rowvector vars)
{
    real scalar i, n, d, ml
    string colvector x, nm
    for (i = 1; i <= cols(vars); i++) {
        x  = st_sdata(., vars[i])
        nm = select(x, x :!= "")
        n  = rows(nm)
        if (n > 0) {
            d  = rows(uniqrows(nm))
            ml = max(ustrlen(nm))
        }
        else {
            d  = 0
            ml = 0
        }
        printf("%-32s  type=%-8s  nonmissing=%8.0f  distinct=%8.0f  maxlen=%5.0f\n",
               vars[i], st_vartype(vars[i]), n, d, ml)
    }
}
end

*--- File list: Y and first-level subfolders ----------------------------------
local relfiles
local top : dir "`confidential'" files "*.dta", respectcase
foreach f of local top {
    local relfiles `"`relfiles' "`f'""'
}
local subdirs : dir "`confidential'" dirs "*"
foreach s of local subdirs {
    local sf : dir "`confidential'/`s'" files "*.dta", respectcase
    foreach f of local sf {
        local relfiles `"`relfiles' "`s'/`f'""'
    }
}
local nfiles : word count `relfiles'
if `nfiles' == 0 {
    di as error "No .dta files found in the confidential folder."
    exit 601
}

*==============================================================================
* One metadata log per file
*==============================================================================
local i 0
local skipped
foreach rf of local relfiles {
    local base = lower(substr("`rf'", strrpos("`rf'", "/") + 1, .))
    if ustrregexm("`base'", "`skip_rx'") {
        local skipped `"`skipped' "`rf'""'
        continue
    }
    local ++i
    local stem = subinstr("`rf'", ".dta", "", .)
    local stem = ustrregexra("`stem'", "[^A-Za-z0-9_]", "_")
    local logfile "`project'/metadata/metadata_`tag'_`stem'_`today'.log"

    confirm file "`confidential'/`rf'"
    use "`confidential'/`rf'", clear

    * Drop staff-name labels before the log opens
    local staffdropped
    foreach v of varlist _all {
        local vl : value label `v'
        if "`vl'" == "" continue
        local lab : variable label `v'
        local isstaff : list v in staff_vars
        if !`isstaff' {
            if ustrregexm(lower("`v'"), "`staff_vn_rx'") local isstaff 1
            if ustrregexm(lower(`"`lab'"'), "`staff_lab_rx'") local isstaff 1
        }
        if `isstaff' {
            capture label drop `vl'
            local staffdropped `staffdropped' `v'(`vl')
        }
    }

    log using "`logfile'", replace text name(md)

    di as text "File: `rf'"
    di as text "Observations: " _N "   Variables: " c(k)
    di as text "Extracted: `c(current_date)' `c(current_time)'"
    di as text "Staff-name labels dropped before listing [variable(label)]: `staffdropped'"

    *--- Structure ------------------------------------------------------------
    di as text _n "==== describe ===="
    describe, fullnames

    *--- Candidate ID variables: uniqueness and digit counts, no values -------
    di as text _n "==== candidate ID variables (uniqueness only, no values) ===="
    local idc
    foreach v of varlist _all {
        if ustrregexm(lower("`v'"), "^(hhid|hh_id|hhcode|caseid|key|interview__key|interview__id)$|__id$|_id$") {
            local idc `idc' `v'
        }
    }
    foreach v of local idc {
        capture isid `v', missok
        local u = (_rc == 0)
        quietly count if missing(`v')
        local nmiss = r(N)
        capture confirm string variable `v'
        if _rc {
            tempvar nd
            quietly gen `nd' = floor(log10(abs(`v'))) + 1 if !missing(`v') & `v' != 0
            quietly summarize `nd'
            local dig = "digits=" + string(r(min)) + "-" + string(r(max))
            drop `nd'
        }
        else {
            tempvar nd
            quietly gen `nd' = ustrlen(`v') if `v' != ""
            quietly summarize `nd'
            local dig = "length=" + string(r(min)) + "-" + string(r(max))
            drop `nd'
        }
        di as text %-32s "`v'" "  type=`:type `v''  unique_alone=`u'  missing=`nmiss'  `dig'"
    }
    if "`idc'" != "" {
        capture isid `idc', missok
        di as text "All candidates jointly (`idc'): unique=" (_rc == 0)
    }

    *--- Value labels (staff labels already removed) --------------------------
    di as text _n "==== value labels: summary (text shown only if entries<=`maxshow' and not geo/name) ===="
    quietly label dir
    local alllabels `r(names)'
    local nlab : word count `alllabels'
    forvalues j = 1/`nlab' {
        local att`j' ""
        local geo`j' 0
    }
    foreach v of varlist _all {
        local vl : value label `v'
        if "`vl'" == "" continue
        local j : list posof "`vl'" in alllabels
        if `j' == 0 continue
        local att`j' `att`j'' `v'
        local lab : variable label `v'
        local hay = lower(`"`v' `lab'"')
        if ustrregexm(`"`hay'"', "`geo_rx'") local geo`j' 1
        local isname : list v in name_vars
        if `isname' local geo`j' 1
    }
    forvalues j = 1/`nlab' {
        local l : word `j' of `alllabels'
        if ustrregexm(lower("`l'"), "`geo_rx'") local geo`j' 1
        quietly label list `l'
        local k = r(k)
        di as text %-24s "`l'" "  entries=" %6.0f `k' "  min=" %10.0f r(min) ///
            "  max=" %10.0f r(max) "  geo_or_name=`geo`j''  attached: `att`j''"
        if `k' <= `maxshow' & `geo`j'' == 0 & `show_text' {
            label list `l'
        }
    }

    *--- Text variables: counts only ------------------------------------------
    di as text _n "==== text variables: non-missing and distinct counts ===="
    quietly ds, has(type string)
    local strvars `r(varlist)'
    if "`strvars'" != "" {
        mata: _strcounts(tokens(st_local("strvars")))
    }

    log close md
    local idx`i' "`rf'   N=`=_N'   k=`=c(k)'"
}

*==============================================================================
* Index log: which files were described, which skipped
*==============================================================================
log using "`project'/metadata/metadata_`tag'_index_`today'.log", replace text name(idx)
di as text "Baseline metadata extraction, `c(current_date)' `c(current_time)'"
di as text "Files described: `i'"
forvalues j = 1/`i' {
    di as text "  `idx`j''"
}
di as text "Skipped (Survey Solutions system files, not described):"
foreach s of local skipped {
    di as text "  `s'"
}
log close idx

clear
di as result "Done. Logs written to: `project'/metadata"
```

Notes:

- Fill in `staff_vars` and `name_vars` from the CAPI form's question structure if available; the regexes are a backstop.
- The candidate-ID regex is generic; add the study's ID variable names.
- The geography regex is a backstop and only knows generic level names. In a test on synthetic data, geography variables with no variable label and a local level name (woreda, kebele) printed their place-name labels. Fill `name_vars` with every geography variable, add the country's level names to `geo_rx`, or set `show_text` to 0. Then check the log (SKILL.md Step 2).
- Per-file logs keep each log short enough to read in sections; the index log lists N and k per file and the skipped files.
- Run it as a do-file from the Stata GUI or in batch mode; the output contains counts and structure only.

---

## R

```r
#==============================================================================
# extract_metadata.R
# Reads the confidential dataset in the restricted folder (Y) and writes a
# metadata log — structure only, no data values — to the project folder (X).
# Run in the restricted environment. Re-run whenever the source data change.
#==============================================================================
suppressPackageStartupMessages(library(haven))

#--- Paths: this script READS from confidential and WRITES to project ---------
confidential <- "<path to Y>"          # restricted folder, read only
project      <- "<path to X>"          # project folder, log lands here
dataset      <- "<filename>.dta"       # .dta, .sav, .rds, or .csv

#--- Variables whose value labels hold fieldwork staff names (FILL IN) --------
staff_vars <- c("<FILL IN, e.g. enumerator>", "<supervisor>")

#--- Checks --------------------------------------------------------------------
infile <- file.path(confidential, dataset)
stopifnot("Input file not found" = file.exists(infile))
dir.create(file.path(project, "metadata"), showWarnings = FALSE, recursive = TRUE)
stem    <- tools::file_path_sans_ext(dataset)
logfile <- file.path(project, "metadata",
                     sprintf("metadata_%s_%s.log", stem, format(Sys.Date(), "%Y%m%d")))

df <- switch(tolower(tools::file_ext(dataset)),
  dta = haven::read_dta(infile),
  sav = haven::read_sav(infile),
  rds = readRDS(infile),
  csv = read.csv(infile, stringsAsFactors = FALSE, check.names = FALSE),
  stop("Unsupported file type"))

# Remove staff-name value labels BEFORE anything is printed
for (v in intersect(staff_vars, names(df))) {
  attr(df[[v]], "labels") <- NULL
  if (is.factor(df[[v]])) df[[v]] <- as.integer(df[[v]])
}

sink(logfile)
cat(sprintf("Dataset: %s\nObservations: %d   Variables: %d\nExtracted: %s\n",
            dataset, nrow(df), ncol(df), format(Sys.time())))
cat("Staff-name labels removed before listing:", paste(staff_vars, collapse = " "), "\n\n")

cat("==== variables: name, class, label ====\n")
for (v in names(df)) {
  x   <- df[[v]]
  lab <- attr(x, "label", exact = TRUE); if (is.null(lab)) lab <- ""
  cat(sprintf("%-32s %-14s %s\n", v, class(x)[1], lab))
}

cat("\n==== value labels (staff labels already removed) ====\n")
for (v in names(df)) {
  labs <- attr(df[[v]], "labels", exact = TRUE)
  if (!is.null(labs)) {
    cat(sprintf("\n%s:\n", v))
    for (i in seq_along(labs)) cat(sprintf("  %s = %s\n", labs[i], names(labs)[i]))
  }
  if (is.factor(df[[v]])) {
    cat(sprintf("\n%s (factor levels):\n", v))
    cat(sprintf("  %d = %s\n", seq_along(levels(df[[v]])), levels(df[[v]])), sep = "")
  }
}

cat("\n==== text variables: non-missing and distinct counts (never values) ====\n")
for (v in names(df)) {
  x <- df[[v]]
  if (is.character(x)) {
    ok <- !is.na(x) & x != ""
    maxw <- if (any(ok)) max(nchar(x[ok])) else 0
    cat(sprintf("%-32s nonmissing=%8d  distinct=%8d  maxwidth=%5d\n",
                v, sum(ok), length(unique(x[ok])), maxw))
  }
}
sink()
message("Metadata log written to: ", logfile)
```

Notes for R:

- `haven` reads `.dta` and `.sav` with variable labels (`label` attribute) and value labels (`labels` attribute) intact. Base `read.csv` has neither; for CSV the log will show names, classes, and text counts only.
- If staff names are stored as a character variable rather than as value labels, the counts loop handles it: only counts print.
- Question text stored in `.dta` notes is not read by `haven`. Ask the researcher for the questionnaire if labels are truncated.

---

## Python

```python
#==============================================================================
# extract_metadata.py
# Reads the confidential dataset in the restricted folder (Y) and writes a
# metadata log — structure only, no data values — to the project folder (X).
# Run in the restricted environment. Re-run whenever the source data change.
# Requires: pandas, pyreadstat  (pip install pandas pyreadstat)
#==============================================================================
import os, sys, datetime
import pandas as pd
import pyreadstat

#--- Paths: this script READS from confidential and WRITES to project ---------
confidential = r"<path to Y>"          # restricted folder, read only
project      = r"<path to X>"          # project folder, log lands here
dataset      = "<filename>.dta"        # .dta, .sav, .sas7bdat, .csv, .parquet

#--- Variables whose value labels hold fieldwork staff names (FILL IN) --------
staff_vars = ["<FILL IN, e.g. enumerator>", "<supervisor>"]

#--- Checks --------------------------------------------------------------------
infile = os.path.join(confidential, dataset)
if not os.path.exists(infile):
    sys.exit(f"Input file not found: {infile}")
os.makedirs(os.path.join(project, "metadata"), exist_ok=True)
stem, ext = os.path.splitext(dataset)
logfile = os.path.join(project, "metadata",
                       f"metadata_{stem}_{datetime.date.today():%Y%m%d}.log")

readers = {".dta": pyreadstat.read_dta, ".sav": pyreadstat.read_sav,
           ".sas7bdat": pyreadstat.read_sas7bdat}
if ext.lower() in readers:
    df, meta = readers[ext.lower()](infile)
    var_labels   = dict(meta.column_names_to_labels)
    value_labels = dict(meta.variable_value_labels)   # {var: {code: label}}
    storage      = dict(meta.original_variable_types)            # {var: '%20s', '%9.0g', ...} display format; width for strings
elif ext.lower() == ".csv":
    df = pd.read_csv(infile, low_memory=False); var_labels = value_labels = storage = {}
elif ext.lower() == ".parquet":
    df = pd.read_parquet(infile); var_labels = value_labels = storage = {}
else:
    sys.exit(f"Unsupported file type: {ext}")

# Remove staff-name value labels BEFORE anything is printed
for v in staff_vars:
    value_labels.pop(v, None)

with open(logfile, "w", encoding="utf-8") as f:
    w = lambda s="": f.write(s + "\n")
    w(f"Dataset: {dataset}")
    w(f"Observations: {len(df)}   Variables: {df.shape[1]}")
    w(f"Extracted: {datetime.datetime.now():%Y-%m-%d %H:%M}")
    w(f"Staff-name labels removed before listing: {' '.join(staff_vars)}")

    w("\n==== variables: name, dtype, format, label ====")
    for v in df.columns:
        w(f"{v:<32} {str(df[v].dtype):<10} {storage.get(v, ''):<8} {var_labels.get(v, '') or ''}")

    w("\n==== value labels (staff labels already removed) ====")
    for v, mapping in value_labels.items():
        w(f"\n{v}:")
        for code, lab in mapping.items():
            w(f"  {code} = {lab}")

    w("\n==== text variables: non-missing and distinct counts (never values) ====")
    for v in df.columns:
        if pd.api.types.is_string_dtype(df[v]) or df[v].dtype == object:   # covers pandas "str", "string", object
            s = df[v].dropna().astype(str)
            s = s[s.str.strip() != ""]
            maxw = int(s.str.len().max()) if len(s) else 0
            w(f"{v:<32} nonmissing={len(s):>8}  distinct={s.nunique():>8}  maxwidth={maxw:>5}")

print("Metadata log written to:", logfile)
```

Notes for Python:

- `pyreadstat` returns metadata separately from data, which makes the value-label dump clean. For `.dta` files `meta.original_variable_types` holds the display format (`%44s`, `%9.0g`); the string width is the free-text signal, and the text section also prints the observed maximum width.
- For `.dta` files with very many rows, `pyreadstat.read_dta(infile, metadataonly=True)` returns the metadata without loading data. Use it for the structure and label sections, then a normal read for the text counts.

---

## SPSS and SAS

No full script is provided. Adapt the R or Python script: `haven::read_sav()`, `haven::read_sas()`, `pyreadstat.read_sav()`, and `pyreadstat.read_sas7bdat()` all return labels in the same structure as for `.dta`. If the researcher must stay inside SPSS or SAS:

- SPSS: `DISPLAY DICTIONARY.` prints names, labels, types, and value labels. Delete staff-name value labels first with `VALUE LABELS enumerator.` (empty list). Do not run `FREQUENCIES` or `DESCRIPTIVES`.
- SAS: `PROC CONTENTS` prints structure; `PROC FORMAT FMTLIB` prints formats. Exclude the staff-name format from `FMTLIB` with `SELECT`. Do not run `PROC MEANS` or `PROC FREQ`.

---

## What to check when the log lands

1. The header line shows the expected dataset name and a plausible N and k.
2. No staff names appear in the label section. If they do, say so without quoting them and continue; note the label name for the de-identification script to drop.
3. Text-variable counts are present for every string variable. A variable with high distinct-to-nonmissing ratio is a redaction candidate.
4. Nothing in the log is a data value. If a value slipped through (for example a `tab` the researcher added), stop and ask for a regenerated log.
