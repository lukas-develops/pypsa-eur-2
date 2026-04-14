# Sector Coupling

## Overview

Sector coupling extends the electricity-only model to include heating, transport, industry, gas networks, hydrogen, biomass, and CO2 management. The primary implementation is in `scripts/prepare_sector_network.py` (~3000 lines), which assembles all sector components onto the electricity network.

## Config Toggles

All in `config.sector`:
```yaml
sector:
  transport: true          # Land transport (EVs, fuel cell vehicles)
  heating: true            # Space/water heating
  biomass: true            # Biomass supply chains
  industry: true           # Industrial energy demand
  agriculture: false       # Agricultural emissions
  gas_network: false       # Explicit gas pipeline network
  H2_network: false        # Hydrogen pipeline network
  co2_spatial: false       # Per-node CO2 tracking (vs single EU node)
  district_heating:
    potential: 0.6         # Max district heating share
```

## The define_spatial() Pattern

At the top of `prepare_sector_network.py`, `define_spatial()` creates `SimpleNamespace` objects that control whether each carrier is spatially resolved (per-node buses) or EU-wide (single bus):

```python
spatial = SimpleNamespace()

# Example: hydrogen
spatial.h2 = SimpleNamespace()
if snakemake.params.sector["H2_network"]:
    spatial.h2.nodes = nodes + " H2"           # ["DE0 0 H2", "FR0 0 H2", ...]
    spatial.h2.locations = nodes                # ["DE0 0", "FR0 0", ...]
else:
    spatial.h2.nodes = ["EU H2"]               # Single EU-wide node
    spatial.h2.locations = ["EU"]

# Similar for: gas, co2, biomass, oil, methanol, etc.
```

**When to use spatial resolution:**
- `H2_network: true` -> per-node H2 buses + H2 pipeline links
- `gas_network: true` -> per-node gas buses + gas pipeline links
- `co2_spatial: true` -> per-node CO2 buses + CO2 pipeline links
- `biomass_spatial: true` -> per-node biomass buses + biomass transport links

## Heat Sector

### Heat system types (defined in `scripts/definitions/`)

```
scripts/definitions/heat_sector.py      -> HeatSector enum: residential, services, urban, rural
scripts/definitions/heat_system.py      -> HeatSystem enum: urban_central, urban_decentral, rural
scripts/definitions/heat_system_type.py -> HeatSystemType enum: central, decentral
```

Three heat systems modeled:
1. **Urban central** (district heating): large-scale heat pumps, CHP, gas boilers, thermal storage
2. **Urban decentral**: individual heat pumps, gas boilers, resistive heaters
3. **Rural**: individual heat pumps, gas boilers, biomass boilers

### Heat components added in prepare_sector_network.py

For each heat system and each node:
- **Heat bus**: `"{node} {heat_system} heat"` (e.g., `"DE0 0 urban central heat"`)
- **Heat load**: demand time series from `build_hourly_heat_demand.py`
- **Heat pump** (Link): electricity -> heat, COP from weather-dependent profiles
- **Gas boiler** (Link): gas -> heat
- **Resistive heater** (Link): electricity -> heat
- **TES** (Store): thermal energy storage
- **Solar thermal** (Generator): time-varying heat generation
- **Retrofitting** (Generator): reduces heat demand (building insulation)

### Heat data pipeline
```
build_temperature_profiles.py    -> temp_air_total.nc, temp_soil_total.nc
build_cop_profiles/run.py        -> cop_profiles.nc (COP = f(temperature))
build_daily_heat_demand.py       -> daily_heat_demand.csv
build_hourly_heat_demand.py      -> hourly_heat_demand.nc
build_heat_totals.py             -> heat_totals.csv (annual by country)
build_district_heat_share.py     -> district_heat_share.csv
build_central_heating_temperature_profiles/ -> central_heating_temperature_profiles.nc
build_direct_heat_source_utilisation_profiles.py -> direct heat profiles
```

## Transport Sector

### Components
- **Battery EVs**: electricity -> transport via Link
- **Fuel cell vehicles**: H2 -> transport via Link
- **ICE vehicles**: oil -> transport (decreasing share over time)
- **EV batteries as storage**: vehicle-to-grid capability

### Key scripts
- `build_transport_demand.py`: nodal transport demand time series
- `build_mobility_profiles.py`: EV availability profiles (when parked and available for V2G)

### Config keys
```yaml
sector:
  land_transport_fuel_cell_share: {2030: 0.05, 2050: 0.15}
  land_transport_electric_share: {2030: 0.25, 2050: 0.85}
  bev_dsm: true              # EV demand-side management
  v2g: true                  # Vehicle-to-grid
  bev_availability: 0.5      # Share of EVs available for V2G
  transport_heating_deadband_upper: 20  # Celsius
  transport_heating_deadband_lower: 15
```

## Hydrogen Economy

