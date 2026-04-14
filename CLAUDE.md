# PyPSA-Eur -- Claude Code Instructions

PyPSA-Eur is an open-source Snakemake-based European energy system model. It optimizes investment and operation of electricity, heating, transport, and industry infrastructure under constraints (CO2 limits, land use, etc.).

**Docs**: https://pypsa-eur.readthedocs.io/ | **Repo**: https://github.com/pypsa/pypsa-eur

## Quick Start

```bash
pixi run snakemake -call -n                    # Dry run (check DAG)
pixi run snakemake -call                       # Full run (all default targets)
pixi run snakemake -call --until cluster_network  # Run up to specific rule
pixi run generate-config                       # Regenerate config.default.yaml from Pydantic
pixi run pytest                                # Run tests
ruff check . && ruff format .                  # Lint + format
```

Users create `config/config.yaml` (gitignored) to override defaults. Tutorial mode: set `tutorial: true`.

## Repository Map

```
Snakefile                     Main orchestrator. Includes rules/*.smk, selects foresight mode.
config/
  config.default.yaml         All defaults (1467 lines). AUTO-GENERATED from Pydantic -- do not edit.
  config.yaml                 User overrides (gitignored, optional).
  plotting.default.yaml       Plotting config: tech_colors, nice_names, map projection.
  schema.default.json         JSON Schema (auto-generated).
rules/
  common.smk                  config_provider(), dataset_version(), memory(), solver_threads()
  retrieve.smk                ~50 data download rules
  build_electricity.smk       27 rules: shapes, base network, profiles, clustering, add_electricity
  build_sector.smk            32+ rules: heat, transport, industry, gas, H2, biomass, CO2
  solve_electricity.smk       solve_network, solve_operations_network
  solve_overnight.smk         Single-year optimization (if foresight=overnight)
  solve_myopic.smk            Rolling-horizon multi-year (if foresight=myopic)
  solve_perfect.smk           Perfect foresight multi-year (if foresight=perfect)
  postprocess.smk             Plotting, summaries, maps
  collect.smk                 Output aggregation checkpoints
  development.smk             Development/comparison utilities
scripts/
  _helpers.py                 mock_snakemake, configure_logging, set_scenario_config, load_costs, etc.
  _benchmark.py               Memory/performance logging
  lib/validation/config/      Pydantic config validators (~30 modules, one per config section)
  definitions/                Enums: heat_sector.py, heat_system.py, heat_system_type.py
  build_*.py                  Data building scripts (~60)
  prepare_*.py                Network preparation (prepare_sector_network.py is ~3000 lines)
  solve_*.py                  Optimization
  plot_*.py                   Visualization (~13 scripts)
  make_*.py                   Summary/aggregation
  retrieve_*.py               Data download
data/
  versions.csv                Central registry: dataset name -> version -> URL
  custom_powerplants.csv      User-defined power plants
  custom_costs.csv            Cost overrides
test/                         pytest tests
doc/                          Sphinx RST documentation
```

## Workflow Pipeline

```
retrieve (download data)
  -> build_shapes (country/offshore boundaries)
  -> base_network (OSM/TYNDP topology)
  -> simplify_network (reduce to 380kV)
  -> cluster_network ({clusters} nodes via k-means)
  -> add_electricity (generators, storage, loads)
  -> prepare_network (CO2 limits, costs, constraints)
  -> solve_network (linear optimization via PyPSA + linopy + solver)
  -> postprocess (summaries, plots, maps)

For sector coupling: build_sector rules -> prepare_sector_network -> solve_sector_network
```

## Script Pattern (MUST follow for any new script)

```python
# SPDX-FileCopyrightText: Contributors to PyPSA-Eur <https://github.com/pypsa/pypsa-eur>
#
# SPDX-License-Identifier: MIT

import logging
from scripts._helpers import configure_logging, set_scenario_config

logger = logging.getLogger(__name__)

if __name__ == "__main__":
    if "snakemake" not in globals():
        from scripts._helpers import mock_snakemake
        snakemake = mock_snakemake("rule_name")
    configure_logging(snakemake)
    set_scenario_config(snakemake)
    # Use: snakemake.input, snakemake.output, snakemake.config, snakemake.params, snakemake.wildcards
```

## Wildcard System

