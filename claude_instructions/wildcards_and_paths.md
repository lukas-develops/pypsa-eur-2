# Wildcards and Paths

## Wildcard Definitions

Defined in `Snakefile` lines 56-60:

| Wildcard | Constraint | Example Values | Purpose |
|----------|-----------|----------------|---------|
| `{clusters}` | `[0-9]+(m|c)?\|all\|adm` | `50`, `100`, `181`, `all`, `adm` | Number of network nodes |
| `{opts}` | `[-+a-zA-Z0-9\.]*` | `""`, `Co2L0.05`, `Co2L0.05-3h` | Electricity-only options |
| `{sector_opts}` | `[-+a-zA-Z0-9\.\s]*` | `""`, `Co2L0-24h`, `T-H-B-I` | Sector coupling options |
| `{planning_horizons}` | `[0-9]{4}` | `2030`, `2040`, `2050` | Investment year |
| `{technology}` | (no constraint) | `solar`, `onwind`, `offwind-ac` | Renewable technology type |

## File Naming Convention

File paths encode the full parameterization:

```
resources/networks/base.nc                                          # Raw base network
resources/networks/base_s.nc                                        # Simplified (s = simplified)
resources/networks/base_s_{clusters}.nc                             # Clustered
resources/networks/base_s_{clusters}_elec_{opts}.nc                 # With electricity components
resources/networks/elec_s_{clusters}_{opts}.nc                      # Prepared for elec-only solving
resources/networks/base_s_{clusters}_{opts}_{sector_opts}_{planning_horizons}.nc  # Sector-coupled

results/{run}/networks/elec_s_{clusters}_{opts}.nc                  # Solved elec-only
results/{run}/networks/base_s_{clusters}_{opts}_{sector_opts}_{planning_horizons}.nc  # Solved sector
```

Other resource file patterns:
```
resources/profile_{clusters}_{technology}.nc          # Renewable capacity factor profiles
resources/availability_matrix_{clusters}_{technology}.nc  # Land availability
resources/regions_onshore_base_s_{clusters}.geojson   # Onshore region shapes
resources/regions_offshore_base_s_{clusters}.geojson   # Offshore region shapes
resources/costs_{planning_horizons}_processed.csv     # Processed cost data
resources/energy_totals_s_{clusters}_{planning_horizons}.csv
resources/heat_totals_s_{clusters}_{planning_horizons}.csv
```

## Path Providers

Defined in `Snakefile` lines 44-49:

```python
logs = path_provider("logs/", RDIR, shared_resources, exclude_from_shared)
benchmarks = path_provider("benchmarks/", RDIR, shared_resources, exclude_from_shared)
resources = path_provider("resources/", RDIR, shared_resources, exclude_from_shared)
scripts = script_path_provider(Path(workflow.snakefile).parent)
RESULTS = "results/" + RDIR
```

- `resources("filename.csv")` -> `resources/{RDIR}filename.csv` (or shared path)
- `logs("filename.log")` -> `logs/{RDIR}filename.log`
- `RESULTS + "file.csv"` -> `results/{RDIR}file.csv`
- `scripts("script.py")` -> `file:///path/to/scripts/script.py`

`RDIR` is computed from `run.name` and `run.prefix`:
- If `run.name: "myrun"` -> RDIR = `myrun/`
- If `run.prefix: "project"` and `run.name: "myrun"` -> RDIR = `project/myrun/`
- If `run.scenarios.enable: true` -> RDIR = `{run}/` (Snakemake wildcard)

## Opts String Parsing

Parsed by `update_config_from_wildcards()` in `scripts/_helpers.py`. The opts string is split by `-`, and each token is matched:

| Token Pattern | Effect | Example |
|--------------|--------|---------|
| `Co2L{value}` | Sets `electricity.co2limit` to `value * co2base` | `Co2L0.05` = 95% reduction |
| `Co2L` (no value) | Enables CO2 limit from config | `Co2L` |
| `CH4L{value}` | Sets gas limit in MtCH4/a | `CH4L50` |
| `{N}h` | Sets temporal resolution to N hours | `3h` = 3-hourly |
| `{N}seg` | Sets temporal resolution to N segments (tsam) | `50seg` |
| `Ep{value}` | Sets CO2 emission price in EUR/tCO2 | `Ep50` |
| `Ept` | Enables time-varying emission prices | `Ept` |
| `ATK` | Autarky: no cross-border transmission | `ATK` |
| `ATKc` | Autarky within each country | `ATKc` |
| `lv{value}` | Line volume limit (expansion factor) | `lv1.5` |
| `lc{value}` | Line cost factor | `lc1.0` |

## Sector Opts String Parsing

Also parsed by `update_config_from_wildcards()`. Split by `-`:

| Token | Effect |
|-------|--------|
| `T` | Enable transport sector |
| `H` | Enable heating sector |
| `B` | Enable biomass |
| `I` | Enable industry |
| `A` | Enable agriculture |
| `CCL` | Enable country capacity limits constraint |
| `EQ{value}` | Enable equity constraint with value |
| `BAU` | Enable business-as-usual constraint |
| `SAFE` | Enable N-1 security constraint |
| `decentral` | Disable gas network |
| `noH2network` | Disable H2 pipeline network |
| `nowasteheat` | Disable waste heat |
| `nodistrict` | Disable district heating |
| `dist{value}` | Distribution grid cost factor |
| `Co2L{value}` | CO2 limit (same as opts) |
| `cb{value}` | CO2 budget |
| `sdr{value}` | Social discount rate |
| `seq{value}` | CO2 sequestration limit MtCO2/a |
| `{N}h` | Temporal resolution (same as opts) |
| `CF+k1+k2+val` | Arbitrary config override: sets `config[k1][k2] = val` |

### CF+ Arbitrary Config Override

The `CF+` prefix allows setting any config value from the wildcard string:
```
CF+sector+transport+true    -> config["sector"]["transport"] = True
CF+costs+year+2030          -> config["costs"]["year"] = 2030
```
Values are auto-cast: "true"/"false" -> bool, numeric strings -> int/float.

## Scenario Expansion

When `scenario` config has multiple values, Snakemake expands all combinations:

```yaml
scenario:
  clusters: [50, 100]
  opts: ["", "Co2L0.05"]
  sector_opts: [""]
  planning_horizons: [2030, 2050]
```

This produces 2 * 2 * 1 * 2 = 8 networks to solve.

The `rule all` target uses `expand()` with `**config["scenario"]` to generate all output paths.
