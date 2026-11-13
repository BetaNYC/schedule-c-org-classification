# schedule-c-org-classification

A Quarto website that renders a single reproducible report from NYC Council Schedule C disclosure data.

## Build

``` sh
quarto render        # writes docs/
quarto preview       # local server with live reload
```

Requires Quarto 1.8+ and R 4.5+. The `setup` chunk installs any missing R packages at render time by diffing `reqd_pkgs` against `installed.packages()`, so a fresh clone needs no manual package step.

`execute: freeze: false` in `_quarto.yml` means every render re-runs every chunk. Nothing is cached between builds.

## Layout

```         
index.qmd              the entire report: setup, data fetch, prose, tables
_quarto.yml            website config; output-dir is docs/
style/                 cil-theme.css, fonts.css, variables.css
fonts/                 Authentic Sans + New Computer Modern webfonts
images/                copied to docs/ via the `resources` key
radar-header.html      injected via include-before-body
radar-footer.html      injected via include-after-body
data/raw/              versioned source spreadsheets (committed)
data/raw-mainifest.csv fetch log: file, version, md5, fetched, url
docs/                  render output; GitHub Pages serves main branch /docs
```

## Data fetching

The `fetch-schedule-c` chunk pins the dataset by its **upstream publication date** rather than by fetch date, so the same file yields the same name for everyone:

``` r
FUNDED_DISCLOSURE_URL     <- "https://rnd.council.nyc.gov/.../funded_disclosure_FY2027.xlsx"
FUNDED_DISCLOSURE_VERSION <- "2026-07-02"
```

`fetch_raw()` writes to `data/raw/<name>_<version>.<ext>`, downloading only when that file is absent. Downloads land in a `.part` file first and are renamed on completion, so an interrupted transfer can't leave a truncated file that a later render mistakes for a good one. Every fetch appends a row to the manifest recording the md5, so revisions are visible in the diff even though `.xlsx` is opaque binary.

`upstream_version()` reads `Last-Modified` via base R's `curlGetHeaders()`. It is used two ways: to refuse a download whose version doesn't match the pin, and to emit a `message()` when a newer release exists. A new release is reported, never applied — bump `FUNDED_DISCLOSURE_VERSION` to move.

Raw data is committed. Older versions stay in place so past renders reproduce.

## `cil_table()`

Wraps a data frame in a CIL-styled flextable and returns an htmltools `div`.

``` r
data |> cil_table()                                   # columns fit their content
data |> cil_table(widths = c("Purpose of Funds" = 4)) # named columns overridden
data |> cil_table(widths = c(1.2, 2, 4), max_width = 3, heights = 0.5)
```

- `widths` — one value per column, or **named** values setting just those columns and leaving the rest content-fitted. A positional vector whose length doesn't match `ncol(data)` is an error rather than a silent misalignment, and an unknown column name is an error too.
- `max_width` — caps the content-fitted widths only; never the ones `widths` names.
- `heights` — applied with `hrule("exact")`, since flextable emits no height at all under the default `"auto"` rule.

Because the function ends in `htmltools_value()`, it returns HTML: flextable setters can't be piped onto its result. Size the table through the arguments.

Explicit widths add a `scroll-output--fit` class to the wrapper, which lets the table size to its columns and scroll horizontally. Without widths the table wraps to fit its container.

### Two flextable quirks the function works around

Both are load-bearing. Removing either silently breaks styling.

1.  **Shadow DOM.** flextable renders into a shadow root, which no page CSS can reach. `ft.shadow = FALSE` drops the static host, but the `tabwid.js` dependency *also* moves every `.tabwid` element into a fresh shadow root on `DOMContentLoaded`. Renaming the wrapper to `cil-tabwid` hides it from that selector. Both steps are required; either alone leaves the table unstyled.

2.  **`layout = "autofit"` discards column widths.** Explicit widths require `layout = "fixed"`, and the theme's global `table-layout: fixed` is then overridden back to `auto` in CSS so the widths survive.

## CSS

`style/cil-theme.css` holds the CIL design system. Three rules exist solely to support `cil_table()` output and are commented as such:

- `.scroll-output tbody tr::before` — suppresses the theme's row-number counter inside these tables. flextable emits a real `<thead>`, so the counter adds a phantom cell to body rows only and shifts every column one position.
- `.scroll-output table` / `.scroll-output--fit table` — restore auto layout, and give fitted tables an intrinsic width so their columns hold.
- `.scroll-output thead th` — `white-space: nowrap`, keeping header names on one line. `dim_pretty()` estimates header widths slightly under what browsers render, so computed widths alone still wrap.

## Debugging rendered output

Styling problems here have twice turned out to be scope problems, not CSS problems. When a rule appears to have no effect, confirm the element is reachable before changing the rule:

``` js
document.querySelectorAll('table').length          // 0 means it's in a shadow root
document.querySelector('.scroll-output table')     // null means selectors can't reach it
```

Measure after `DOMContentLoaded`. Scripts running earlier see a different DOM than the one that gets painted.

## Attribution

Anthropic's Opus 5 via Claude Code was used to write the file caching functions, to extend the `cil_table()` function from the preexisting Quarto report template, and to generate this README. The report was manually authored by Andrew Kittredge, BetaNYC Civic Innovation Lab Director.
