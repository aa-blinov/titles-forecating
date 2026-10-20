# AGENTS.md — agent context for `titles-forecating`

A Python pipeline that forecasts the headlines three Russian news outlets will publish on a given date, built for the HSE Centaur Race forecasting competition for "Human + AI" teams. It collects the outlets, cleans the corpus, clusters the topics, and writes the predicted headline and lead for each outlet through OpenRouter.

This file is what you need to work on the project without reading all of it.

## Run / build / test

```bash
uv sync                                     # install, from uv.lock
uv run python main.py --mode scrape         # collect the news (network, slow)
uv run python main.py --mode etl            # clean and deduplicate
uv run python main.py --mode analyze        # topics, entities, style profile
uv run python main.py --mode backtest       # replay the last 7 days
uv run python main.py --mode metrics        # score the backtest
uv run python main.py --mode forecast --target 2026-04-02
uv run python main.py --mode all --target 2026-04-02
jupyter lab pipeline.ipynb                  # the same pipeline, cell by cell
```

There is no build step. `pyproject.toml` sets `package = false`, so the project is a set of scripts rather than an installable package, and `uv sync` is the whole bootstrap.

There is no test runner in the dependency set either, and the two files at the root that look like tests do not run. The first trap below explains why.

## Where things live

| Path | What it holds |
| --- | --- |
| `main.py` | The CLI. `--mode` picks the stage, the flags go to `forecaster.forecast_all`. |
| `config.py` | Paths, dates, the outlet table, the OpenRouter settings, the thresholds. Importing it creates the `data/` directories. |
| `scraper.py` | RSS, the archive pages, the Wayback fallback, the raw CSV. The only module that talks to the news sites. |
| `etl.py` | Cleaning, the opinion filter, the near-duplicate check, the audit files. Every other stage reads what this one writes. |
| `analyzer.py` | Topic clustering, the frequency table, the noise check, the entity heuristics, the style profile. |
| `event_calendar.py` | `events.csv` plus a recurring fallback layer of parliamentary weeks, weekly US labour statistics, weekend matches and the end of the month. |
| `forecaster.py` | The four methods, the topic selection, the outlet priors, the prompt, the export. The largest module. |
| `backtester.py` | The holdout replay, one day at a time. |
| `metrics.py` | The five metrics and the report. |
| `pipeline.ipynb` | 21 cells: the same stages, with the tables and the plots. |
| `data/raw/` | The scraped articles. Written on every scrape, never tracked by git. |
| `data/clean/` | The clean corpus and the two ETL audit files per outlet. |
| `data/events/events.csv` | 25 known events for 2026-04-01 to 2026-04-30. |
| `data/forecasts/` | The forecast runs (JSON, XLSX, shortlists) and the backtest artefacts. |

## Traps

