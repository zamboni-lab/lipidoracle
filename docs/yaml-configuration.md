# YAML configuration reference

A LipidOracle configuration is a YAML file with two top-level sections:

- `WORKFLOW` selects **what** to run and which files to use. Start here.
- `PARAM` controls **how** matching, filtering, scoring, and reporting behave. The supplied defaults are deliberately broad and should normally be retained until a run is evaluated.

The shipped [`data/lipidoracle.yaml`](../data/lipidoracle.yaml) is the full, commented template. Configuration names are case-sensitive. Use YAML booleans such as `true` and `false`, and quote strings that contain special characters.

## Start with `WORKFLOW`

```yaml
VERSION: x.y.z

WORKFLOW:
  mz_target: ''
  lipidoracle: true
  lipidex: false
  lipidblast: false
  library_extra: []
  stage3: false
  rt_check: liri
  rt_ref: ''
  is_peaks: ''
```

### Workflow controls

| Key | Values | What it does |
| --- | --- | --- |
| `mz_target` | empty string or target CSV | Restricts analysis to precursor targets. The CSV supplies target *m/z* values. Useful for focused QA or validation, not routine untargeted discovery. |
| `lipidoracle` | `true` / `false` | Enables the internal MS1 and MS2 workflow. Stage 1 runs whenever it is enabled. |
| `lipidex` | `true` / `false` | Adds LipiDex comparison entries. Requires `lipidoracle: true`; functionality is more limited than for the internal library. |
| `lipidblast` | `true` / `false` | Adds LipidBlast entries. See `lipidblast_skip` before enabling it for routine work. |
| `library_extra` | list of `.csv` or `.csv.gz` paths | Appends external library files. Paths may be local relative paths or container paths such as `/data/...`. |
| `stage3` | `false`, `ead1`, `ead2`, `uvpd`, `oad`, `oad2`, or `ozid` | Runs one acquisition-specific chain-localization engine after stages 1 and 2. See [Stage 3 with EAD](stage3-ead.md) and [Oxidation](oxidation.md). |
| `rt_check` | `false`, `liri`, `hydra`, or `legacy` | Runs one RT checking engine. `liri` is the default conservative choice. See [Retention time](retention-time.md). |
| `rt_ref` | empty string or RT reference CSV | Optional external RT anchors with columns `hg,c,db,mod,rt`. The engine aligns this file onto the current run's RT axis before using it. |
| `is_peaks` | empty string or internal-standard CSV | Adds internal-standard peak matching at stage 1. Bundled EquiSPLASH, LightSPLASH, and UltimateSPLASH files can be used. |

### Select an appropriate stage-3 engine

Do not use an engine simply because it produces more detailed names. The engine must match how the fragments were acquired.

| `stage3` value | Intended evidence | Scope and decision |
| --- | --- | --- |
| `false` | Routine CID/HCD or no localization data | Skip stage 3. Stage 2 results remain valid. |
| `ead1` | EAD chain-cleavage spectra | Heuristic C=C localization for unoxidized species. Oxidized species are skipped. |
| `ead2` | EAD chain-cleavage spectra | Exact-formula C=C, hydroxyl, and ketone candidate model. Required for supported oxidized-lipid localization. |
| `uvpd` | 193/213 nm UVPD | UVPD-specific allylic cleavage model. |
| `oad` | OAD spectra | OAD profile-gated workflow. Existing validation is strongest for PC `[M+H]+`, TG `[M+NH4]+`, and SM `[M+H]+`; other supported classes are extrapolations. |
| `oad2` | OAD spectra | Joint candidate competition across all stage-2 parents for a feature. |
| `ozid` | Ozone-induced dissociation | Requires paired aldehyde and Criegee evidence. Unsupported chemistry is skipped rather than approximated. |

`true` is accepted as an alias for `ead1`. Typing an unrecognized engine is an error, not a silent skip. Only one stage-3 engine runs for a feature.

## `PARAM`: high-impact settings

The following groups cover normal parameter tuning. Run the full template unchanged first, preserve the YAML with each output folder, and tune one group at a time.

