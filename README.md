# News forecast pipeline

Collects what three Russian news outlets published over the past 90 days, works out which stories they are likely to run next, and writes the expected headline and first paragraph for each outlet on a chosen date. It is the submission for the HSE Centaur Race forecasting competition for "Human + AI" teams ([contest page](https://mollab.hse.ru/centaur_hse)), and the whole pipeline runs from one command.

[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.10%2B-blue)](pyproject.toml)

---

## Quick start

```bash
uv sync
uv run python main.py --mode forecast --target 2026-04-02
```

Python 3.10 or newer, and `uv` to install it. The forecast stage needs an OpenRouter key. Put one in `.env`, copied from [`.env.example`](.env.example):

```bash
OPENROUTER_API_KEY=...
OPENROUTER_MODEL=anthropic/claude-3-haiku-20240307
OPENROUTER_BASE_URL=https://openrouter.ai/api/v1
```

Without a key every other stage still runs and the forecast comes out empty. `--no-llm` skips the call on purpose.

The forecast reads the corpus in `data/clean/`, which makes it the fast way in. To rebuild that corpus from the news sites, run the scrape first: it pulls 90 days of archive pages and is the slow stage by a wide margin.

| Command | What it does |
| --- | --- |
| `uv run python main.py --mode all --target 2026-04-02` | Everything: scrape, clean, analyze, backtest, metrics, forecast. |
| `uv run python main.py --mode scrape` | Collect the news into `data/raw/`. |
| `uv run python main.py --mode etl` | Clean and deduplicate into `data/clean/`. |
| `uv run python main.py --mode analyze` | Topics, entities, style profile. |
| `uv run python main.py --mode backtest` | Replay the last 7 days, one day at a time. |
| `uv run python main.py --mode metrics` | Score the backtest. |
| `uv run python main.py --mode forecast --target 2026-04-02` | Write the forecast into `data/forecasts/`. |

[`pipeline.ipynb`](pipeline.ipynb) runs the same stages cell by cell, with the tables and the plots.

## What it does

- **Collect three outlets.** RSS feeds and archive pages for Kommersant, Lenta.ru and Interfax, 90 days back from the date of the run. The Wayback Machine fills in when an outlet returns fewer than 50 records.
- **Clean the corpus.** Residual markup and control characters out, outlet-specific title debris out, exact URL duplicates out, opinion and blog pages out, and near-duplicates out: two titles with a character n-gram similarity above 0.85, published within 72 hours of each other under the same rubric.
- **Find the topics.** TF-IDF over the titles and the leads, 20 clusters per outlet, each named by its five strongest terms, plus a rolling 14-day frequency table, a volume and rubric noise check, entity extraction and a style profile.
- **Forecast four ways.** `inertia` carries yesterday's topics forward, `frequency` takes the 14-day leaders, `calendar` turns [`data/events/events.csv`](data/events/events.csv) into headlines, and `llm` writes headlines and leads through OpenRouter.
- **Select the topics two ways.** The `llm` strategy takes the frequency ranking as it is. The `hybrid` strategy scores every topic on frequency, inertia, calendar support and an outlet-specific prior, then sorts on the sum.
- **Write in the voice of the outlet.** The prompt carries the topic, the calendar, the entities, eight retrieved headlines from the same outlet and four style examples from it, and asks for two or three numbered headlines with a lead of up to 50 words.
- **Score the result.** The last 7 days are replayed with the training restricted to what came before, and five metrics come out: topic hit rate, entity F1, semantic similarity, style match and diversity.
- **Leave the artefacts behind.** Every run writes a timestamped JSON file and a workbook with one sheet per outlet into `data/forecasts/`, and every backtest writes one JSON file per outlet.

## How it works

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

`main.py` is the only entry point. It parses the flags and calls one function per stage, and each stage reads and writes files under `data/` without knowing about the others. `config.py` is imported everywhere and holds the paths, the dates, the outlet table and the thresholds.

The scrape collects three ways and joins them on the article URL. The RSS feed gives the most recent entries, with the lead taken from the summary. The archive pages give the history, and Kommersant needs one page per rubric per day, which is 14 rubrics for each of the 90 days. The Wayback CDX API runs only when an outlet has produced fewer than 50 records, and its records carry no title, so the ETL drops them at the first step. The date filter is strict and inclusive at both ends, and `data/raw/{slug}_raw.csv` is rewritten on every run rather than appended to.

The ETL keeps a row count and a date range after each of its steps and writes them to `{slug}_etl_audit.json`, with the per-day counts to `{slug}_etl_daily_counts.csv`. The near-duplicate check compares character n-grams of the titles, and only within one rubric and inside a 72-hour window, so the same headline republished under a second rubric survives the clean.

The topic step is TF-IDF over the title and the lead together, 20 clusters per outlet, and a cluster is named by the five terms with the highest weights. The frequency table counts the clusters over the last 14 days. The backtest replays the last 7 days one day at a time, trains only on the days before, and skips any day with fewer than 14 days of history behind it.

The forecast stage differs by strategy and by profile. The `llm` strategy takes the frequency ranking as it is. The `hybrid` strategy adds the inertia count, the calendar overlap and the outlet prior to the frequency score, sorts on the sum, and hands the prompt an editorial instruction for that outlet. The `forward_look` profile then looks at the rows published in the last 14 days that already announce the target date, carry a planning phrase such as "запланировано", or name the date outright, and re-ranks the topics on that score.

The prompt asks for two or three numbered entries, each with a `ЗАГОЛОВОК` line and a `ЛИД` line of up to 50 words, and it goes to OpenRouter with a temperature of 0.4 and a ceiling of 800 tokens. The parser looks for those two labels, so a reply that changes them loses the lead. The scored results are filtered for topic and outlet fit, deduplicated, and reranked with at most four headlines per topic.

The export carries the `llm` predictions only, because `main.py` passes `llm_only=True` on the way out. The three baselines stay in the pipeline for the backtest and for the comparison in the report.

## Configuration

The command line:

| Flag | Default | What it does |
| --- | --- | --- |
| `--mode` | required | `scrape`, `etl`, `analyze`, `backtest`, `metrics`, `forecast` or `all`. |
| `--outlets` | all three | Comma-separated slugs: `kommersant`, `lenta`, `interfax`. |
| `--target` | `2026-04-02` | The date to forecast, as `YYYY-MM-DD`. |
| `--forecast-strategy` | `llm` | `llm` takes the frequency ranking; `hybrid` scores the ensemble. |
| `--forecast-profile` | `default` | `forward_look` boosts the topics that already announce the target date. |
| `--no-llm` | off | Skip the OpenRouter call. |
| `--enrich-leads` | off | Fetch the first paragraph of every article during the scrape. |

The environment, read from `.env`:

| Variable | Default |
| --- | --- |
| `OPENROUTER_API_KEY` | empty |
| `OPENROUTER_BASE_URL` | `https://openrouter.ai/api/v1` |
| `OPENROUTER_MODEL` | `anthropic/claude-3-haiku-20240307` |

The thresholds live in [`config.py`](config.py):

| Constant | Value | What it does |
| --- | --- | --- |
| `HISTORY_DAYS` | `90` | How far back the scrape goes. |
| `TARGET_DATE` | `2026-04-02` | The default forecast date. |
| `N_TOPICS` | `20` | Clusters per outlet. |
| `TOPIC_WINDOW` | `14` | Days in the rolling topic frequency. |
| `FREQ_WINDOW` | `14` | Days in the frequency baseline, and the history a backtest day needs. |
| `BACKTEST_DAYS` | `7` | The holdout window. |
| `DEDUP_THRESHOLD` | `0.85` | Similarity above which two titles are near-duplicates. |
| `DEDUP_WINDOW_HOURS` | `72` | How far apart two near-duplicates may be published. |
| `DEDUP_REQUIRE_SAME_RUBRIC` | `True` | Compare only within one rubric. |
| `REQUEST_DELAY` | `1.5` | Seconds of back-off after a throttled or failed request. |
| `REQUEST_TIMEOUT` | `15` | Seconds per request. |
| `MAX_RETRIES` | `3` | Attempts per request. |

The metrics stage asks the same endpoint for a second model, `qwen/qwen3-embedding-8b`, for the semantic similarity. Without a key that metric returns zero rather than a warning in the report.

## Development

```bash
uv sync                                          # install, from uv.lock
uv run python main.py --mode forecast --target 2026-04-02   # the forecast
uv run python main.py --mode all --target 2026-04-02         # the whole pipeline
jupyter lab pipeline.ipynb                      # the notebook, cell by cell
```

There is no build step. `pyproject.toml` sets `package = false`, so the project is a set of scripts rather than an installable package. There is no test runner in the dependency set either: the two files at the root, `test_scrapers.py` and `test_rss_content.py`, are scripts that print what the RSS feeds return, and both import function names that `scraper.py` no longer has, so they stop at the import.

| Path | What it holds |
| --- | --- |
| `main.py` | The CLI entry point. |
| `config.py` | Paths, dates, the outlet table, the thresholds. |
| `scraper.py` | The collection stage. |
| `etl.py` | The cleaning stage and the audit files. |
| `analyzer.py` | Topics, entities, the noise check, the style profile. |
| `event_calendar.py` | The calendar of known events. |
| `forecaster.py` | The four methods, the topic selection, the prompt, the export. |
| `backtester.py` | The holdout replay. |
| `metrics.py` | The five metrics and the report. |
| `pipeline.ipynb` | The same pipeline as a notebook, with the plots. |
| `data/raw/` | The scraped articles. Not tracked by git. |
| `data/clean/` | The clean corpus and the ETL audit files. |
| `data/events/` | `events.csv`, the calendar of known events. |
| `data/forecasts/` | The forecast runs, the workbooks and the backtest artefacts. |

The two forecast runs kept in the repository come with a hand-written shortlist of five headlines each: [llm + default](data/forecasts/forecast_2026-04-02_llm_default_shortlist.md) and [hybrid + forward_look](data/forecasts/forecast_2026-04-02_hybrid_forward_look_shortlist.md).

## License

MIT, © Alexander Blinov. See [LICENSE](LICENSE).
