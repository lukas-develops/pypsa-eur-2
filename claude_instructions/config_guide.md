# Configuration Guide

## Config Hierarchy (lowest to highest priority)

1. `config/config.default.yaml` -- all defaults, auto-generated from Pydantic schema. **Do not edit directly.**
2. `config/plotting.default.yaml` -- plotting/visualization defaults. **Do not edit directly.**
3. `config/config.yaml` -- user overrides (gitignored, created by user). **This is what users edit.**
4. Scenario overrides via `config/scenarios.yaml` (when `run.scenarios.enable: true`)
5. Wildcard overrides via `update_config_from_wildcards()` in `_helpers.py` (parsed from `{opts}` and `{sector_opts}`)

The Snakefile loads configs in order (lines 26-32), and later files override earlier ones.

## The Pydantic Validation System

Config is validated on every Snakemake run via `validate_config(config)` (Snakefile line 35).

### Validator Structure

```
scripts/lib/validation/config/
  __init__.py           validate_config() entry point
  _base.py              Base classes
  _schema.py            ConfigSchema combining all section models
  adjustments.py        Adjustments section
  atlite.py             Weather data config
  biomass.py            Biomass config
  clustering.py         Clustering config
  co2_budget.py         CO2 budget config
  conventional.py       Conventional generators
  costs.py              Cost assumptions
  countries.py          Country list
  data.py               Data sources and versions
  electricity.py        Electricity section
  enable.py             Feature toggles
  energy.py             Energy data config
  existing_capacities.py  Existing capacity data
  foresight.py          Foresight mode
  industry.py           Industry sector
  lines.py              Transmission lines
  links.py              HVDC links
  load.py               Load config
  overpass_api.py       OSM API settings
  pypsa_eur.py          Default component carriers
  renewable.py          Renewable tech config
  run.py                Run management
  scenario.py           Scenario wildcards
  sector.py             Sector coupling (largest)
  snapshots.py          Time period
  solar_thermal.py      Solar thermal
  solving.py            Solver settings
  transformers.py       Transformer specs
  transmission_projects.py  Grid expansion
```

### Adding a New Config Option

1. **Find the right Pydantic model** in `scripts/lib/validation/config/{section}.py`
2. **Add the field** with type, default, and description:
   ```python
   my_new_option: float = Field(
       default=0.5,
       description="Description of what this does",
       alias="my-new-option",  # Use alias if YAML key has dashes
   )
   ```
3. **Regenerate defaults**: `pixi run generate-config`
   - This updates `config/config.default.yaml` and `config/schema.default.json`
4. **Verify**: `pytest test/test_config_schema.py`
5. **Use in scripts** via `snakemake.params` or `snakemake.config`
6. **Use in rules** via `config_provider("section", "my_new_option")`

### Pydantic Field Conventions

- Use `alias="kebab-case"` when YAML key uses dashes (Python identifiers can't have dashes)
- Use `Field(default=..., ge=0, le=1)` for bounded values
- Use `Literal["option1", "option2"]` for enums
- Use `dict[str, float]` for year-indexed values like `{2030: 0.5, 2050: 0.0}`
- Nested sections use nested Pydantic models

## config_provider() Function

Defined in `rules/common.smk` (line 72). Returns a partial function for deferred config lookup.

```python
# In a rule:
params:
    co2limit=config_provider("electricity", "co2limit"),
    solver=config_provider("solving", "solver", "name"),
    feature=config_provider("sector", "my_feature", default=False),
```

When Snakemake evaluates the rule, `config_provider()` resolves the value considering:
- Base config
- Scenario overrides (if `run.scenarios.enable`)
- Wildcard-based overrides (from opts/sector_opts strings)

**Always use `config_provider()` for params in rules** instead of direct `config["key"]` access. Direct access doesn't work with scenarios.

## Key Config Sections Reference

### `scenario` -- Defines what to run
```yaml
scenario:
  clusters: [50]           # Network node counts
  opts: [""]               # Electricity options
  sector_opts: [""]        # Sector options
  planning_horizons: [2050] # Years to solve
```
Snakemake expands all combinations of these lists.

### `countries` -- Geographic scope
List of ISO 2-letter codes. Adding/removing countries affects all downstream rules.

### `snapshots` -- Time period
```yaml
snapshots:
  start: "2013-01-01"
  end: "2014-01-01"
  inclusive: left
```

### `electricity` -- Carrier lists and limits
```yaml
electricity:
  extendable_carriers:
    Generator: [solar, onwind, offwind-ac, offwind-dc, OCGT]
    StorageUnit: [battery]
    Store: [H2]
    Link: [H2 pipeline]
  conventional_carriers: [nuclear, oil, OCGT, CCGT, coal, lignite]
  renewable_carriers: [solar, onwind, offwind-ac, offwind-dc]
  co2limit: 7.75e+7       # tCO2/a
  co2limit_enable: false
  gaslimit_enable: false
```

### `renewable` -- Per-technology settings
```yaml
renewable:
  onwind:
    cutout: default
    resource:
      method: wind
      turbine: Vestas_V112_3MW
    capacity_per_sqkm: 3
    corine: [12, 13, 14, ...]  # Allowed land use codes
    natura: true               # Exclude Natura 2000 areas
```

### `sector` -- Sector coupling toggles
```yaml
sector:
  transport: true
  heating: true
  biomass: true
  industry: true
  gas_network: false
  H2_network: false
  co2_spatial: false
```

### `solving` -- Solver and constraints
```yaml
solving:
  solver:
    name: gurobi       # or highs, cplex, copt, xpress, glpk, cbc
    options: gurobi-default
  solver_options:
    gurobi-default:
      threads: 8
      method: 2
      crossover: 0
      BarConvTol: 1.e-5
    highs-default:
      threads: 8
      solver: ipm
      run_crossover: "off"
  options:
    load_shedding:
      enable: false
    rolling_horizon:
      enable: false
    noisy_costs: true
    skip_iterations: true
  constraints:
    CCL: false        # Country capacity limits
    EQ: false         # Equity constraints
    BAU: false        # Business-as-usual
    SAFE: false       # N-1 security
```

### `costs` -- Cost assumptions
```yaml
costs:
  year: 2050
  version: v0.9.3
  social_discountrate: 0.02
  emission_prices:
    enable: false
    co2: 0
    co2_monthly_prices: false
  overwrites: {}     # Override any cost parameter
```

### `clustering` -- Spatial aggregation
```yaml
clustering:
  cluster_network:
    algorithm: kmeans   # or hac
  aggregation_strategies:
    generators: ...
  temporal:
    resolution_elec: false
    resolution_sector: false
```
