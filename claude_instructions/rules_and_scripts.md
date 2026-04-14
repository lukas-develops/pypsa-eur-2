# Rules and Scripts Guide

## Rule Anatomy

A typical Snakemake rule in PyPSA-Eur:

```python
rule build_shapes:
    params:
        countries=config_provider("countries"),
    input:
        # Named inputs from other rules or data files
        natura=ancient(resources("natura.tiff")),
        country_shapes=resources("country_shapes.geojson"),
    output:
        # Named outputs consumed by downstream rules
        country_shapes=resources("country_shapes.geojson"),
        offshore_shapes=resources("offshore_shapes.geojson"),
    log:
        logs("build_shapes.log"),
    benchmark:
        benchmarks("build_shapes")
    threads: 1
    resources:
        mem_mb=2000,
    shadow:
        shadow_config
    script:
        scripts("build_shapes.py")
```

Key elements:
- **`resources()`**: path provider function -- handles shared vs run-specific paths
- **`logs()`**, **`benchmarks()`**: similar path providers for logs/benchmarks
- **`scripts()`**: returns `file://` path to script (supports PyPSA-Eur as submodule)
- **`config_provider()`**: deferred config access (supports scenarios + wildcard overrides)
- **`ancient()`**: marks input as "don't re-run if this changed"
- **`shadow: shadow_config`**: runs in shadow directory (avoids file conflicts)

## Script Pattern

Every script follows this exact boilerplate:

```python
# SPDX-FileCopyrightText: Contributors to PyPSA-Eur <https://github.com/pypsa/pypsa-eur>
#
# SPDX-License-Identifier: MIT

import logging
import pandas as pd
# ... other imports

from scripts._helpers import configure_logging, set_scenario_config

logger = logging.getLogger(__name__)


# ... function definitions ...


if __name__ == "__main__":
    if "snakemake" not in globals():
        from scripts._helpers import mock_snakemake
        snakemake = mock_snakemake("rule_name")

    configure_logging(snakemake)
    set_scenario_config(snakemake)

    # Script logic using:
    # snakemake.input["name"] or snakemake.input[0]
    # snakemake.output["name"] or snakemake.output[0]
    # snakemake.config["section"]["key"]
    # snakemake.params.param_name
    # snakemake.wildcards.clusters, .opts, etc.
```

### What the boilerplate does:
- `mock_snakemake("rule_name")`: For standalone testing. Simulates Snakemake context by reading the Snakefile and filling in inputs/outputs/config for the named rule. Pass wildcard values as kwargs: `mock_snakemake("rule", clusters="50")`.
- `configure_logging(snakemake)`: Sets up logging to both file (`snakemake.log[0]`) and stderr.
- `set_scenario_config(snakemake)`: Applies scenario overrides and wildcard-based config changes.

### Accessing data in scripts:
```python
# Inputs/outputs (named or positional)
n = pypsa.Network(snakemake.input.network)
df = pd.read_csv(snakemake.input["costs"])
n.export_to_netcdf(snakemake.output[0])

# Config values
countries = snakemake.params.countries  # Preferred (works with scenarios)
solver = snakemake.config["solving"]["solver"]["name"]  # Direct access (no scenario support)

# Wildcards
clusters = snakemake.wildcards.clusters
planning_horizon = snakemake.wildcards.planning_horizons
```

## Rule File Organization

| File | Stage | # Rules | Key Rules |
|------|-------|---------|-----------|
| `rules/retrieve.smk` | Data download | ~50 | retrieve_cutout, retrieve_electricity_demand_*, retrieve_cost_data |
| `rules/build_electricity.smk` | Network building | 27 | base_network, build_shapes, build_renewable_profiles, simplify_network, cluster_network, add_electricity, prepare_network |
| `rules/build_sector.smk` | Sector data | 32+ | build_population_layouts, build_heat_totals, build_energy_totals, build_gas_network, build_cop_profiles, prepare_sector_network |
| `rules/solve_electricity.smk` | Elec solving | 2 | solve_network, solve_operations_network |
| `rules/solve_overnight.smk` | Overnight mode | 1 | solve_sector_network |
| `rules/solve_myopic.smk` | Myopic mode | 3 | add_existing_baseyear, add_brownfield, solve_sector_network_myopic |
| `rules/solve_perfect.smk` | Perfect foresight | 4 | add_existing_baseyear, prepare_perfect_foresight, solve_sector_network_perfect, make_summary_perfect |
| `rules/postprocess.smk` | Visualization | 13+ | plot_power_network, plot_hydrogen_network, make_summary, plot_summary, plot_balance_timeseries |
| `rules/collect.smk` | Aggregation | 9 | collect_networks, collect_summaries |
| `rules/development.smk` | Dev utilities | 5 | base_network_incumbent, make_network_comparison |