| Wildcard | Constraint | Examples | Purpose |
|----------|-----------|---------|---------|
| `{clusters}` | `[0-9]+(m\|c)?\|all\|adm` | 50, 100, all | Network node count |
| `{opts}` | `[-+a-zA-Z0-9\.]*` | Co2L0.05, 3h | Electricity options |
| `{sector_opts}` | `[-+a-zA-Z0-9\.\s]*` | T-H-B-I-24h | Sector options |
| `{planning_horizons}` | `[0-9]{4}` | 2030, 2050 | Investment year |
| `{technology}` | - | solar, onwind | Renewable tech |

File naming: `base_s_{clusters}_{opts}_{sector_opts}_{planning_horizons}.nc`

Opts are parsed by `update_config_from_wildcards()` in `_helpers.py`. Key tokens:
- `Co2L{value}` -- CO2 limit, `{N}h` -- temporal resolution, `Ep{value}` -- emission price
- `T`/`H`/`B`/`I` -- sector toggles, `CCL`/`EQ`/`BAU`/`SAFE` -- constraint toggles
- `CF+key1+key2+value` -- arbitrary config override

## Configuration System

**Priority** (lowest to highest): `config.default.yaml` < `config.yaml` < scenario overrides < wildcard overrides

- Validated by Pydantic models in `scripts/lib/validation/config/`
- In rules, use `config_provider("section", "key")` (deferred, scenario-aware)
- In scripts, prefer `snakemake.params.x` over `snakemake.config["x"]`
- After Pydantic changes: `pixi run generate-config` then `pytest test/test_config_schema.py`

Key sections: `scenario`, `countries`, `snapshots`, `electricity`, `renewable`, `sector`, `solving`, `costs`, `clustering`

## Code Style

- **Linter/formatter**: ruff (see `ruff.toml`). Rules: F, E, W, I, D, UP, TID
- **License header**: `SPDX-FileCopyrightText: Contributors to PyPSA-Eur` + `SPDX-License-Identifier: MIT`
- **Docstrings**: NumPy style
- **Pre-commit**: `pre-commit run --all-files`

## Key Conventions

- **Config defaults are auto-generated**: Never edit `config/config.default.yaml` directly. Edit Pydantic models in `scripts/lib/validation/config/`, then `pixi run generate-config`.
- **Carrier naming**: lowercase with spaces ("solid biomass", "urban central heat"), except H2 (uppercase). Must be consistent across config, cost data, scripts, and `plotting.default.yaml`.
- **Path providers**: `resources("file.csv")`, `logs("file.log")`, `RESULTS + "file.csv"` -- handle shared/run-specific paths automatically.
- **Network files**: PyPSA Network objects stored as NetCDF (.nc). Load: `pypsa.Network(path)`. Save: `n.export_to_netcdf(path)`.
- **Year-dependent config**: Use `get(config_value, year)` from `_helpers.py` -- handles both scalar and `{2030: x, 2050: y}` dict formats.

## Deep Dive Guides

For detailed instructions on specific topics, see `claude_instructions/`:

- **[architecture.md](claude_instructions/architecture.md)** -- Full data flow, directory layout, PyPSA network objects, foresight modes
- **[config_guide.md](claude_instructions/config_guide.md)** -- Config hierarchy, Pydantic system, adding config options, key sections reference
- **[rules_and_scripts.md](claude_instructions/rules_and_scripts.md)** -- Rule anatomy, creating new rules/scripts, _helpers.py toolkit
- **[wildcards_and_paths.md](claude_instructions/wildcards_and_paths.md)** -- Wildcard constraints, file naming, path providers, opts/sector_opts parsing
- **[adding_technologies.md](claude_instructions/adding_technologies.md)** -- Adding renewable/conventional/storage/sector technologies
- **[solving_and_optimization.md](claude_instructions/solving_and_optimization.md)** -- Solver config, constraints, foresight modes, troubleshooting
- **[sector_coupling.md](claude_instructions/sector_coupling.md)** -- define_spatial(), heat/transport/industry/gas/H2/CO2 systems
- **[plotting_and_postprocess.md](claude_instructions/plotting_and_postprocess.md)** -- Plot scripts, plotting config, adding new plots
- **[testing.md](claude_instructions/testing.md)** -- pytest, config schema tests, mock_snakemake, CI/CD
- **[common_tasks.md](claude_instructions/common_tasks.md)** -- Cookbook: change countries/solver/resolution, add data, debug failures