### Performance, library size, and adducts

```yaml
PARAM:
  ncores: 6
  rarity_max: 3
  rarity_penalty: 0.15
  ms2_maxO: 2
  ms1_maxO: 2
  remove_d: false
  adducts: ['[M+H]+', '[M+NH4]+', '[M+Na]+', '[M+H-H2O]+', '[M-H2O+H]+', '[M-H]-', '[M+Ac-H]-', '[M+FA-H]-', '[M-2H]2-', '[M+HCOO]-']
  adduct_likelihood: [1, 0.5, 0.3, 0.3, 0.3, 1, 0.5, 0.5, 0.5, 1]
```

- `ncores` controls parallel MS2 scoring. Use a value appropriate for the machine and shared workload.
- `rarity_max` defines which rarity indices are admitted to library generation. Lower values are more conservative. `rarity_penalty` downweights rare candidates that remain eligible.
- `ms1_maxO` and `ms2_maxO` cap extra side-chain oxygen considered during library generation. Increasing them expands both chemical coverage and the candidate space.
- `remove_d` removes deuterated library entries. Supply deuterated internal standards through `is_peaks` instead of relying on them in a general library.
- `adducts` and `adduct_likelihood` are parallel lists. Every likelihood must correspond to the adduct at the same position. Restrict the list to adducts genuinely expected in the solvent and polarity to reduce competing candidates.

### Spectrum cleanup and mass tolerances

```yaml
PARAM:
  ms2_noise_abs: 10
  ms2_noise_prc: 0.3
  ms2_mztrim: 5
  ms1_tolerance: 0.01
  ms2_tolerance: 0.02
  ms1_ppm_penalty: 0.03
```

| Setting | Meaning | Tuning principle |
| --- | --- | --- |
| `ms2_noise_abs` | Absolute fragment intensity floor. | Raise only when the data contain genuine low-level noise that survives centroiding. |
| `ms2_noise_prc` | Percentile floor from 0 to 1. The applied floor is the larger of this percentile and `ms2_noise_abs`. | Lower to retain sparse low-intensity spectra. |
| `ms2_mztrim` | Removes fragments at or above `precursor_mz - ms2_mztrim`. | Increase to remove precursor/neutral-loss clutter, but do not remove expected diagnostic ions. |
| `ms1_tolerance` | Precursor matching tolerance in Da. | Set from calibrated mass accuracy and acquisition resolution. |
| `ms2_tolerance` | Fragment matching tolerance in Da. | Set from fragment mass accuracy. It directly controls false and missed fragment matches. |
| `ms1_ppm_penalty` | Per-absolute-ppm penalty in the stage-1 score. | Normally leave at the default after the tolerance has been calibrated. |

OzID deliberately applies only `ms2_noise_abs`: a weak member of the required aldehyde/Criegee pair must not be removed by percentile filtering.

### Stage 1 and stage 2 thresholds

```yaml
PARAM:
  s1_lo_score_cutoff: 0.2
  s1_score_top_cutoff: 0.5
  s2_lo_score_cutoff: 0.2
  s2_score_top_cutoff: 0.9
  s2_compress_undefined_chains_to_species: true
  S2_unresolvable_chains_to_species: true
  s2_unresolvable_sn_to_unknown: true
  s2_chain_temperature: 1.0
  s2_chain_dump_candidates: false
  s2_suppress_idlevel1_if_idlevel2: true
```

- The `*_lo_score_cutoff` parameters are absolute floors. `s2_score_top_cutoff` retains candidates relative to the best score for the feature. Although the template describes `s1_score_top_cutoff` similarly, the current stage-1 implementation retains only strict best-score ties, so it does not preserve near-best MS1 candidates.
- `s2_compress_undefined_chains_to_species` prevents a multi-chain annotation from claiming a chain split when only headgroup evidence was observed.
- `S2_unresolvable_chains_to_species` is stronger. It reports a species-level result when no exclusive observed fragment distinguishes the winning chain arrangement from its rival. Review `diag/s2_chain_resolvability.csv` before weakening or disabling it.
- `s2_unresolvable_sn_to_unknown` is the milder fallback. It changes `/` to `_`, retaining chain composition while removing an unproven *sn* claim.
- `s2_chain_temperature` influences diagnostic chain posteriors only. It is not calibrated as a probability.
- `s2_chain_dump_candidates` writes all scored arrangements for calibration and troubleshooting. It can make diagnostics large.
- `s2_suppress_idlevel1_if_idlevel2` avoids duplicating the same feature at both specificity levels in the compact export.

