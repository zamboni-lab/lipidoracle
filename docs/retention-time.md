# Retention-time checking

Retention time (RT) is an orthogonal plausibility check applied after MS1/MS2 matching and cleanup. In reversed-phase lipid LC, more carbon generally elutes later, while more double bonds or oxygen commonly elute earlier. RT checking can remove annotations that are chemically plausible by mass and fragments but inconsistent with the run's chromatographic structure.

RT filtering is optional. Use it conservatively for a new chromatography method, sample type, or limited dataset.

## Configure RT checking

```yaml
WORKFLOW:
  rt_check: liri       # false, liri, hydra, or legacy
  rt_ref: ''            # optional external anchors

PARAM:
  rt_deviation_cutoff: 0.10
```

Only one selected RT engine runs per analysis:

| Value | Engine | Main use |
| --- | --- | --- |
| `false` | None | No RT-based removal. Use while validating a new method or investigating unexpected behavior. |
| `liri` | Lipid Retention Index | Default, conservative in-run per-class model. Best when the run has sufficient confirmed homologous series. |
| `hydra` | HYDrophobic Retention Axis | Cross-class model that can assess zero-shot classes. More aggressive, especially for level-1 candidates. |
| `legacy` or `true` | Legacy models | Historical comparison or rollback. Runs generic formula and class-specific regressions together. |

An unrecognized value fails configuration validation rather than silently disabling RT checking.

## External reference RT file

`WORKFLOW.rt_ref` optionally points to a CSV with these columns:

```csv
hg,c,db,mod,rt
PC,34,1,,320.5
PC,36,2,,345.1
```

`hg` is headgroup/class, `c` and `db` are total carbon and double-bond counts, `mod` holds modification information, and `rt` is reference retention time. The bundled `data/rt_ref_2min.csv` is an example schema.

The engine does not blindly assume an external reference uses the same gradient. It first aligns anchors onto the current run's RT axis and can reject an incompatible file. In LiRI, alignment requires at least `rt_liri_align_min_inliers` matched pairs and a leave-one-out error no larger than `rt_liri_align_max_loo`. When alignment fails, LiRI falls back to in-run anchors rather than forcing an erroneous transfer.

Internal standards specified by `WORKFLOW.is_peaks` are separate from `rt_ref`. They add stage-1 standard matching; they are not a replacement for the RT reference schema.

## Processing order and final generic check

After the selected engine runs, LipidOracle applies a final generic formula-based RT check before stage 3. It trains a robust ridge model from confident level-2-or-higher annotations with formula and score at least 0.7, using C, H, N, O, P, S, and degree of unsaturation.

The exported `rt_deviation` is the absolute difference between observed and predicted RT from this **final generic** model. It is not necessarily the same value as the error in the LiRI, HYDRA, or legacy engine diagnostic.

`rt_deviation_cutoff` is resolved as follows:

| Value | Interpretation |
| --- | --- |
| `0` | Disable the final generic cutoff. |
| `0 < value <= 1` | Fraction of observed RT span. `0.10` means 10% of `max(RT) - min(RT)`. |
| `value > 1` | Absolute seconds. |

Rows above the resolved cutoff are removed when an RT engine is selected. If `rt_check: false`, `rt_deviation` may still be calculated and displayed for review, but this cutoff does not remove rows.

## LiRI: Lipid Retention Index

LiRI forms a structural index roughly equivalent to carbon content penalized for double bonds and oxygen, then fits a monotone index-to-RT map independently for each `(class, ether)` group. It uses high-confidence in-run IDs and optional aligned external anchors.

```yaml
PARAM:
  rt_liri_train_minlevel: 2
  rt_liri_check_minlevel: 1
  rt_liri_min_score: 0.7
  rt_liri_min_score_protection: 0.7
  rt_liri_align_min_inliers: 15
  rt_liri_align_max_loo: 5.0
  rt_liri_z_soft: 3.0
  rt_liri_z_hard: 5.0
  rt_liri_jitter_floor: 0.5
  rt_liri_min_group_n: 5
```

LiRI tiers describe how much local anchor support exists:

| Tier | Meaning | RT decision |
| --- | --- | --- |
| A | Exact structural anchor match | Hard rejection can apply. |
| B | Inside a group's anchored range | Hard rejection can apply. |
| C | Extrapolation from a local group | Soft decision only. |
| D | No local model for the class/ether group | Not checked. |

LiRI is deliberately conservative. It can leave a class untouched when the run contains no local homologous-series evidence, including some MS1-only categories such as FA and CE. Its score-protection gate can preserve a high-confidence level-2 result from a soft deviation, but hard deviations are not protected.

## HYDRA: HYDrophobic Retention Axis

HYDRA fits a shared monotone hydrophobicity axis while modeling class effects from headgroup SMILES-fragment features and chain inventory, including carbon, double bonds, oxygen, ether, and lyso features. It can provide a chemistry-informed estimate for classes with little direct local evidence.

```yaml
PARAM:
  rt_hydra_anchor_min_score: 0.7
  rt_hydra_check_minlevel: 1
  rt_hydra_z_soft: 3.0
  rt_hydra_z_soft_ms1: 0.75
  rt_hydra_z_hard: 5.0
  rt_hydra_z_compete: 4.0
  rt_hydra_r_max: 8.0
  rt_hydra_llr_weight: 0.0
  rt_hydra_min_anchors: 24
  rt_hydra_lambda: 3.0
```

HYDRA combines absolute and nearby-anchor relative evidence. It can compete candidates within the same precursor when one candidate is RT-comfortable and another is an RT outlier. It is powerful for zero-shot classes, but the special default level-1 soft threshold (`rt_hydra_z_soft_ms1: 0.75`) is intentionally strict and can remove many level-1 calls. Raise it toward `rt_hydra_z_soft`, or set it to `0` to disable the special level-1 threshold, only after reviewing representative diagnostics.

## Legacy

Legacy runs a pooled generic formula regression and a per-class regression together. It retains historical guards for lyso and triacylglycerol behavior. The key settings are `rt_legacy_generic_cutoff`, `rt_legacy_isigma_cutoff`, training/protection scores, `rt_legacy_lyso_max`, `rt_legacy_tg_min`, and `rt_legacy_duplicate_max_delta`.

Use Legacy primarily for comparison with prior LipidOracle workflows. It assumes a more linear relation between RT and composition than LiRI or HYDRA and is less robust on gradients with nonlinear late elution.

## Review the diagnostics

| File | Meaning |
| --- | --- |
| `diag/rt_liri.csv` | LiRI prediction, uncertainty, z score, tier, and verdict. |
| `diag/rt_hydra.csv` | HYDRA predicted RT, absolute/relative z scores, tier, and score effect. |
| `diag/rt_hydra_anchors.csv` | HYDRA anchors used to build the model. |
| `diag/rt_class.csv` | Legacy class-model trail. |
| `diag/rt_generic.csv` | Final generic formula check. This is the source of exported `rt_deviation`. |

A correct review asks: Was the candidate actually modeled? Which tier and error did the selected engine produce? Did score protection apply? Did the final generic cutoff agree? Is the removal consistent with standards and chromatographic knowledge? Never treat a missing prediction as a negative result.

## Recommendation

- Choose **LiRI** for a conservative routine workflow with adequate in-run homologous series.
- Choose **HYDRA** when cross-class coverage and level-1 screening are important, then inspect its strict level-1 behavior.
- Choose **Legacy** only for historical comparison.
- Choose **false** when validating unfamiliar chromatography, because an unvalidated RT model should not silently remove biological candidates.

See [Outputs and dashboard](outputs-dashboard.md) for access to RT columns and dashboard review.
