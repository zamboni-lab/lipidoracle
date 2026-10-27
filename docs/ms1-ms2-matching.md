# MS1 and MS2 matching

LipidOracle separates precursor evidence from fragment evidence. This prevents a mass-only assignment from being presented as a resolved molecular species.

## Annotation levels and stages

| Stage | Evidence | Typical result | ID level |
| --- | --- | --- | --- |
| 1 | Precursor accurate mass and adduct | `PC 34:1` | 1, sum composition/species |
| 2 | MS2 diagnostic and chain-related fragments | `PC 16:0_18:1` or, where supported, `PC 16:0/18:1` | 2, molecular species or *sn*-resolved species |
| 3 | Acquisition-specific within-chain fragments | `PC 16:0/18:1(9)` plus an aggregate consensus | 3 and 4 |

MS1 cannot distinguish arrangements with identical total formula. For example, `PC 16:0_18:1` and `PC 16:1_18:0` both have the same precursor mass as `PC 34:1`. MS2 is needed to establish chain composition. Stage 3 is needed to localize C=C or supported oxygen sites within a chain.

## Stage 1: accurate-mass matching

Stage 1 compares each precursor to generated library candidates using the configured precursor tolerance and applies a mass-error penalty.

```yaml
PARAM:
  ms1_tolerance: 0.01
  ms1_ppm_penalty: 0.03
  s1_lo_score_cutoff: 0.2
  s1_score_top_cutoff: 0.5
```

- `ms1_tolerance` is an absolute Da lookup window. Set it from observed calibrated mass accuracy, not a vendor nominal setting.
- `ms1_ppm_penalty` reduces the score as absolute ppm error grows inside the lookup window.
- `s1_lo_score_cutoff` removes weak candidates absolutely.
- `s1_score_top_cutoff` is documented in the template as a relative MS1 cutoff. The current stage-1 implementation retains only candidates tied at the strict best score, so do not rely on it to preserve near-best MS1 alternatives.

Stage 1 also matches configured internal standards through `WORKFLOW.is_peaks`. It does **not** prove chain composition, double-bond location, or oxygen position. Retain sum-composition language in downstream reporting unless stage 2 or stage 3 provides the needed evidence.

## Stage 2: fragment and library matching

For every candidate arrangement, LipidOracle predicts expected fragments with positive weights and can include negative-weight exclusion evidence. Predicted and observed fragments match within `PARAM.ms2_tolerance` after spectrum cleanup.

### Stage-2 simulation engines (`WORKFLOW.stage2`)

`WORKFLOW.stage2` selects which MS2 fragment simulator constructs the internal headgroup and acyl-chain library for stage 2:

```yaml
WORKFLOW:
  lipidoracle: true
  stage2: legacy        # false | legacy | qc-cid | qc-ead
```

