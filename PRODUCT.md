# Product

<!-- impeccable:product-schema 1 -->

## Platform

cli

A Python 3.10 or newer project, installed with `uv` from `uv.lock`. The interface is a command line (`main.py`) and a Jupyter notebook (`pipeline.ipynb`). There is no server, no web page and no database: every stage reads and writes CSV, JSON and XLSX files under `data/`. The only runtime network dependency is OpenRouter, and only for the headline generation and for the embedding metric.

## Users

A team taking part in the HSE Centaur Race forecasting competition for "Human + AI" pairs. The situation: the submission has to name the outlet, the expected headline and the first paragraph for a fixed future date, and has to describe the model and the prompt that produced them. The person is assembling that submission with a day or two left, and needs a forecast they can rerun rather than a set of headlines they wrote by hand.

The second audience is a reader of the repository. It is public, has no topics, no releases and nothing beyond the README, and it was forked from `aazhivotrev-collab/titles_forecating`. The repository description is "Forecasting titles". There is no evidence of anybody else using it.

## Product Purpose

The product turns 90 days of Russian news into a forecast: for one target date and one outlet, the headline and the first paragraph that outlet is likely to publish.

The pipeline is the deliverable as much as the forecast is. The contest asks for the method, the model and the prompt alongside the answer, and the repository's own claim is reproducibility: one command rebuilds the corpus, the backtest and the forecast.

Success for the user is a forecast for 2026-04-02 that names the outlet, the headline and the lead, produced by a command that anyone can rerun. Success for the product is that the report says which method is trustworthy. The current state is that `frequency` is the strongest baseline at topic level, and `llm` writes publishable text but does not guess the editorial agenda well.

## Positioning

The distinguishing part is not the language model. It is the machinery around it: an ensemble that chooses the topics, a retrieval step that grounds each prompt in headlines the same outlet published, an explicit editorial prior per outlet, and a gate that rejects a story that is off-brand for that outlet.

A neighbouring approach would take a topic list and ask a model for headlines. This one takes a topic list with a score, eight retrieved headlines, four style examples from the same outlet, the calendar, and the people and organisations in the story, and it refuses the output when it drifts from the topic or from the outlet.

The second difference is the evaluation. Topic-level and text-level metrics are separate, a method that generates no text is not scored, and the baseline that wins at topic level stays in the report next to the model instead of being dropped.

## Operating Context

The run happens on a working day, with the target date a day or two ahead. The scrape needs the network: three news sites, their archive pages, and the Wayback Machine. The forecast needs an OpenRouter key and a model that is available on that day.

The review happens in the notebook, cell by cell, where the topics, the entities and the per-day publication volume are looked at as tables and plots. The submission is assembled from the JSON or the workbook in `data/forecasts/`, and the quality claim comes from the backtest artefacts beside them.

Two Russian artefacts matter to the reader: the prompts, the marker lists and the generated headlines are Russian text, and the workbook sheet names are Russian ("Все прогнозы", "LLM заголовки").

The reproducibility limits are known. The archive pages change shape, the Wayback fallback is best effort, the first run downloads the NLTK Russian stopwords, and the OpenRouter catalogue moves.

## Capabilities and Constraints

Confirmed by the code:

- Three outlets in `config.py`: Kommersant, Lenta.ru and Interfax, each with RSS feeds and an archive path.
- A 90-day history window, computed at import time from the current date.
- Three collection sources: RSS, archive pages, and the Wayback CDX API as a fallback under 50 records for an outlet.
- A nine-column schema for every article: id, outlet, url, published_at, title, lead, rubric, language, country.
- An ETL that strips markup, cleans outlet-specific title artefacts, drops empty titles and unparseable dates, removes exact URL duplicates, filters opinion and blog pages, and removes near-duplicates above a 0.85 cosine similarity within 72 hours under the same rubric.
- An ETL audit per outlet: the row count and the date coverage after each step, plus the per-day counts.
- Topic modelling with TF-IDF and KMeans, 20 clusters per outlet, named by their strongest terms, with a 14-day frequency table.
- Entity extraction by capitalisation heuristics, into persons, organisations and locations.
- Four forecast methods: inertia, frequency, calendar and llm.
- Two topic selection strategies (llm, hybrid), two prompt profiles (default, forward_look), an editorial prior per outlet and a story gate per outlet.
- A holdout backtest over 7 days, training only on the data before each day, with the LLM limited to two topics per day to save tokens.
- Five metrics: topic hit rate, entity F1, semantic similarity, style match and diversity.
- A timestamped JSON and XLSX export per run, with one sheet per outlet, and a hand-written shortlist for each of the two runs kept in the repository.
- A notebook of 21 cells that runs the same stages and draws the plots.

