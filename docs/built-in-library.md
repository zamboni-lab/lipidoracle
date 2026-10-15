# Built-in lipid library

LipidOracle's internal library is the primary source for MS1 precursor matching and MS2 fragment-based annotation. The exact class, adduct, maximum ID level, carbon/double-bond limits, and active species count are therefore written into the **Lipid library** page of every dashboard, using the configuration from that run.

## What the internal library provides

The internal library combines:

- lipid-class definitions and neutral formulas;
- generated sum compositions and chain arrangements within configured limits;
- supported adducts and their likelihood priors;
- predicted diagnostic, neutral-loss, and chain-related fragments for MS2 scoring;
- structural metadata used for nomenclature, CXSMILES generation, rarity, RT modeling, and stage 3.

This design gives the library consistent coverage across supported classes and allows the same chemistry to be used in positive and negative mode. It also means that coverage depends on the active configuration. `PARAM.adducts`, `ms1_maxO`, `ms2_maxO`, `rarity_max`, and `remove_d` change which candidates are generated.

## Coverage is run-specific

Open the dashboard's **Lipid library** page to see the authoritative coverage table for the run. It lists, for each active class/adduct combination:

- the class name and abbreviation;
- relevant adducts in the selected polarity;
- maximum annotation level supported by that library entry;
- carbon and double-bond limits;
- the number of unique active species.

Do not infer coverage from the presence of a similar class in another polarity, instrument mode, or configuration. In particular:

- cardiolipin (`CL`) has more limited structural resolution than simpler classes;
- ether-linked entries use their own library keys;
- class/adduct support is chemistry-specific, so an adduct in `PARAM.adducts` does not guarantee useful fragments for every class;
- stage 3 has additional acquisition- and class-specific restrictions.

## Adducts and priors

The default list includes common positive and negative channels:

```yaml
PARAM:
  adducts: ['[M+H]+', '[M+NH4]+', '[M+Na]+', '[M+H-H2O]+', '[M-H2O+H]+', '[M-H]-', '[M+Ac-H]-', '[M+FA-H]-', '[M-2H]2-', '[M+HCOO]-']
  adduct_likelihood: [1, 0.5, 0.3, 0.3, 0.3, 1, 0.5, 0.5, 0.5, 1]
```

The two lists are positional pairs. A lower likelihood is a weaker prior for that channel, not a mass-tolerance change. Prefer a short list representing solvent additives and acquisition polarity. Leaving implausible channels enabled expands the isobaric candidate field and can weaken discrimination.

## Rarity

To differentiate common and rare lipids, LipidOracle uses a rarity score. The rarity is built as the sum of rarity of the headgroup, chain lengths (very long and short, uneven), and modifications such as hydroxyl (`-OH`) or epoxy groups. A rarity of `0` indicates a common lipid (e.g., PC 34:1), whereas a rarity of 2 or 3 is for lipids which are rare.

Two parameters control rarity filtering:

```yaml
PARAM:
  rarity_max: 3
  rarity_penalty: 0.15
```

- **A priori filtering**: `rarity_max` sets the maximum rarity allowed into library generation. A low value is more conservative. In the current implementation, `0` keeps only the most common forms.
- **A posteriori filtering**: leaving `rarity_max` at a value of 2–3 will include rare forms in the analysis. To prioritize common over rare, `rarity_penalty` subtracts a score penalty per rarity unit from admitted candidates. This allows uncommon but well-supported candidates to survive while preferring common alternatives when spectral evidence is otherwise similar.

Rarity is a prior, not proof that a lipid is present or absent. Avoid using it as a substitute for MS2 evidence, RT review, or standards.

## Oxidation and oxygen limits

The internal library can generate oxygenated candidates under `ms1_maxO` and `ms2_maxO`. The `;O` suffix records an oxygen count at this stage, not a localized chemical assignment. 

The class table generated for your run reflects your oxidation policy:

- The internal library covers the major lipid classes implemented in LipidOracle.
- Cardiolipin (`CL`) has more limited structural resolution than simpler lipid classes.
- Ether-linked entries are represented by their distinct library keys.
- Raising oxygen limits enlarges the MS1/MS2 candidate pool and produces more isobaric competition.

See [Oxidation analysis](oxidation.md) for the difference between oxygen-count evidence and EAD2 positional hypotheses.

## Negative-mode library

Negative-mode coverage comes from the `adducts_neg` entries of `data/lipids_definition.json`. The prior is the adduct's default weight (`adducts_neg_score`) and `rarity` is its adduct rarity (`adducts_neg_rarity`), shown only when non-zero. Only adducts that are also listed in `PARAM.adducts` are used in a run.

- `[M-H]-` is defined for almost every class. Two exceptions: `MGDG` and `DGDG` use `[M+HCOO]-` as their primary channel (`[M-H]-` is a rarer alternative).
- `[M+Ac-H]-`, `[M+FA-H]-`, and `[M-2H]2-` are in the default `PARAM.adducts`, but no class lists them under `adducts_neg`. Check the dashboard's **Lipid library** page for their effect in your run.
- Some classes have no negative entry: `FA-AMPP`, `SB`, `CE`, `TG`, `DG`, and `MG`. They are not annotated in negative mode.
- `HexCer`, `Hex2Cer`, `LacCer`, `Hex3Cer`, `Cer`, `Sph`, `NAE`, `ST`, `FA`, and `CAR` carry a reduced prior or rarity 1 for `[M-H]-`, so they rank below better-supported alternatives.
- Ether-linked `PC`, `PE`, `LPC`, and `LPE` (`O-`/`P-`) have their own keys, with the same `[M-H]-` channel. `LPC` and `LPE` each cover both the sn-1 and sn-2 acyl regioisomer keys.