See [MS1 and MS2 matching](ms1-ms2-matching.md) for the score and resolvability logic.

### Stage 3 enumeration and selection

```yaml
PARAM:
  s3_include_ms1_monochain: true
  s3_max_unsaturated: 6
  s3_max_oxygen: 2
  s3_max_candidates: 100000
  s3_intensity_scaling: 'none'
  s3_oxo_rarity_prior: 0.85
  s3_selection: posterior
  s3_post_temperature: 0.01
  s3_post_temperature_ead1: 0.03
  s3_post_temperature_ead2: 0.003
  s3_post_credible_level: 0.95
  s3_post_odds_floor: 0.125
  s3_post_min_mass: 0.5
  s3_output_max: 100
  s3_output_extra: 2
```

`ead2` expands each eligible chain into C=C and supported oxygen-site isomers. A `20:4;O` token can have roughly 22,300 valid hydroxyl/ketone hypotheses before multi-chain combinations. Keep `s3_max_candidates` high enough that the true isomer is not stride-sampled out before scoring. This limit controls computational cost **and** potential recall.

`posterior` is the recommended selection mode. It uses a softmax over the closed stage-3 isomer set and retains the smallest score-ranked set reaching `s3_post_credible_level`. The engine-specific temperatures were fitted to their score models and should not be treated as cosmetic display settings. A position enters the idlevel-4 consensus only when its marginal posterior mass reaches `s3_post_min_mass`.

`legacy` (also accepted as `cutoff`) retains candidates within `s3_legacy_cutoff` of the top score and reports agreement across them. It is retained for reproduction of historical workflows; it makes more assertions but is less conservative.

Other important stage-3 controls are:

| Key | Purpose |
| --- | --- |
| `s3_min_margin`, `s3_min_matched` | EAD1 assertion gates for top-two raw-score separation and matched fragment count. |
| `s3_ead2_gate_db_on_resolvability` | Experimental EAD2 gate for C=C assertions. Disabled by default because it has a steep recall cost. |
| `s3_report_position_tuples` | EAD2 output option that reports joint position hypotheses instead of independent per-position marginals. |
| `s3_null_decoys` | Diagnostic-only scoring of impossible allene placements. Never produces annotations. |
| `ozid_require_complete_pair`, `ozid_min_complete_pairs`, `ozid_formula_mz_tolerance` | OzID-specific evidence gates. |
| `s3_uvpd_corr_weight` | UVPD-only chain-correlation score weight. |

### Retention time and diagnostics

`rt_deviation_cutoff` is the final generic formula-model check. Values from 0 to 1 are fractions of the observed RT span, values above 1 are seconds, and `0` disables that final cutoff. The default `0.10` is intentionally loose. The `rt_liri_*`, `rt_hydra_*`, and `rt_legacy_*` families only apply to their respective selected engine. See [Retention time](retention-time.md).

For troubleshooting, `debug: true` enables verbose logs, while `ms2id_cleanup` and `ms1_filter` control consistency filtering. Disable them only for diagnosis. `lipidomics_data_quality_scoring: true` adds lipidomics data-quality columns.

## A safe configuration workflow

1. Begin with the supplied template and edit `WORKFLOW` only.
2. Confirm input polarity, expected adducts, and mass tolerances on known standards or representative files.
3. Inspect `annotation.csv`, `diag/`, and the dashboard before tuning cutoffs.
4. Change one parameter group at a time, record why, and keep the YAML in the output folder.
5. Treat stage-3 position calls and RT-based removals as evidence to review, not unqualified ground truth.
