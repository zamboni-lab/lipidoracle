---
title: Locating C=C and oxidation by EAD
---

# Locating C=C and oxidation by EAD

MS2 matching tells you a lipid is `PE 18:1_20:4`. It does not tell you *where* on those chains the double bonds sit, or where a hydroxyl group has been added. Acyl-chain analysis answers that second question.

It runs on lipids that have **already been identified**, taking their EAD or UVPD spectra and localising double-bond and oxygen positions along each chain.

```yaml
WORKFLOW:
  acyl_analysis: v2   # False | v1 | v2 | v3 | v4 | uvpd
```

This step is off by default. It requires EAD or UVPD spectra: it cannot extract positional information from CID.

## What goes in and what comes out

**Input** is annotations at idlevel 2 or above, plus idlevel 1 single-chain species that still have something to localise when `PARAM.include_ms1_monochain` is enabled (default `true`).

**Output** is two new annotation levels:

- **idlevel 3**, one row per surviving position candidate. Several isomers can survive for the same feature.
- **idlevel 4**, one consensus row per feature, reporting positions as percentages across the surviving candidates.

The `score_L3` column carries the localisation score, and `score_L3_data` and `ms2_evidence` record the supporting evidence. All engines bound `score_L3` to the range 0 to 1, but **the mapping differs by engine**. Equal `score_L3` values from different engines are not equivalent evidence, and should not be compared across runs configured differently.

## How it works

Every engine follows the same skeleton.

1. **Select parents** from the existing annotations.
2. **Parse chain tokens**, so `DG 16:0_16:1` becomes per-chain carbon, double-bond, and oxygen counts.
3. **Enumerate isomers** per chain, for every allowed arrangement of double bonds and oxygen.
4. **Build a fragment ladder** for each isomer, expressed as *losses relative to the precursor*. This is why the same arithmetic works whether the charge sits on the chain itself, as in a derivatised free fatty acid, or elsewhere, as on a lyso headgroup or one arm of a triacylglycerol: the headgroup mass cancels out of the difference.
5. **Combine chains** into whole-molecule candidates.
6. **Score** against the observed peaks within `ms2_tolerance`.
7. **Keep and aggregate**, retaining idlevel 3 candidates and folding them into one idlevel 4 consensus row.

Engines differ at steps 4 and 6: how they compute fragment masses, and how they score matches.

### The oxygen chemistry

A `;O` suffix in a lipid name gives an oxygen **count**, not a chemistry. Since every hypothesis is anchored on a measured precursor, each candidate must be isobaric, so each oxygen group has to pay for itself:

| Group | Change | Backbone cost |
|---|---|---|
| `-OH` hydroxyl | +1 O | none |
| `=O` ketone | +1 O, −2 H | one double bond |

So `20:4;O` admits `20:4;OH` and `20:3;oxo`, but never `20:4;oxo`. An oxygenated carbon may not be sp2, so no double bond may land on or adjacent to it.

Two consequences worth stating plainly: the supported oxygen space is deliberately limited to hydroxyl and ketone readings, and hydroperoxide or extra-carboxyl interpretations are **not** enumerated at all.

## Choosing an engine

| Engine | Fragment model | Oxygen | Use for |
|---|---|---|---|
| `v1` | Fixed CH₂-loss ladder | Not supported | Non-oxidised lipids. Tuned and regression-tested on glycerolipids. |
| `v2` | Exact elemental formula per cleavage, intensity-aware scoring | Yes | Oxidised PUFAs. Strongest measured double-bond localisation. |
| `v3` | v1's ladder extended with an oxygen step | Yes | v1's arithmetic plus oxygen. For a chain with no oxygen it reproduces v1 exactly. |
| `v4` | Two-stage cascade: exact formulas to place the oxygen, then a ladder to place the double bonds | Yes | Strongest measured oxygen *type* calls, distinguishing hydroxyl from ketone. |
| `uvpd` | v3's enumeration, emitting only at allylic and alpha bonds | Yes | UVPD spectra, which are different fragment chemistry rather than a scoring variant. |