- **`requirements.txt` is behind `pyproject.toml`.** `aiohttp`, `aiofiles`, `nest-asyncio`, `openpyxl` and `ipykernel` are missing from it. `uv pip install -r requirements.txt` gives a tree where `import scraper` fails on `aiohttp` and the forecast stage fails when pandas asks for `openpyxl`. Use `uv sync`.
- **The two test scripts do not import.** `test_scrapers.py` and `test_rss_content.py` import `scrape_rss` and `scrape_outlet` from `scraper`. The scraper has `scrape_rss_async`, `scrape_outlet_async` and `scrape_all` now, so both files raise `ImportError` before a line runs. Neither file asserts anything; they print.
- **A scrape overwrites the raw corpus.** `save_raw` opens `data/raw/{slug}_raw.csv` in mode `w`, so each run replaces the previous file. Nothing is appended.
- **The forecast date is pinned in two places.** `config.TARGET_DATE` is `2026-04-02` and `main.py` defaults `--target` to the same string. `forecaster.combine_forecast` takes its default from `config.TARGET_DATE`, so a call that skips the argument still forecasts April 2026.
- **The history window slides with the clock.** `HISTORY_FROM` is `TODAY - 90 days`, evaluated when `config` is imported. The committed corpus covers 2025-12-30 to 2026-03-29, and a scrape run today produces a different window.
- **A missing key reads as a score of zero.** `metrics.semantic_similarity` returns `0.0` when `OPENROUTER_API_KEY` is missing, and the mean reaches the report as `0.000` rather than `n/a`. A zero there means the metric was not measured.
- **The prompt has to answer in Russian, in the agreed format.** It asks for numbered entries with `ЗАГОЛОВОК` and `ЛИД` lines, and `_parse_llm_response` looks for exactly those two labels. A reply that changes the labels loses the lead, and a reply without them loses the title as well, because the parser falls back to the first line of the block.
- **The OpenRouter call identifies another repository.** `_llm_chat` sends `HTTP-Referer: https://github.com/aazhivotrev/titles_forecating`, the upstream project this one was forked from, and `X-Title: Titles Forecasting Project`. That header is what OpenRouter shows in its dashboard.
- **The Wayback fallback fires under 50 records, and its records carry no title.** The CDX query is capped at 500 rows, the titles come back empty, and the ETL drops those rows at the first step. A run that leans on the fallback ends up with rows that the clean has already thrown away.
- **The Kommersant archive is the slow part.** 14 rubrics across 90 days is about 1,260 separate requests at a concurrency of 15, on top of the daily archive pages of the other two outlets. `REQUEST_DELAY` is a back-off after a throttled or failed request, not a pause between successful ones.
- **Near-duplicates need the same rubric and a 72-hour window.** `DEDUP_REQUIRE_SAME_RUBRIC` is on, so the same headline republished under a second rubric survives the ETL.
- **The topic count adapts instead of failing.** `extract_topics` lowers `n_topics` when the corpus has fewer distinct texts than clusters and falls back to a single cluster named `general`. A small corpus produces one topic quietly.
- **`config.py` creates directories on import.** The `os.makedirs` loop near the top of the module creates `data/raw`, `data/clean`, `data/forecasts`, `data/forecasts/backtests` and `data/events` as a side effect of the import.
- **`.gitignore` lists paths the repository tracks anyway.** `data/raw/`, `data/clean/` and `data/events/events.csv` are in `.gitignore`, yet the clean corpus and the calendar are committed. A new file under those paths is ignored; the tracked ones stay tracked.
- **Two outlets sit in the data but not in the config.** `data/clean/vedomosti_clean.csv` holds 198 rows and `data/clean/rbc_clean.csv` holds a header, and `events.csv` names `rbc` and `vedomosti`, but `config.OUTLETS` has three entries. Nothing in the code reads those two files.

## Style / safety rails

- **The Russian stays Russian.** The prompts, the marker lists, the outlet priors and the generated headlines are the product, and they are Russian. The documents are English; the code and the product are not.
- **The forecast export carries the `llm` method only.** `main.py` passes `llm_only=True` on purpose. The `inertia`, `frequency` and `calendar` baselines stay in the pipeline for the backtest and the comparison.
- **A method that cannot be scored gets `n/a`, never zero.** `inertia` and `frequency` generate no text, so their text-level metrics are absent on purpose.
- **Do not move a threshold without re-running the backtest.** `DEDUP_THRESHOLD`, `FREQ_WINDOW`, `N_TOPICS` and `BACKTEST_DAYS` change the numbers in the report, and the committed backtest artefacts were produced with the values in `config.py`.
- **Keep the ETL audit.** `data/clean/{slug}_etl_audit.json` and `{slug}_etl_daily_counts.csv` are how a change in the corpus becomes visible.
- **Comments explain why, never what.** The comments in this codebase are good on that point, for instance the note in `metrics.py` about fitting the style vectorizer on the real corpus only.
- **The key comes from the environment.** `OPENROUTER_API_KEY` is read from `.env` or the shell, and `.env` is in `.gitignore`. No key belongs in a file under `data/`.

## Where the product truth lives

`PRODUCT.md` holds it. Read it before changing the behaviour of the forecast, and write the change there when the behaviour changes.
