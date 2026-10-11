---
name: defensive-data-coding
description: Defensive-coding checks for data pipelines (cleaning, merging, variable construction) in Stata, R or Python. Geared towards survey data collected with CAPI tools (SurveyCTO, ODK, Survey Solutions, CSPro) but equally useful for administrative and other tabular data; for surveys it first asks for the deployed CAPI form and uses its codes, labels and relevance conditions as an oracle. Use whenever writing or reviewing code that loads data, merges files, constructs variables or indicators, or defines an estimation sample — ALWAYS when the code is AI-generated — and when retrofitting checks onto an existing pipeline. Trigger on any merge, new constructed variable, sample restriction, or request to check, verify, validate or add asserts. Covers uniqueness, merge patterns and silent name collisions, ranges, missingness and sentinel codes, wildcard variable lists, degenerate distributions, row counts, outliers, oracles, side-effect-free retrofitting, and provenance of results and constants.
---

# Defensive Data Coding

This is Layer 1 of a verification architecture: mechanical checks that catch faults a machine can detect without judgment. It does not replace looking at distributions (Layer 2) or adversarial review of intent (Layer 3) — it catches the errors that should never survive the point at which they happen.

## The one idea

A check is only worth writing if it could fail on a plausible bug. "Does it run and look reasonable" verifies almost nothing, because wrong pipelines usually run and produce reasonable-looking numbers. Every check below is designed to falsify a specific error: state what must be true, then halt loudly when it isn't. When you write an important check, confirm it works by running it once against deliberately broken data — a check you have never seen fail is a check you have not verified.

Prefer fail-fast: a hard halt on a correctness violation is safer than a fallback that silently produces a plausible wrong number. This inverts the usual production-software instinct (which favors graceful recovery) — a one-shot analysis for a paper is correctness-critical, not uptime-critical.

## First, for survey data: get the instrument

Before writing any check on survey data, ask the user where the questionnaire is, and ask specifically for the **deployed CAPI form** (SurveyCTO/ODK XLSForm `.xlsx`, Survey Solutions questionnaire export, CSPro dictionary), not the Word/PDF version. Paper questionnaires are often outdated: question wording, choice lists, codes and skip logic change during piloting and fieldwork, and the form on the server is what generated the data. If several form versions exist, use the one matching the data's form-version variable (e.g. SurveyCTO `formdef_version`). If no instrument is available, say so and treat every code's meaning as unverified.

In an XLSForm, the `survey` sheet gives each variable's `type` (`select_one <list>`, `select_multiple <list>`, `integer`, …), `label` and `relevance` (when it is asked); the `choices` sheet gives each list's codes and labels. Select-multiple questions typically export as one dummy per option, named `<var>_<code>`, with negative codes doubling the underscore (`<var>__88`), alongside the original string variable and any "other, specify" text field.

## The preconditions (concept first)

Each concept below is language-neutral. Syntax for Stata / R / Python is in the reference table at the end — write the check by concept, look up the verb.

### 1. Uniqueness

Assert that your ID variable uniquely identifies rows before any merge. A silent duplicate on the key turns a one-to-one merge into a many-to-many explosion that produces a larger, plausible, wrong dataset. This is the single most common way a merge corrupts data. Check uniqueness in both the master and the using/right file, not just one.

### 2. Match pattern

A merge's result is a claim about which records matched. Enforce it: declare the match types you expect (e.g. "every master row matches; no using-only rows") and halt if reality differs. Do not verify a merge by checking N alone — N can be preserved by a compensating error (one row dropped, one duplicated). Inspect the distribution of match outcomes, and account for every unmatched record as either known attrition or a bug.

Also check the variables the two files share *besides* the key. Stata's `merge` keeps the master's value of a shared non-key variable and silently discards the using file's (unless `update`/`replace` is specified). R and pandas add `.x`/`_x` suffixes instead, which isn't silent but breaks downstream references to the original name. Shared variables are often harmless copies (the same source variable kept in two build files), so compare them **by value**: halt if any differs, log them if all are identical. Renaming is the fix only when they differ.

### 3. Range / bounds

Every constructed variable has a logically admissible range. An age is in [0, 120]; a proportion is in [0, 1]; a row-mean of items each in [0, 1] is itself in [0, 1]. State the property and assert it over every row, not three hand-picked cases. Properties check the whole space; spot-checks check points and miss the case you didn't think of.

### 4. Missingness logic

State exactly when a constructed variable is allowed to be missing, and assert it. The classic trap: a row-mean function that silently averages over available items, so a respondent who answered 1 of 4 questions gets an index built from a single item — quietly non-comparable to a full responder. Force the decision into the open: either require all inputs present, or accept partial and document it, but never let it happen by accident.