### H2 components
- **Electrolysis** (Link): electricity -> H2
- **Fuel cell** (Link): H2 -> electricity
- **H2 storage** (Store): underground (salt caverns) or overground tanks
- **H2 pipeline** (Link): transport between nodes (if `H2_network: true`)
- **H2 demand**: industry (steel, chemicals), transport (fuel cells), power-to-X

### Key scripts
- `build_salt_cavern_potentials.py`: underground H2 storage potential per region
- `cluster_gas_network.py`: also clusters H2 network topology

### Config keys
```yaml
sector:
  H2_network: false           # Enable H2 pipelines
  H2_network_limit: 2000      # Max H2 pipeline capacity (MW)
  H2_retrofit: false          # Allow gas -> H2 pipeline conversion
  H2_retrofit_capacity_per_CH4: 0.6  # Capacity ratio after retrofit
  hydrogen_underground_storage: true
  hydrogen_underground_storage_locations: ["onshore"]  # or "nearshore"
```

## Gas Network

### Components
- **Gas buses**: per-node (if `gas_network: true`) or single EU bus
- **Gas pipelines** (Link): existing + expandable
- **Gas import** (Generator): at entry points (LNG terminals, pipeline interconnectors)
- **Methanation** (Link): H2 + CO2 -> CH4 (synthetic gas)
- **Biogas** (Store): from biomass potentials

### Key scripts
- `build_gas_network.py`: extracts gas pipeline topology
- `build_gas_input_locations.py`: identifies gas entry points
- `cluster_gas_network.py`: clusters gas network to match electricity

## CO2 Management

### Components
- **CO2 atmosphere** (Bus): tracks atmospheric CO2
- **CO2 capture** (Link): from point sources (CHP, industrial) or direct air capture
- **CO2 storage** (Store): geological sequestration (limited by `seq` in sector_opts)
- **CO2 pipeline** (Link): transport to storage (if `co2_spatial: true`)
- **CO2 utilization** (Link): Fischer-Tropsch, methanation

### Key scripts
- `build_co2_sequestration_potentials.py`: storage capacity per region
- `build_clustered_co2_sequestration_potentials.py`: aggregated to cluster level

## Industry Sector

### Components
- Process heat demand (high/medium/low temperature)
- Feedstock demand (chemicals, steel)
- Specific industrial processes with technology options

### Key scripts (complex pipeline)
```
build_industrial_production_per_country.py        # Current production volumes
build_industrial_production_per_country_tomorrow.py  # Future projections
build_industrial_distribution_key.py              # Spatial disaggregation keys
build_industrial_production_per_node.py           # Per-node production
build_industry_sector_ratios.py                   # Technology mix per sector
build_industrial_energy_demand_per_node.py        # Final energy demand per node
build_ammonia_production.py                       # Ammonia-specific data
```

## Biomass

### Components
- **Solid biomass** (Store): wood, agricultural residues
- **Biogas** (Store): from anaerobic digestion
- **Biomass CHP** (Link): biomass -> electricity + heat
- **Biomass transport** (Link): between nodes (if `biomass_spatial: true`)

### Key scripts
- `build_biomass_potentials.py`: supply curves per region (JRC ENSPRESO data)
- `build_biomass_transport_costs.py`: transport cost by distance

### Config keys
```yaml
sector:
  biomass_spatial: false      # Per-node biomass (vs EU-wide)
  biomass_transport: false    # Enable biomass transport links
  conventional_generation:
    biomass: true             # Biomass power plants
```

## Build Rule Mapping (rules/build_sector.smk)

| Category | Rules | Key Outputs |
|----------|-------|-------------|
| Population | build_population_layouts, build_clustered_population_layouts | Population per node |
| Heat demand | build_daily_heat_demand, build_hourly_heat_demand | Heat load time series |
| Heat supply | build_cop_profiles, build_solar_thermal_profiles, build_geothermal_heat_potential | COP profiles, supply potentials |
| Temperature | build_temperature_profiles, build_central_heating_temperature_profiles | Temperature time series |
| Energy data | build_eurostat_balances, build_energy_totals, build_heat_totals | Energy/heat demands |
| Transport | build_transport_demand, build_mobility_profiles, build_shipping_demand | Transport demand profiles |
| Industry | build_industrial_* (7 rules), build_industry_sector_ratios | Industrial demand per node |
| Gas | build_gas_network, build_gas_input_locations, cluster_gas_network | Gas infrastructure |
| Biomass | build_biomass_potentials, build_biomass_transport_costs | Biomass supply data |
| Storage | build_salt_cavern_potentials, build_ates_potentials, build_egs_potentials | Storage potentials |
| CO2 | build_co2_sequestration_potentials, build_clustered_co2_sequestration_potentials | CO2 storage potential |
| Assembly | **prepare_sector_network** | Full sector-coupled network |
