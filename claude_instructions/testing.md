# Testing

## Test Framework

- **Framework**: pytest
- **Location**: `test/`
- **Fixtures**: `test/conftest.py`
- **Run all tests**: `pixi run pytest` or `pytest test/`
- **Run specific test**: `pytest test/test_config_schema.py -v`

## Existing Test Files

| File | Tests |
|------|-------|
| `test/test_config_schema.py` | Config defaults match Pydantic schema; JSON schema in sync |
| `test/test_base_network.py` | Base network building functions |
| `test/test_build_powerplants.py` | Power plant allocation |
| `test/test_build_shapes.py` | Geographic shape generation |
| `test/test_data_versions_layer.py` | Data versioning system |

## Config Schema Test (Critical)

`test/test_config_schema.py` verifies that:
1. `config/config.default.yaml` matches what Pydantic models generate
2. `config/schema.default.json` matches the Pydantic JSON schema

**After any Pydantic model change:**
```bash
pixi run generate-config    # Regenerates config.default.yaml and schema.default.json
pytest test/test_config_schema.py  # Verify sync
```

If this test fails, it means the config file is out of sync with the Pydantic schema. The fix is always `pixi run generate-config`.

## Testing Script Changes Standalone

Use `mock_snakemake()` to run any script outside Snakemake:

```python
# In an interactive session or separate test script:
from scripts._helpers import mock_snakemake

snakemake = mock_snakemake(
    "build_shapes",
    clusters="50",
)
# Now snakemake.input, snakemake.output, snakemake.config are populated
# based on the rule definition in the Snakefile
```

**Requirements:**
- Input files must exist (from a prior run or tutorial mode)
- Working directory must be the repo root
- The rule name must match exactly

## Tutorial Mode for Quick Testing

Set `tutorial: true` in config to use a minimal dataset:
- Small geographic area (typically just a few countries)
- Reduced time period
- Fewer clusters needed

Good for validating that scripts run without errors before a full run.

## Integration Testing

Full workflow tests run the Snakemake pipeline end-to-end:

```bash
# Dry run (check DAG without executing):
snakemake -call -n

# Run with tutorial config and small cluster count:
snakemake -call results/networks/elec_s_5.nc --configfile config/test/config.electricity.yaml

# Run a specific rule:
snakemake -call --until build_shapes
```

## CI/CD Workflows

Located in `.github/workflows/`:

| Workflow | Purpose |
|----------|---------|
| `test.yaml` | Runs Snakemake workflow tests (integration) |
| `validate.yaml` | Validates config schema and formatting |
| `codeql.yaml` | Security scanning |
| `update-lockfile.yaml` | Dependency lock file updates |

## Writing New Tests

### Unit test pattern
```python
# test/test_my_feature.py
import pytest
import pandas as pd
from scripts.build_my_feature import my_function

@pytest.fixture
def sample_data():
    return pd.DataFrame({"col": [1, 2, 3]})

def test_my_function(sample_data):
    result = my_function(sample_data)
    assert len(result) == 3
    assert result["output_col"].sum() > 0
```

### Using conftest fixtures
`test/conftest.py` provides shared fixtures. Check it for available fixtures like sample networks, config dicts, etc.

## Code Quality

- **Linter**: `ruff check .` (configured in `ruff.toml`)
- **Formatter**: `ruff format .`
- **Pre-commit**: `pre-commit run --all-files`
- **Rules**: F (pyflakes), E/W (pycodestyle), I (isort), D (pydocstyle), UP (pyupgrade), TID (tidy-imports)