Two more traps. First, in Stata a missing value is larger than any number, so `x >= 2` is true when `x` is missing: every comparison that builds an indicator needs `& !missing(x)` or `if !missing(x)`. Second, sentinel codes (88/99/-88 for don't know, refused, other) look like valid numbers. Recode them to missing (Stata extended missing `.d`/`.r`) before any arithmetic or comparison, and assert that none remain. Check the full tail first: in a count variable, 88 may be genuine.

### 5. Row-count continuity

After every sample restriction, assert the resulting N against what you expect, and log the count dropped with the reason. The estimation sample's N should be reconcilable, drop by drop, back to the raw file. If you can't explain a drop, that's a finding, not a nuisance.

### 6. Outliers and implausible values

Flag values outside plausible bounds even when they're not logically impossible (a household size of 47, a monthly consumption 200× the median). These are warnings, not halts — they may be legitimate — but they must surface for a human to eyeball rather than flow silently into an estimate.

### 7. Variable-list composition

Any variable list built by pattern (`ds x*`, `rowtotal(x*)`, `starts_with()`, `filter(regex=)`) is a claim about which variables it contains, and it is usually wrong in the same way: survey exports name don't-know, "other", "none" and yes/no gate variables with the same prefix as the substantive options (`q5_1 … q5_4`, `q5_88`, `q5__88`, `q5_yn`). A count built from such a list silently counts DK as a correct answer or the gate as an event. List option variables explicitly from the instrument's choice list. Where a pattern is unavoidable, assert that the matched set equals the expected set, so that an extra variable halts the run rather than entering the sum.

Pinning a list to what the pattern matches *today* documents it but does not validate it: if the wildcard already sweeps in a wrong variable, the pin locks the bug in. Take the expected set from the instrument's option list, never from the current data.

### 8. Degenerate distributions

A constructed variable that is constant, all missing, or nearly so (100% of households "save", everyone scores 1 on an item) passes every range assert and is almost always a construction bug. Warn on it. A warning costs nothing when the result is genuine and catches a whole class of errors that range checks miss.

A constant *validation* variable is a finding in itself. If a discrepancy flag or a difference between two measures is zero for every row, the two "independent" measures are almost certainly the same source (e.g. a total the CAPI form computed from a roster), and the cross-check is vacuous.

## Anchor to oracles

The strongest checks compare against truth that exists independently of this code — numbers you knew before the data was loaded. Design constants, registered allocations, sample sizes from survey documentation, published marginal totals. Assert against them directly.

Example: if the design has a known number of clusters, `assert` that the analysis sample hits exactly that count, or explain every deviation. Hunt for every such external number you have and turn it into an assertion — this is where mechanical checking gets its grip on correctness rather than mere internal consistency.

**The instrument is an oracle too.** Numbers are not the only external truth: for survey data, the CAPI form defines what each code *means*. Many construction bugs pass every range check because the codes are valid but mean something else — "16 = Illiterate (not educated)" counted as "any education", "0 = I don't turn" counted as turning compost, a select-one "who provided the training" treated as an attendance dummy, "2 = owned but locked away" silently coded as not owned. For every source variable an indicator is built from:

1. assert its values lie in the form's code set for that question, not an assumed 0/1 (e.g. `assert inlist(x, 0, 1, 2) if !missing(x)`);
2. print the label of every code the indicator counts as "yes" or "correct" — every code in an `inlist()`, `== k`, `!= 0` or `> 0` condition — and read it against the indicator's definition;
3. check the question's relevance condition: who was asked determines what missing means and which denominator the indicator has.

## Provenance: two rules

Verification of code cannot catch a number that entered the paper without going through code. Close that gap with two enforceable rules.

**Rule 1 — no results-side literals.** A number representing an output (a coefficient, an N, a mean, a p-value) must never be typed as a literal into the manuscript or a table. It must flow from code via the tool's export path (Stata `estout`/`putexcel`; R/Python written tables; `\input{}` of generated files in LaTeX). Enforceable by grepping the manuscript source for numeric literals in results tables — anything found is a violation.

**Rule 2 — input constants must be named and sourced.** A legitimate constant (a poverty line, a protocol threshold, a unit conversion) is allowed, but never as a bare number inline. Declare it once at the top of the script, named, with a source comment:

```stata
local poverty_line = 1.90    // 2011 PPP, World Bank; source: <doc>
local n_clusters   = 234     // design constant; source: trial registration
```

Enforceable by flagging any bare numeric literal below the header block. Bonus: every named, sourced constant doubles as a Layer 1 oracle — assert against `n_clusters` directly. Rule 2 and the oracle section reinforce each other.

## Severity

Not every check is equal. Tag each so the system stays one you'll actually keep running:

- **halt** — correctness violation; everything downstream is invalid (failed uniqueness, wrong match pattern, out-of-range constructed variable, variable list ≠ expected set, code outside the instrument's code set). Stop the run.
- **warn** — needs a human eye but may be legitimate (outliers, degenerate distribution, unexpected but explicable attrition). Print prominently, continue. Known, documented violations in existing data are pinned by count (see below), not left as open-ended warnings.
- **log** — context for the record (N dropped at each step, match-type counts).

Halting on trivia trains you to comment out asserts, which is how verification dies. Warning on catastrophes misses them. Set the level deliberately.

## Retrofitting checks onto an existing pipeline

Adding checks to code that already runs is different from writing them alongside new code:

- **Prove the checks have no side effects.** Write outputs to a scratch location, rebuild, and compare every output dataset with the current one (Stata `cf _all using <file>, all`; R `all.equal()`; pandas `DataFrame.equals()`). Only then touch the real outputs, and only after the user confirms.
- **Pin known violations by count, don't downgrade them.** If the data already contain one sentinel code that the construction mishandles, `assert r(N) == 1` (with a comment pointing to the finding) catches both a fix and a new occurrence; a warning catches neither.
- **Package checks as small reusable programs or functions** (one file, loaded at the top of each script), each tested once against deliberately broken data, rather than copying asserts into every file.
- **Keep a findings log.** Checks surface construction problems that need an analytical decision. Record each with file:line and evidence, and do not fix them silently.
- **A failed check is a hypothesis, not yet a bug.** Before reporting it, confirm it against the data and the instrument — a variable you expected in a sum may be a string that never entered it.

## Per-language reference table

Write the check by concept above; look up the verb here. Stata is first-class; R and Python give the nearest idiomatic equivalent.

| Concept | Stata | R | Python (pandas) |
|---|---|---|---|
| Uniqueness of key | `isid id` | `stopifnot(!anyDuplicated(df$id))` or `assertr::assert(df, is_uniq, id)` | `assert df["id"].is_unique` |
| One-to-one join, enforce match | `merge 1:1 id using f, assert(match)` | `dplyr::inner_join(a, b, "id", relationship = "one-to-one")` | `a.merge(b, on="id", validate="one_to_one")` |
| Declare allowed match types | `merge 1:1 id using f, assert(match master)` | inspect join, `stopifnot()` on unmatched | `validate=` + check indicator via `how="outer", indicator=True` |
| Inspect match distribution | `tab _merge` | count by join indicator | `df["_merge"].value_counts()` (from `indicator=True`) |
| Range / bounds (all rows) | `assert inrange(x, 0, 1) if !missing(x)` | `stopifnot(all(x >= 0 & x <= 1, na.rm = TRUE))` | `assert df["x"].between(0, 1).all()` |
| Values in instrument code set | `assert inlist(x, 0, 1, 2) if !missing(x)` | `stopifnot(all(x %in% c(0, 1, 2) \| is.na(x)))` | `assert df["x"].dropna().isin([0, 1, 2]).all()` |
| Missingness logic | `assert missing(idx) == missing(a, b, c, d)` | `stopifnot(is.na(idx) == (is.na(a) & is.na(b)))` | `assert (df["idx"].isna() == df[cols].isna().all(1)).all()` |
| Row-count assertion | `count`, then `assert r(N) == <expected>` | `stopifnot(nrow(df) == expected)` | `assert len(df) == expected` |
| Require all inputs present | build index only if `!missing(a,b,c,d)` | `rowMeans(m)` with `na.rm = FALSE` | `df[cols].mean(axis=1)` (NaN propagates by default) |
| Outlier flag (warn) | `count if x > <bound>` then display | `sum(x > bound, na.rm = TRUE)` | `(df["x"] > bound).sum()` |
| Shared non-key variables: compare by value | see block below | `s <- setdiff(intersect(names(a), names(b)), "id")`; `m <- merge(a[c("id", s)], b[c("id", s)], by = "id")`; `stopifnot(all(sapply(s, function(v) isTRUE(all.equal(m[[paste0(v, ".x")]], m[[paste0(v, ".y")]])))))` | `s = (set(a.columns) & set(b.columns)) - {"id"}`; `m = a.merge(b, on="id", suffixes=("_a", "_b"))`; `assert all(m[f"{v}_a"].equals(m[f"{v}_b"]) for v in s)` |
| Indicator with missing guard | `gen d = (x >= 2) if !missing(x)` | `d <- x >= 2` (NA propagates by default) | `d = (df["x"] >= 2).where(df["x"].notna())` |
| No sentinel codes remain | `assert !inlist(x, 88, 99, -88)` | `stopifnot(!any(x %in% c(88, 99, -88)))` | `assert not df["x"].isin([88, 99, -88]).any()` |
| Pattern list = expected set | see block below | `stopifnot(setequal(grep("^q5_", names(df)[sapply(df, is.numeric)], value = TRUE), opts))` | `assert set(df.filter(regex=r"^q5_").select_dtypes("number").columns) == set(opts)` |
| Degenerate distribution (warn) | see block below | `if (length(unique(na.omit(x))) <= 1) warning("x degenerate")` | `if df["x"].nunique() <= 1: warnings.warn("x degenerate")` |

Stata blocks for the rows above (tested in StataNow 19: each fires on broken data and passes on clean data):

```stata
* Shared non-key variables: compare by value (master in memory, key id)
qui describe using "f.dta", varlist
local u `r(varlist)'
qui ds
local m `r(varlist)'
local key id                       // must be a local: list c - id would read id as a macro name
local c : list m & u
local c : list c - key
if "`c'" != "" {
    tempfile uf
    preserve
        qui use id `c' using "f.dta", clear
        local i 0
        foreach v of local c {
            local ++i
            rename `v' _u`i'       // numbered names: a prefix can exceed 32 characters
        }
        qui save `uf'
    restore
    preserve
        qui merge 1:1 id using `uf', keep(match) nogenerate
        local bad ""
        local i 0
        foreach v of local c {
            local ++i
            cap assert `v' == _u`i'
            if _rc local bad `bad' `v'
        }
    restore
    if "`bad'" != "" {
        di as error "shared non-key variables differ: `bad'"
        error 9
    }
    di as text "shared non-key variables identical (harmless): `c'"
}

