# Country parameters

Read this at Step 3 (geography levels, ID structure) and Step 6 (phone, ID, coordinate patterns).

This file holds a template and two **worked examples** (Ethiopia, Malawi). They show the level of detail that helps; they are not a coverage list, and they may be out of date (phone number plans and ID schemes change).

- **If the study country has an entry**, use it, tell the researcher which entry you used, and ask them to correct anything that disagrees with the metadata log or their knowledge.
- **If not**, draft an entry from the template at Step 3, from general knowledge, and mark it *unverified*. Before any pattern from it is used in a script, show the researcher the phone and ID formats and regexes and ask them to confirm or correct them. Save the confirmed entry to the project folder (`deidentify_country.md`) so later waves reuse it.

A wrong phone or ID pattern fails silently: it matches nothing, and the check reports PASS.

Regex patterns are written for ICU and PCRE, which Stata's `ustrregex*` functions, R's `perl = TRUE`, and Python's `re` all accept. Patterns are printed in their bare form; in an R string literal either double every backslash or use a raw string `r"(...)"`. Stata and Python raw strings take them as printed. Phone numbers in free text are often written with spaces or dashes; the "tolerant" pattern allows a single separator between digit groups. For scans and redaction use the **bounded** form, which does not match inside longer digit strings, UUIDs or decimals but does match a number at the end of a sentence; see `code-conventions.md`, Lessons from field use.

## Contents

