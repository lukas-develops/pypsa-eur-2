# Solving and Optimization

## How Solving Works

The core solver is in `scripts/solve_network.py`. It handles both electricity-only and sector-coupled networks.

### Call chain
1. Rule (`solve_network` or `solve_sector_network`) invokes `scripts/solve_network.py`
2. Script loads the prepared network: `n = pypsa.Network(snakemake.input.network)`
3. Applies custom constraints (CCL, EQ, BAU, SAFE)
4. Calls `n.optimize()` (PyPSA wraps linopy for LP/MIP formulation)
5. Linopy passes the problem to the configured solver (Gurobi, HiGHS, etc.)
6. Results are written back to the network object
7. Network saved: `n.export_to_netcdf(snakemake.output.network)`

### Key function: `solve_network()`

In `scripts/solve_network.py`:
```python
def solve_network(n, config, solving, **kwargs):
    # Set solver options
    solver_options = solving["solver_options"][solving["solver"]["options"]]
    solver_name = solving["solver"]["name"]

    # Optional: skip iterations (faster, less accurate for AC)
    if solving["options"].get("skip_iterations"):
        n.optimize(solver_name=solver_name, solver_options=solver_options, **kwargs)
    else:
        # Iterative LOPF: update line reactances based on results
        ilopf(n, solver_name=solver_name, solver_options=solver_options, **kwargs)
```

## Solver Configuration

### Selecting a solver
```yaml
solving:
  solver:
    name: gurobi    # gurobi, highs, cplex, copt, xpress, glpk, cbc
    options: gurobi-default  # References a named option set
```

### Solver option sets
```yaml
solving:
  solver_options:
    gurobi-default:
      threads: 8
      method: 2        # Barrier method
      crossover: 0     # Skip crossover
      BarConvTol: 1.e-5
      Seed: 123
      AggFill: 0
      PreDual: 0
      GURO_PAR_BARDENSETHRESH: 200
    highs-default:
      threads: 8
      solver: ipm
      run_crossover: "off"
      small_matrix_value: 1e-6
      large_matrix_value: 1e9
      primal_feasibility_tolerance: 1e-5
      dual_feasibility_tolerance: 1e-5
      ipm_optimality_tolerance: 1e-4
      parallel: "on"
      random_seed: 123
```

Thread count from solver options determines Snakemake thread allocation (via `solver_threads()` in `rules/common.smk`).

### Memory allocation
Dynamic in `rules/common.smk:memory()`:
```python
mem_mb = factor * (10000 + 195 * int(clusters))
# factor adjusted for temporal resolution: /N for Nh, *N/8760 for Nseg
# "all" clusters: factor * (18000 + 180 * 4000) = ~720GB base
```

## Key Solving Options

```yaml
solving:
  options:
    # Iteration control
    skip_iterations: true     # Skip iterative LOPF (default, faster)

    # Numerical stability
    noisy_costs: true         # Add small random cost perturbation to break degeneracy

    # Feasibility helpers
    load_shedding:
      enable: false           # Add slack generators (prevents infeasibility)
      cost: 8000              # EUR/MWh cost of load shedding
    clip_p_max_pu: 1.e-2      # Clip near-zero renewable availability

    # Transmission
    transmission_losses: 0    # Number of piecewise segments for loss approx (0 = lossless)
    post_discretization:
      enable: false           # Round continuous expansion to discrete units

    # Advanced
    linearized_unit_commitment: false  # Binary commit decisions for conventional
    rolling_horizon:
      enable: false           # Solve in time chunks (for very large problems)
      overlap: 24             # Hours of overlap between chunks
    store_model: false        # Save LP model to file (for debugging)

    # Custom
    custom_extra_functionality: false  # Path to Python file with extra constraints
```

## Extra Constraints

### Built-in constraints (in solve_network.py)

| Constraint | Config Key | Activated by | Purpose |
|-----------|-----------|--------------|---------|
| CCL | `solving.constraints.CCL` | `CCL` in sector_opts | Country-level capacity limits (min/max per technology per country) |
| EQ | `solving.constraints.EQ` | `EQ{value}` in sector_opts | Equity: limits how much one country's share deviates from equal |
| BAU | `solving.constraints.BAU` | `BAU` in sector_opts | Business-as-usual: minimum capacity = current installed |
| SAFE | `solving.constraints.SAFE` | `SAFE` in sector_opts | N-1 security: enough dispatchable capacity to cover peak load |

### Custom constraints

Set `solving.options.custom_extra_functionality` to a Python file path. The file must define a function `extra_functionality(n, snapshots)` that adds constraints using linopy:

```python
def extra_functionality(n, snapshots):
    # Example: minimum renewable share
    from linopy import LinearExpression
    renewable_gen = n.generators.query("carrier in ['solar', 'onwind']").index
    total_gen = n.generators.index
    # ... add constraint using n.model.add_constraints(...)
```

### Perfect foresight constraints

`add_land_use_constraint_perfect()` in `solve_network.py`: ensures total land use across all investment periods doesn't exceed available land per region.

## Foresight Modes

### Overnight (`rules/solve_overnight.smk`)
- Single investment period
- One rule: `solve_sector_network`
- Input: prepared sector network
- Output: solved network
- Simplest and fastest mode

### Myopic (`rules/solve_myopic.smk`)
- Rolling horizon: solves year-by-year
- Three rules:
  1. `add_existing_baseyear`: Sets up existing capacity for base year (e.g., 2020)
  2. `add_brownfield`: For subsequent years, carries forward previous year's results as existing capacity. Handles decommissioning (lifetime expiry) and cost updates.
  3. `solve_sector_network_myopic`: Solves each year sequentially
- Key script: `scripts/add_brownfield.py` -- transfers optimized capacities from year N to year N+1, removes expired assets

### Perfect Foresight (`rules/solve_perfect.smk`)
- All years solved simultaneously
- Four rules:
  1. `add_existing_baseyear`: Same as myopic
  2. `prepare_perfect_foresight`: Concatenates all planning horizon networks into one multi-period network
  3. `solve_sector_network_perfect`: Solves the combined network
  4. `make_summary_perfect`: Extracts per-year results from combined solution
- Most expensive computationally (time and memory)
- Captures optimal long-term investment paths

## Solver Troubleshooting

### Infeasible model
1. Enable load shedding: `solving.options.load_shedding.enable: true`
2. Check CO2 limits aren't too tight for available technologies
3. Check that all required data files exist (missing profiles = zero generation)
4. Examine the log file for solver-specific infeasibility diagnostics

### Numerical issues
1. Try `solving.solver_options.{solver}-numeric-focus` option set
2. Increase tolerances (BarConvTol, primal_feasibility_tolerance)
3. Enable `noisy_costs: true` (breaks degeneracy)
4. For Gurobi: try `method: 1` (dual simplex) instead of barrier

### Out of memory
1. Reduce `{clusters}` count
2. Increase temporal aggregation (e.g., `3h` or `24h` in opts)
3. Use `rolling_horizon` for very large problems
4. Adjust `solving.mem_mb` to match available RAM
