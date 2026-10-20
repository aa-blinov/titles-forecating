# Architecture

How the news forecast pipeline is put together.

Seven modules and one entry point. Each stage reads and writes files under
`data/` and knows nothing about the other stages, so a stage can be run on
its own with `--mode`.

## The shape

| Module | What it is responsible for |
| --- | --- |
| `main.py` | The command line. `--mode` picks the stage, and each stage calls one function in one module. 228 lines. |
| `config.py` | The paths, the dates, the outlet table, the OpenRouter settings and the thresholds. Importing it creates the `data/` directories. 135 lines. |
| `scraper.py` | The collection stage: the RSS feeds, the archive pages, the Wayback fallback and the raw CSV. The only module that talks to the news sites. |
| `etl.py` | The cleaning stage: the markup, the opinion filter, the near-duplicate check and the two audit files. Every later stage reads what this one writes. |
| `analyzer.py` | Topic clustering, the rolling frequency table, the volume and rubric noise check, the entity heuristics and the style profile. |
| `event_calendar.py` | The calendar of known events, read from `data/events/events.csv`, plus a recurring fallback layer. |
| `forecaster.py` | The four methods, the topic selection, the outlet priors, the prompt, the parsing and the export. The largest module. |
| `backtester.py` | The holdout replay, one day at a time. |
| `metrics.py` | The five metrics and the report. |

## The data flow

```
RSS feeds, archive pages, Wayback CDX
        |
        v
data/raw/{slug}_raw.csv
        |
        v
clean, filter, deduplicate  --->  data/clean/{slug}_clean.csv
                                 +  {slug}_etl_audit.json
                                 +  {slug}_etl_daily_counts.csv
        |
        v
TF-IDF + KMeans  ->  topics, entities, style
        |
        v
topic selection:  llm | hybrid   x   default | forward_look
        |
        v
OpenRouter  ->  headline + lead
        |
        v
data/forecasts/forecast_{date}_{strategy}_{profile}_{stamp}.{json,xlsx}
```

The scrape collects three ways and joins them on the article URL. The RSS
feed gives the most recent entries with the lead taken from the summary.
The archive pages give the history, and Kommersant needs one page per rubric
per day, which is 14 rubrics for each of the 90 days. The Wayback CDX API
runs only when an outlet has produced fewer than 50 records, and its
records carry no title, so the ETL drops them at the first step.

The ETL keeps a row count and a date range after each of its steps and
writes them to `{slug}_etl_audit.json`, with the per-day counts to
`{slug}_etl_daily_counts.csv`.

The topic step is TF-IDF over the title and the lead together, 20 clusters
per outlet, and a cluster is named by the five terms with the highest
weights. The frequency table counts the clusters over the last 14 days.

## Decisions worth knowing about

### The stages are files, not objects

Each stage reads a CSV, writes a CSV and returns. Nothing is passed from
the scrape to the ETL in memory.

The cost: the corpus on disk is the interface between the stages, so a
stage cannot be tested without a file from the stage before it. It also
means the committed `data/clean/` corpus is what the backtest and the
forecast read.

### The date is pinned in two places

`config.TARGET_DATE` is `2026-04-02` and `main.py` defaults `--target` to
the same string. `forecaster.combine_forecast` takes its default from
`config.TARGET_DATE`.

The cost: a call that skips the argument still forecasts April 2026.

### The history window slides with the clock

`HISTORY_FROM` is `TODAY - 90 days`, evaluated when `config` is imported.
The committed corpus covers 2025-12-30 to 2026-03-29, and a scrape run
today produces a different window.

The cost: the committed artefacts and a fresh run describe different
periods, and the two are not comparable.

### The near-duplicate check is deliberately narrow

Two titles are near-duplicates when their character n-gram similarity is
above 0.85, they were published within 72 hours of each other, and they
sit under the same rubric.

The cost: the same headline republished under a second rubric survives the
clean, and a duplicate across rubrics reaches the corpus.

### The topic count adapts instead of failing

`extract_topics` lowers `n_topics` when the corpus has fewer distinct
texts than clusters and falls back to a single cluster named `general`.

The cost: a small corpus produces one topic, quietly, and the forecast for
that run is written as if the topic had been found.

### The baselines stay in the pipeline

`inertia`, `frequency` and `calendar` generate no text, and the export
carries the `llm` method only. `main.py` passes `llm_only=True` on the
way out.

The cost: the three baselines exist for the backtest and the comparison in
the report, and their text-level metrics are absent on purpose rather than
zero.

### A missing key reads as a score of zero

`metrics.semantic_similarity` returns `0.0` when `OPENROUTER_API_KEY` is
missing, and the mean reaches the report as `0.000`.

The cost: a zero there means the metric was not measured, and it is
indistinguishable from a real zero in the summary line.

### The parser looks for two labels

`_parse_llm_response` looks for the `ЗАГОЛОВОК` and `ЛИД` lines in the
reply. A reply that changes the labels loses the lead, and a reply without
them loses the title as well, because the parser falls back to the first
line of the block.

The cost: the prompt and the parser are one contract, and the prompt is
written in Russian because the headlines are.

## Boundaries

- `config.py` is imported everywhere and holds no behaviour beyond the
  paths, the table and the thresholds.
- `scraper.py` is the only module that opens a connection to a news site.
  Everything after it works on files.
- `etl.py` owns the corpus. `analyzer.py`, `backtester.py` and
  `forecaster.py` read `data/clean/` and never clean anything themselves.
- `analyzer.py` owns the topics, the entities and the style profile, and it
  writes them where the forecaster can find them.
- `forecaster.py` owns the prompt and the export. It does not call the
  model itself; `_llm_chat` is the only place that does.
- `backtester.py` and `metrics.py` are the evaluation half. They read the
  corpus and the backtest artefacts, and they write their own reports.
