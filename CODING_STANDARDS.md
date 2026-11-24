# R Coding Standards

Generic standards for R code (analysis projects and packages). Targets R >= 4.3,
dplyr >= 1.1, tidyr >= 1.3, purrr >= 1.0 (`in_parallel()`: >= 1.1),
vctrs >= 0.6.

**Precedence:** project-specific conventions (e.g. in the repo's `CLAUDE.md`)
override this file where they differ. When editing existing code, consistency
with the surrounding file beats a rule here; apply these rules to new code and
to code you are rewriting anyway.

**Context:** rules apply to analysis code and package code alike. Where package
code differs, a **Package:** note directly after the rule states the exception.

Most rules are phrased so that a reviewer can check them against a diff.

---

## 1. Project setup (analysis)

- Never call `setwd()` or `rm(list = ls())`. Build paths from the project root
  with `here::here("data", "raw", "sales.csv")`; no absolute paths.
- Load packages with `library()` at the top of the script, never mid-script.
- Lock package versions with `renv` (`renv::snapshot()` after adding a
  package). **Package:** not applicable; dependencies live in `DESCRIPTION`.
- A script that uses randomness calls `set.seed()` once, at the top.
- Pipelines are numbered scripts (`01_load.R`, `02_clean.R`, ...) that run in
  order from a fresh session. Raw data is read-only: no code writes to it.

---

## 2. Style

### Naming

- Use `snake_case` for all objects and functions. Avoid `camelCase`,
  `PascalCase` and dots in names (`calculate.mean`); dots are reserved for S3
  methods (`print.my_class`).
- Variables are nouns (`user_data`, `day_one`); functions are verbs
  (`calculate_mean`, `fit_model`).
- In functions that take `...`, prefix all other named arguments with `.`
  (`.data`, `.by`, `.names`), so they can't clash with or be partially matched
  by user-supplied names.
- Don't overwrite base/common names (`c`, `t`, `df`, `data`, `mean`, `T`, `F`).
- Write `TRUE`/`FALSE`, never `T`/`F`.

### Layout

- Indent with **4 spaces**, never tabs.
- Lines are at most **80 characters**.
- Space after commas, around binary operators (`=`, `<-`, `+`, `==`, `|>`),
  none inside parentheses: `mean(x, na.rm = TRUE)`, `x[, 1]`.
- Space after `if`/`for`/`while`, before `{`; `else` on the same line as `}`.
- Use `<-` for assignment, `=` only for arguments.
- One pipe step per line; the pipe ends the line. Calls (and function
  definitions) that do not fit on one line get one argument per line, closing
  `)` on its own line:

```r
data |>
    filter(year >= 2020) |>
    summarise(
        mean_value = mean(value),
        n = n(),
        .by = category
    )
```

### Pipes and anonymous functions

- Prefer the native pipe `|>` over magrittr `%>%` in new code. Don't mix the two
  within one pipeline.
- Use `\(x)` for all anonymous functions, never purrr-style `~ .x` formulas
  (also not inside `across()`).

### Returns

- Every function with a `{ }` body ends with an explicit `return()`;
  side-effect functions end with `return(invisible(x))`. Anonymous functions
  and one-line functions without braces are exempt.

### Pitfalls

- Index with `seq_along(x)` / `seq_len(n)`, never `1:length(x)` (gives
  `c(1, 0)` when `x` is empty).
- Use `&&` / `||` in `if` conditions; `&` / `|` in vectorised code.
- Use `if_else()` / `case_when()`, not `ifelse()` (drops classes such as dates
  and factors).
- Handle `NA` explicitly: set `na.rm` deliberately and decide how `NA` flows
  through comparisons and filters (`filter()` silently drops `NA` rows).

### Comments

- Comments explain *why*, not *what*. Delete commented-out code.

---

## 3. Data manipulation

### Grouping

- Use `.by` for per-operation grouping; avoid `group_by() |> ... |> ungroup()`
  for a single operation. `.by` always returns ungrouped data.
- Persistent `group_by()` is acceptable only when several consecutive verbs
  share the grouping; then end with `ungroup()` (or `.groups = "drop"`).

```r
# Do
summarise(data, total = sum(revenue), .by = c(company, year))

# Avoid
data |>
    group_by(company, year) |>
    summarise(total = sum(revenue)) |>
    ungroup()
```

### Column operations

- `across()` to apply functions to several columns; name outputs with `.names`.
  Pass extra arguments via a lambda, not through `...` of `across()`
  (deprecated): `across(x:z, \(v) mean(v, na.rm = TRUE))`.
- `pick()` (not `across()`) when you only need to *select* columns inside a
  data-masking verb.
- `reframe()` (not `summarise()`) when a summary returns more than one row per
  group.

### Joins

- Use `join_by()`, not character vectors: `join_by(company == id)`, not
  `by = c("company" = "id")`.
- Inequality and rolling joins via `join_by(id, year >= since)` and
  `join_by(id, closest(year >= since))`.
- State expectations about cardinality explicitly where they matter:
  `relationship = "one-to-one"` / `"many-to-one"`, `unmatched = "error"`
  (or `"drop"` deliberately). A silent row explosion from a join is a bug.

### Reshaping

- `pivot_longer()` / `pivot_wider()`, never `gather()` / `spread()`.
- `separate_wider_delim()` / `separate_wider_position()` /
  `separate_wider_regex()`, not `separate()` / `extract()`.

### Strings

- Use stringr (`str_*`) instead of base string functions in new code.
  **Package:** base string functions are fine when stringr is not otherwise a
  dependency (see section 8).

| Base R | stringr |
|---|---|
| `grepl(p, x)` | `str_detect(x, p)` |
| `gsub(a, b, x)` / `sub()` | `str_replace_all(x, a, b)` / `str_replace()` |
| `regmatches(x, regexpr(p, x))` | `str_extract(x, p)` |
| `substr(x, 1, 5)` | `str_sub(x, 1, 5)` |
| `nchar(x)` | `str_length(x)` |
| `strsplit(x, ",")` | `str_split(x, ",")` |
| `tolower()` / `toupper()` | `str_to_lower()` / `str_to_upper()` |
| `paste0()` / `sprintf()` templates | `str_c()` / `str_glue()` |

- Use `fixed()` for literal patterns (`str_detect(x, fixed("$"))`).
- Note `str_c()` propagates `NA` while `paste0()` turns it into `"NA"`; choose
  deliberately.

### Base R -> tidyverse

| Base R | Tidyverse |
|---|---|
| `subset(df, cond)` | `filter(df, cond)` |
| `df[order(df$x), ]` | `arrange(df, x)` |
| `aggregate(x ~ y, df, mean)` | `summarise(df, mean(x), .by = y)` |

---

## 4. Functions and tidy evaluation

### General

- A function does one thing and is named for it. Prefer pure functions; keep
  side effects (writing files, plotting) in clearly named functions.
- Return a consistent type regardless of input values (see section 7). Data
  frame–returning functions return tibbles.
- Prefer vectorised operations over loops for element-wise arithmetic
  (`x + y`, not a `for` loop over indices).

### Choosing the right tool

| Input to your function | Use |
|---|---|
| A bare column / expression from the user | `{{ var }}` (embrace) |
| A column name as a string | `.data[[var]]` (data-masking) / `all_of(vars)` (tidy-select) |
| A tidy-selection (`starts_with("x")`) | `{{ cols }}` inside a tidy-select verb, or `across({{ cols }})` in a data-masking verb |
| Several expressions | forward `...` unchanged |
| An environment value that might clash with a column | `.env$x` or `!!x` |

```r
my_summary <- function(data, group_var, summary_var) {
    result <- data |>
        summarise(mean = mean({{ summary_var }}), .by = {{ group_var }})
    return(result)
}

my_mean <- function(data, var) {               # var is a string
    return(summarise(data, mean = mean(.data[[var]])))
}

my_group_by <- function(data, vars) {         # tidy-select -> data-mask bridge
    return(group_by(data, across({{ vars }})))
}
```

### Rules

- Forward `...` as-is; no special syntax needed. Wrap in `c(...)` when the
  target argument takes a single tidy selection (`pivot_longer(c(...))`).
- Use `{{ }}` only on function arguments, never on local variables.
- Pick one style per function: embrace (`{{ }}`) *or* defuse-and-inject
  (`enquo()` + `!!`). Prefer embrace.
- Use `.data$col` / `.env$x` wherever a name could be ambiguous between data and
  environment. **Package:** always, together with the rlang import in section 8
  (avoids `R CMD check` notes).
- Don't use `.data` inside tidy-select verbs (deprecated); use `all_of()` /
  `any_of()`.
- Use `!!!` to splice lists/vectors of arguments: `group_by(!!!syms(vars))`,
  `mean(x, !!!args)`. Prefer `across(all_of(vars))` where it works.
- Name injection: use glue syntax with `:=` (`"mean_{{ var }}" := ...`,
  `"{name}" := ...`). To build such a name as a string (e.g. a default for a
  `.names`-style argument), use `rlang::englue("mean_{{ var }}")`.
- Collect dots with `rlang::list2(...)` (not `list(...)`) when you want
  splicing, name injection and trailing commas.

### Forbidden

- `eval(parse(text = ...))` to build code from strings. Use `sym()` /
  `.data[[ ]]` / injection instead.
- `get()` / `assign()` to look up or create variables by name. Use `.data[[ ]]`,
  a named list, or an environment explicitly.
- `paste0()` + `!!` on the LHS for new column names; use glue-style
  `"{name}" :=`.

---

## 5. Iteration and performance

### Iteration

- Use purrr `map_*()` instead of `sapply()` (type-unstable) and `lapply()`/
  `vapply()` in new code: `map_dbl()`, `map_chr()`, `map_lgl()`, `map_int()`,
  `map()` for lists. **Package:** if purrr is not a dependency, use `lapply()`
  or `vapply()` with an explicit `FUN.VALUE`; never `sapply()`.
- Bind results with `list_rbind()` / `list_cbind()`; `map_dfr()`, `map_dfc()`,
  `map2_dfr()`, `pmap_dfr()`, `imap_dfr()` are superseded.

```r
results <- splits |>
    map(\(split) fit_model(split)) |>
    list_rbind()
```

- Use `walk()` / `walk2()` / `pwalk()` for side effects (saving files, plots),
  not `map()` and not a `for` loop whose only job is a side effect.
- Never grow objects in a loop (`x <- c(x, new)`, `rbind()` inside a loop).
  Pre-allocate (`vector("list", n)`) or use `map()`.
- `for` loops are fine when iterations depend on each other; don't rewrite them
  for "speed" without measurement.

### Optimising

- **Profile before optimising.** Performance-motivated changes need evidence:
  `profvis::profvis()` to locate the bottleneck, `bench::mark()` to compare
  alternatives (realistic data sizes, multiple iterations, check memory).
  `system.time()` only for quick sanity checks.
- Optimise only the hot spots (80/20); prefer algorithmic improvements over
  micro-optimisations. Don't trade readability for small gains.

| Backend | Use when |
|---|---|
| dplyr | Default. Readability matters, joins/window functions, moderate data |
| data.table | Very large data (> ~1 GB), heavy grouping, profiled bottleneck in dplyr |
| duckdb / arrow (dbplyr) | Data larger than memory or already in a database |
| base R | No dependencies allowed, trivial operations |

- Don't mix backends within one pipeline without a reason; convert at clear
  boundaries.

### Parallelism

- Parallelise only CPU-heavy (or I/O-bound), independent tasks where each task
  takes far longer than the dispatch overhead. Never for fast element-wise work
  or tasks needing shared state.
- With purrr >= 1.1: `mirai::daemons(n)` + `map(x, in_parallel(f))`, and
  `daemons(0)` afterwards. Functions passed to `in_parallel()` must be
  self-contained: pass needed objects as arguments and use `pkg::fun()`.
- Respect machine limits; set the worker count explicitly, don't use all cores
  by default. Make parallel random numbers reproducible (seeded streams).

---

## 6. Errors and validation

### Messages

- Raise conditions with `cli::cli_abort()` / `cli_warn()` / `cli_inform()`,
  not `stop()` / `warning()` / `message()` with `paste()`ed text.
- Messages are specific (what went wrong), actionable (how to fix it, as an
  `"i"` bullet) and use inline markup: `{.arg x}`, `{.col name}`, `{.cls class}`,
  `{.val value}`, `{.fn fun}`, `{.file path}`.
- Validation helpers take `call = rlang::caller_env()` (and
  `arg = rlang::caller_arg(x)`) and pass them on, so the error points at the
  user-facing function.
- **Package:** give every error a class, `class = "<pkg>_error_<what>"`
  (e.g. `"mypkg_error_not_numeric"`), so callers and tests can match it.
  Analysis code needs no error classes.

```r
check_numeric <- function(
    x,
    arg = rlang::caller_arg(x),
    call = rlang::caller_env()
) {
    if (!is.numeric(x)) {
        cli::cli_abort(
            c(
                "{.arg {arg}} must be numeric, not {.cls {class(x)}}.",
                "i" = "Convert it with {.fn as.numeric} first."
            ),
            class = "mypkg_error_not_numeric",
            call = call
        )
    }
    return(invisible(x))
}
```

### How much to validate

| Function type | Validation |
|---|---|
| User-facing / exported | Comprehensive: type, length, range of every argument |
| Internal helper | Minimal: document assumptions, check critical invariants (`stopifnot()` is fine) |
| vctrs-based | Cast with `vec_cast()` in helpers; check types in constructors with `rlang::is_*()` / `vec_check_size()` |

- Fail early and loudly; never silently coerce, drop rows, or return `NULL` on
  bad input. Don't swallow errors with bare `try()`; if you catch, use
  `tryCatch()` with a specific condition class and handle it.

---

## 7. Classes and type stability

### Type stability

- A function's output type and size depend only on its input *types/sizes*,
  never on input *values*. `sapply()` and `if (...) 1L else 1.0` violate this.
- Check types at the input (`rlang::is_double(x)`, `vec_check_size(x, 1L)` or
  an explicit check) or cast at the output (`vec_cast(result, double())`).
  Don't use `vec_assert()` / `vec_is()` (questioning since vctrs 0.6).
- Base `c()`, `as.*()` and `length()` are fine in analysis code.
  **Package:** when combining heterogeneous inputs, prefer vctrs: `vec_c()`
  over `c()` (e.g. `c(factor("a"), "b")` is surprising), `vec_cast()` over
  `as.*()`, `vec_ptype_common()` to find a common type, `vec_size()` over
  `length()` for data frames.

### Choosing an OOP system

| Building | Use |
|---|---|
| Vector-like object that must live in data frames (units, custom dates, percentages, categorical types) | **vctrs** (`new_vctr()` / `new_rcrd()`) |
| New formal class with validated properties, inheritance or multiple dispatch | **S7** |
| Simple/internal class, quick prototype, extending an S3 ecosystem, very hot code | **S3** |
| Bioconductor, complex multiple inheritance, existing working S4 code | **S4** |

- Don't create a class when a list or tibble suffices.
- Migrate S3 -> S7 when the class needs validation or a hierarchy; question
  whether S4 features are really needed before keeping S4.

### S3

- Always provide a low-level constructor `new_<class>()` (via `structure()`),
  plus a user-facing helper and a validator for anything non-trivial.
- Provide `format()` and `print()` methods; `print()` ends with
  `return(invisible(x))`.
- Methods take `...` and are named `generic.class`.

### S7

- Define properties with types (`class_double`, ...) and a `validator` that
  returns a string describing the problem, or `NULL`.
- Access properties with `@`, never via `attr()`.
- Define generics with `new_generic()`, methods with `method(gen, Class) <-`;
  use `parent =` for inheritance and `super()` rather than `NextMethod()`.

```r
Range <- new_class(
    "Range",
    properties = list(start = class_double, end = class_double),
    validator = function(self) {
        if (self@end < self@start) {
            return("@end must be >= @start")
        }
        return(NULL)
    }
)
```

### vctrs classes

- Constructor (`new_*`, checks the type with `rlang::is_*()`, calls
  `new_vctr()`) + helper (casts input with `vec_cast()`, calls constructor) +
  `format()` method.
- Prefix the class with the package name (`pkg_percent`) to avoid clashes.
- Define coercion with `vec_ptype2.<a>.<b>()` (both directions, plus self) and
  conversion with `vec_cast.<to>.<from>()`; keep the rules symmetric.

```r
new_percent <- function(x = double()) {
    if (!rlang::is_double(x)) {
        cli::cli_abort(
            "{.arg x} must be a double vector.",
            class = "pkg_error_not_double"
        )
    }
    return(new_vctr(x, class = "pkg_percent"))
}
percent <- function(x = double()) new_percent(vec_cast(x, double()))
format.pkg_percent <- function(x, ...) paste0(vec_data(x) * 100, "%")

vec_ptype2.pkg_percent.pkg_percent <- function(x, y, ...) new_percent()
vec_ptype2.pkg_percent.double <- function(x, y, ...) double()
vec_ptype2.double.pkg_percent <- function(x, y, ...) double()
vec_cast.pkg_percent.double <- function(x, to, ...) new_percent(x)
vec_cast.double.pkg_percent <- function(x, to, ...) vec_data(x)
```

---

## 8. Package development

### Dependencies

- Add a dependency only for substantial gains (complex functionality such as
  regex, dates, web; less maintenance; better UX). Use base R for simple
  utilities, especially in widely used packages.
- Core tidyverse (dplyr, purrr, stringr, tidyr) is usually worth it; lubridate,
  forcats, readr, ggplot2 only with heavy use; never depend on the `tidyverse`
  meta-package; shiny only for apps.
- Call imported functions as `pkg::fun()` or import specific functions with
  `@importFrom`; never `library()` / `require()` inside package code.
- rlang/tidy eval: import `.data`, `.env` and `:=`
  (`@importFrom rlang .data .env :=`); `{{`, `!!`, `!!!` need no import.

### API design

- Mirror tidyverse conventions: data first (`.data`), then `...`, then optional
  `.by = NULL` forwarded with `{{ .by }}`, then other options with defaults.
- Accept user columns via `{{ }}` / tidy-select, not strings, unless the function
  is explicitly name-based.

### Exported vs internal

- Export only functions users call directly or may extend, with a stable API.
- Keep implementation details internal; they may change freely, get minimal
  validation and documentation. A leading `.` (`.validate_columns()`) may mark
  internals; be consistent within a package.

### Documentation

- Every exported function has roxygen docs: title, `@param` for every argument,
  `@return`, `@export`, and a working `@examples`.
- Mark tidy-eval arguments: `` <[`data-masked`][dplyr::dplyr_data_masking]> ``,
  `` <[`tidy-select`][dplyr::dplyr_tidy_select]> ``,
  `` <[`dynamic-dots`][rlang::dyn-dots]> ``.
- Document complex algorithms/formulas and non-obvious parameter interactions.
  Trivial internal helpers need no docs.

### Testing (testthat)

- Every exported function has unit tests covering edge cases (empty input,
  `NA`, `NULL`, wrong types) and error paths. Test errors by class
  (`expect_error(..., class = )`, see section 6) or snapshot
  (`expect_snapshot(error = TRUE)`), not by matching message text.
- Integration tests for end-to-end pipelines (output class, columns, rows).
- Property tests for invariants: idempotency, output size, type stability
  (`expect_type(f(integer(0)), "double")`).
- Tidy-eval functions: test both bare columns and expressions (`f(df, x * 2)`)
  and injection (`f(df, !!sym(var))`).
- vctrs classes: test `vec_ptype2` / `vec_cast` in both directions, `vec_c()`,
  and incompatible types erroring.
- Snapshot tests (`expect_snapshot()`) for printed/formatted output.
- Every bug fix comes with a test that fails without the fix.

---

*Condensed from the R skills in Sarah Johnson's gist
(https://gist.github.com/sj-io/3828d64d0969f2a0f05297e59e6c15ad), reorganised
by topic and adapted.*
