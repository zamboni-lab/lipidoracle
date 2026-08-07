---
title: Locating C=C and oxidation by EAD
---

# Locating C=C and oxidation by EAD

MS2 matching tells you a lipid is `PE 18:1_20:4`. It does not tell you *where* on those chains the double bonds sit, or where a hydroxyl group has been added. Acyl-chain analysis answers that second question.

It runs on lipids that have **already been identified**, taking their EAD spectra and localising double-bond and oxygen positions along each chain.

```yaml
WORKFLOW:
  acyl_analysis: v2   # False | v1 | v2 (experimental)
```

This step is off by default. It requires EAD spectra: it cannot extract positional information from CID or other dissociation methods.

## What goes in and what comes out

**Input** is annotations at idlevel 2 or above, plus idlevel 1 single-chain species that still have something to localise when `PARAM.include_ms1_monochain` is enabled (default `true`).

**Output** is two new annotation levels:

- **idlevel 3**, one row per surviving position candidate. Several isomers can survive for the same feature.
- **idlevel 4**, one consensus row per feature, reporting positions as percentages across the surviving candidates.

The `score_L3` column carries the localisation score, and `score_L3_data` and `ms2_evidence` record the supporting evidence. The mapping between raw scores and the reported 0–1 range differs between engines, so **equal `score_L3` values from different engines are not equivalent evidence** and should not be compared across runs configured differently.

## The oxygen chemistry

A `;O` suffix in a lipid name gives an oxygen **count**, not a chemistry. Since every hypothesis is anchored on a measured precursor, each candidate must be isobaric, so each oxygen group has to pay for itself:

| Group | Change | Backbone cost |
|---|---|---|
| `-OH` hydroxyl | +1 O | none |
| `=O` ketone | +1 O, −2 H | one double bond |

So `20:4;O` admits `20:4;OH` and `20:3;oxo`, but never `20:4;oxo`. An oxygenated carbon may not be sp2, so no double bond may land on or adjacent to it.

Two consequences worth stating plainly: the supported oxygen space is deliberately limited to hydroxyl and ketone readings, and hydroperoxide or extra-carboxyl interpretations are **not** enumerated at all.

## The two engines

### v1: Fixed ladder, non-oxidised

v1 walks the chain from tail to head, losing a fixed unit (CH₂, or CH/CH₃ near a double bond) per step. It emits three neutral-loss variants per step (`0`, `-H`, `+H`). Step weight comes from a hand-tuned distance array with a spike at 2 bonds away from the nearest double bond.

v1 does **not** support oxidised lipids at all. Oxidation parameters are ignored; chains with `;O` are filtered out entirely.

**Validation:** tuned and regression-tested on non-oxidised glycerolipids (`ara-ead` fixture, 114 spectra of arachidonic acid fatty acids).

**Reference:** published in Wu et al, *Analyst*, 2025, [10.1039/d5an00567a](https://doi.org/10.1039/d5an00567a).

### v2: Exact formulas, oxidised, experimental

v2 takes the opposite approach: computes exact elemental-formula losses for every cleavage, and scores independently against each observed peak (not one-to-one matching). It supports hydroxyl and ketone oxygens natively.

v2 is **experimental**. It is validated on two test datasets but has known limitations (see below). Measured performance:

| Dataset | Scans | Double-bond accuracy | Hydroxyl accuracy | Notes |
|---|---|---|---|---|
| HETE (hydroxylated FA) | 96 | 79% | N/A | Rust v2 vs Python: Rust 79%, Python 77% |
| ARA (polyunsaturated FA) | 114 | 83% | N/A | Rust v2: 83%, Python: 75% |
| ARA (excluding NH₄ adducts) | 82 | 100% | N/A | Both engines reach 100% when NH₄ is excluded |

The remaining mismatches are concentrated in adduct-specific cases:

- **Na-adduct spectra** (HETE): several isomers fail to localise correctly
- **NH₄-adduct spectra** (ARA): both Rust and Python fail; the modelling gap is documented (fragments do not retain the mobile ammonium charge)
- **AMP-derivatised FA** (mixed): mostly correct, scattered failures

**Important:** v2 is suitable for research and methods development. It is not a production-ready engine. Use it when you need to localise oxidised lipids and understand its failure modes.

## Configuration

| Parameter | Default | Meaning |
|---|---|---|
| `ead_max_unsaturated` | `2` | Caps double bonds per chain (higher = more candidates, slower) |
| `ead_max_oxygen` | `2` | Caps total oxygen atoms across all chains (v2 only) |
| `ead_max_candidates` | `30000` | Candidates scored before stride-sampling. Do not lower below ~22,300 for oxidised work. |
| `ead_chain_cutoff` | `0.95` | Keep candidates within this fraction of the best score |
| `ead_min_score` | `0.2` | Absolute `score_L3` floor |
| `ead_chain_corr_weight` | `0.5` | Weight of the correlation term (v1 only; unused by v2) |
| `ead_intensity_scaling` | `none` | Compressive transform on matched intensity (v2 only): `none`, `sqrt`, `log10`, `ln`, `log2` |
| `include_ms1_monochain` | `true` | Admit idlevel 1 single-chain parents |
| `db_n_notation` | `false` | Report positions as `n-x` instead of Δ |

**`ead_max_candidates` deserves particular care.** A single oxidised token such as `20:4;O` enumerates roughly 22,300 candidates. If the cap sits below that, the true structure can be stride-sampled out of consideration *before scoring ever runs*. The failure is silent and looks identical to a scoring failure in the output. The default of 30,000 is set above the oxidised case deliberately: lower it only if you understand the space you are searching.

## Known limitations

**v2 hydroxyl vs ketone ambiguity.** The `;O` count data model cannot always distinguish hydroxyl from ketone from spectral evidence alone. On a validation set that is 100% hydroxyl by construction, v2's top candidate is still a ketone-branch structure in roughly 30% of spectra. This is a genuine data-driven ambiguity, not a scoring bug.

**No ketone ground truth.** Every oxygen-type figure is measured on hydroxyl-only material. Any bias toward hydroxyl reading scores well in validation while being untestable. Treat ketone calls with caution.

**`[M+NH4]+` precursors of underivatized free fatty acids fail completely** (both engines). Fragments do not retain the ammonium charge that losses are anchored on, and a large share of signal sits in a hydrocarbon-cation series neither engine can model. Derivatised or sodiated spectra of the same lipids do not show this failure. Prefer a different adduct.

**Tuned constants come from a narrow fixture set**, pooled across derivatisation chemistries (AMP and Na-adduct, mainly) that behave very differently. Transfer to other chain lengths, derivatisations, or glycerolipids is an assumption rather than a validated result.

## Diagnostics

`diag/annotation_idlevel3.csv` lists the retained positional candidates and `diag/annotation_idlevel4.csv` the per-feature consensus. In the dashboard, the **MS2 match** tab shows the mirror plot for a selected candidate, and clicking a peak maps it onto the structure so you can see which cleavage supports a given position call.

## Related

- [Parameters](PARAMETERS.html) for the full configuration reference
- [The built-in MS2 library](LIPID_LIBRARY.html) for what is identified before this step runs
- [Retention-time consistency](RT_CHECK.html)

## Questions and problems

Report issues or ask questions at [github.com/zamboni-lab/lipidoracle/issues](https://github.com/zamboni-lab/lipidoracle/issues).
