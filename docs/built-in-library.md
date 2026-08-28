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
5. Inspect source, `score_s2`, matched fragments, resolvability, and RT for every non-routine call.
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
