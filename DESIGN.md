---
name: titles-forecating
description: The pipeline has no interface of its own. What it renders is terminal text, a JSON file, a workbook and a notebook.
colors:
  # No palette of its own. The artefacts carry no colour.
  primary: null
  neutral-bg: null
typography:
  body:
    # No typeface of its own. The terminal and the spreadsheet choose.
    fontFamily: null
    fontSize: null
    fontWeight: null
    lineHeight: null
  title:
    fontFamily: null
    fontSize: null
    fontWeight: null
    lineHeight: null
    letterSpacing: null
rounded:
  md: null
spacing:
  sm: null
  md: null
---

# The design record of titles-forecating

This pipeline has no interface. It reads news feeds, writes a corpus, calls
a model and writes files. Nothing in the repository draws a pixel, and no
colour, typeface, radius or spacing value appears anywhere in the code.

The frontmatter above is therefore empty on purpose. Every token is `null`,
because the project holds no token to record. A filled-in palette here would
describe a screen the project does not have.

What the project does own is four surfaces, and each one is a file format
rather than a design: the text it prints to the terminal, the JSON it writes
into `data/forecasts/`, the workbook it writes next to it, and the notebook
that runs the same stages cell by cell. What follows records what the
repository decides about each of them.

## Overview

**Creative North Star: "the plain report"**

The pipeline is a tool for one person reading its output. The reports go to
standard output, the artefacts go to disk, and nothing is published. A
forecast is a table of headlines with a column of reasons next to it, and
the reasons are in Russian because the topics and the headlines are.

The confirmed rejections follow from that:

- No dashboard, no chart, no page.
- No colour in the terminal output. The stage headers are made of equals
  signs, and a stage reads the same in a monochrome terminal.
- No styling in the workbook. `pandas.to_excel` writes the cells and
  nothing else.
- No theme for the notebook. The plots come from pandas and matplotlib with
  their own defaults.

The key characteristics: a fixed set of stages, a fixed order of report
lines, plain text on standard output, and files on disk that carry the
numbers.

## Colors

The pipeline names no colour. There is no CSS, no theme and no colour
constant in the code, and the four consumers of the output apply their own
palette to whatever arrives.

- **The terminal**: the colours of the shell that runs the command. The
  pipeline has no say in them and records none.
- **The workbook**: the colours of Excel or of whatever opens the `.xlsx`.
  `openpyxl` writes the cells and lets the reader's application paint them.
- **The notebook**: Jupyter's own styling, and the matplotlib defaults of
  the plots.

The one thing the code decides about the output is which words are used,
and the words are Russian where the subject is Russian: the sheet names
`Все прогнозы` and `LLM заголовки`, the reason strings in the selection
scores, and the headlines themselves.

### Named rules

**The report is text or it is nothing.** A line that arrives without a
label cannot be read. Every stage announces itself with a header, every
metric has its name in front of it, and a metric that was not measured
prints `n/a` rather than a number.

## Typography

**Display Font:** none. **Body Font:** none.

The font belongs to the reader: the terminal's, the spreadsheet's or the
notebook's. The repository decides the shape of a line, and nothing about
the way it looks.

The console report has three kinds of line, and they are distinguished by
their text rather than by their weight:

| Kind | What it looks like | Where it appears |
| --- | --- | --- |
| Stage header | `========== STAGE: ETL ==========` | Once per stage |
| Labelled value | `  topic_hit=0.412  entity_f1=n/a` | The metrics summary |
| Indented item | `      • headline` and `        > lead` | The forecast summary |

The values are aligned with the width of the slug: `f"{slug:<12}"` in the
metrics summary, so the columns line up under one another. The forecast
summary indents by two spaces per level, so a headline, its lead and the
label above them read as one block.

The JSON carries the same hierarchy in its keys, and the workbook carries it
in its columns: `outlet`, `outlet_name`, `target_date`,
`forecast_strategy`, `forecast_profile`, `method`, `rubric`, `topic_label`,
`title`, `lead` and `top_topics`.

## Layout

There is one layout: the order of the stages. Each stage prints a header,
does its work, prints its own lines and returns.

```
[main] Mode=...  Outlets=...  Target=...  LLM=...  ForecastStrategy=...  ForecastProfile=...
========== STAGE: SCRAPE ==========
  kommersant: 412 records saved to data/raw/
========== STAGE: ETL ==========
========== STAGE: ANALYZE ==========
========== STAGE: BACKTEST ==========
========== STAGE: METRICS ==========
--- Summary ---
  kommersant   days=7  topic_hit=0.412  entity_f1=n/a  sem_sim=0.000  style=0.118  diversity=0.900
========== STAGE: FORECAST → 2026-04-02 [llm / default] ==========
  [Коммерсантъ]
    Top topics : ...
    LLM titles : 8
      • headline
        > lead
[main] Done.
```

The workbook mirrors that order. The first sheet is every forecast on every
outlet, `Все прогнозы`. The second holds the `llm` rows only,
`LLM заголовки`. Then there is one sheet per outlet, named with its slug.

## Components

### The stage header

A line of ten equals signs on each side of the stage name and its
arguments. The forecast header carries the target date and the strategy and
the profile, because those two change what the numbers mean.

### The metrics summary

One line per outlet and per method, with six metrics in a fixed order:
topic hit rate, entity F1, semantic similarity, style match and diversity,
with the number of days evaluated in front of them.

Every metric is printed to three decimal places. A metric that was not
measured prints `n/a`, and the semantic similarity prints `0.000` when the
API key is missing, which is a fact about the code and a trap for the
reader.

### The forecast summary

Per outlet: the name, the three strongest topics, the number of headlines
generated, and the first five headlines with their leads.

The lead is cut to 120 characters before it is printed, so the summary fits
a screen. The full lead is in the JSON and in the workbook.

### The JSON export

One object with one key per outlet, written with `ensure_ascii=False` and
an indent of two, so the Russian is readable in the file. Each outlet
carries the target date, the strategy, the profile, the topics, the
selection scores with their reasons, and the predictions.

### The workbook

Written by `pandas.ExcelWriter` with the `openpyxl` engine. The code sets
the sheet names and the column order, and nothing else: no fill, no font,
no column width, no frozen pane.

### The notebook

Twenty-one cells that run the same stages as the command line, with the
tables and the plots in between. The markdown cells are in Russian, and so
are the section comments in the code cells.

## Do's and Don'ts

### Do:

- **Do** print a header for every stage, so the reader can find the
  boundary of a stage in a long log.
- **Do** print `n/a` for a metric that was not measured.
- **Do** keep the metric order the same in the console, in the JSON and in
  the workbook.
- **Do** keep the Russian in the artefacts. The topics, the reasons and the
  headlines are the product.

### Don't:

- **Don't** add colour to the terminal output. A stage header made of
  equals signs reads the same in a monochrome terminal.
- **Don't** style the workbook. The code sets the sheet names and the
  columns, and a reader's application paints the rest.
- **Don't** add a plot to the command line output. The notebook is where
  the plots live.
- **Don't** print a zero for a metric that was not measured. `n/a` says
  what `0.000` hides.
