---
title: Parameters
---

# LipidOracle Parameters Guide

All LipidOracle runs are configured via a `lipidoracle.yaml` file. If the file is missing, LipidOracle will generate a default configuration and exit so you can review and customize it before running.

## File Structure

The YAML file has two main sections:

```yaml
VERSION: x.y.z                  # Tracks configuration version

WORKFLOW:                       # Workflow switches and choices
  ...

PARAM:                          # Numeric and threshold parameters
  ...
```

## Workflow Section

The `WORKFLOW` section controls which annotation engines run and how they operate.

### Library & Engine Switches

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `lipidoracle` | bool | `True` | Enable the internal LipidOracle MS1/MS2 annotation engine |
| `lipidex` | bool | `False` | Enable LipiDex library matching (experimental, fewer features) |
| `lipidblast` | bool | `False` | Enable LipidBlast library matching (experimental, fewer features) |
| `library_extra` | list | `[]` | Additional CSV-format lipid libraries (e.g., `['/path/to/custom.csv']`) |

### Data Filtering

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `mz_target` | string | `''` | Optional m/z value list file to restrict analysis to specific features (useful for targeted runs) |

### Advanced Analysis

| Parameter | Type | Choices | Description |
|-----------|------|---------|-------------|
| `acyl_analysis` | bool | `False` or `True` | Analyze acyl chain composition (requires EAD or UVPD spectra); produces idlevel 3 and 4 matches |
| `rt_check` | string | `False`, `'legacy'`, `'liri'`, `'hydra'` | Retention-time validation engine (see [RT Models](RT_CHECK.html) for details) |

## PARAM Section

The `PARAM` section sets numeric thresholds and limits. Key parameters are listed below; the default file includes others.

### Identification & Scoring

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `rarity_max` | int | `3` | Maximum rarity index kept (lower = stricter, more conservative IDs; higher = includes rare chain compositions). Set to `0` to keep all. |
| `rarity_penalty` | float | `0.15` | Score penalty applied to rare lipids instead of excluding them. See [Lipid library](LIPID_LIBRARY.html#rarity). |

### MS1 Matching

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ms1_tolerance` | float | `0.01` | Precursor m/z tolerance in Da (tolerance used for MS1 library matching) |
| `ms1_maxO` | int | `0` | Maximum extra oxygen atoms added during MS1 library generation (for oxidized variants) |

### MS2 Matching & Spectral Processing

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ms2_noise_abs` | int | `10` | Absolute MS2 peak intensity floor (peaks below this are removed) |
| `ms2_maxO` | int | `0` | Maximum extra oxygen atoms added during MS2 isomer generation |

### EAD/UVPD Analysis

When `acyl_analysis` is enabled, these parameters control the exhaustive C=C and oxidation position search:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `ead_max_unsaturated` | int | `2` | Caps unsaturated chains per feature (higher = more candidates, slower runtime) |
| `ead_max_oxygen` | int | `2` | Caps total oxygen atoms across all chains |
| `ead_max_candidates` | int | `30000` | Candidates scored before stride-sampling. Do not lower below ~22,300 for oxidised work, or true structures can be sampled out silently. See [EAD](EAD.html). |
| `ead_chain_cutoff` | float | `0.95` | Relative score threshold: retain isomers scoring ≥ (max_score × cutoff) |
| `ead_min_score` | float | `0.2` | Absolute `score_L3` floor |

## Quick-Start Defaults

For most users, the default configuration works well:

```yaml
VERSION: 1.0.126

WORKFLOW:
  mz_target: ''
  lipidoracle: True
  lipidex: False
  lipidblast: False
  library_extra: []
  acyl_analysis: False
  rt_check: liri

PARAM:
  rarity_max: 3
  rarity_penalty: 0.15
  ms1_tolerance: 0.01
  ms2_tolerance: 0.02
  ms2_noise_abs: 10
  ms1_maxO: 0
  ms2_maxO: 0
  ead_max_unsaturated: 2
  ead_max_oxygen: 2
  ead_max_candidates: 30000
  ead_chain_cutoff: 0.95
  ead_min_score: 0.2
  # ... (other parameters with defaults)
```

### When to Customize

- **Looser matching** (more candidates): lower `rarity_max`, increase `ms1_tolerance`
- **Stricter matching** (fewer false positives): raise `rarity_max`, decrease `ms1_tolerance`, tighten `ead_chain_cutoff`
- **Faster runs** (skip acyl analysis): keep `acyl_analysis: False` or lower `ead_max_candidates`
- **Oxidized lipids**: raise `ms1_maxO` and `ms2_maxO` (default 0 means no oxidation variants)
- **Targeted RT validation**: choose `rt_check: legacy` for the original two-regression model, or `rt_check: False` to skip RT filtering entirely

## References

- **Retention-time consistency**: see [RT_CHECK.md](RT_CHECK.html)
- **Built-in MS2 library and rarity**: see [LIPID_LIBRARY.md](LIPID_LIBRARY.html)
- **C=C and oxidation localisation**: see [EAD.md](EAD.html)
- **Running LipidOracle**: see [DOCKER_README.md](DOCKER_README.html)

## Questions and problems

Report issues or ask questions at [github.com/zamboni-lab/lipidoracle/issues](https://github.com/zamboni-lab/lipidoracle/issues).
