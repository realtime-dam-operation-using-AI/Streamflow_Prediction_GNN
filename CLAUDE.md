# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Data-preparation code for graph neural network streamflow/discharge prediction and imputation on USGS
gauge stations in Iowa. It contains **no models and no data** — models come from
[tsl (torch-spatiotemporal)](https://github.com/torchspatiotemporal/tsl); the experimental data is
obtained separately (see README.md). Everything here turns raw per-station CSVs into the
`(target, connectivity, mask)` triple that tsl consumes.

## Running things

There is no build, test, or lint setup — these are standalone scripts and exploratory notebooks.

The conda env `hydrotgnn` has the dependencies (tsl 0.9.6, torch, pandas, networkx, pytables,
scikit-learn, matplotlib/seaborn).

All paths in the scripts and notebooks are **relative to the current working directory**, not to the
script location, and the data directories they expect are not in the repo. Run them from the data
root, e.g.:

```bash
cd <data-root>                 # must contain data_time_series/ and catchment_relationship.csv
python .../tsl_customed_data_loader/merge_river_data.py   # -> merged_discharge_data.csv
python .../tsl_customed_data_loader/adj_matrix.py         # -> adj_matrix.csv
python .../tsl_customed_data_loader/tsl_data_loader.py    # builds SpatioTemporalDataset
```

## Data conventions (important, shared by all code here)

- **Per-station series**: `data_time_series/<station_id>_data.csv`, with a `datetime` column, a
  `discharge` column, and a contiguous feature block from `precipitation` through `silty_clay_loam`
  (the notebooks slice it positionally as `X.loc[:, 'precipitation':'silty_clay_loam']`).
- **Topology**: `catchment_relationship.csv` with columns `station_id` (the *downstream* gauge) and
  `upstream_id`. Edges are directed **upstream → downstream**: `adj[upstream, downstream] = 1`.
  Both the script (`adj_matrix.py`) and notebook 3 (`dist.loc[upstream_id, station_id] = 1`) use this
  orientation; keep it when adding code.
- **Station IDs are strings**, used as dict keys and DataFrame column names. Read relationship files
  with `dtype=str`, otherwise the `station_to_index` lookup in `adj_matrix.py` silently drops edges
  (int vs. str keys) and produces an all-zero adjacency matrix.
- **Missing discharge is encoded as `<= 0`**, not NaN. The notebooks scan for it to find timestamps
  with complete data across all stations and write the surviving filenames to `data.txt`.
- **Column order matters**: the adjacency matrix rows/columns are ordered by the column order of
  `merged_discharge_data.csv`. Regenerate both files together.

## Two pipelines

### 1. `tsl_customed_data_loader/` — current, script-based (wide CSV → tsl)

Sequential, each step consuming the previous step's output file:

1. `merge_river_data.py` — pivots all `*_data.csv` into one wide frame (index `datetime`, one column
   per station) holding **discharge only**. Normalizes `/` to `-` in dates, drops duplicate
   timestamps, outer-joins across stations, then fills gaps with `interpolate() → ffill → bfill`.
   Note this imputation is unconditional, so downstream code cannot distinguish observed from filled
   values — if you need a validity mask, capture it before the fill.
2. `adj_matrix.py` — builds the dense directed adjacency from the relationship table, ordered to
   match the merged CSV's columns.
3. `tsl_data_loader.py` — standardizes the values and wraps them in `SpatioTemporalDataset`
   (`window=24`, `horizon=12`). Rough edge: the data is already scaled by hand *and* the scaler is
   passed as `transform=`, which in tsl 0.9 is a per-sample callable, not the scaler hook
   (`scalers={'target': ...}`). Fix this before trusting any denormalized output.

### 2. `codes/` — earlier, notebook-based (per-timestamp CSVs → `water.h5`)

Exploratory, Windows-era notebooks with hardcoded absolute paths, some Chinese comments, and dead
cells (e.g. the `pymysql` block in notebook 1). Read them as a record of how the HDF5 dataset was
produced rather than as runnable code.

- `1readindata.ipynb` — inventory of stations and relationship file.
- `2buildgraph.ipynb` — **transposes the layout**: writes one CSV per timestamp
  (`time_series/<datetime>.csv`) containing every station's row at that instant. This is per-row
  appending over ~60k timestamps and is very slow; prefer the pandas pivot in `merge_river_data.py`.
- `1.5connectedcomponent.ipynb` — builds an **undirected** `networkx` graph of the stations and
  splits it into connected components, i.e. independent river basins, emitted as `set1..setN`
  subdirectories with their own `relationshipset<N>.csv`. As the notebook warns, component ordering
  from `nx.connected_components` differs between runs, so set numbers are not stable — never rely on
  a hardcoded `connectedcom[i]` index across sessions.
- `3datastructure and export h5 water data.ipynb` — the payoff. Its first ~24 cells are a **verbatim
  copy of tsl/GRIN's `AirQuality` dataset class** (with `PandasDataset`, `geographical_distance`,
  `thresholded_gaussian_kernel`) used as a template; their `from ..utils.utils` imports do not
  resolve standalone. The real work is from the "discharge data read in and apply mask" cell on:
  it takes one basin's discharge, slices the last 1800 timesteps, **borrows the AirQuality
  `eval_mask` block** as a synthetic missingness pattern for imputation experiments, builds `dist`
  as the directed adjacency, and writes `water.h5`.

### `water.h5` layout

Three pandas keys, mirroring what tsl's `AirQuality.load_raw` expects:

| key | contents |
| --- | --- |
| `discharge` | wide frame, DatetimeIndex × station columns |
| `eval_mask` | same shape; 1 = value is ground truth held out for imputation |
| `dist` | station × station directed adjacency (1 = upstream→downstream edge), *not* a distance matrix despite the name |

Note the semantic split: in the tsl/AirQuality convention `mask` marks valid data and `eval_mask`
marks held-out ground truth, and `dist` normally holds geographic distances passed through a
Gaussian kernel. Here `dist` is a binary river-network adjacency instead, so
`get_similarity`/`thresholded_gaussian_kernel` from the template must not be applied to it.