| `stage2` value | Engine | Chemistry modeled |
| --- | --- | --- |
| `legacy` or `true` | Empirical rule-based simulator (default) | Combines curated headgroup rules (`data/diag_cid_hg.json`) with class-specific acyl/alkyl/sphingoid cleavage rules (`src/fragments.rs`) across all built-in lipid classes (glycerophospholipids including `MMPE`, `DMPE`, `PT`, `PMe`, `PEth`, `cPA`, `PGlc`, `PIM1`, `PIM2`, `CDP-DG`, `BMP`, `HBMP`, `LBMP`, `NAPE`, `LNAPE`, `NAPS`, `LNAPS`, `CL`, `MLCL`, `DLCL`, and `O-`/`P-` ether keys; glycerolipids and thylakoid/betaine lipids `MGDG`, `MGMG`, `DGDG`, `DGMG`, `SQDG`, `SQMG`, `GlcADG`, `DGTS`, `DGTA`, `LDGTS`, `DGCC`, `LDGCC`; fatty acyls `FA`, `CAR`, `NAE`, `NAGly`, `NAT`, `NASer`, `FAHFA`, `FAA`, `FAL`, `FOH`, `WE`, `CoA`; sphingolipids `SM`, `LysoSM`, `Cer`, `CerP`, `CerPE`, `HexCer`, `LysoHexCer`, `Hex2Cer`/`LacCer`, `Hex3Cer`, `LysoGb3`, `SHexCer`, `IPC`/`PI-Cer`, `MIPC`, `MIP2C`, `GIPC`, `AcylCer`, `SL`, `SPH`; and sterol/amino lipids `ST`, `CE`, `ST-S`, `SG`, `ASG`, `BA-FA`, `OL`). Enforces ether-`PE`/`PS` acylium gating (`!has_ether`), conserves symmetry reaction degeneracy when deduplicating identical chains, and supports negative-mode `FA [M-H]-`, `[M+HCOO]-`, and `[M+Ac-H]-` / `[M+CH3COO]-` fragmentation. |
| `qc-cid` | First-principles QC-CID simulator | Evaluates gas-phase RRKM/TS-2D master-equation barriers ($E^* = 46\text{ kcal/mol}$, $s_{\text{eff}} = 8.5$) across all supported glycerophospholipid, glycerolipid, glycolipid, betaine, fatty-acyl amide/taurine/amino-acid, sphingolipid, and sterol/amino lipid classes. Filters kinetically inaccessible ($E_0^\ddagger > 50\text{ kcal/mol}$) headgroup artifacts (`PE` $m/z\ 142.0264$, `PS` $m/z\ 186.0162$) and adds `TG`/`DG [M+Na]+` $\text{Na}^+$-retained `[M+Na-RCOOH]+` ions, `PE`/`PS [M+H]+` ketene neutral losses (`-RCH=C=O`), `PC [M+H]+` minor *sn*-1 ketene/FA losses, and PUFA ($\text{db} \ge 4$) retro-ene decarboxylation `[RCOO-CO2]-` anions. |
| `qc-ead` | First-principles 10–20 eV QC-EAD simulator | Extends the `qc-cid` thermal channels with EAD-specific electronic/radical headgroup and glycerol-backbone diagnostics: `PC`/`LPC [M+H]+` `GPC+H-CH3OH` ($m/z\ 226.0839$) and `GPC+H` ($m/z\ 258.1101$), `SM`/`LysoSM [M+H]+` C2–C3 phosphocholine cleavages ($m/z\ 225.0999$ and $253.0948$), homolytic ester C–O acyloxyl radical loss (`[M+Adduct-RCOO•]+`), Baba's regioisomeric *sn*-1 glycerol C1–C2 radical loss (`[M+Adduct-R1COOCH2•]+`), and sphingoid C2–C3 backbone ions across ceramide, glycosphingolipid, phosphosphingolipid, and sulfonolipid classes. |
| `false` | Disabled | Skips stage 2 and reports stage-1 accurate-mass matches only. |

The stage-2 score is:

```text
s2_score = 0.75 × (matched_weight − 2 × exclusion_penalty) / total_weight
         + 0.25 × (matched_intensity / total_intensity)
```

| Term | Interpretation |
| --- | --- |
| `matched_weight` | Sum of positive predicted-fragment weights that were observed. |
| `exclusion_penalty` | Magnitude of negative-weight fragments that were observed. These are evidence against a candidate and are weighted twice. |
| `total_weight` | Sum of all positive predicted-fragment weights. |
| `matched_intensity` | Observed intensity attributed to matched fragments. |
| `total_intensity` | Total observed intensity of the scan. |

The first component asks whether expected diagnostic chemistry is present. The second ensures that a candidate explaining only a few tiny peaks cannot dominate a spectrum whose high-intensity peaks it fails to explain. `s2_modcos` may be shown as a familiar cosine-style reference, but it is not the value used to filter or select stage-2 calls.

### Stage-2 filters

```yaml
PARAM:
  ms2_noise_abs: 10
  ms2_noise_prc: 0.3
  ms2_mztrim: 5
  ms2_tolerance: 0.02
  s2_selection: posterior
  s2_lo_score_cutoff: 0.2
  s2_min_matched_rel: 0.01
  s2_score_top_cutoff: 0.9
```