* Pattern list = expected set (numeric only: "specify" text fields share the prefix)
local opts q5_1 q5_2 q5_3 q5_4     // from the CAPI form's choice list
ds q5_*, has(type numeric)
local found `r(varlist)'
local extra : list found - opts
local absent : list opts - found
if "`extra'`absent'" != "" {
    di as error "q5_*: unexpected [`extra'] absent [`absent']"
    error 9
}

* Degenerate distribution (warn)
qui sum x
if r(N) == 0 | r(min) == r(max) di as error "WARN: x is constant or all missing"
```

Use `if … error 9` rather than `assert` for macro-level checks: `assert` evaluates once per observation and passes vacuously when there are 0 observations.

**Notes on cross-language traps:**

*Row-mean missingness defaults differ.* Stata `egen rowmean` and R `rowMeans(na.rm=TRUE)` silently average over available items; pandas `.mean(axis=1)` skips NaN too. To require completeness you must set the flag explicitly (`na.rm = FALSE`) or guard on missingness first. This is precisely the trap concept 4 guards against — the default is the dangerous one in all three languages.

*Join-validation strength differs.* Stata `assert()` and pandas `validate=` halt on violation; base-R `merge()` does not — use `dplyr`'s `relationship=` (R ≥ 4.1 / dplyr ≥ 1.1) or an explicit `stopifnot()` on duplicate keys before joining.

**Stata-specific traps:**

*Stale `r()` results.* Commenting out a `ds` (or `summarize`) while keeping the line that reads `r(varlist)` makes that line silently use the previous command's result.

*Undefined locals expand to nothing.* A misspelled or undefined local in `keep`, `foreach` or a varlist drops variables or skips loops without any error.

*Macro list operators take macro names.* In `local c : list a - b`, `b` is the name of a local, not a literal; `list a - id` subtracts the (probably empty) local `id`, not the word "id".

*Pattern lists include string fields.* `ds q5_*` also matches `q5_specify`/`q5_oth`; compare numeric variables only (`ds q5_*, has(type numeric)`).

*32-character variable names.* Prefixing names in comparison scripts (`rename x n_x`) fails on long names (r(198)); use numbered temporaries (`_n1`, `_n2`, …) instead.

## Workflow when applying this skill

0. For survey data, ask for the deployed CAPI form before writing checks (see "First, for survey data").
1. Identify the operation class (load / join / construct / restrict) and pull the relevant concepts above.
2. Write checks by concept, at the point the operation happens — not batched at the end, where a failure can't tell you which step caused it.
3. Anchor at least one check to an external oracle (design constant, documented N), and for survey data check every indicator's source codes and labels against the instrument.
4. Tag each check halt / warn / log.
5. For any check that matters, run it once against deliberately corrupted data and confirm it fires before trusting it.
6. When retrofitting an existing pipeline, prove the checks have no side effects before touching real outputs.