- [Template](#template)
- [Ethiopia](#ethiopia)
- [Malawi](#malawi)

---

## Template

Copy this block for a new country. Delete guidance in angle brackets.

```
## <Country>

### Administrative hierarchy
<Level 1> → <Level 2> → <Level 3> → <Level 4> → <village or settlement level>
Typical analysis or sampling cluster: <level>.
Name-disclosure ladder (codes are kept at every level): keep names: <levels>; ask: <levels>; remove names: <levels>.
Local terms and spellings: <e.g. district = woreda; also spelled wereda>.
Urban hierarchy if different: <...>.

### Phone numbers
Mobile: <format, digit count, leading digits>. Landline: <format>.
International prefix: <+NNN>. Numbers commonly written as: <examples of spacing>.
Regex strict:   <...>
Regex tolerant: <...>

### National and other ID numbers
<ID name>: <format, digit count, regex>. Rollout or coverage note: <...>.
Other common IDs in survey data: <tax, voter, passport, health insurance, ration card, farmer registry>.

### Coordinates
Latitude <min> to <max>, longitude <min> to <max> (decimal degrees).
Regex for a decimal in range: <...>

### Calendar and dates
<Gregorian only, or local calendar in use; how dates appear in data; fiscal or agricultural year conventions>

### Languages and scripts
<Languages in which free text appears; scripts; Unicode range for a non-Latin script; transliteration variability>

### Naming conventions
<Structure of personal names; how many tokens a name field usually holds; whether family surnames exist; very common names>
Name connectors and honorifics (redact the word after them when it follows a redacted name): <...>
Suffixes that attach to names in free text: <...>. Whether enumerators commonly type names in lower case: <...>

### Survey ID structures
<How household or firm IDs are typically built; whether they embed geography codes; national statistical office EA code structure>
```

---

## Ethiopia

Worked example. Drafted 2026-09, phone regex corrected 2026-10 after use on a baseline survey; verify before relying on it.

### Administrative hierarchy

Region (kilil) → zone → woreda (district) → kebele (sub-district, lowest formal unit) → got or sub-kebele (village, informal).
Typical analysis or sampling cluster: kebele or enumeration area (EA).
Name-disclosure ladder (confirmed by a researcher working in Ethiopia, 2026-10; codes are kept at every level):

- keep names: region, zone
- ask: woreda (borderline)
- remove names: kebele, got or village

Group names (development groups, farmer groups, cooperatives) identify the kebele and are removed under the general rule for group names (SKILL.md Step 3); confirmed by the same researcher.
Local terms and spellings: woreda also spelled wereda; kebele also spelled qebele; some regions use "district" for woreda in English output. Chartered cities (Addis Ababa, Dire Dawa) use sub-city → woreda → ketena.
Region boundaries have changed since 2019 (new regions created from SNNPR); a region code list should state its vintage.

### Phone numbers

Mobile: 10 digits, `09XXXXXXXX` (Ethio telecom) or `07XXXXXXXX` (Safaricom Ethiopia, since 2022). International form `+251 9XXXXXXXX` or `+251 7XXXXXXXX` (9 digits after the country code, leading zero dropped). Landline: 10 digits starting `01` (Addis Ababa) or `02` to `05` elsewhere, e.g. `011XXXXXXX`.
Numbers commonly written as `0911 22 33 44`, `0911-223344`, `+251 911 223344`.

```
Regex strict (mobile):    (\+251|0)[79][0-9]{8}
Regex tolerant (mobile):  (\+?251|0)[\s-]?[79](?:[\s-]?[0-9]){8}
Regex bounded (use in scans and redaction):
    (?<![0-9A-Za-z.])(\+?251|0)[\s-]?[79](?:[\s-]?[0-9]){8}(?![0-9A-Za-z]|[.][0-9])
Regex strict (any line):  (\+251|0)[1-9][0-9]{8}
```

### National and other ID numbers

- **Fayda (national digital ID):** 12-digit Fayda Identification Number (FIN), often written in three groups of four. Rollout began 2021 and is expanding; older survey rounds will not contain it. Regex: `\b[0-9]{4}[\s-]?[0-9]{4}[\s-]?[0-9]{4}\b`. This pattern also matches 12-digit household IDs and some barcodes; check hits against variable labels.
- **Kebele ID card:** the traditional ID, issued locally. No national format; alphanumeric with kebele-specific numbering. Cannot be caught by a pattern; rely on variable labels ("ID card number").
- **Passport:** `EP` followed by 7 digits. Regex: `\bEP[0-9]{7}\b`.
- **Taxpayer identification number (TIN):** 10 digits. Appears in firm surveys. Regex: `\b[0-9]{10}\b` is too broad on its own; use only on variables labelled TIN.
- **Farmer registry, PSNP client, and health insurance (CBHI) numbers:** formats vary by region and program. Treat any variable labelled as a program ID as a direct identifier.

### Coordinates

Latitude 3.4 to 15.0 N, longitude 33.0 to 48.0 E (decimal degrees). Any pair of decimal variables in these ranges is a coordinate pair.
Regex for a plausible latitude: `\b(?:[3-9]|1[0-4])\.[0-9]{3,}\b`. For longitude: `\b(?:3[3-9]|4[0-7])\.[0-9]{3,}\b`. Both match ordinary decimals too; use them on the check CSV to find candidate variables, not as proof.

### Calendar and dates

The Ethiopian calendar is in everyday use: 13 months, year number about 7 to 8 behind the Gregorian year (2017 E.C. runs from 11 September 2024 to 10 September 2025). Dates in questionnaires and free text are often Ethiopian and may be stored as strings such as `15/03/2016`. CAPI system timestamps are Gregorian. When coarsening dates, check which calendar a variable is in first; a Gregorian date function applied to an Ethiopian date string will silently produce wrong years.
The agricultural year (meher season) runs roughly June to December, harvest through January; surveys often reference "last meher".

### Languages and scripts

Free text appears in Amharic and Tigrinya (Ethiopic script, Unicode U+1200 to U+137F), Afaan Oromo and Somali (Latin script, Qubee orthography for Oromo), and English. Enumerators frequently transliterate Amharic into Latin letters with inconsistent spelling (Alemayehu, Alemayhu, Alamayehu).
Regex to flag any Ethiopic-script text: `[\x{1200}-\x{137F}]` (ICU) or `[\u1200-\u137F]` (Python, R with `perl = TRUE`). Any field with Ethiopic hits needs translation before the free-text screen; a Latin-script name list will not catch names written in Ethiopic.

### Naming conventions

Personal names are given name followed by father's given name, and often the grandfather's given name. There are no family surnames. A "name" field therefore usually holds two or three tokens, and the second token of one person matches the first token of another (the father). Redacting a full name string will miss the same person referred to by given name only, and redacting single given-name tokens over-matches because many given names are extremely common (Mohammed, Abebe, Tigist, Almaz). Prefer full-string redaction of harvested names, then judge single-token survivors in the free-text screen by context.
Religious and honorific prefixes (Ato, Weyzero, Sheikh, Kes, Abba) often precede names in free text and are a useful anchor for a pattern search.

In Oromo areas (observed in a 2023 household survey in south-western Oromia):

- **Connectors.** Names are often written "X Abba Y", "X A/Y" or "H/X A/Y" (Haji, Abba). After full-name redaction, the word after the connector survives; redact it with a connector rule. "Abba" also means father ("Abba koo", my father), so restrict the rule to positions after `[NAME]`.
- **Suffixes.** Afaan Oromo attaches case and focus suffixes to names (-ti, -n, -f, -dha, -tu), which defeats whole-word matching.
- **Lower case.** Enumerators writing in Afaan Oromo often type names in lower case.
- **Plant names.** Many local tree names (shade trees, for example) coincide with personal names; whitelist the form's tree and crop lists.

### Survey ID structures

The Ethiopian Statistical Service (formerly Central Statistical Agency) codes enumeration areas as concatenated region, zone, woreda, kebele, and EA numbers. Household IDs built on EA codes therefore encode location down to the kebele even after geographic variables are stripped. Check the ID construction in the metadata log before deciding it can be kept; if it encodes geography below the analysis level, the researcher decides between keeping it (internal file, normally fine) and re-mapping (release file, often needed).
Listing files with GPS points and household head names commonly share the household ID with the survey file. If the listing is or will be public or shared with partners, the ID is a join key back to identified data.

---

## Malawi

Worked example. Drafted 2026-09 from general knowledge and used on one baseline survey; not checked against a national ID card or NSO codebook. Correct anything that disagrees with what you see in the metadata log.

### Administrative hierarchy

Region (Northern, Central, Southern) → district (28) → Traditional Authority (TA; also Sub-TA, STA) → Group Village Head (GVH; also "group village headman") → village.
Typical analysis or sampling cluster: village or GVH; the NSO enumeration area (EA) nests within TA and is used as the cluster in IHS/LSMS-type surveys.
Name-disclosure ladder (draft, not confirmed; codes are kept at every level): keep names: region, district; ask: Traditional Authority; remove names: GVH, village (headman names, see below).
Local terms and spellings: TA and GVH are almost always written as acronyms; "boma" is the district headquarters town; "chief" may refer to a TA, GVH, or village head. Urban hierarchy: city (Lilongwe, Blantyre, Mzuzu, Zomba) → ward → EA.
**Village and GVH names are frequently the name of the current or founding headman** (e.g. "GVH Kamwendo", "Chimwala village"). A geographic name field in Malawi is therefore also a personal-name field, and a headman named in free text locates the village. Strip GVH and village name labels and strings without exception; codes stay. The metadata extraction must not print these labels' text either, since the code-to-name mapping is what the working file withholds.

### Phone numbers

Mobile: 10 digits, `099XXXXXXX` or `098XXXXXXX` (Airtel), `088XXXXXXX` (TNM); Access Communications uses `07XXXXXXXX`. International form `+265 99X XXX XXX` (9 digits after the country code, leading zero dropped). Landline: `01XXXXXXX` (Lilongwe, Blantyre, Zomba, Mzuzu all under the single `01` code), 9 or 10 digits depending on how the zero is written.
Numbers commonly written as `0999 12 34 56`, `0888-123-456`, `+265 999 123 456`, `265999123456` (no plus).

```
Regex strict (mobile):    (\+?265|0)[789][0-9]{8}
Regex tolerant (mobile):  (\+?265|0)[\s-]?[789](?:[\s-]?[0-9]){8}
Regex bounded (use in scans over all variables):
    (?<![0-9A-Za-z.])(\+?265|0)[\s-]?[789](?:[\s-]?[0-9]){8}(?![0-9A-Za-z]|[.][0-9])
Regex strict (any line):  (\+?265|0)[1789][0-9]{7,8}
```

Do not allow a dot as a digit-group separator: SurveyCTO random-draw and calculated-acreage strings such as `0.7891234567...` then match as phone numbers. The bounded form also stops matches inside UUIDs (`key`) and long digit strings, while still matching a number that ends a sentence.

### National and other ID numbers

- **National ID (National Registration Bureau, NRB):** rolled out 2017 to 2018, coverage of adults is high, so recent surveys often record it. The ID number is an 8-character uppercase alphanumeric string (letters and digits mixed, no fixed positions). Regex: `\b[A-Z0-9]{8}\b` restricted to strings containing at least one letter and one digit: `\b(?=[A-Z0-9]{8}\b)(?=[A-Z0-9]*[A-Z])(?=[A-Z0-9]*[0-9])[A-Z0-9]{8}\b`. Too broad for a blind scan of all text; apply to variables labelled ID, and to comment fields.
- **Passport:** two letters followed by six digits, e.g. `MA123456`. Regex: `\b[A-Z]{2}[0-9]{6}\b`.
- **Taxpayer identification (TPIN):** 8 digits. Firm surveys only. Regex `\b[0-9]{8}\b` is too broad on its own; use only on labelled variables.
- **Program IDs:** Affordable Inputs Programme (AIP, formerly FISP) beneficiary or coupon numbers, Social Cash Transfer Programme (SCTP) household IDs, voter registration numbers, MASAF/public works IDs, farmer registry numbers, VSLA member numbers. Formats vary by program and district. Treat any variable labelled as a program or membership ID as a direct identifier.

### Coordinates

Latitude -17.2 to -9.3 (southern hemisphere, negative), longitude 32.6 to 36.0 (decimal degrees).
Regex for a plausible latitude: `-?(?:9|1[0-7])\.[0-9]{3,}`. For longitude: `\b3[2-5]\.[0-9]{3,}\b`. Both match ordinary decimals; use them on the check CSV to find candidate variables, not as proof. SurveyCTO exports GPS as four numeric fields (`*latitude`, `*longitude`, `*altitude`, `*accuracy`).

### Calendar and dates

Gregorian calendar throughout. Government fiscal year runs April to March. The agricultural year is a single rainy season, roughly November to April, harvest April to June; surveys reference "the last rainy season" or "the 2024/25 season". Lean season is December to February, which matters for interpreting food-security dates but not for de-identification.

### Languages and scripts

Free text appears in English and Chichewa (also called Chewa or Nyanja), with Tumbuka in the Northern Region and Yao, Lomwe, and Sena in parts of the South. All are written in Latin script, so non-Latin regexes are not needed, but enumerator spelling is inconsistent (Chichewa/Chicheŵa, Kamwendo/Kamwendu). Chichewa text often mixes in English numbers and names unchanged, so phone and ID patterns work on untranslated text; personal-name detection does not. Recommend translating flagged Chichewa fields before the final free-text screen.

### Naming conventions

Given name followed by a family surname; two tokens is the norm, three when a middle or Christian name is recorded. Surnames are shared across large clans and are extremely common (Banda, Phiri, Mwale, Chirwa, Nyirenda, Kamanga, Gondwe, Tembo, Moyo, Mhango, Kachale). A single-token surname match therefore over-matches badly; redact harvested full-name strings first, then judge single-token survivors by context. Honorifics and kin terms that anchor a name in free text: Mr, Mrs, Madam, Chief, TA, GVH, VH, Agogo (grandparent), Amayi/Amai (mother), Abambo (father), Bambo, Mayi, Aphunzitsi (teacher), Adotolo (doctor).
Because village and GVH names are headman names, a surname search will also hit place names; do not treat that as a false positive.

### Survey ID structures

NSO EA codes concatenate district, TA, and EA numbers, so an EA-based household ID encodes location to the TA. Project-built IDs in RCTs are often `district-TA-GVH-village-household` sequences and encode location to the village; a listing file with GPS and household-head names normally shares the same ID. For an internal working file, keep the ID and note in the removal log that it is a join key to the listing kept in Y. For a panel, never re-map.
