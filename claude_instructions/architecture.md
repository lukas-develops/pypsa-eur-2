# PyPSA-Eur Architecture

## What This Model Is

PyPSA-Eur is an open-source energy system model for Europe. It optimizes investment and operation of electricity, heating, transport, and industry infrastructure to meet energy demand at minimum cost under given constraints (e.g., CO2 limits). Built on top of [PyPSA](https://pypsa.org/) (Python for Power System Analysis) and orchestrated by [Snakemake](https://snakemake.readthedocs.io/).

## Workflow Pipeline

The model runs as a Snakemake DAG (directed acyclic graph). Each step produces files consumed by later steps.

```
1. RETRIEVE         Download raw data (weather, demand, power plants, costs, shapes)
   rules/retrieve.smk   ~50 download rules
        |
2. BUILD SHAPES     Geographic boundaries (countries, NUTS3, offshore EEZ)
   build_shapes.py       -> resources/country_shapes.geojson, etc.
        |
3. BASE NETWORK     Electrical topology from OSM/TYNDP/ENTSO-E GridKit
   base_network.py       -> resources/networks/base.nc
        |
4. SIMPLIFY         Reduce to 380kV equivalent, remove stubs
   simplify_network.py   -> resources/networks/base_s.nc
        |
5. CLUSTER          K-means/HAC clustering to {clusters} nodes
   cluster_network.py    -> resources/networks/base_s_{clusters}.nc
        |
6. ADD ELECTRICITY  Attach generators, storage, loads, renewables
   add_electricity.py    -> resources/networks/base_s_{clusters}_elec_{opts}.nc
        |
7a. PREPARE ELEC    Add CO2 limits, costs, constraints (elec-only path)
    prepare_network.py   -> resources/networks/elec_s_{clusters}_{opts}.nc
        |
7b. BUILD SECTOR    Heat/transport/industry/gas/H2 data (sector-coupled path)
    rules/build_sector.smk  ~40 build rules producing CSVs, GeoJSONs, NetCDFs
        |
    PREPARE SECTOR   Assemble full sector-coupled network
    prepare_sector_network.py -> resources/networks/base_s_{clusters}_{opts}_{sector_opts}_{planning_horizons}.nc
        |
8. SOLVE            Linear optimization (PyPSA + linopy + solver)
   solve_network.py      -> results/{run}/networks/...nc
        |
9. POSTPROCESS      Summaries, plots, maps
   rules/postprocess.smk -> results/{run}/graphs/, maps/, csvs/
```

## Directory Layout

```
Snakefile                   Main orchestrator (includes rules/*.smk)
config/
  config.default.yaml       All defaults (1467 lines, auto-generated from Pydantic)
  config.yaml               User overrides (gitignored, optional)
  plotting.default.yaml     Plotting config (colors, nice_names, projections)
  schema.default.json       JSON Schema (auto-generated)
  scenarios.template.yaml   Example scenario file
rules/
  common.smk                Helper functions: config_provider(), dataset_version(), memory()
  retrieve.smk              Data download rules
  build_electricity.smk     Electricity network building (27 rules)
  build_sector.smk          Sector coupling building (32+ rules)
  solve_electricity.smk     Electricity-only solving
  solve_overnight.smk       Single-year optimization
  solve_myopic.smk          Rolling-horizon multi-year
  solve_perfect.smk         Perfect foresight multi-year
  postprocess.smk           Plotting and summary rules
  collect.smk               Output aggregation checkpoints
  development.smk           Development/comparison utilities
scripts/
  _helpers.py               Shared utilities (mock_snakemake, logging, path helpers)
  _benchmark.py             Memory/performance logging
  build_*.py                Data building scripts (~60 files)
  prepare_*.py              Network preparation scripts
  solve_*.py                Optimization scripts
  plot_*.py                 Visualization scripts (~13 files)
  make_*.py                 Summary/aggregation scripts
  retrieve_*.py             Data download scripts
  add_*.py                  Network component addition scripts
  cluster_*.py              Clustering scripts
  determine_*.py            Availability matrix scripts
  lib/validation/config/    Pydantic config validators (~30 modules)
  definitions/              Enums: heat_sector.py, heat_system.py, heat_system_type.py
data/
  versions.csv              Central registry of all external datasets (name, version, URL)
  custom_powerplants.csv    User-defined power plants
  custom_costs.csv          Cost overrides
  entsoegridkit/            ENTSO-E GridKit network data
  transmission_projects/    TYNDP/NEP/manual grid expansion projects
  existing_infrastructure/  Current infrastructure data
  retro/                    Building retrofitting data
resources/                  Generated intermediate files (gitignored except .gitkeep)
results/                    Final outputs (gitignored except .gitkeep)
test/                       pytest tests
doc/                        Sphinx RST documentation
envs/                       Conda environment definition
```

## Output Directory Structure

```
results/{run_name}/
  networks/          Solved PyPSA network files (.nc)
  csvs/              Summary statistics (costs, capacities, energy, etc.)
  graphs/            Chart PDFs (cost breakdown, capacity, etc.)
  maps/
    static/          PDF/HTML maps (network, H2, gas, balance, heat)
    interactive/     Interactive HTML maps
  graphics/
    balance_timeseries/    Time series balance plots
    heatmap_timeseries/    Heatmap dispatch visualizations
    interactive_bus_balance/  Interactive bus balance plots
  configs/           Solver configuration snapshots
```

## Resource Sharing

Controlled by `run.shared_resources.policy` in config:
- `false` (default): each run gets its own `resources/{run_name}/`
- `true`: all runs share `resources/`
- `"base"`: smart sharing -- shares weather data, shapes, availability matrices; keeps network files run-specific
- Custom string: uses that string as shared folder name

Logic in `scripts/_helpers.py:get_run_path()`. Files NOT shared under "base" policy:
- Files ending in `elec.nc`
- Files starting with `add_electricity`
- Files in `run.shared_resources.exclude` list

## Data Versioning

External datasets are managed through `data/versions.csv`:
- Each row: dataset name, source (primary/archive), version, tags (latest/supported/deprecated), URL
- Config `data.{dataset}.source` and `data.{dataset}.version` select which entry to use
- `dataset_version("name")` in `rules/common.smk` returns the matching row
- Downloaded to: `data/{dataset}/{source}/{version}/`

## PyPSA Network Objects

The core data structure is a `pypsa.Network` stored as NetCDF (.nc). Key components:
- **buses**: nodes (geographic locations with x,y coordinates)
- **generators**: power plants (gas, coal, nuclear, wind, solar, etc.)
- **lines**: AC transmission lines (between buses)
- **links**: DC links and sector coupling converters (electrolysis, heat pumps, etc.)
- **stores**: energy storage without power rating (H2 caverns, CO2 stores)
- **storage_units**: storage with power rating (batteries, pumped hydro)
- **loads**: electricity/heat/transport demand at each bus

Each component has static attributes (capital_cost, efficiency, p_nom) and time-varying attributes (p_max_pu, marginal_cost) stored as DataFrames indexed by snapshots (timestamps).

## Foresight Modes

Selected by `config.foresight`:
- **overnight**: Single investment period. All capacity built at once. Simplest mode.
- **myopic**: Rolling horizon. Solves year-by-year. Previous year's capacity becomes existing (`add_brownfield`). Captures path dependency.
- **perfect**: Multi-period optimization. All years solved simultaneously. Sees future costs. Most computationally expensive.

Each mode includes different solve rules from `rules/solve_*.smk`.