| Class | Negative adducts (prior) | Carbon range | Max DB |
| --- | --- | --- | --- |
| `PC (O-)` | `[M-H]-` 1 | 12–60 | 8 |
| `PC (P-)` | `[M-H]-` 1 | 12–60 | 8 |
| `PE (O-)` | `[M-H]-` 1 | 12–60 | 8 |
| `PE (P-)` | `[M-H]-` 1 | 12–60 | 8 |
| `LPC (O-)` | `[M-H]-` 1 | 3–30 | 6 |
| `LPC (P-)` | `[M-H]-` 1 | 3–30 | 6 |
| `LPE (O-)` | `[M-H]-` 1 | 3–30 | 6 |
| `LPE (P-)` | `[M-H]-` 1 | 3–30 | 6 |
| `PI` | `[M-H]-` 1 | 12–60 | 8 |
| `LPI` | `[M-H]-` 1 | 3–30 | 8 |
| `PIP` | `[M-H]-` 1 | 2–60 | 8 |
| `PIP2` | `[M-H]-` 1 | 12–60 | 8 |
| `PIP3` | `[M-H]-` 1 | 12–60 | 8 |
| `PA` | `[M-H]-` 1 | 3–60 | 8 |
| `LPA` | `[M-H]-` 1 | 3–30 | 6 |
| `LPG` | `[M-H]-` 1 | 3–30 | 6 |
| `LPS` | `[M-H]-` 1 | 3–30 | 6 |
| `BMP` | `[M-H]-` 1 | 12–60 | 6 |
| `HexCer` | `[M-H]-` 0.5 (rarity 1) | 12–60 | 6 |
| `Hex2Cer` | `[M-H]-` 0.5 (rarity 1) | 2–60 | 6 |
| `LacCer` | `[M-H]-` 0.5 (rarity 1) | 12–60 | 6 |
| `Hex3Cer` | `[M-H]-` 0.5 (rarity 1) | 12–60 | 6 |
| `SHexCer` | `[M-H]-` 1 | 12–60 | 6 |
| `CerP` | `[M-H]-` 1 | 12–60 | 6 |
| `IPC` | `[M-H]-` 1 | 2–60 | 6 |
| `MIPC` | `[M-H]-` 1 | 12–60 | 6 |
| `MIP2C` | `[M-H]-` 1 | 12–60 | 6 |
| `S1P` | `[M-H]-` 1 | 12–30 | 4 |
| `MGDG` | `[M+HCOO]-` 1; `[M-H]-` 0.5 (rarity 1) | 12–60 | 6 |
| `DGDG` | `[M+HCOO]-` 1; `[M-H]-` 0.5 (rarity 1) | 12–60 | 6 |
| `SQDG` | `[M-H]-` 1 | 12–60 | 6 |
| `GM3` | `[M-H]-` 1 | 12–60 | 6 |
| `GM2` | `[M-H]-` 1 | 12–60 | 6 |
| `GM1` | `[M-H]-` 1 | 12–60 | 6 |
| `GD3` | `[M-H]-` 1 | 12–60 | 6 |
| `GD2` | `[M-H]-` 1 | 12–60 | 6 |
| `GD1` | `[M-H]-` 1 | 2–60 | 6 |
| `GT1` | `[M-H]-` 1 | 12–60 | 6 |
| `NAE` | `[M-H]-` 0.5 (rarity 1) | 3–30 | 6 |
| `ST` | `[M-H]-` 0.3 (rarity 1) | 2–34 | 6 |
| `FA` | `[M-H]-` 1 | 2–34 | 6 |
| `CAR` | `[M-H]-` 0.5 (rarity 1) | 2–30 | 6 |
| `PE` | `[M-H]-` 1 | 2–60 | 8 |
| `LPE` | `[M-H]-` 1 | 2–30 | 6 |
| `PC` | `[M-H]-` 1 | 2–60 | 8 |
| `LPC` | `[M-H]-` 1 | 2–30 | 6 |
| `PG` | `[M-H]-` 1 | 2–60 | 8 |
| `PS` | `[M-H]-` 1 | 2–60 | 8 |
| `Cer` | `[M-H]-` 0.5 (rarity 1) | 2–60 | 6 |
| `Sph` | `[M-H]-` 0.5 (rarity 1) | 2–60 | 6 |
| `SM` | `[M-H]-` 1 | 28–60 | 6 |
| `CL` | `[M-H]-` 1 | 2–80 | 12 |

## Optional libraries

Three optional sources can be added to the internal search:

| Source | Enable with | Intended use |
| --- | --- | --- |
| Custom CSV or CSV.GZ | `WORKFLOW.library_extra` | Standards, project-specific targets, and curated extensions. |
| LipiDex | `WORKFLOW.lipidex: true` | Testing or comparison with that library. Requires `lipidoracle: true`. |
| LipidBlast | `WORKFLOW.lipidblast: true` | Testing or comparison. Some classes are excluded by default because they create frequent false positives. |

The dashboard lists the libraries actually used for a run. Treat that list, the retained YAML, and the copied input MGF as the minimum reproducibility record.