As a starting point: **`v1` for lipids without oxidation, `v2` for oxidised PUFAs.** Reach for `v4` when telling hydroxyl from ketone is the point of the experiment, since separating that decision into two stages is exactly what it was built for. Use `uvpd` only with UVPD spectra.

Exactly one engine runs per run. There is no per-feature mixing.

## Configuration

| Parameter | Default | Applies to | Meaning |
|---|---|---|---|
| `ead_max_unsaturated` | `2` | all | Caps unsaturated chains per feature |
| `ead_max_oxygen` | `2` | v2, v3, v4, uvpd | Caps total oxygen atoms across all chains |
| `ead_max_candidates` | `30000` | all | Candidates scored before stride-sampling kicks in |
| `ead_chain_cutoff` | `0.95` | all | Keep candidates within this fraction of the best score for the feature |
| `ead_min_score` | `0.2` | all | Absolute `score_L3` floor |
| `ead_chain_corr_weight` | `0.5` | v1, v3, v4, uvpd | Weight of the correlation term, unused by v2 |
| `ead_intensity_scaling` | `none` | v2, v4 | Compressive transform on matched intensity: `none`, `sqrt`, `log10`, `ln`, `log2` |
| `include_ms1_monochain` | `true` | all | Admit idlevel 1 single-chain parents |
| `db_n_notation` | `false` | all | Report positions as `n-x` instead of Δ |

**`ead_max_candidates` deserves particular care.** A single oxidised token such as `20:4;O` enumerates roughly 22,300 candidates. If the cap sits below that, the true structure can be stride-sampled out of consideration *before scoring ever runs*. The failure is silent and looks identical to a scoring failure in the output. An unoxidised `20:4` enumerates only 1,365, which is why this only bites on oxidised work. The default of 30,000 is set above the oxidised case deliberately: lower it only if you understand the space you are searching.

## Limitations

These are real and worth knowing before interpreting results.

**Hydroxyl and ketone are genuinely hard to separate.** The `;O` count data model cannot always distinguish them from spectral evidence alone. On a validation set that is 100% hydroxyl by construction, where any ketone call is wrong by definition, `v2`'s top candidate is still a ketone-branch structure in roughly 30% of spectra. `v4` exists specifically to improve this, and does, but the ambiguity is inherent rather than fully solved.

**No ketone ground truth exists in our validation data.** Every oxygen-type figure is measured on hydroxyl-only material. Any bias toward the hydroxyl reading would therefore score well in validation while being untestable. Treat ketone calls with more caution than hydroxyl calls.

**`[M+NH4]+` precursors of underivatized free fatty acids fail completely**, scoring zero double-bond top-1 across every engine on our test data. That flat zero is the signature of a modelling gap, not noise. Two problems compound: fragments do not retain the ammonium charge carrier that losses are anchored on, and a large share of the signal sits in a hydrocarbon-cation series the fragment model cannot produce. Derivatised or sodiated spectra of the same lipids do not show this. Prefer a different adduct.

**Tuned constants come from a narrow fixture set**, pooled across derivatisation chemistries that behave quite differently. Transfer to other chain lengths, derivatisations, or oxidised glycerolipids is an assumption rather than a validated result.

## Diagnostics

`diag/annotation_idlevel3.csv` lists the retained positional candidates and `diag/annotation_idlevel4.csv` the per-feature consensus. In the dashboard, the **MS2 match** tab shows the mirror plot for a selected candidate, and clicking a peak maps it onto the structure so you can see which cleavage supports a given position call.

## Related

- [Parameters](PARAMETERS.html) for the full configuration reference
- [The built-in MS2 library](LIPID_LIBRARY.html) for what is identified before this step runs
- [Retention-time consistency](RT_CHECK.html)
- [Installation and usage](DOCKER_README.html)

## Questions and problems

Report issues or ask questions at [github.com/zamboni-lab/lipidoracle/issues](https://github.com/zamboni-lab/lipidoracle/issues).
