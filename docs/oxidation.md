# Oxidation analysis: MS1, MS2, and EAD2

Oxygenated lipids require particularly careful evidence language. A precursor mass can establish an elemental oxygen count within mass accuracy, but it cannot by itself establish oxygen type, chain, or position. LipidOracle carries those levels of information forward rather than treating them as equivalent.

## The evidence ladder

| Evidence stage | What can be supported | What remains unresolved |
| --- | --- | --- |
| MS1 | An oxygenated sum composition or formula candidate, for example a lipid carrying `;O`. | Which chain carries oxygen, whether the group is hydroxyl or ketone, and its position. |
| MS2 | Class confirmation, chain composition, and sometimes evidence narrowing oxygenated arrangements. | Within-chain oxygen position and, in many cases, oxygen type. |
| EAD2 | Candidate-specific C=C, hydroxyl, and ketone positions from EAD chain-cleavage evidence. | An unambiguous structure unless the selected candidates and assertion gates support one. |

The presence of `;O` is **not** a statement that the group is hydroxyl, ketone, hydroperoxide, or a specific site. It records the oxygen count represented in the candidate. The correct name should remain as broad as the evidence.

## MS1: oxygen count, not location

Stage 1 generates and matches precursor candidates under `PARAM.ms1_maxO`.

```yaml
PARAM:
  ms1_maxO: 2
  ms1_tolerance: 0.01
  s1_lo_score_cutoff: 0.2
```

MS1 identifies candidates by neutral formula, adduct, and precursor mass. Isomeric oxygen positions, alternative oxygen-bearing chains, and several oxygen types can be isobaric. Therefore, MS1-only results should be reported as oxygen-count sum compositions and reviewed with RT and standards where possible.

Expanding `ms1_maxO` improves coverage of oxygenated candidates but also increases isobaric precursor competition. Do not solve an MS1 ambiguity by simply tightening the mass tolerance unless calibration justifies it.

## MS2: class and chain evidence

Stage 2 uses diagnostic and chain-related fragments to score candidates. `PARAM.ms2_maxO` controls the extra side-chain oxygen considered during MS2-isomer generation:

```yaml
PARAM:
  ms2_maxO: 2
  ms2_tolerance: 0.02
  s2_lo_score_cutoff: 0.2
  s2_score_top_cutoff: 0.9
```

MS2 can confirm the lipid class and can sometimes assign chain composition or an oxygenated chain at level 2. However, stage-2 fragment evidence normally does not localize a functional group along the acyl chain. A chain name such as `20:4;O` should be read as an oxygen-count description, not a positional oxidized-lipid structure.

Use `rival_resolvability` to determine whether observed **exclusive** fragments actually distinguish an oxygenated chain arrangement from a surviving rival. When they do not, the configured stage-2 fallback should preserve a species-level or underscore-separated result instead of an overstated chain or *sn* claim.

## EAD1 versus EAD2

```yaml
WORKFLOW:
  stage3: ead2

PARAM:
  s3_max_oxygen: 2
  s3_max_candidates: 100000
  s3_oxo_rarity_prior: 0.85
  s3_selection: posterior
```

- **EAD1** is an unoxidized C=C-localization engine. It skips oxidized species entirely.
- **EAD2** is the required EAD path for supported oxidized-lipid localization. It uses exact-formula chain cleavage and explicitly enumerates hydroxyl and ketone branches.

### EAD2's oxygen model

A `;O<n>` suffix denotes oxygen count. EAD2 currently enumerates these interpretations:

| Interpretation | Oxygen cost | Backbone effect |
| --- | ---: | --- |
| Hydroxyl (`OH`) | +1 O | No change in hydrogen deficiency. |
| Ketone (`oxo`) | +1 O | Two fewer hydrogens and one fewer C=C relative to the hydroxyl branch. |

Consequently, a `20:4;O` oxygen-count token can represent `20:4;OH(position)` or `20:3;oxo(position)`, but **not** `20:4;oxo(position)`. The engine enforces carbon valence: oxygenated carbons remain sp3, and a C=C cannot be placed on or adjacent to one of those functional carbons.

Hydroperoxide and extra-carboxyl interpretations are not enumerated by EAD2. Do not overextend an EAD2 hydroxyl/ketone result to other oxidation chemistry.

## EAD2 scoring and assertion

EAD2 generates candidate-specific chain-cleavage masses, matches them within `ms2_tolerance`, and computes a weighted raw score from matched intensities. Fragment weights favor bonds alpha to a functional carbon, then positions one bond from an allylic carbon, then default cleavages.

A single observed peak may support more than one predicted EAD2 fragment. Therefore, a top raw score alone is insufficient to interpret as unique positional proof. Candidate selection and aggregation are central:

- Under `s3_selection: posterior`, `score_s3` is a posterior probability over the closed candidate set. The smallest candidate set reaching `s3_post_credible_level` is retained.
- `s3_post_min_mass` controls whether the level-4 consensus may name an oxygen or C=C placement.
- `s3_oxo_rarity_prior` applies a multiplicative penalty per asserted ketone. Its default value, `0.85`, makes an oxo hypothesis require stronger fragment support than an isobaric hydroxyl alternative.
- `s3_report_position_tuples: true` can keep coupled placement alternatives legible instead of reporting only independent positional marginals.

The ketone prior is a fitted score preference, not analytical confirmation. Interpret an oxygen type only after reviewing the candidate set, matched diagnostic peaks, raw score, posterior/credible set, and relevant standards.

![Illustration of an oxygenated structural hypothesis](img/16-demo-ceramide.png)

## Candidate-space and performance caution

An oxidized `20:4;O` chain has approximately 22,300 valid hydroxyl/ketone hypotheses. If `s3_max_candidates` is below the active search space, the candidate enumeration can stride-sample away the true site before scoring. A rapid run with a low cap can be useful for diagnostics but is not a safe localization analysis.

For a production oxygen-localization run:

1. Use EAD2-compatible spectra and validated mass calibration.
2. Keep `s3_max_candidates` at or above the relevant search-space scale, or state the cap as a limitation.
3. Keep `s3_max_oxygen` consistent with the chemical question.
4. Review all retained level-3 candidates, not just the level-4 summary.
5. Check `diag/s3_resolvability.csv`; it reports distinguishability but is not a calibrated confidence probability.
6. Confirm high-value calls with authentic standards, orthogonal fragmentation, or targeted follow-up where possible.

## Reporting language

Use the most specific wording that matches the evidence:

- **MS1:** “oxygenated sum-composition candidate” or “`...;O` candidate”.
- **MS2:** “oxygenated molecular-species candidate”, with chain ambiguity preserved when appropriate.
- **EAD2 level 3:** “supported positional candidate” or “credible positional isomer”.
- **EAD2 level 4:** “consensus positional assignment”, only when the reported posterior/selection and assertion rules are met.

Avoid describing `score_s3` as an instrument-independent probability of a biological structure. It is conditional on the generated candidate space, the selected EAD2 scoring model, the configuration, and the input spectrum.

For the complete stage-3 workflow and all selection parameters, see [Stage 3 with EAD](stage3-ead.md).
