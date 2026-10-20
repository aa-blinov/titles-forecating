# Documentation

The documents of the news forecast pipeline.

| Document | What it covers |
| --- | --- |
| [architecture.md](architecture.md) | The stages, the modules, the data flow and the boundaries between them. |
| [screenshots/README.md](screenshots/README.md) | The images of the pipeline. The repository holds none. |
| [../README.md](../README.md) | The front door: what the pipeline does and how to run it. |
| [../AGENTS.md](../AGENTS.md) | The commands, the traps and the rules of the project. |
| [../PRODUCT.md](../PRODUCT.md) | The product truth: the competition, the audience, the constraints. |
| [../DESIGN.md](../DESIGN.md) | The output surfaces: the terminal report, the JSON, the workbook and the notebook. |

## Reading order

New here: [../README.md](../README.md), then
[architecture.md](architecture.md) for the code and the shape of the stages.

Working on the forecast: [architecture.md](architecture.md) for the stages
and the modules, then [../AGENTS.md](../AGENTS.md) for the traps. The
thresholds in `config.py` change the numbers in the report, and the
committed backtest artefacts were produced with the values that are in
`config.py` now.

## The notebook

`pipeline.ipynb` at the root of the repository runs the same stages cell by
cell, with the tables and the plots. It is not a document in this folder,
because it is executable rather than written.