Deliberately not true, and worth knowing before a change:

- The forecast export contains the `llm` predictions only. The baselines are not part of the deliverable.
- `llm` is the only method that produces a headline and a lead. `inertia` and `frequency` produce topics, and `calendar` produces template titles built from the events.
- Nothing runs on a schedule. The artefacts in the repository were produced by hand.
- The entity extraction is a regex on capitalisation. `pymystem3` is a declared dependency and is never imported, because the comment in `analyzer.py` says it hangs on macOS ARM.
- Seven declared dependencies are never imported by any module or cell: `pymystem3`, `razdel`, `sentence-transformers`, `requests`, `aiofiles`, `nest-asyncio` and `ipykernel`.
- The repository tracks `data/clean/` and `data/events/events.csv` even though `.gitignore` lists both.
- `data/clean/rbc_clean.csv` holds only a header and `data/clean/vedomosti_clean.csv` holds 198 rows, and neither `rbc` nor `vedomosti` is in the outlet table, so nothing reads them.

Open questions, for the owner to answer rather than for a later session to guess:

- The licence. The repository carried no `LICENSE` file; the MIT licence of the house standard was added along with these documents.
- Whether the two kept runs are the two to submit, and whether `forward_look` should become the default profile.
- Whether the 90-day window should grow, and whether the target date stays at 2026-04-02 or moves with the run.
- Whether `rbc` and `vedomosti` come back as outlets. The calendar and the clean corpus still mention them.

## Brand Commitments

**Name:** the repository is `titles-forecating`, and the project calls itself the News Forecast Pipeline in the README and in the module docstrings. **Tone:** plain and technical, in Russian inside the code and in English in the documents.

The editorial voice of the forecast follows the outlet, not the project. Kommersant is given the business angle, Lenta.ru the conflict angle and Interfax the agency and diplomatic angle. Those are term lists in `forecaster.py`, and they are why two forecasts for the same day can read differently.

The product states its limits. The README names the methods and their weaknesses, and the metric layer refuses to score a method it cannot score.

## Evidence on Hand

- The repository's own README, 218 lines in Russian before this rewrite, describes the contest task, the pipeline, the CLI flags, the four methods, the metrics and the limitations.
- `data/clean/`: 11,391 Kommersant rows, 5,311 Interfax rows and 2,866 Lenta.ru rows covering 2025-12-30 to 2026-03-29, a 198-row `vedomosti` file and an empty `rbc` file.
- `data/events/events.csv`: 25 events from 2026-04-01 to 2026-04-30, each with the outlets it concerns.
- `data/forecasts/backtests/`: three files, one per outlet, each holding the 7 holdout days from 2026-03-23 to 2026-03-29 with the predictions of all four methods against the titles that were published.
- `data/forecasts/`: two runs from 2026-03-30, `llm + default` at 15:36:40 and `hybrid + forward_look` at 15:38:25, each with a JSON file, a workbook and a shortlist of five headlines written by hand.
- `pipeline.ipynb`: 21 cells, the pipeline with the plots.

Absent, and not to be invented: no download counts, no usage numbers, no testimonials, no comparison with other forecasting systems, and no score from the competition. The repository has no stars, no releases and no topics.

## Product Principles

1. **The pipeline is the submission.** The forecast is worth as much as the command that reproduces it, so the corpus, the backtest and the forecast are one artefact rather than three.
2. **A method that cannot be scored is not scored.** `inertia` and `frequency` generate no text, so their text-level numbers are `n/a`. A zero would be a lie.
3. **The text is the model's, the topic is the algorithm's.** The ensemble picks what the outlet is likely to write about, and the model writes it.
4. **The outlet's own history grounds the prompt.** Every prompt carries headlines and style examples that outlet published, and the selection is penalised when the story drifts from the outlet's editorial angle.
5. **The baseline stays in the box.** `frequency` remains the strongest topic-level baseline and is reported as such, next to the LLM.

## Accessibility & Inclusion

- The product has no interface of its own: the outputs are CSV, JSON and a workbook, and the notebook prints tables and plots. The constraint that matters is the language, not the modality. The forecast is Russian text for Russian readers.
- The workbook sheet names are Russian, and so are the prompts. The documentation is English. A reader who cannot read Russian gets the method and the numbers, not the forecast.
- Not decided: no accessibility standard has been set for the artefacts, and no English export of a forecast exists.
