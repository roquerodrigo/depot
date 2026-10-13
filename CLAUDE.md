# CLAUDE.md

`depot` is a Python library (`src/depot/`) for building Subway Builder custom maps
(`depot.maps.MapGen`) and editing demand files (`depot.demand.DemandData`). This repo is a
fork of `Subway-Builder-Modded/depot`.

## Environment gotchas

- `pyproject.toml` declares only `requests`. The real runtime dependencies live in
  `environment.yml` (conda-forge), so `pip install .` alone does not give a working install.
  When adding an import, add the package to `environment.yml` and to the README version table.
- `demand.py` imports `sklearn` (scikit-learn), which is not listed in `environment.yml`;
  install it explicitly if `import depot.demand` fails with a missing `sklearn`.
- `MapGen.__init__` calls `_validate_env()`, which raises `RuntimeError` unless every CLI tool
  (`node`, `mapshaper`, `osmium`, `java`, `tile-join`, `tippecanoe`, `sqlite3`, `jq`, `pmtiles`)
  is on `PATH` and executable, plus `planetiler.jar` (or `planetiler`, existence only). Merely
  constructing a `MapGen`, including in tests, needs these installed.
- The `RAM` argument only feeds mapshaper; planetiler is always launched with `-Xmx16g`
  (`generate_pmtiles`).
- `DemandData` routing with `routing_method="osrm"` expects a local OSRM server on
  `localhost:<osrm_port>`; `prepare_osrm` creates it with Docker containers named after the
  map code. `osmnx` routing needs no Docker.
- Network access is required at runtime: Overture buildings (STAC catalog + DuckDB), GEBCO
  bathymetry over OPeNDAP (`load_bathymetry_data`), and planetiler `--download`.

## Behaviour worth knowing

- `DemandData(...)` always creates `<outputdir>/.railyard_map/` (default `outputdir` is the
  map code, relative to the CWD). If the loaded file already contains special-demand points,
  the `special_demand_points.json` / `special_demand_types.json` schema files must already
  exist there unless `IGNORE_SCHEMA=True`.
- `src/depot/roads_config.json` and `src/depot/special_demand_types.json` are package data,
  located at runtime relative to the module file; `pyproject.toml` ships every `*.json` under
  `src/`.
- `MapGen` step order matters: `extract_base_data` first, `process_buildings` before
  `generate_pmtiles`, and `generate_pmtiles` before `add_labels`.
- The README parameter table can lag the code (e.g. defaults for `reprocess_bathymetry_data`
  and `max_building_tile_size`). The `__init__` signatures and docstrings in `maps.py` /
  `demand.py` are authoritative; update the README when changing a parameter.

## Tests

Plain `unittest`, no CI. The files are `tests/demanddata.py` and `tests/mapgen.py` (not
`test_*.py`, so default discovery finds nothing), and `demanddata.py` opens
`test_demand_data.json` by relative path. Run them from inside `tests/` against an installed
package:

    cd tests && python -m unittest demanddata mapgen

## Versioning and releases

- The version lives only in `pyproject.toml`. `depot.__version__` reads it from
  `pyproject.toml` when running from a source checkout, falling back to installed metadata.
- Work lands on `dev`; releases are PRs from `dev` into `main` titled `vX.Y.Z`.