1. Peaks below `max(ms2_noise_abs, percentile(ms2_noise_prc))` are removed, then peaks too close to precursor are removed by `ms2_mztrim`. A candidate is also discarded unless its most intense matched fragment is at least `PARAM.s2_min_matched_rel` (default `0.01`) of the spectrum's base peak.
2. `PARAM.s2_selection` picks how surviving candidates are cut, mirroring stage 3's `s3_selection` (the two are independent):
   - `posterior` (default) — raw scores are converted to a softmax probability (`s2_post_temperature`), and the smallest set reaching `s2_post_credible_level` (default `0.95`) is kept, subject to `s2_post_odds_floor` (default `0.125`). `s2_lo_score_cutoff` does **not** apply on this path; `s2_score` is rewritten to the posterior probability, and the raw `hybrid_score` moves to `s2_score_raw`.
   - `legacy` — candidates below `s2_lo_score_cutoff` are discarded, then surviving candidates must be within `s2_score_top_cutoff` of the best stage-2 score in that scan. At the default `0.9`, candidates within 90% of the best score remain.

Increasing tolerance or lowering thresholds can improve sensitivity but also allows incorrect fragment coincidences. Calibrate these values on representative spectra with known standards.

**Both stages select the same way.** Stage 2 and stage 3 each enumerate a candidate set and convert the raw scores of that set into a posterior by softmax; the retained set is the smallest one holding `s2_post_credible_level` (or `s3_post_credible_level`), subject to the odds floor. The sets differ in what they enumerate - stage 3 covers the localized isomers of the selected chains, stage 2 the library species and adducts compatible with the precursor - and the posterior is over the set each stage generates. If the true species is not in that set (an unmodelled class, the wrong adduct, a chimaeric spectrum), the posterior still normalises over the candidates that were generated, so read the credible-set size, `s2_score_raw` and the resolvability verdict together rather than `s2_score` alone. `s2_selection: legacy` keeps the historical absolute-cutoff behaviour (`s2_lo_score_cutoff`, then a relative top cutoff) for runs that need it; `s2_lo_score_cutoff` does not apply on the posterior path.

## Chain resolvability is separate from a score lead

A higher score does not always demonstrate that a spectrum could distinguish the winner from a rival. LipidOracle audits each contested chain composition using fragments predicted by **exactly one** arrangement. Shared fragments cannot resolve a structural question because both candidates explain them.

The `rival_resolvability` field means:

| Value | Meaning |
| --- | --- |
| `yes` | Observed exclusive evidence favored the reported arrangement over its best disagreeing rival. |
| `no` | A disagreeing rival survived, but exclusive evidence did not favor the reported result. |
| `uncontested` | No surviving candidate disagreed on that structural aspect. This is not equivalent to `no`. |
| blank | The question was not applicable, such as a level-1 call. |

The audit is written to `diag/s2_chain_resolvability.csv`. Enable `s2_chain_dump_candidates: true` only when a full candidate-level diagnostic table is needed.

### Honest fallback controls

```yaml
PARAM:
  s2_compress_undefined_chains_to_species: true
  S2_unresolvable_chains_to_species: false
  s2_unresolvable_sn_to_unknown: true
```

- `s2_compress_undefined_chains_to_species` handles the extreme case where only headgroup fragments match. It avoids inventing chains.
- `S2_unresolvable_chains_to_species` is the strongest fallback, and off by default. It reports `PC 34:1` instead of an arrangement when the data do not distinguish it from a rival. The default keeps the arrangement and removes only the *sn* claim, because the audit abstains in cases where the withheld arrangement is generally right.
- `s2_unresolvable_sn_to_unknown` is milder. It changes a slash to an underscore, for example `PC 16:0/18:1` to `PC 16:0_18:1`, retaining chain identities without claiming *sn* order.

The chain posterior fields `top_prob` and `cred95` are diagnostic only. `s2_chain_temperature` has not been calibrated as a probability and must not be interpreted as a validated confidence percentage.

## Practical review sequence

For a stage-2 result, verify in this order:

1. Correct polarity, precursor *m/z*, adduct, and mass error.
2. Class/headgroup evidence and `s2_score`.
3. Chain-specific fragments and `rival_resolvability`.
4. Whether `/` versus `_` honestly reflects *sn* evidence.
5. RT plausibility and duplicate/consistency outcomes.
6. If stage 3 ran, use its output as a separate within-chain evidence layer rather than replacing the stage-2 checks.

See [Outputs and dashboard](outputs-dashboard.md) for where these values appear and [Stage 3 with EAD](stage3-ead.md) for positional analysis.
