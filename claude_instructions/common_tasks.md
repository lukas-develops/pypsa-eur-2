# Common Tasks Cookbook

## Running the Model

### Full run (all default targets)
```bash
snakemake -call                # Uses all available cores, runs locally
```

### Specific target
```bash
snakemake -call results/{run}/networks/elec_s_50.nc
```

### Dry run (check DAG without executing)
```bash
snakemake -call -n
```

### Run up to a specific rule
```bash
snakemake -call --until cluster_network
```

### Force re-run a specific rule
```bash
snakemake -call --forcerun build_shapes
```

### With pixi (recommended)
```bash
pixi run snakemake -call ...
```

## Changing the Country Set

Edit `countries` list in `config/config.yaml`:
```yaml
countries: [DE, FR, NL, BE]  # Only these countries
```

**Important**: After changing countries, purge resources (they depend on country set):
```bash
snakemake purge  # Or manually delete resources/
```

## Changing Temporal Resolution

### Via opts wildcard
```yaml
scenario:
  opts: ["Co2L0.05-3h"]  # 3-hourly resolution
```

### Via config
```yaml
clustering:
  temporal:
    resolution_elec: 3h
    resolution_sector: 24h
```

## Changing the Solver

### Switch to HiGHS (free, open-source)
```yaml
solving:
  solver:
    name: highs
    options: highs-default
```

### Switch to Gurobi (commercial, faster)
```yaml
solving:
  solver:
    name: gurobi
    options: gurobi-default
```

## Adding a New Data Download

### Step 1: Register in data/versions.csv
```csv
my_dataset,primary,v1.0,"latest supported",https://example.com/data.csv
my_dataset,archive,v1.0,"latest supported",https://data.pypsa.org/my_dataset/v1.0/data.csv
```

### Step 2: Add config entry
In `scripts/lib/validation/config/data.py`, add a model for the dataset config:
```python
class MyDatasetConfig(BaseModel):
    source: Literal["primary", "archive"] = "primary"
    version: str = "latest"
```
Add it to the parent Data model.

### Step 3: Add retrieve rule
In `rules/retrieve.smk`:
```python
rule retrieve_my_dataset:
    input:
        storage(dataset_version("my_dataset")["url"]),
    output:
        dataset_version("my_dataset")["folder"] + "/data.csv",
    run:
        move(input[0], output[0])
```

### Step 4: Regenerate config
```bash
pixi run generate-config
```

## Modifying Cost Assumptions

### Quick override in config
```yaml
costs:
  overwrites:
    solar:
      capital_cost: 300  # EUR/kW (override default)
    onwind:
      capital_cost: 900
```

### Persistent overrides
Edit `data/custom_costs.csv` (same format as technology-data CSV).

### Change cost year
```yaml
costs:
  year: 2030  # Use 2030 cost projections instead of 2050
```

## Adding a New Scenario

### Step 1: Create scenarios file
```yaml
# config/scenarios.yaml
high_RE:
  electricity:
    co2limit: 0
  costs:
    year: 2030

low_cost:
  costs:
    year: 2050
    overwrites:
      solar:
        capital_cost: 200
```

### Step 2: Enable in config
```yaml
run:
  name: [high_RE, low_cost]  # or "all" for all scenarios
  scenarios:
    enable: true
    file: config/scenarios.yaml
```

Each scenario overrides the base config for its run. Results go to `results/{scenario_name}/`.

## Debugging a Failed Rule

### Step 1: Check the log
```bash
cat logs/{run_name}/{rule_name}_s_{clusters}_{opts}_{sector_opts}_{planning_horizons}.log
```

### Step 2: Run script standalone
```python
# In Python/IPython:
from scripts._helpers import mock_snakemake
snakemake = mock_snakemake("failing_rule", clusters="50", opts="", sector_opts="", planning_horizons="2050")
# Then step through the script logic
```

### Step 3: Check config
```bash
snakemake -call --printshellcmds -n  # Show what would be executed
```

### Step 4: Common issues
- **Missing input files**: Run prerequisite rules first (`--until`)
- **Memory errors**: Reduce clusters or increase `solving.mem_mb`
- **Infeasible optimization**: Enable `solving.options.load_shedding.enable: true`
- **Config validation error**: Check Pydantic model matches your config changes

## Modifying an Existing Config Section

### Step 1: Edit Pydantic model
Find the right file in `scripts/lib/validation/config/{section}.py` and modify the field.

### Step 2: Regenerate
```bash
pixi run generate-config
```

### Step 3: Test
```bash
pytest test/test_config_schema.py
```

### Step 4: Update scripts
If scripts reference the changed config key, update them. Search for usage:
```bash
grep -r "config\[\"section\"\]\[\"old_key\"\]" scripts/
grep -r "\"old_key\"" rules/
```

## Running Specific Parts of the Workflow

### Only electricity (no sector coupling)
```bash
snakemake -call results/{run}/networks/elec_s_50.nc
```

### Only build resources (no solving)
```bash
snakemake -call --until prepare_sector_network
```

### Only postprocessing (assumes solved networks exist)
```bash
snakemake -call results/{run}/graphs/costs.pdf
```

## Purging Generated Files

```bash
snakemake purge  # Interactive confirmation, deletes resources/ and results/
```

Or manually:
```bash
rm -rf resources/* results/*
touch resources/.gitkeep results/.gitkeep
```

## Remote Execution

### Sync to cluster
```yaml
remote:
  ssh: user@cluster.example.com
  path: /home/user/pypsa-eur
```
```bash
snakemake sync       # Upload code + download results
snakemake sync_dry   # Preview what would be synced
```

## Working with the DAG

### Generate rule dependency graph
```bash
snakemake -call resources/dag_rulegraph.pdf
```

### Generate file dependency graph
```bash
snakemake -call resources/dag_filegraph.pdf
```