## Creating a New Rule

### Step 1: Create the script

```python
# scripts/build_my_feature.py
# SPDX-FileCopyrightText: Contributors to PyPSA-Eur
# SPDX-License-Identifier: MIT

import logging
import pandas as pd
from scripts._helpers import configure_logging, set_scenario_config

logger = logging.getLogger(__name__)

def build_my_feature(input_data, param_value):
    """Main logic here."""
    # ...
    return result

if __name__ == "__main__":
    if "snakemake" not in globals():
        from scripts._helpers import mock_snakemake
        snakemake = mock_snakemake("build_my_feature", clusters="50")

    configure_logging(snakemake)
    set_scenario_config(snakemake)

    result = build_my_feature(
        pd.read_csv(snakemake.input.some_data),
        snakemake.params.my_param,
    )
    result.to_csv(snakemake.output[0])
```

### Step 2: Add the rule

Add to the appropriate `rules/*.smk` file:

```python
rule build_my_feature:
    params:
        my_param=config_provider("section", "my_param"),
    input:
        some_data=resources("some_input.csv"),
    output:
        resources("my_feature_s_{clusters}.csv"),
    log:
        logs("build_my_feature_s_{clusters}.log"),
    benchmark:
        benchmarks("build_my_feature_s_{clusters}")
    threads: 1
    resources:
        mem_mb=2000,
    shadow:
        shadow_config
    script:
        scripts("build_my_feature.py")
```

### Step 3: Wire into the DAG

Reference the output as input to a downstream rule. For example, in `prepare_sector_network`:

```python
rule prepare_sector_network:
    input:
        my_feature=resources("my_feature_s_{clusters}.csv"),
        # ... other inputs
```

### Step 4: Add to rule all (if producing final output)

If the output is a final result (plot, summary), add to `rule all` in Snakefile.

## _helpers.py Key Functions

| Function | Purpose | Usage |
|----------|---------|-------|
| `mock_snakemake(rule, **wildcards)` | Simulate Snakemake for standalone testing | Script `if __name__` block |
| `configure_logging(snakemake)` | Set up file + stderr logging | Every script |
| `set_scenario_config(snakemake)` | Apply scenario/wildcard config | Every script |
| `update_config_from_wildcards(config, w)` | Parse opts/sector_opts into config | Called by set_scenario_config |
| `get_snapshots(config)` | Create DatetimeIndex | Time series scripts |
| `load_costs(fn, config)` | Load processed cost CSV | Scripts needing costs |
| `progress_retrieve(url, fn)` | Download with progress bar | Retrieve scripts |
| `get(config_value, year)` | Year-aware config lookup | Handles `{2030: x, 2050: y}` or scalar |
| `generate_periodic_profiles(dt_index, ...)` | Weekly profiles with TZ | Demand profile scripts |
| `rename_techs(label)` | Standardize tech names for display | Plotting scripts |
| `aggregate_costs(n, ...)` | Sum costs by carrier | Summary scripts |
| `setup_dask()` | Configure Dask cluster | Heavy computation scripts |

## Common Script Patterns

### Loading a network
```python
n = pypsa.Network(snakemake.input.network)
```

### Saving a network with metadata
```python
n.meta = dict(snakemake.config, **dict(wildcards=dict(snakemake.wildcards)))
n.export_to_netcdf(snakemake.output.network)
```

### Year-dependent config values
```python
from scripts._helpers import get
planning_horizons = int(snakemake.wildcards.planning_horizons)
value = get(snakemake.config["sector"]["some_param"], planning_horizons)
# If some_param is {2030: 0.5, 2050: 1.0}, returns interpolated value
# If some_param is a scalar, returns that scalar
```

### Conditional rule inputs
```python
rule my_rule:
    input:
        optional=lambda w: resources("optional_file.csv") if config_provider("sector", "my_feature")(w) else [],
```
