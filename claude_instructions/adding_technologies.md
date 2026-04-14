# Adding Technologies

## Overview

In PyPSA, a technology is defined by:
- A **carrier** name (e.g., "solar", "onwind", "H2")
- A **component type** (Generator, Link, Store, StorageUnit)
- **Cost data** from the technology-data CSV files
- **Resource profiles** (for variable renewables) from atlite
- **Configuration** entries in `config.default.yaml`

## Adding a New Renewable Generator

Example: adding a new wind turbine type or solar-hsat.

### Step 1: Config entry in `renewable` section

Add to `scripts/lib/validation/config/renewable.py`:
```python
class MyTechConfig(BaseModel):
    cutout: str = Field(default="default")
    resource: dict = Field(default={"method": "wind", "turbine": "MyTurbine"})
    capacity_per_sqkm: float = Field(default=3.0)
    corine: list[int] = Field(default=[12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27, 28, 29, 31, 32])
    natura: bool = Field(default=True)
    potential: str = Field(default="conservative")
```

### Step 2: Add to carrier lists

In `scripts/lib/validation/config/electricity.py`, add the carrier name to:
- `extendable_carriers.Generator` list
- `renewable_carriers` list

### Step 3: Ensure cost data exists

The carrier name must appear in the technology-data CSV (`resources/costs_{year}_processed.csv`). Check the cost column names match. If the tech doesn't exist in technology-data, add an entry to `data/custom_costs.csv` or use `config.costs.overwrites`.

### Step 4: Availability matrix (if land-use constrained)

`scripts/determine_availability_matrix.py` uses the `renewable.{tech}` config to compute suitable areas. It automatically handles new technologies if their config follows the standard structure (corine, natura, ship_threshold, etc.).

### Step 5: Build profiles

`scripts/build_renewable_profiles.py` reads `config["renewable"][tech]` and uses atlite to compute capacity factors. New techs are automatically handled if:
- The `resource.method` is "wind" or "pv" (atlite's built-in methods)
- A valid turbine/panel name is specified

### Step 6: Regenerate and run

```bash
pixi run generate-config  # Update config.default.yaml
snakemake -call resources/profile_{clusters}_{technology}.nc  # Test profile generation
```

## Adding a New Conventional Generator

Conventional generators (dispatchable, fuel-burning) are simpler.

### Step 1: Add to carrier lists
In config validation, add to `electricity.conventional_carriers`.

### Step 2: Ensure cost data
Must have capital_cost, marginal_cost, efficiency, lifetime in cost CSV.

### Step 3: Power plant data
If using real-world power plants, ensure the fuel type is recognized by `powerplantmatching` library. Check `scripts/build_powerplants.py` for the fuel type mapping.

### Step 4: Automatic pickup
`scripts/add_electricity.py` function `attach_conventional_generators()` iterates over `conventional_carriers` and attaches matching plants. No script changes needed if the carrier name matches powerplantmatching conventions.

## Adding a New Storage Technology

### StorageUnit (combined power + energy, e.g., battery)

1. Add to `electricity.extendable_carriers.StorageUnit`
2. Set `electricity.max_hours.{carrier}` (energy/power ratio)
3. Cost data needs: capital_cost, marginal_cost, efficiency (round-trip)

### Store + Link pair (separate power and energy, e.g., H2 storage)

Used when charge/discharge have different characteristics.

1. Add to `electricity.extendable_carriers.Store`
2. Check `STORE_LOOKUP` dict in `scripts/add_electricity.py`:
   ```python
   STORE_LOOKUP = {
       "H2": ("H2 electrolysis", "H2 fuel cell", "H2 Store"),
       "battery": ("battery charger", "battery discharger", "battery"),
   }
   ```
3. If your tech isn't in STORE_LOOKUP, add it. The tuple is `(charger_link, discharger_link, store_name)`.
4. Cost data needs entries for each component (charger, discharger, store).

## Adding a Sector-Coupled Technology

Sector-coupled technologies (heat pumps, electrolysis, CCS, etc.) are added in `scripts/prepare_sector_network.py`.

### Step 1: Config toggle
Add to `scripts/lib/validation/config/sector.py`:
```python
my_technology: bool = Field(default=False, description="Enable my technology")
```

### Step 2: Implement in prepare_sector_network.py

Find the appropriate section (heat, transport, industry, gas, H2) and add:

```python
if snakemake.params.sector["my_technology"]:
    # Add buses for the carrier
    n.add("Bus", "EU my_carrier", carrier="my_carrier", location="EU")

    # Add link (converter)
    n.add("Link",
        "EU my_converter",
        bus0="EU electricity",
        bus1="EU my_carrier",
        carrier="my_converter",
        efficiency=costs.at["my_converter", "efficiency"],
        capital_cost=costs.at["my_converter", "capital_cost"],
        p_nom_extendable=True,
    )

    # Add store (if applicable)
    n.add("Store",
        "EU my_carrier store",
        bus="EU my_carrier",
        carrier="my_carrier",
        e_nom_extendable=True,
        capital_cost=costs.at["my_carrier store", "capital_cost"],
    )
```

### Step 3: Spatial resolution (if per-node)

If the technology needs spatial resolution, use the `define_spatial()` pattern:

```python
# In define_spatial() function at top of prepare_sector_network.py
spatial.my_carrier = SimpleNamespace()
if snakemake.params.sector.get("my_carrier_spatial"):
    spatial.my_carrier.nodes = nodes + " my_carrier"
    spatial.my_carrier.locations = nodes
else:
    spatial.my_carrier.nodes = ["EU my_carrier"]
    spatial.my_carrier.locations = ["EU"]
```

### Step 4: Wire input data

If the technology needs input data (potentials, profiles):
1. Create a `build_my_feature.py` script
2. Add a rule in `rules/build_sector.smk`
3. Reference the output in `prepare_sector_network` rule inputs

## Cost Data Integration

### How costs are loaded
1. `scripts/process_cost_data.py` downloads and processes technology-data CSV
2. Outputs `resources/costs_{year}_processed.csv`
3. Scripts load via `load_costs()` from `_helpers.py`

### Cost CSV structure
```
technology,parameter,value,unit,source
onwind,capital_cost,1040,EUR/kW,DEA
onwind,lifetime,27,years,DEA
onwind,FOM,3,% of capital_cost,DEA
```

### Overriding costs
In config:
```yaml
costs:
  overwrites:
    onwind:
      capital_cost: 900  # EUR/kW
```

Or in `data/custom_costs.csv` for persistent overrides.

## Naming Conventions

### Carrier names
- Lowercase with spaces: `"solid biomass"`, `"urban central heat"`, `"H2"` (exception: H2 is uppercase)
- Must be consistent across: config, cost data, scripts, `plotting.default.yaml` (tech_colors, nice_names)

### Spatially-resolved components
- Bus naming: `"{location} {carrier}"` (e.g., `"DE0 0 urban central heat"`)
- Link naming: `"{bus0_location} {technology}"` (e.g., `"DE0 0 H2 electrolysis"`)
- Generator naming: `"{bus_location} {carrier}"` (e.g., `"DE0 0 solar"`)

### Plotting integration
When adding a new carrier, also add to `config/plotting.default.yaml`:
```yaml
plotting:
  tech_colors:
    my_carrier: "#hexcolor"
  nice_names:
    my_carrier: "My Nice Name"
```