For authoring a custom CSV and considering optional MS2 reference spectra, see [Run LipidOracle](running-lipidoracle.md#5-add-an-extra-library).

## Recommended library practice

1. Use the internal library first and validate expected standards or reference samples.
2. Restrict adducts to the chromatographic and ion-source conditions used.
3. Add project-specific records in a separate, versioned extra-library CSV.
4. Confirm that custom records have formula, neutral mass, adducts, and nomenclature that agree.
5. Inspect source, `s2_score`, matched fragments, resolvability, and RT for every non-routine call.
6. Do not compare raw stage-3 scores across EAD, OAD, UVPD, and OzID. Their candidate spaces and score scales differ.

## Built-in class and adduct coverage

The internal library provides coverage for a defined set of lipid classes, each with specific structural rules and supported adducts. A table listing every internal library entry for your run's polarity is generated in your dashboard under the **Lipid library** page. This table reports:

- the class name and abbreviation;
- relevant adducts in the selected polarity;
- `Max level`: the highest annotation level for each listed adduct;
- carbon and double-bond limits;
- the total number of unique species (reflecting active library settings).

### Important coverage notes

- The internal library covers the major lipid classes implemented in LipidOracle.
- **Cardiolipin (`CL`)** has more limited structural resolution than simpler lipid classes.
- **Ether-linked entries** are represented by their distinct library keys (not mixed with non-ether forms).
- **Class/adduct support is chemistry-specific**, so an adduct in `PARAM.adducts` does not guarantee useful fragments for every class.
- **Stage 3 has additional restrictions** on acquisition and class-specific capabilities.
- Do not infer coverage from the presence of a similar class in another polarity, instrument mode, or configuration.

Unique-species counts reflect the active library settings: `PARAM.adducts`, `ms1_maxO`, `ms2_maxO`, `rarity_max`, and `remove_d` all change which candidates are generated and included in the active count.

### Supported lipid classes

The internal library (`data/lipids_definition.json`) defines the following classes. Adducts, carbon/double-bond limits, and max annotation level are run-specific (see the dashboard table above); this list is static and does not depend on configuration.

| Abbreviation | Class |
| --- | --- |
| `AMPP-FA` | AMPP-derivatized fatty acid |
| `BMP` | Bis(monoacylglycero)phosphate |
| `CAR` | Acylcarnitine |
| `CE` | Cholesteryl ester |
| `Cer` | Ceramide |
| `CerP` | Ceramide phosphate |
| `CL` | Cardiolipin |
| `DG` | Diacylglycerol |
| `DGDG` | Digalactosyldiacylglycerol |
| `FA` | Fatty acid |
| `GD1`, `GD2`, `GD3` | Ganglioside (di-sialo) |
| `GM1`, `GM2`, `GM3` | Ganglioside (mono-sialo) |
| `GT1` | Ganglioside (tri-sialo) |
| `HexCer` | Hexosylceramide (monohexosylceramide) |
| `Hex2Cer` | Dihexosylceramide |
| `Hex3Cer` | Trihexosylceramide |
| `IPC` | Inositol phosphorylceramide |
| `LacCer` | Lactosylceramide |
| `LPA` | Lysophosphatidic acid |
| `LPC` | Lysophosphatidylcholine (diacyl and ether-linked O-/P- keys) |
| `LPE` | Lysophosphatidylethanolamine (diacyl and ether-linked O-/P- keys) |
| `LPG` | Lysophosphatidylglycerol |
| `LPI` | Lysophosphatidylinositol |
| `LPS` | Lysophosphatidylserine |
| `MG` | Monoacylglycerol |
| `MGDG` | Monogalactosyldiacylglycerol |
| `MIP2C` | Mannosyl-diinositolphosphorylceramide |
| `MIPC` | Mannosylinositolphosphorylceramide |
| `NAE` | N-acylethanolamine |
| `PA` | Phosphatidic acid |
| `PC` | Phosphatidylcholine (diacyl and ether-linked O-/P- keys) |
| `PE` | Phosphatidylethanolamine (diacyl and ether-linked O-/P- keys) |
| `PG` | Phosphatidylglycerol |
| `PI` | Phosphatidylinositol |
| `PIP`, `PIP2`, `PIP3` | Phosphatidylinositol mono-/bis-/trisphosphate |
| `PS` | Phosphatidylserine |
| `S1P` | Sphingosine-1-phosphate |
| `SB` | Sphingoid base |
| `SHexCer` | Sulfatide (sulfo-hexosylceramide) |
| `SM` | Sphingomyelin |
| `Sph` | Sphingosine |
| `SQDG` | Sulfoquinovosyldiacylglycerol |
| `ST` | Sterol |
| `TG` | Triacylglycerol |

Ether-linked glycerophospholipids (`PCo`/`PCp`, `PEo`/`PEp`, `LPCo`/`LPCp`, `LPEo`/`LPEp`) share their diacyl abbreviation in output but are defined under separate internal keys, consistent with the [ether-linked entries note](#important-coverage-notes) above.
