# Screenshots

The repository holds no images.

The pipeline has no interface to photograph. Its output is text on standard
output, a JSON file, a workbook and a notebook, and the plots live in the
notebook cells rather than in files.

The workbook is the one artefact that can hold a picture, and it does not:
`pandas.to_excel` writes the cells and nothing else.

## Putting a picture here

This folder is part of the standard, and the images the README shows are
referenced by a relative path, so they have to live somewhere the checker
can resolve. If a plot is worth keeping and the README is going to show it,
export it from the notebook into this folder and point at it from the README
as `docs/screenshots/<name>.png`.
