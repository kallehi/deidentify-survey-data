# De-identification script conventions

Read this at Step 5, after the researcher has approved the flag list. It gives a complete skeleton per language plus the individual blocks. Every script:

- reads from Y, writes the final file to X, keeps temp files in Y
- checks the input file, the output directory, and every expected variable **before** changing anything
- harvests staff and respondent names from value labels and name variables inside the script, so names never enter the script source
- uses replace-all string functions
- asserts after redaction that no harvested name survives, in Y
- drops unattached value-label definitions before saving (they are saved with the file otherwise)
- zeroes slack bytes in the saved binary file and scans its raw bytes for harvested names before it leaves Y
- writes a delimited export next to the output for pattern checks
- is tested on synthetic data before hand-over

## Contents

- [Replace-first-match traps](#replace-first-match-traps)
- [Stata skeleton](#stata-skeleton)
- [R skeleton](#r-skeleton)
- [Python skeleton](#python-skeleton)
- [Date coarsening](#date-coarsening)
- [Several source files and module splitting](#several-source-files-and-module-splitting)
- [Verification snippets Claude runs in X](#verification-snippets-claude-runs-in-x)
- [Name harvest, redaction and byte scrub (Stata and Mata)](#name-harvest-redaction-and-byte-scrub-stata-and-mata)
- [Free-text review outputs (Stata and Mata)](#free-text-review-outputs-stata-and-mata)
- [Synthetic test harness](#synthetic-test-harness)
- [Lessons from field use](#lessons-from-field-use)

---

## Replace-first-match traps

| Language | Replaces first match only (do not use) | Replaces all matches (use) |
|---|---|---|
| Stata | `regexr()`, `ustrregexrf()` | `subinstr(s, "x", "y", .)`, `ustrregexra()` |
| R | `sub()` | `gsub()`, `stringr::str_replace_all()` |
| Python | `str.replace(old, new, 1)`, `re.sub(..., count=1)` | `str.replace(old, new)`, `re.sub()`, `Series.str.replace(..., regex=True)` |

Case matters too. Enumerator "Abebe Kebede" may appear in a comment as "abebe". Use a case-insensitive literal match: Stata `ustrregexra(s, "(?i)\Qname\E", "[NAME]")`, R `gsub("(?i)\\Qname\\E", "[NAME]", s, perl = TRUE)` or `stringr::str_replace_all(s, stringr::fixed(name, ignore_case = TRUE), "[NAME]")`, Python `re.sub(re.escape(name), "[NAME]", s, flags=re.I)`. The `\Q...\E` wrapper treats the name as a literal so a name containing a dot or bracket does not become a regex.

---

## Stata skeleton

```stata
*==============================================================================
* deidentify.do
* Reads the confidential dataset in Y, writes a de-identified copy to X.
* Temp files stay in Y. Every list below comes from the approved flag list.
*==============================================================================
version 16
clear all
set more off

*--- Paths: READS from confidential, WRITES final file to project ------------
local confidential "<path to Y>"             // source and temp files
local project      "<path to X>"             // de-identified output
local dataset      "<filename>.dta"
local outname      "<filename>_deid"         // no extension

*--- Approved lists (Step 4) ---------------------------------------------------
local drop_vars        "<variables to drop>"
local geo_strip        "<numeric geography: strip labels, keep codes>"
local geo_encode       "<string geography: convert to codes>"
local text_redact      "<free-text variables to redact>"
local staff_label_vars "<variables whose value labels hold staff names>"
local phone_regex      "<from references/countries.md, e.g. (\+?251|0)[79][0-9]{8}>"
local id_regex         "<from references/countries.md, or leave empty>"

*=== Stage 0: checks — nothing below runs unless all of these pass ============
confirm file "`confidential'/`dataset'"
mata: st_local("projexists", strofreal(direxists("`project'")))
if !`projexists' {
    di as error "Project folder not found: `project'"
    exit 601
}
capture mkdir "`confidential'/tmp_deidentify"

use "`confidential'/`dataset'", clear
local n_in = _N

local expected `drop_vars' `geo_strip' `geo_encode' `text_redact' `staff_label_vars'
local missing_vars
foreach v of local expected {
    capture confirm variable `v'
    if _rc local missing_vars `missing_vars' `v'
}
if "`missing_vars'" != "" {
    di as error "Stopping. Not found in data: `missing_vars'"
    di as error "The flag list was built from a metadata log that may be stale."
    exit 111
}

*=== Stage 1: harvest staff names from value labels (they stay in Y) =========
local staff_names
foreach v of local staff_label_vars {
    local lbl : value label `v'
    if "`lbl'" != "" {
        quietly levelsof `v', local(codes)
        foreach c of local codes {
            local nm : label `lbl' `c'
            if `"`nm'"' != "`c'" & strlen(`"`nm'"') > 2 {
                local staff_names `"`staff_names' `"`nm'"'"'
            }
        }
    }
}
local n_staff : word count `staff_names'
di as text "Harvested `n_staff' staff names from value labels."

*=== Stage 2: drop and transform ==============================================
drop `drop_vars'

* Numeric geography: keep codes, detach place-name labels
foreach v of local geo_strip {
    label values `v' .
}

* String geography: convert to codes, then detach the resulting label.
* Codes are alphabetical within this file. If codes must match an official
* list or an earlier wave, merge a code list from Y instead of using encode.
foreach v of local geo_encode {
    encode `v', gen(`v'_code) label(`v'_lbl)
    order `v'_code, after(`v')
    drop `v'
    rename `v'_code `v'
    label values `v' .
    label drop `v'_lbl
}

*=== Stage 3: redact free text ================================================
foreach t of local text_redact {
    * staff names, case-insensitive, literal
    foreach nm of local staff_names {
        quietly replace `t' = ustrregexra(`t', `"(?i)\Q`nm'\E"', "[NAME]")
    }
    * phone numbers
    if "`phone_regex'" != "" {
        quietly replace `t' = ustrregexra(`t', "`phone_regex'", "[PHONE]")
    }
    * national ID numbers
    if "`id_regex'" != "" {
        quietly replace `t' = ustrregexra(`t', "`id_regex'", "[ID]")
    }
    * literals found in Step 6 iterations go here, one line each:
    * quietly replace `t' = ustrregexra(`t', "(?i)\Qexact string\E", "[NAME]")
}

*=== Stage 4: post-write assertions (run in Y, on the data about to be saved) =
local leaks 0
foreach t of local text_redact {
    foreach nm of local staff_names {
        quietly count if ustrregexm(`t', `"(?i)\Q`nm'\E"')
        local leaks = `leaks' + r(N)
    }
}
if `leaks' > 0 {
    di as error "Stopping. `leaks' staff-name occurrences survived redaction."
    exit 459
}
assert _N == `n_in'

* No place-name labels attached to geography
foreach v in `geo_strip' `geo_encode' {
    local lbl : value label `v'
    assert "`lbl'" == ""
}

*=== Stage 5: label cleanup, compress, save ===================================
quietly labelbook, problems
capture label drop `r(notused)'          // unattached labels are saved otherwise
quietly compress
label data "De-identified working file. Source: `dataset'. Created `c(current_date)'."
note: De-identified by deidentify.do on `c(current_date)'. See removal log in project folder.

save "`confidential'/tmp_deidentify/`outname'.dta", replace
copy  "`confidential'/tmp_deidentify/`outname'.dta" "`project'/`outname'.dta", replace

*=== Stage 6: plain-text export for pattern checks in X =======================
export delimited using "`project'/`outname'_check.csv", replace quote

di as result "Done. Output: `project'/`outname'.dta  (N = " _N ", k = " c(k) ")"
di as result "Delete `outname'_check.csv from X once verification passes."
```

Notes for Stata:

- `labelbook, problems` returns `r(notused)`: label definitions no longer attached to any variable. Without the `label drop`, stripped place-name labels remain in the saved file and are visible with `label list`.
- `exit 111` and `exit 459` stop the do-file with a non-zero return code, so a batch run fails visibly.
- If `text_redact` contains `strL` variables, `ustrregexra` works but is slow on large files; that is acceptable.
- Names with a double quote inside will break the macro loop. Rare; if the assertion stage reports leaks that redaction should have caught, that is the first thing to check.

---

## R skeleton

**Not executed; partial.** Covers checks, label harvest, drop and strip, redaction, assertions, save and export. Not included: name harvest from list variables, ID crosswalk, guarded single-word redaction, byte scan, free-text review outputs. Port these from the Stata and Mata sections below, and test on synthetic data first.

```r
#==============================================================================
# deidentify.R — reads Y, writes de-identified copy to X, temp files stay in Y.
#==============================================================================
suppressPackageStartupMessages({ library(haven); library(stringr) })

confidential <- "<path to Y>"
project      <- "<path to X>"
dataset      <- "<filename>.dta"
outname      <- "<filename>_deid"

drop_vars        <- c("<variables to drop>")
geo_strip        <- c("<numeric geography: strip labels, keep codes>")
geo_encode       <- c("<string geography: convert to codes>")
text_redact      <- c("<free-text variables to redact>")
staff_label_vars <- c("<variables whose value labels hold staff names>")
phone_regex      <- r"(<from references/countries.md>)"   # R raw string: paste the regex as printed
id_regex         <- ""        # leave empty if none

#--- Stage 0: checks -----------------------------------------------------------
infile <- file.path(confidential, dataset)
stopifnot("Input file not found" = file.exists(infile),
          "Project folder not found" = dir.exists(project))
tmpdir <- file.path(confidential, "tmp_deidentify"); dir.create(tmpdir, showWarnings = FALSE)

df   <- haven::read_dta(infile)
n_in <- nrow(df)
expected <- c(drop_vars, geo_strip, geo_encode, text_redact, staff_label_vars)
missing_vars <- setdiff(expected, names(df))
if (length(missing_vars)) stop("Stopping. Not found in data: ", paste(missing_vars, collapse = " "))

#--- Stage 1: harvest staff names from value labels ----------------------------
staff_names <- character(0)
for (v in staff_label_vars) {
  labs <- attr(df[[v]], "labels", exact = TRUE)
  if (!is.null(labs)) staff_names <- c(staff_names, names(labs))
}
staff_names <- unique(staff_names[nchar(staff_names) > 2])
message("Harvested ", length(staff_names), " staff names.")

#--- Stage 2: drop and transform -----------------------------------------------
df <- df[, setdiff(names(df), drop_vars)]
for (v in geo_strip)  df[[v]] <- haven::zap_labels(df[[v]])      # keeps codes
for (v in geo_encode) df[[v]] <- as.integer(factor(df[[v]]))     # alphabetical codes

#--- Stage 3: redact free text ------------------------------------------------
redact <- function(x) {
  for (nm in staff_names) x <- str_replace_all(x, fixed(nm, ignore_case = TRUE), "[NAME]")
  if (nzchar(phone_regex)) x <- str_replace_all(x, regex(phone_regex), "[PHONE]")
  if (nzchar(id_regex))    x <- str_replace_all(x, regex(id_regex), "[ID]")
  # literals from Step 6 iterations:
  # x <- str_replace_all(x, fixed("exact string", ignore_case = TRUE), "[NAME]")
  x
}
for (t in text_redact) df[[t]] <- redact(df[[t]])

#--- Stage 4: post-write assertions -------------------------------------------
leaks <- sum(sapply(text_redact, function(t)
  sum(sapply(staff_names, function(nm) sum(str_detect(df[[t]], fixed(nm, ignore_case = TRUE)), na.rm = TRUE)))))
if (leaks > 0) stop("Stopping. ", leaks, " staff-name occurrences survived redaction.")
stopifnot(nrow(df) == n_in)
for (v in c(geo_strip, geo_encode)) stopifnot(is.null(attr(df[[v]], "labels", exact = TRUE)))

#--- Stage 5: save via temp in Y, copy to X -----------------------------------
attr(df, "label") <- sprintf("De-identified working file. Source: %s. Created %s.", dataset, Sys.Date())
tmpfile <- file.path(tmpdir, paste0(outname, ".dta"))
haven::write_dta(df, tmpfile)
invisible(file.copy(tmpfile, file.path(project, paste0(outname, ".dta")), overwrite = TRUE))

#--- Stage 6: plain-text export for pattern checks in X ------------------------
write.csv(haven::zap_labels(df), file.path(project, paste0(outname, "_check.csv")), row.names = FALSE, na = "")
message("Done. Delete ", outname, "_check.csv from X once verification passes.")
```

Notes for R:

- Regex patterns from `references/countries.md` contain backslashes. In an ordinary R string every backslash must be doubled (`"(\\+251|0)..."`). The skeleton uses an R raw string, `r"(...)"`, available since R 4.0, so the pattern can be pasted exactly as printed. Do not add a second layer of escaping inside a raw string.
- `haven::zap_labels()` removes value labels and keeps the underlying codes. `haven::as_factor()` would do the opposite and must not be used on geography.
- `haven` does not carry unattached labels, so the Stata `labelbook` step has no R equivalent.
- `write_dta` requires variable names valid in Stata; rename anything with dots first.

---

## Python skeleton

**Not executed; partial.** Covers checks, label harvest, drop and strip, redaction, assertions, save and export. Not included: name harvest from list variables, ID crosswalk, guarded single-word redaction, byte scan, free-text review outputs. Port these from the Stata and Mata sections below, and test on synthetic data first.

```python
#==============================================================================
# deidentify.py — reads Y, writes de-identified copy to X, temp files stay in Y.
# Requires: pandas, pyreadstat
#==============================================================================
import os, re, sys, shutil
import pandas as pd
import pyreadstat

confidential = r"<path to Y>"
project      = r"<path to X>"
dataset      = "<filename>.dta"
outname      = "<filename>_deid"

drop_vars        = ["<variables to drop>"]
geo_strip        = ["<numeric geography: strip labels, keep codes>"]
geo_encode       = ["<string geography: convert to codes>"]
text_redact      = ["<free-text variables to redact>"]
staff_label_vars = ["<variables whose value labels hold staff names>"]
phone_regex      = r"<from references/countries.md>"
id_regex         = r""          # leave empty if none

#--- Stage 0: checks -----------------------------------------------------------
infile = os.path.join(confidential, dataset)
if not os.path.exists(infile): sys.exit(f"Input file not found: {infile}")
if not os.path.isdir(project):  sys.exit(f"Project folder not found: {project}")
tmpdir = os.path.join(confidential, "tmp_deidentify"); os.makedirs(tmpdir, exist_ok=True)

df, meta = pyreadstat.read_dta(infile)
n_in = len(df)
expected = drop_vars + geo_strip + geo_encode + text_redact + staff_label_vars
missing_vars = [v for v in expected if v not in df.columns]
if missing_vars: sys.exit(f"Stopping. Not found in data: {' '.join(missing_vars)}")

var_labels   = dict(meta.column_names_to_labels)
value_labels = dict(meta.variable_value_labels)          # {var: {code: label}}

#--- Stage 1: harvest staff names from value labels ----------------------------
staff_names = sorted({lab for v in staff_label_vars for lab in value_labels.get(v, {}).values()
                      if isinstance(lab, str) and len(lab) > 2})
print(f"Harvested {len(staff_names)} staff names.")

#--- Stage 2: drop and transform -----------------------------------------------
df = df.drop(columns=drop_vars)
for v in drop_vars + geo_strip: value_labels.pop(v, None)    # keep codes, detach labels
for v in geo_encode:
    codes = {k: i + 1 for i, k in enumerate(sorted(df[v].dropna().unique()))}
    df[v] = df[v].map(codes).astype("Int64")

#--- Stage 3: redact free text ------------------------------------------------
def redact(s: pd.Series) -> pd.Series:
    s = s.astype("string")
    for nm in staff_names:
        s = s.str.replace(re.escape(nm), "[NAME]", regex=True, flags=re.I)
    if phone_regex: s = s.str.replace(phone_regex, "[PHONE]", regex=True)
    if id_regex:    s = s.str.replace(id_regex, "[ID]", regex=True)
    # literals from Step 6 iterations:
    # s = s.str.replace(re.escape("exact string"), "[NAME]", regex=True, flags=re.I)
    return s
for t in text_redact: df[t] = redact(df[t])

#--- Stage 4: post-write assertions -------------------------------------------
leaks = sum(int(df[t].str.contains(re.escape(nm), case=False, regex=True, na=False).sum())
            for t in text_redact for nm in staff_names)
if leaks: sys.exit(f"Stopping. {leaks} staff-name occurrences survived redaction.")
assert len(df) == n_in
assert not any(v in value_labels for v in geo_strip + geo_encode)

#--- Stage 5: save via temp in Y, copy to X -----------------------------------
tmpfile = os.path.join(tmpdir, outname + ".dta")
pyreadstat.write_dta(df, tmpfile,
                     column_labels={v: var_labels.get(v) for v in df.columns},
                     variable_value_labels={v: m for v, m in value_labels.items() if v in df.columns},
                     file_label=f"De-identified working file. Source: {dataset}.")
shutil.copyfile(tmpfile, os.path.join(project, outname + ".dta"))

#--- Stage 6: plain-text export for pattern checks in X ------------------------
df.to_csv(os.path.join(project, outname + "_check.csv"), index=False)
print(f"Done. Delete {outname}_check.csv from X once verification passes.")
```

Notes for Python:

- `pyreadstat.write_dta` writes only the value labels passed in `variable_value_labels`, so dropping a variable from that dict is what detaches its label. Nothing unattached is saved.
- `pd.Series.str.replace` with `regex=True` replaces all occurrences. The trap is `str.replace(a, b, 1)` on a plain Python string.

---

## Date coarsening

Only if the researcher asked for it (Step 3). Keep the original variable name so analysis code keeps running; document the change in the removal log.

| Target | Stata | R | Python |
|---|---|---|---|
| Interview date to month-year | `replace intdate = mofd(intdate)` then `format intdate %tm` | `df$intdate <- format(df$intdate, "%Y-%m")` | `df["intdate"] = df["intdate"].dt.to_period("M").astype(str)` |
| Date of birth to birth year | `gen birthyear = year(dob)` then `drop dob` | `df$birthyear <- as.integer(format(df$dob, "%Y")); df$dob <- NULL` | `df["birthyear"] = df["dob"].dt.year; df = df.drop(columns="dob")` |
| Date of birth to age at interview | `gen age = floor((intdate - dob) / 365.25)` | `df$age <- floor(as.numeric(df$intdate - df$dob) / 365.25)` | `df["age"] = ((df["intdate"] - df["dob"]).dt.days // 365.25)` |

Compute age before coarsening the interview date. Where the survey uses a non-Gregorian calendar stored as strings, convert first or coarsen at string level (keep the year and month tokens). See `references/countries.md`.

---

## Several source files and module splitting

Decided at intake (Step 0, items 2 and 3). Two cases.

**The source is already several files** (one per module or roster, linked by ID). Run the same pipeline on each: put the per-file lists in a block per file, or loop over a list of file names with the lists looked up by file. Keep the linking IDs in every output file. The staff-name harvest usually comes from the cover or household file; harvest once, save the names to a local or object, and reuse across files.

**The source is one file and the researcher wants module outputs.** Add a stage after the redaction and assertions, before the delimited export. Each module keeps the ID variables plus its own variables, and uniqueness of the IDs is checked per file.

Stata, variable prefixes as module markers:

```stata
*=== Stage 5b: split by module ================================================
local idvars   "hhid"                          // add pid for individual-level modules
local modules  "a b c d"                       // prefixes: a_* b_* c_* d_*
local mod_a    "Module A: Household roster"
local mod_b    "Module B: Housing and assets"
local mod_c    "Module C: Agriculture"
local mod_d    "Module D: Consumption"

foreach m of local modules {
    preserve
        keep `idvars' `m'_*
        isid `idvars'
        label data "`mod_`m''. De-identified working file from `dataset'."
        save "`confidential'/tmp_deidentify/`outname'_`m'.dta", replace
        copy "`confidential'/tmp_deidentify/`outname'_`m'.dta" "`project'/`outname'_`m'.dta", replace
        export delimited using "`project'/`outname'_`m'_check.csv", replace quote
    restore
}
```

If modules are not prefixed, replace `keep `idvars' `m'_*` with an explicit variable list per module built from the section markers in the metadata log: `local vars_a "hhid a1 a2 ... "` and `keep `vars_`m''`. Individual-level modules need `isid hhid pid`, not `isid hhid`.

R:

```r
idvars  <- c("hhid")
modules <- list(a = "Module A: Household roster", b = "Module B: Housing and assets")
for (m in names(modules)) {
  keep <- c(idvars, grep(paste0("^", m, "_"), names(df), value = TRUE))
  sub  <- df[, keep]
  stopifnot(!anyDuplicated(sub[idvars]))
  attr(sub, "label") <- modules[[m]]
  haven::write_dta(sub, file.path(tmpdir, sprintf("%s_%s.dta", outname, m)))
  invisible(file.copy(file.path(tmpdir, sprintf("%s_%s.dta", outname, m)),
                      file.path(project, sprintf("%s_%s.dta", outname, m)), overwrite = TRUE))
}
```

Python:

```python
idvars  = ["hhid"]
modules = {"a": "Module A: Household roster", "b": "Module B: Housing and assets"}
for m, label in modules.items():
    keep = idvars + [c for c in df.columns if c.startswith(f"{m}_")]
    sub = df[keep]
    assert not sub.duplicated(idvars).any()
    tmp = os.path.join(tmpdir, f"{outname}_{m}.dta")
    pyreadstat.write_dta(sub, tmp, column_labels={v: var_labels.get(v) for v in sub.columns},
                         variable_value_labels={v: vl for v, vl in value_labels.items() if v in sub.columns},
                         file_label=label)
    shutil.copyfile(tmp, os.path.join(project, f"{outname}_{m}.dta"))
```

The Step 6 checks then run per output file.

## Verification snippets Claude runs in X

Once the de-identified file and its `_check.csv` are in X, Claude can read them. Run the checks through whichever tool is available. Never print rows; print counts, variable names, and detached distinct values only.

**Header scan (any language).** Match variable names against the token list in SKILL.md Step 6, case-insensitive. Report matches with their labels.

**Pattern scan of the CSV (Python).**

```python
import re, pandas as pd
df = pd.read_csv("<X>/<outname>_check.csv", dtype=str, keep_default_na=False)
phone = re.compile(r"<phone_regex>"); coord = re.compile(r"-?\d{1,2}\.\d{4,}")
for v in df.columns:
    s = df[v]
    n_phone = s.str.contains(phone).sum()
    n_coord = s.str.contains(coord).sum()
    if n_phone or n_coord:
        print(f"{v}: phone-like={n_phone} coordinate-like={n_coord}")
```

Coordinate-like hits in variables that are legitimately decimal (yields, prices) are expected. Judge by variable name and label, and by whether two such variables sit next to each other.

**Geography check (Stata, on the `.dta`).**

```stata
use "<X>/<outname>.dta", clear
label dir                         // should list no place-name label sets
foreach v of varlist <geo vars> {
    local lbl : value label `v'
    di "`v': label = `lbl'"       // all should be empty
    confirm numeric variable `v'
}
```

**Free-text screen (detached distinct values, Stata).**

```stata
foreach t of varlist <text vars> {
    di as text _n "==== `t' ===="
    quietly levelsof `t' if !missing(`t'), local(vals)
    foreach x of local vals {
        di `"`x'"'
    }
}
```

`levelsof` returns values in sorted order with no row linkage, each wrapped in quotes so values containing spaces stay whole (do not add the `clean` option, which splits them into words). It fails if the combined length exceeds the macro limit; in that case use `contract t` then `list t, clean noobs` on the collapsed dataset, which is also detached from rows and IDs. The same in R: `sort(unique(df$t))`; in Python: `sorted(df[t].dropna().unique())`.

**Map byte-scan hits to variables (Python, `.dta` formats 118 and 119).** Run in X on the de-identified file. Prints section and variable names, never the matched bytes.

```python
import re, struct
from collections import Counter
b = open("<X>/<outname>.dta", "rb").read()
order = "<" if b"<byteorder>LSF" in b[:200] else ">"
mp = b.index(b"<map>") + 5
m = struct.unpack(order + "14Q", b[mp:mp + 112])
sections = ["stata_data", "map", "variable_types", "varnames", "sortlist", "formats",
            "value_label_names", "variable_labels", "characteristics", "data",
            "strls", "value_labels", "stata_data_end", "eof"]
K = (m[4] - m[3] - len(b"<varnames></varnames>")) // 129
ts = m[2] + len(b"<variable_types>")
typ = struct.unpack(order + "%dH" % K, b[ts:ts + 2 * K])
vs = m[3] + len(b"<varnames>")
names = [b[vs + 129 * i: vs + 129 * (i + 1)].split(b"\0")[0].decode() for i in range(K)]
w = {65526: 8, 65527: 4, 65528: 4, 65529: 2, 65530: 1, 32768: 8}
wid = [t if t <= 2045 else w[t] for t in typ]
rw = sum(wid); off = [sum(wid[:i]) for i in range(K)]
ds = m[9] + len(b"<data>")
pat = re.compile(rb"<bounded phone regex>")
hits = Counter()
for h in pat.finditer(b):
    sec = sections[max(i for i in range(14) if m[i] <= h.start())]
    if sec == "data":
        a = max(i for i in range(K) if off[i] <= (h.start() - ds) % rw)
        z = max(i for i in range(K) if off[i] <= (h.end() - 1 - ds) % rw)
        hits[(sec, names[a], names[z])] += 1
    else:
        hits[(sec, "", "")] += 1
for k, v in hits.items():
    print(k, v)          # (section, first variable, last variable): count
```

A hit whose first and last variables differ spans two fields and is an artefact of the layout; a hit inside one string variable is real and goes back to the script.

---

## Name harvest, redaction and byte scrub (Stata and Mata)

Tested in StataNow 19 on synthetic data mimicking a 16-file Survey Solutions export (2026-10). Names live only in Mata memory during the run; the script prints counts, never names.

How it fits in the do-file:

```stata
* After loading the main file, before dropping anything:
foreach v in <staff_label_vars> <respondent_name_label_vars> {
    local l : value label `v'
    mata: nm_add_label("`l'")
}
mata: nm_add_vars(tokens("<string variables holding names, e.g. member list slots>"))
mata: nm_finalize()                       // prints counts only

* Per free-text variable, after drop/transform:
quietly recast strL `t'                   // so "[NAME]" never truncates
mata: st_local("nchg", strofreal(nm_redact("`t'")))

* Assertion, per free-text variable:
mata: st_local("nl", strofreal(nm_leaks("`t'")))   // must be 0

* After saving to Y's temp folder, before copying to X:
mata: dta_scrub("`tmp'/`out'.dta")
mata: st_local("nb", strofreal(dta_leaks("`tmp'/`out'.dta")))   // must be 0
```

The routines:

- `nm_finalize()` builds two sets from the harvested strings: full names (two or more words) plus name bigrams, and single name words of four or more letters. Stopwords (honorifics, typed non-answers such as "none" or "don't know", and ordinary words that recur in names in the study language) are excluded from words and bigrams. The stopword list in the code is generic; extend it per country and topic.
- `nm_redact()` works on the distinct values of a variable, not on cells. It replaces phone numbers, then full names and bigrams case-insensitively anywhere, then single words only where they appear capitalised (or in capitals) and in fewer than three distinct values of the variable. Replacement regexes never match inside an existing `[MARKER]`.
- `nm_leaks()` counts distinct values that still contain a full name, a bigram, or a phone number.
- `dta_scrub()` zeroes the slack bytes after the terminator in the fixed-width fields of a saved `.dta` (formats 118 and 119): variable names (129 bytes), formats (57), value-label names (129), variable labels (321), and every `str#` cell. It reads the section offsets from the file's `<map>`. Data are unchanged: in the test, `cf _all` against unscrubbed copies passed for all files.
- `dta_leaks()` searches the raw bytes of a saved file, lowercased, for every full name and bigram.

```stata
mata:
PHONE_RX = "<bounded phone regex from countries.md; see Lessons below>"
NM_RAW = J(0, 1, "")
NM_PAT = J(0, 1, "")
NM_TOK = J(0, 1, "")

void nm_add_label(string scalar lbl)
{
    external string colvector NM_RAW
    real colvector v
    string colvector t
    if (lbl == "") return
    st_vlload(lbl, v, t)
    NM_RAW = NM_RAW \ t
}

void nm_add_vars(string rowvector vars)
{
    external string colvector NM_RAW
    real scalar i
    for (i = 1; i <= cols(vars); i++) {
        NM_RAW = NM_RAW \ uniqrows(st_sdata(., vars[i]))
    }
}

// Full names (2+ words) and name bigrams are redacted anywhere,
// case-insensitive. Single name words (4+ letters) only where capitalised
// and rare (fewer than 3 distinct values of the variable contain them).
void nm_finalize()
{
    external string colvector NM_RAW, NM_PAT, NM_TOK
    string colvector raw, pat, tok
    string rowvector w, stop
    string scalar s
    real scalar i, j
    // Generic stopwords; add the study's honorifics, kin terms and ordinary
    // words that occur in names (e.g. Ethiopia: ato wro obbo aadde abba haji
    // sheikh; plus topic words such as farm, plot, home)
    stop = tokens("mr mrs ms dr none other unknown dont know refused applicable near main small large")
    raw = uniqrows(ustrlower(ustrtrim(NM_RAW)))
    pat = J(0, 1, "")
    tok = J(0, 1, "")
    for (i = 1; i <= rows(raw); i++) {
        s = raw[i]
        if (s == "" | strpos(s, "#") > 0 | ustrregexm(s, "[0-9]")) continue
        s = ustrtrim(ustrregexra(s, "[^\p{L} ]+", " "))
        s = ustrregexra(s, " +", " ")
        w = tokens(s)
        if (cols(w) >= 2) pat = pat \ s
        for (j = 1; j <= cols(w); j++) {
            if (ustrlen(w[j]) >= 4 & !anyof(stop, w[j])) tok = tok \ w[j]
            if (j < cols(w)) {
                if (ustrlen(w[j]) >= 3 & ustrlen(w[j+1]) >= 3 & !anyof(stop, w[j]) & !anyof(stop, w[j+1])) {
                    pat = pat \ (w[j] + " " + w[j+1])
                }
            }
        }
    }
    NM_PAT = uniqrows(pat)
    NM_TOK = uniqrows(tok)
    printf("{txt}Name harvest: %9.0f raw entries, %9.0f full-name/bigram patterns, %9.0f single words\n", rows(raw), rows(NM_PAT), rows(NM_TOK))
}

real scalar nm_redact(string scalar var)
{
    external string colvector NM_PAT, NM_TOK
    external string scalar PHONE_RX
    string colvector x, U, R, lu
    real colvector hit, idx, wh
    real scalar i, k, nchg
    string scalar rx, T, TU
    x = st_sdata(., var)
    U = uniqrows(select(x, x :!= ""))
    if (rows(U) == 0) return(0)
    R = U
    for (i = 1; i <= rows(R); i++) R[i] = ustrregexra(R[i], PHONE_RX, "[PHONE]")
    lu = ustrlower(R)
    for (k = 1; k <= rows(NM_PAT); k++) {
        hit = strpos(lu, NM_PAT[k]) :> 0
        if (!any(hit)) continue
        rx = "(?i)(?<![\p{L}\[])\Q" + NM_PAT[k] + "\E(?![\p{L}\]])"
        idx = vec(selectindex(hit))
        for (i = 1; i <= rows(idx); i++) {
            R[idx[i]]  = ustrregexra(R[idx[i]], rx, "[NAME]")
            lu[idx[i]] = ustrlower(R[idx[i]])
        }
    }
    for (k = 1; k <= rows(NM_TOK); k++) {
        hit = strpos(lu, NM_TOK[k]) :> 0
        if (!any(hit)) continue
        idx = vec(selectindex(hit))
        rx = "(?<![\p{L}])\Q" + NM_TOK[k] + "\E(?![\p{L}])"
        wh = J(rows(idx), 1, 0)
        for (i = 1; i <= rows(idx); i++) wh[i] = ustrregexm(lu[idx[i]], rx)
        idx = vec(select(idx, wh))
        if (rows(idx) == 0 | rows(idx) >= 3) continue
        T  = ustrupper(usubstr(NM_TOK[k], 1, 1)) + usubstr(NM_TOK[k], 2, .)
        TU = ustrupper(NM_TOK[k])
        rx = "(?<![\p{L}\[])(?:\Q" + T + "\E|\Q" + TU + "\E)(?![\p{L}\]])"
        for (i = 1; i <= rows(idx); i++) {
            R[idx[i]]  = ustrregexra(R[idx[i]], rx, "[NAME]")
            lu[idx[i]] = ustrlower(R[idx[i]])
        }
    }
    nchg = 0
    for (i = 1; i <= rows(U); i++) {
        if (R[i] != U[i]) {
            idx = vec(selectindex(x :== U[i]))
            x[idx] = J(rows(idx), 1, R[i])
            nchg = nchg + rows(idx)
        }
    }
    st_sstore(., var, x)
    return(nchg)
}

// Leak test: values still containing a full name/bigram or a phone number
real scalar nm_leaks(string scalar var)
{
    external string colvector NM_PAT
    external string scalar PHONE_RX
    string colvector U, lu
    real colvector idx
    real scalar i, k, n
    U = uniqrows(select(st_sdata(., var), st_sdata(., var) :!= ""))
    if (rows(U) == 0) return(0)
    lu = ustrlower(U)
    n = 0
    for (i = 1; i <= rows(U); i++) n = n + ustrregexm(U[i], PHONE_RX)
    for (k = 1; k <= rows(NM_PAT); k++) {
        idx = vec(selectindex(strpos(lu, NM_PAT[k]) :> 0))
        for (i = 1; i <= rows(idx); i++) {
            n = n + ustrregexm(lu[idx[i]], "(?<![\p{L}\[])\Q" + NM_PAT[k] + "\E(?![\p{L}\]])")
        }
    }
    return(n)
}

// Stata leaves the unused bytes of fixed-width .dta fields (variable names,
// formats, value-label names, variable labels, str# cells) uninitialised, and
// they can hold fragments of earlier strings in memory, i.e. names. Zero every
// byte after the first NUL in those fields, in place. Formats 118/119 only.
void dta_scrub(string scalar fn)
{
    real scalar fh, len, C, K, rw, N, i, j, w, off, p, dstart, lsf
    real colvector m, m2, typ, wid, coff
    real rowvector sec, secw
    string scalar s, f, rel
    fh = fopen(fn, "rw")
    fseek(fh, 0, 1)
    len = ftell(fh)
    fseek(fh, 0, -1)
    s = fread(fh, len)
    rel = substr(s, strpos(s, "<release>") + 9, 3)
    if (rel != "118" & rel != "119") {
        fclose(fh)
        errprintf("dta_scrub: unsupported .dta release %s\n", rel)
        exit(459)
    }
    C = bufio()
    lsf = (substr(s, strpos(s, "<byteorder>") + 11, 3) == "LSF")
    bufbyteorder(C, (lsf ? 2 : 1))
    // map: 14 8-byte unsigned offsets, read as pairs of 4-byte words
    m2 = bufget(C, s, strpos(s, "<map>") + 4, "%4bu", 28)'
    m = J(14, 1, .)
    for (i = 1; i <= 14; i++) {
        m[i] = (lsf ? m2[2*i-1] + m2[2*i] * 2^32 : m2[2*i] + m2[2*i-1] * 2^32)
    }
    K = (m[5] - m[4] - strlen("<varnames></varnames>")) / 129
    typ = bufget(C, s, m[3] + strlen("<variable_types>"), "%2bu", K)'
    // fixed-width header fields: varnames, formats, value_label_names, variable_labels
    sec  = (m[4] + strlen("<varnames>"), m[6] + strlen("<formats>"), m[7] + strlen("<value_label_names>"), m[8] + strlen("<variable_labels>"))
    secw = (129, 57, 129, 321)
    for (j = 1; j <= 4; j++) {
        for (i = 1; i <= K; i++) {
            off = sec[j] + (i - 1) * secw[j]
            f = substr(s, off + 1, secw[j])
            p = strpos(f, char(0))
            if (p > 0 & p < secw[j]) {
                if (substr(f, p + 1, .) != (secw[j] - p) * char(0)) {
                    fseek(fh, off + p, -1)
                    fwrite(fh, (secw[j] - p) * char(0))
                }
            }
        }
    }
    // data section: str# cells
    wid = J(K, 1, .)
    for (i = 1; i <= K; i++) {
        if (typ[i] <= 2045)        wid[i] = typ[i]
        else if (typ[i] == 32768)  wid[i] = 8
        else if (typ[i] == 65526)  wid[i] = 8
        else if (typ[i] == 65527)  wid[i] = 4
        else if (typ[i] == 65528)  wid[i] = 4
        else if (typ[i] == 65529)  wid[i] = 2
        else if (typ[i] == 65530)  wid[i] = 1
        else {
            fclose(fh)
            errprintf("dta_scrub: unknown variable type code %f\n", typ[i])
            exit(459)
        }
    }
    rw = sum(wid)
    coff = runningsum(wid) - wid
    dstart = m[10] + strlen("<data>")
    N = (m[11] - m[10] - strlen("<data></data>")) / rw
    if (N != floor(N)) {
        fclose(fh)
        errprintf("dta_scrub: data section size does not match row width\n")
        exit(459)
    }
    for (j = 1; j <= K; j++) {
        if (typ[j] > 2045 | typ[j] < 2) continue
        w = typ[j]
        for (i = 1; i <= N; i++) {
            off = dstart + (i - 1) * rw + coff[j]
            f = substr(s, off + 1, w)
            p = strpos(f, char(0))
            if (p > 0 & p < w) {
                if (substr(f, p + 1, .) != (w - p) * char(0)) {
                    fseek(fh, off + p, -1)
                    fwrite(fh, (w - p) * char(0))
                }
            }
        }
    }
    fclose(fh)
}

// Byte-level leak test on a saved file: harvested full names / bigrams
real scalar dta_leaks(string scalar fn)
{
    external string colvector NM_PAT
    real scalar fh, len, k, n
    string scalar s
    fh = fopen(fn, "r")
    fseek(fh, 0, 1)
    len = ftell(fh)
    fseek(fh, 0, -1)
    s = strlower(fread(fh, len))
    fclose(fh)
    n = 0
    for (k = 1; k <= rows(NM_PAT); k++) n = n + (strpos(s, NM_PAT[k]) > 0)
    return(n)
}
end
```

Mata pitfalls hit while writing this:

- `selectindex()` on a 1x1 input returns a row vector, so loops over `idx[i]` break when a variable has exactly one distinct value. Wrap it in `vec()`.
- `bufget(..., "%8z")` reads a double, not an 8-byte integer. Read `.dta` map offsets as pairs of `%4bu` words.
- `st_sstore()` truncates to the variable's storage width; `recast strL` the variable before redacting.
- Mata globals defined in a do-file (`NM_PAT = ...`) persist across commands but not across `clear all`. Do not `clear all` between files.

---

## Free-text review outputs (Stata and Mata)

The two outputs of the free-text screen (SKILL.md Step 6), written by the de-identification script after redaction and the leak assertions, from the redacted data. Tested in StataNow 19 on synthetic data (2026-10).

- `txt_screen(vars, file)` writes, per variable, the distinct non-missing values, sorted, one per line, with no IDs, row positions or other variables. Line breaks inside a value become spaces. **Mode A:** write it to X (for example `check/<file>_text.txt`); Claude reads it. **Mode B:** write it to Y; only the researcher reads it.
- `tok_report(vars, file, cues)` writes a tab-separated list of tokens that deserve a human look, with the reason, the variable, and the number of distinct values containing the token. Reasons: `after_cue` (word following an honorific or kin term in `cues`), `name_word` (a harvested name word that the rarity guard left unredacted), `capitalised` (capitalised word that is not the first word of a value), `digit_run` (six or more consecutive digits). **Always written to Y**: it lists candidate names by construction. The researcher reads it in both modes.

```stata
* After redaction and the leak assertions, per output file:
mata: txt_screen(tokens("`text'"), "`screendir'/`out'_text.txt")   // X in mode A, Y in mode B
mata: tok_report(tokens("`text'"), "`tmp'/`out'_tokens.tsv", ("mr", "mrs", "ms", "dr", "chief"))
* extend the cue list with the study's honorifics and kin terms (countries.md)
```

`tok_report()` uses the `NM_TOK` set built by `nm_finalize()` (previous section); define that section's Mata first.

```stata
mata:
// Free-text screen: the distinct non-missing values of each listed string
// variable, sorted, one per line, with no IDs, row positions or other
// variables. Run on the REDACTED data. Mode A writes it to X, mode B to Y.
void txt_screen(string rowvector vars, string scalar fn)
{
    real scalar fh, i, j
    string colvector x, U
    string scalar u
    if (fileexists(fn)) unlink(fn)
    fh = fopen(fn, "w")
    for (i = 1; i <= cols(vars); i++) {
        x = st_sdata(., vars[i])
        U = uniqrows(select(x, ustrtrim(x) :!= ""))
        fput(fh, "==== " + vars[i] + "  (" + strofreal(rows(U)) + " distinct values) ====")
        for (j = 1; j <= rows(U); j++) {
            u = subinstr(subinstr(U[j], char(13), " ", .), char(10), " ", .)
            fput(fh, u)
        }
        fput(fh, "")
    }
    fclose(fh)
}

// Suspicious-token report, for the researcher, written to Y only. Lists
// tokens in the REDACTED text that deserve a human look, with the reason,
// the variable, and how many distinct values contain them. No IDs.
// Reasons:
//   after_cue    word following an honorific or kin term (cues, lowercase)
//   capitalised  capitalised word that is not the first word of a value
//   name_word    harvested name word left unredacted by the rarity guard
//   digit_run    six or more consecutive digits
void tok_report(string rowvector vars, string scalar fn, string rowvector cues)
{
    external string colvector NM_TOK
    real scalar fh, i, j, k, n
    string colvector x, U, keys
    string rowvector w
    string scalar lw, key
    transmorphic A
    A = asarray_create()
    for (i = 1; i <= cols(vars); i++) {
        x = st_sdata(., vars[i])
        U = uniqrows(select(x, ustrtrim(x) :!= ""))
        for (j = 1; j <= rows(U); j++) {
            w = tokens(ustrregexra(U[j], "[^\p{L}\p{N}\[\]]+", " "))
            for (k = 1; k <= cols(w); k++) {
                if (strpos(w[k], "[") > 0) continue
                lw = ustrlower(w[k])
                if (anyof(cues, lw)) continue
                key = ""
                if (k > 1) {
                    if (anyof(cues, ustrlower(w[k-1]))) key = "after_cue"
                }
                if (key == "" & rows(NM_TOK) > 0) {
                    if (anyof(NM_TOK, lw)) key = "name_word"
                }
                if (key == "" & k > 1 & ustrregexm(w[k], "^\p{Lu}\p{Ll}+$")) key = "capitalised"
                if (key == "" & ustrregexm(w[k], "[0-9]{6,}")) key = "digit_run"
                if (key == "") continue
                key = key + char(9) + vars[i] + char(9) + w[k]
                // count distinct values: add once per value
                if (asarray_contains(A, key)) {
                    if (asarray(A, key)[2] != j | asarray(A, key)[3] != i) {
                        asarray(A, key, (asarray(A, key)[1] + 1, j, i))
                    }
                }
                else asarray(A, key, (1, j, i))
            }
        }
    }
    keys = sort(asarray_keys(A), 1)
    if (fileexists(fn)) unlink(fn)
    fh = fopen(fn, "w")
    fput(fh, "reason" + char(9) + "variable" + char(9) + "token" + char(9) + "n_values")
    for (n = 1; n <= rows(keys); n++) {
        fput(fh, keys[n] + char(9) + strofreal(asarray(A, keys[n])[1]))
    }
    fclose(fh)
    printf("{txt}Suspicious-token report: %9.0f entries\n", rows(keys))
}
end
```

The report is a reading aid, not a test: it over-flags (place names, crop and tree names, sentence starts after a full stop) and cannot see names written in lower case that were never harvested. In the synthetic test it listed a surviving name after an honorific, two harvested name words, and a 12-digit number (the test data had not been through the phone rule).

---

## Synthetic test harness

Run before handing the scripts over (SKILL.md Step 5). In a scratch folder outside X and Y:

1. Write a generator script that rebuilds every source file from the metadata log: same file names, variable names, storage types, value-label names, and a few dozen rows. Include every variable the drop, strip, encode and text lists name, or the fail-loudly checks stop the test early.
2. Plant fakes: invented staff and respondent names in the value labels the script harvests; the same names in list or roster variables; the same names in free text in lower case, upper case, and with a misspelling; a name that appears only in free text; phone numbers in each common written form, including one at the end of a sentence, plus a decimal like `0.7912345678` that must *not* match; a duplicated household ID; a text variable with one distinct value; the platform system files the script should skip.
3. Run the extraction script and the de-identification script with the path lines pointed at the scratch folder (substitute them with `sed`).
4. Check: the log for `r(` errors; every expected output file exists; a case-insensitive grep of the plain-text exports **and of the raw binary files** for every planted fake returns nothing; the misspelling and the name-only-in-free-text cases show what the rules miss (expected; Step 6 handles them); a resave-and-compare (`cf _all`) shows the byte scrub changes no data.

---

## Lessons from field use

From two applications: an 11,000-variable SurveyCTO household file (about 3,000 households) and a 16-file Survey Solutions export (about 2,200 households). Each point cost an iteration.

**Leak test on full names, not tokens.** A single-token test ("any harvested name token appears") fails on real data: given names include ordinary words (Monday, Gift, Grace, Precious); surnames coincide with village names and everyday words. Hundreds of helper variables (weekday strings, schooling labels, item names) tripped it. Test instead for full names and for two consecutive harvested name tokens.

**Redaction of single tokens needs two guards.** Replace a lone name token only if it is capitalised in the text and appears in fewer than about 3 distinct values. Ordinary words that coincide with names recur across answers; a person named in a note does not. Full names and bigrams are replaced regardless. Expect residual over-redaction of a few sentence-initial words; add them to a stopword list.

**Typed non-answers poison the harvest.** "Don't know", "Not applicable", "None", and platform placeholders such as `##N/A##` typed or exported into name fields become "names". Stopword or skip them in every harvest.

**Protect markers from later rules.** After `[GROUP]` was inserted, a later rule matching the word "group" rewrote it to `[[PLACE]]`. Wrap every replacement regex so it never matches inside brackets, `(?<![\p{L}\[])\Q...\E(?![\p{L}\]])`, and skip marker words in token loops.

**Group names need a cue.** Savings groups are named with everyday words (Business, Kitchen, Tomato). Replace a harvested group word only when it stands next to a cue (VSLA, group, bank, club, savings); replace whole group names and bigrams anywhere.

**Place cues with a whitelist.** Trading centres and employers typed in free text are caught by "at / in / from / to X", "X trading centre", "X estate", "market X", with a whitelist of countries, cities, districts, months and programme words. Where the researcher keeps a geography level's labels, read those label texts into the whitelist inside the script.

**Phone regex: bounded, no dot separator, but allow a sentence-ending dot.** Random-draw strings (`0.789123...`), calculated areas and UUIDs match a tolerant pattern, so bound it on both sides and do not accept `.` between digit groups. But a right-hand boundary of `(?![0-9A-Za-z.])` misses a number that ends a sentence ("call 0911 22 33 44."). Use this form, with the country code, leading digits and digit count from `countries.md`:

```
(?<![0-9A-Za-z.])(\+?CC|0)[\s-]?[<leading digits>](?:[\s-]?[0-9]){<n>}(?![0-9A-Za-z]|[.][0-9])
```

**Name connectors leave the second name behind.** In "X Abba Y" (Ethiopia) the harvested full name was redacted and the patronymic after the connector survived. After the main redaction, also redact the word following `[NAME]` plus a connector, with a bounded lookbehind such as `(?<=\[NAME\]\s{1,3}(?:Abba|A/|H/)\s{0,3})\p{L}+`. Restrict it to comment-type fields if the connector is also an ordinary word elsewhere ("Abba koo", my father).

**Lower-case and suffixed names defeat the single-word guard.** Enumerators typing in a local language often write names in lower case, and agglutinative languages attach suffixes ("Abdallati"). Neither matches a capitalised whole-word rule. Accepting lower case raises over-redaction; matching known suffixes is country-specific (see `countries.md`). In one survey these two cases were most of what the free-text screen found.

**Whitelist ordinary vocabulary from the form.** Local tree and crop names coincided with harvested person names and were redacted from shade-tree and crop lists. Read the form's non-personal answer lists (crops, trees, units, materials) into a whitelist and never redact those words as single tokens.

**Comments about names are hot spots.** Most surviving names were in enumerator comments on name corrections, nicknames, and replacement households. Screen those first.

**Rare traits in prose.** Free text identified people without naming them (a farmer visited by a national leader, a household exporting its coffee). The screen should report these as quasi-identifiers for the researcher to decide.

**Byte-scan hits can span two fields.** All 44 phone-shaped hits in one file ran from the end of an interview key into the next field. Map hits to variables (snippet under Verification) before treating them as leaks.

**Do not edit a script while it runs.** An edit to a handed-over do-file failed only because Windows had it locked by the researcher's running Stata session. Without the lock it could have altered the run midway, since Stata reads do-files as it executes. Ask first.

**Short tokens in the header scan.** `lon` matched `long_income` and `shortlong`. Match short tokens only as whole name parts.

**Binary output can carry names in slack bytes.** See SKILL.md Step 5 and `dta_scrub()` above. Found only by grepping the raw bytes of synthetic output; no in-package check would have shown it.

**Encoded look-alike IDs.** A secondary file held a variable with the household-ID name whose values were 1..N codes and whose value label held the real IDs. Merging on it would have silently mismatched every row. The metadata label summary (entries equal to N, code range 1..N) gives it away.

**Value labels are frame-specific, and `preserve` writes identified data to the local temp folder.** To read label text inside the script, use Mata `st_vlload()`, or `label save` to a file in Y and parse it in Mata, stripping compound quotes with `char(96)`, `char(34)`, `char(39)` rather than literal quote characters.

**The do-file reader scans quotes and backticks inside Mata blocks.** A `` `" `` or `\"` sequence in Mata code, or even in a Mata comment, produces "mismatched quotes". Build quote characters from `char()`. Avoid generating Mata code from Python non-raw strings: `\b` becomes a backspace and `\n` a real newline.

**Speed.** Work on `uniqrows()` of each string variable, not on cells; prefilter with `strpos()` on the lowercased distinct values before running any regex. Build the label-to-variable map in one pass.

**Blank rather than drop where existing code drops by name.** Stata's `drop x` fails if `x` is absent, so removing device fields, comments or GPS variables can break downstream do-files that drop them. Keep the variable and set every value to missing; note the affected lines for anything the researcher insists on dropping.
