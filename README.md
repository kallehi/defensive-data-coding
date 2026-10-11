# defensive-data-coding

A Claude skill that makes Claude write and review data-pipeline code
defensively: every merge, constructed variable and sample restriction
gets a check that would fail on a plausible bug, rather than code that
merely runs and produces reasonable-looking numbers.

Geared towards survey data (household, individual, farm, firm or facility
surveys collected with CAPI tools such as SurveyCTO, ODK, Survey Solutions
or CSPro), but the core checks apply equally to administrative and other
tabular data. Stata is first-class; R and Python (pandas) equivalents are
included.

## Install

This repository is a Claude Code plugin and its own marketplace.

```
claude plugin marketplace add kallehi/defensive-data-coding
claude plugin install defensive-data-coding@defensive-data-coding
```

The skill triggers on its own when Claude writes or reviews code that
loads data, merges files, constructs variables or restricts a sample, or
when you ask it to check, validate or add asserts to a data step. To call
it explicitly, run `/defensive-data-coding:defensive-data-coding`.

To use the skill without the plugin system, copy
`skills/defensive-data-coding/` into `~/.claude/skills/` (all projects) or
`<project>/.claude/skills/` (one project).

For survey work, have the deployed CAPI form at hand: the skill asks for
it before writing checks.

## Why

Wrong data pipelines rarely crash. A duplicated merge key, a wildcard
that sweeps a "don't know" dummy into a count, a -99 code that enters an
average, or a code that means something other than the analyst assumed
(say, "illiterate (not educated)" counted as having education, or a
"who provided the training" question treated as an attendance dummy):
all of these run without error and produce plausible numbers. The skill
encodes checks designed to catch these failures where they happen.

## What it does

- **Asks for the instrument first.** For survey data, the skill asks for
  the deployed CAPI form (e.g. the SurveyCTO or ODK XLSForm), not the
  Word or PDF questionnaire, which is often outdated. The form's choice
  lists, labels and relevance conditions become an oracle: what each code
  means, who was asked, and which option dummies belong in a count.
- **Writes checks by concept**, at the point each operation happens:
  uniqueness of merge keys; declared merge match patterns, and a
  by-value comparison of variables the two files share besides the key
  (Stata silently keeps the master's copy); logical ranges; missingness
  logic, including Stata's missing-is-larger-than-any-number trap and
  unrecoded sentinel codes; row counts across sample restrictions;
  outliers; the exact contents of wildcard variable lists; and constant
  or all-missing variables.
- **Anchors checks to oracles**: design constants (sample size, clusters,
  arm sizes) and the instrument's code sets.
- **Tags every check** as halt, warn or log.
- **Enforces provenance**: no results typed into a manuscript by hand;
  every input constant named and sourced.
- **Retrofits safely** onto existing pipelines: it rebuilds to a scratch
  location and compares every output to prove the checks change nothing,
  pins known violations by count, and keeps a findings log instead of
  silently "fixing" analytical choices.

The skill covers mechanical checks only. It complements, and does not
replace, looking at distributions and adversarial review of whether the
code does what the analysis intends. Indicator definitions, sample
restrictions and outlier treatment remain the researcher's decisions.

## Testing status

The Stata code blocks in `SKILL.md` were tested in StataNow 19: each
check passes on clean data and fires on deliberately broken data. The R
and Python equivalents have not been executed; test them before relying
on them.

## Licence

MIT. See `LICENSE`.
