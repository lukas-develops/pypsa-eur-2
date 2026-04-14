# Plotting and Postprocessing

## Plotting Configuration

Defined in `config/plotting.default.yaml`. Key sections:

### tech_colors
Maps carrier names to hex colors. **Must match carrier names exactly.**
```yaml
plotting:
  tech_colors:
    solar: "#f9d002"
    onwind: "#235ebc"
    offwind-ac: "#6895dd"
    H2: "#bf13a0"
    battery: "#b8ea04"
    # ... etc.
```

### nice_names
Display names for carriers:
```yaml
plotting:
  nice_names:
    onwind: "Onshore Wind"
    offwind-ac: "Offshore Wind (AC)"
    OCGT: "Open-Cycle Gas"
    # ... etc.
```

### Map settings
```yaml
plotting:
  map:
    boundaries: [-11, 30, 34, 70]  # [lon_min, lon_max, lat_min, lat_max]
    color_geomap:
      ocean: white
      land: whitesmoke
  projection:
    name: "EqualEarth"
```

### When adding a new technology
Always add entries to both `tech_colors` and `nice_names` in `config/plotting.default.yaml`, otherwise plots will fail or show "unknown" labels.

## Plot Scripts

| Script | Rule | Output | Description |
|--------|------|--------|-------------|
| `plot_base_network.py` | `plot_base_network` | `resources/maps/power-network.pdf` | Base topology before clustering |
| `plot_power_network_clustered.py` | `plot_power_network_clustered` | `resources/maps/power-network-s-{clusters}.pdf` | Clustered network topology |
| `plot_power_network.py` | `plot_power_network` | `results/.../maps/static/...-costs-all_{ph}.pdf` | Solved network with generation/storage capacities on map |
| `plot_hydrogen_network.py` | `plot_hydrogen_network` | `results/.../maps/static/...-h2_network_{ph}.pdf` | H2 infrastructure map (if H2_network) |
| `plot_gas_network.py` | `plot_gas_network` | `results/.../maps/static/...-ch4_network_{ph}.pdf` | Gas network map (if gas_network) |
| `plot_heat_source_map.py` | `plot_heat_source_*_map_*` | `results/.../maps/static/...-heat_source_*.html` | Heat source temperature/energy maps |
| `plot_balance_map.py` | `plot_balance_map_*` | `results/.../maps/static/...-{carrier}_{ph}.pdf` | Per-carrier energy balance on map |
| `plot_balance_map_interactive.py` | `plot_balance_map_interactive_*` | `results/.../maps/interactive/...` | Interactive Folium balance maps |
| `plot_summary.py` | `plot_summary` | `results/.../graphs/costs.pdf`, etc. | Bar charts: costs, capacities, energy |
| `plot_balance_timeseries.py` | `plot_balance_timeseries` | `results/.../graphics/balance_timeseries/` | Stacked area time series per carrier |
| `plot_heatmap_timeseries.py` | `plot_heatmap_timeseries` | `results/.../graphics/heatmap_timeseries/` | Heatmap view of dispatch |
| `plot_statistics.py` | `plot_base_statistics` | `results/.../graphs/statistics/` | Statistical analysis plots |
| `plot_interactive_bus_balance.py` | `plot_interactive_bus_balance` | `results/.../graphics/interactive_bus_balance/` | Interactive per-bus balance |
| `plot_cop_profiles/` | `plot_cop_profiles` | `results/.../graphs/cop_profiles.html` | Heat pump COP visualization |

## Postprocessing Scripts

### make_summary.py
Produces CSV summaries from solved networks:
- `costs.csv` -- total costs by carrier
- `capacities.csv` -- installed capacity by carrier
- `energy.csv` -- energy generation/consumption by carrier
- `energy_balance.csv` -- per-carrier energy balance
- `capacity_factors.csv` -- utilization rates
- `prices.csv` -- marginal prices at buses
- `curtailment.csv` -- curtailed renewable energy
- `metrics.csv` -- system-wide metrics

### make_global_summary.py
Aggregates summaries across all scenarios into multi-indexed DataFrames.

### make_cumulative_costs.py
For myopic foresight: tracks cumulative costs across planning horizons.

## rename_techs() Function

In `scripts/_helpers.py`. Standardizes technology labels for display:
```python
def rename_techs(label):
    # Maps internal names to display names
    # e.g., "urban central solid biomass CHP" -> "solid biomass CHP"
    # Strips location prefixes and standardizes naming
```

Update this function when adding technologies with non-standard naming.

## Adding a New Plot

### Step 1: Create the script
```python
# scripts/plot_my_visualization.py
import matplotlib.pyplot as plt
from scripts._helpers import configure_logging, set_scenario_config

if __name__ == "__main__":
    if "snakemake" not in globals():
        from scripts._helpers import mock_snakemake
        snakemake = mock_snakemake("plot_my_visualization", clusters="50", ...)

    configure_logging(snakemake)
    set_scenario_config(snakemake)

    n = pypsa.Network(snakemake.input.network)
    plotting = snakemake.params.plotting

    fig, ax = plt.subplots()
    # ... create visualization using plotting["tech_colors"], plotting["nice_names"]
    fig.savefig(snakemake.output[0], bbox_inches="tight")
```

### Step 2: Add the rule
In `rules/postprocess.smk`:
```python
rule plot_my_visualization:
    params:
        plotting=config_provider("plotting"),
    input:
        network=RESULTS + "networks/base_s_{clusters}_{opts}_{sector_opts}_{planning_horizons}.nc",
    output:
        RESULTS + "graphs/my_visualization_s_{clusters}_{opts}_{sector_opts}_{planning_horizons}.pdf",
    log:
        logs("plot_my_visualization_s_{clusters}_{opts}_{sector_opts}_{planning_horizons}.log"),
    script:
        scripts("plot_my_visualization.py")
```

### Step 3: Add to rule all
In `Snakefile`, add the output path to `rule all` inputs:
```python
expand(
    RESULTS + "graphs/my_visualization_s_{clusters}_{opts}_{sector_opts}_{planning_horizons}.pdf",
    run=config["run"]["name"],
    **config["scenario"],
),
```

## Map Projection Patterns

Most map plots use Cartopy:
```python
import cartopy.crs as ccrs

projection = ccrs.EqualEarth()
fig, ax = plt.subplots(subplot_kw={"projection": projection})
n.plot(ax=ax, bus_sizes=..., line_widths=..., ...)
ax.set_extent(snakemake.params.plotting["map"]["boundaries"])
```

## Balance Map System

The balance map system (`plot_balance_map.py`) creates one map per carrier showing:
- Bus-level generation/consumption as pie charts
- Transmission flows as line widths
- Color-coded by technology type

Carriers for balance maps are determined from the solved network's carrier list. The `rule all` uses `balance_map_paths()` function (defined in `rules/postprocess.smk`) to generate all needed output paths.
