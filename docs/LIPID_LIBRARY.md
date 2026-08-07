---
title: The built-in MS2 library
---

# The built-in MS2 library

LipidOracle ships with an internal lipid library that is the primary source for both MS1 and MS2 annotation. LipiDex, LipidBlast, and any libraries you supply through `WORKFLOW.library_extra` are optional additional sources.

## Understanding the dashboard coverage table

Open `diag/dashboard.html` and go to **Documentation > Lipid Library** to see the exact class and adduct coverage for that run. The dashboard shows two sections:

**Libraries used in this run** lists which sources are active (internal LipidOracle v1.1, LipiDex, LipidBlast, and any extras you configured). This section reflects your `WORKFLOW` parameters.

**Built-in class and adduct coverage** is a table showing every internal library entry for the ion polarity used in your run. Columns are:

| Column | Meaning |
|---|---|
| Class name | Full lipid class name, e.g. Phosphatidylcholine |
| Abbreviation | Standard shorthand, e.g. PC |
| Adducts | Ion forms, e.g. `[M+H]+`, `[M+Na]+`, `[M+NH4]+` |
| Max level | Highest annotation level reachable for that adduct (MS1 or MS2) |
| Carbon range | Minimum to maximum carbons in the library |
| Max DB | Maximum double bonds in the library for that class |
| Unique species | Count of distinct lipid species under the current oxidation settings |

The table is generated from the library definition in the version you ran. The unique-species counts reflect your active `PARAM.ms1_maxO` and `PARAM.ms2_maxO` settings.

### Built-in LipidOracle v1.1 library

| Class | Abbreviation | Adducts | Max level | Carbon range | Max DB | Unique species |
|---|---|---|---|---|---|---|
| Bis(monoacylglycero)phosphate | BMP | `[M+H]+`; `[M+Na]+` | `[M+H]+`: MS1; `[M+Na]+`: MS1 | 12–60 | 6 | 313 |
| Carnitine | CAR | `[M+H]+`; `[M+Na]+`; `[M+NH4]+` | `[M+H]+`: MS2; `[M+Na]+`: MS1; `[M+NH4]+`: MS1 | 2–30 | 6 | 83 |
| Cholesteryl ester | CE | `[M+H]+`; `[M-H2O+H]+`; `[M+Na]+`; `[M+NH4]+` | `[M+H]+`: MS2; `[M-H2O+H]+`: MS1; `[M+Na]+`: MS1; `[M+NH4]+`: MS2 | 2–34 | 6 | 93 |
| Cardiolipin | CL | `[M+H]+`; `[M+Na]+`; `[M+NH4]+` | all: MS2 | 2–80 | 12 | 325 |
| Ceramide | Cer | `[M+H]+`; `[M-H2O+H]+`; `[M+Na]+`; `[M+K]+` | `[M+H]+`: MS2; `[M-H2O+H]+`: MS1; `[M+Na]+`: MS1; `[M+K]+`: MS1 | 2–60 | 6 | 133 |
| Ceramide phosphate | CerP | `[M+H]+`; `[M+Na]+` | `[M+H]+`: MS2; `[M+Na]+`: MS1 | 12–60 | 6 | 133 |
| Diacylglycerol | DG | `[M+NH4]+`; `[M+Na]+`; `[M+H]+`; `[M+K]+`; `[M+Li]+` | all: MS2 | 2–40 | 12 | 221 |
| Digalactosyldiacylglycerol | DGDG | `[M+NH4]+`; `[M+Na]+` | `[M+NH4]+`: MS2; `[M+Na]+`: MS1 | 12–60 | 6 | 313 |
| Fatty acid | FA | `[M+H]+`; `[M+Na]+`; `[M+NH4]+` | all: MS1 | 2–34 | 6 | 93 |
| Ganglioside GD1 | GD1 | `[M+H]+` | MS1 | 2–60 | 6 | 0 |
| Ganglioside GD2 | GD2 | `[M+H]+` | MS1 | 12–60 | 6 | 0 |
| Ganglioside GD3 | GD3 | `[M+H]+` | MS1 | 12–60 | 6 | 0 |
| Ganglioside GM1 | GM1 | `[M+H]+` | MS1 | 12–60 | 6 | 0 |
| Ganglioside GM2 | GM2 | `[M+H]+` | MS1 | 12–60 | 6 | 0 |
| Ganglioside GM3 | GM3 | `[M+H]+` | MS1 | 12–60 | 6 | 0 |
| Ganglioside GT1 | GT1 | `[M+H]+` | MS1 | 12–60 | 6 | 0 |
| Dihexosylceramide | Hex2Cer | `[M+H]+`; `[M+Na]+` | all: MS2 | 2–60 | 6 | 133 |
| Trihexosylceramide | Hex3Cer | `[M+H]+`; `[M+Na]+` | all: MS2 | 12–60 | 6 | 133 |
| Hexosylceramide | HexCer | `[M+H]+`; `[M+Na]+`; `[M+K]+` | `[M+H]+`: MS2; `[M+Na]+`: MS2; `[M+K]+`: MS1 | 12–60 | 6 | 133 |
| Inositolphosphorylceramide | IPC | — | — | 2–60 | 6 | 0 |
| Lysophosphatidic acid | LPA | `[M+H]+`; `[M+Na]+` | all: MS1 | 3–30 | 6 | 81 |
| Lysophosphatidylcholine | LPC | `[M+H]+`; `[M+Na]+`; `[M+Li]+` | all: MS2 | 2–30, 3–30 | 6 | 83, 53, 0 |
| Lysophosphatidylethanolamine | LPE | `[M+H]+`; `[M+Na]+` | `[M+H]+`: MS2; `[M+Na]+`: MS1 | 2–30, 3–30 | 6 | 83, 53, 0 |
| Lysophosphatidylglycerol | LPG | `[M+H]+` | MS1 | 3–30 | 6 | 81 |
| Lysophosphatidylinositol | LPI | `[M+H]+`; `[M+Na]+` | all: MS1 | 3–30 | 6 | 81 |
| Lysophosphatidylserine | LPS | `[M+H]+` | MS1 | 3–30 | 6 | 81 |
| Lactosylceramide | LacCer | `[M+H]+`; `[M+Na]+` | `[M+H]+`: MS2; `[M+Na]+`: MS1 | 12–60 | 6 | 0 |
| Monoacylglycerol | MG | `[M+NH4]+`; `[M+Na]+`; `[M+H]+` | `[M+NH4]+`: MS2; `[M+Na]+`: MS1; `[M+H]+`: MS2 | 2–30 | 6 | 83 |
| Monogalactosyldiacylglycerol | MGDG | `[M+NH4]+`; `[M+Na]+` | `[M+NH4]+`: MS2; `[M+Na]+`: MS1 | 12–60 | 6 | 264 |
| Mannosylinositolphosphorylceramide bisphosphate | MIP2C | — | — | 12–60 | 6 | 0 |
| Mannosylinositolphosphorylceramide | MIPC | — | — | 12–60 | 6 | 0 |
| N-acylethanolamine | NAE | `[M+H]+`; `[M+Na]+`; `[M+NH4]+` | all: MS2 | 3–30 | 6 | 0 |
| Phosphatidic acid | PA | `[M+H]+`; `[M+Na]+`; `[M+NH4]+` | `[M+H]+`: MS1; `[M+Na]+`: MS1; `[M+NH4]+`: MS2 | 3–60 | 6 | 337 |
| Phosphatidylcholine | PC | `[M+H]+`; `[M+Na]+` | all: MS2 | 2–60, 12–60 | 6 | 337, 292, 0 |
| Phosphatidylethanolamine | PE | `[M+H]+`; `[M+Na]+` | `[M+H]+`: MS2; `[M+Na]+`: MS1 | 2–60, 12–60 | 6 | 337, 292, 0 |
| Phosphatidylglycerol | PG | `[M+H]+` | MS1 | 2–60 | 6 | 337 |
| Phosphatidylinositol | PI | `[M+H]+`; `[M+Na]+`; `[M+NH4]+` | all: MS2 | 12–60 | 6 | 313 |
| Phosphatidylinositol phosphate | PIP | `[M+H]+`; `[M+Na]+` | all: MS1 | 2–60 | 6 | 337 |
| Phosphatidylinositol bisphosphate | PIP2 | `[M+H]+`; `[M+Na]+` | all: MS1 | 12–60 | 6 | 313 |
| Phosphatidylinositol trisphosphate | PIP3 | `[M+H]+`; `[M+Na]+` | all: MS1 | 12–60 | 6 | 313 |
| Phosphatidylserine | PS | `[M+H]+` | MS2 | 2–60 | 6 | 337 |
| Sphingosine-1-phosphate | S1P | `[M+H]+`; `[M+Na]+` | `[M+H]+`: MS2; `[M+Na]+`: MS1 | 12–30 | 4 | 5 |
| Sphingoid base | SB | `[M+H]+`; `[M+Na]+`; `[M+NH4]+` | `[M+H]+`: MS2; `[M+Na]+`: MS2; `[M+NH4]+`: MS1 | 2–34 | 6 | 0 |
| Sulfohexosylceramide | SHexCer | `[M+H]+`; `[M+Na]+` | all: MS1 | 12–60 | 6 | 133 |
| Sphingomyelin | SM | `[M+H]+`; `[M+Na]+` | all: MS2 | 28–60 | 6 | 103 |
| Sulfoquinovosyldiacylglycerol | SQDG | `[M+NH4]+`; `[M+Na]+` | all: MS1 | 12–60 | 6 | 313 |
| Sterol | ST | `[M+H]+`; `[M+H-H2O]+`; `[M+NH4]+`; `[M+Na]+` | all: MS2 | 2–34 | 6 | 91 |
| Sphingosine | Sph | `[M+H]+`; `[M-H2O+H]+`; `[M+Na]+`; `[M+K]+` | `[M+H]+`: MS2; `[M-H2O+H]+`: MS1; `[M+Na]+`: MS1; `[M+K]+`: MS1 | 2–60 | 6 | 5 |
| Triacylglycerol | TG | `[M+NH4]+`; `[M+Na]+`; `[M+H]+`; `[M+K]+`; `[M+Li]+` | all: MS2 | 14–60 | 12 | 461 |

## What the library covers

The internal library covers the major lipid classes implemented in LipidOracle:

- **Glycerolipids:** TG, DG, MG
- **Glycerophospholipids:** PC, PE, PI, PG, PA, PS, CL (cardiolipin), LPC, LPE, LPI, LPG, LPA, LPS
- **Sphingolipids:** SM, Cer, HexCer, LacCer
- **Sterols and sterol esters:** CE, Cholesterol
- **Fatty acids and derivatives:** FA

Each class is defined with its structural limits (carbon and double-bond ranges). These limits are fixed in the library code and do not change with configuration.

**Key coverage notes:**

- **Ether-linked species** are represented by their own distinct library keys. An ether-linked PC (`PC O-`) is a separate entry from a diacyl PC.
- **Cardiolipin (CL)** has more limited structural resolution than simpler lipid classes. It is typically identified but with less fine-grained positional information.
- The dashboard table is the authoritative list for any given run. If a class or adduct you expect to see is missing, it was not included in the version you ran.

Exact class and adduct combinations are defined by the library code and evolve with each version of LipidOracle.

## Library sources

| Source | Switch | Notes |
|---|---|---|
| Internal LipidOracle library | `WORKFLOW.lipidoracle` | Default `True`. The primary, fully supported path. Includes MS2. |
| LipiDex | `WORKFLOW.lipidex` | Default `False`. Optional comparative source, fewer features. |
| LipidBlast | `WORKFLOW.lipidblast` | Default `False`. Optional comparative source, fewer features. |
| Your own CSV | `WORKFLOW.library_extra` | Default empty. One or more CSV files. |

Two extra libraries ship inside the image and can be switched on by path:

```yaml
WORKFLOW:
  library_extra: ["/data/ST501.csv", "/data/AMP_PUFA.csv"]
```

- `/data/ST501.csv` is the steroid library taken from LipidBlast. No MS2.
- `/data/AMP_PUFA.csv` is the AMP-derivatized PUFA library. No MS2.

### Supplying your own CSV library

A custom CSV can be a plain MS1 list or can carry MS2 fragments. Recognised columns:

| Column | Meaning |
|---|---|
| `name` | Lipid name |
| `species` | Species identifier |
| `formula` | Molecular formula |
| `chains` | Number of acyl chains, defaults to 0 if omitted |
| `hg_class` | Headgroup class |
| `adduct` | Ion form, for example `[M+H]+` |
| `mod` | Modifications |
| `frag_mz`, `frag_w`, `frag_label`, `frag_type` | Optional MS2 fragment definition |

Point at it with an absolute path inside the container:

```yaml
WORKFLOW:
  library_extra: ["/input/my_library.csv"]
```

## Oxidation

The library can generate oxidised variants of each entry. Two separate limits control how far that goes, because MS1 and MS2 libraries are built independently:

| Parameter | Default | Controls |
|---|---|---|
| `PARAM.ms1_maxO` | `0` | Maximum additional oxygen atoms per lipid during MS1 library generation |
| `PARAM.ms2_maxO` | `0` | Maximum additional oxygen atoms per chain during MS2 isomer generation |

Both default to `0`, which means no oxidised variants are generated. Raise them if you are looking for oxidised lipids:

- `ms1_maxO: 1` includes mono-oxidised species in MS1 matching (one extra oxygen per lipid)
- `ms2_maxO: 2` allows up to two extra oxygen atoms per chain during MS2 analysis

**How oxidation limits work:**

- **MS1 generation:** the limit is per *lipid*. A TG with `ms1_maxO: 1` can have one oxygen across its three chains; setting `ms1_maxO: 2` allows two total oxygens across all chains.
- **MS2 generation:** the limit is per *chain*. Each acyl chain can carry up to `ms2_maxO` oxygens independently, subject to the class definition's oxygen limit. A lipid with two chains and `ms2_maxO: 2` can have up to 2 oxygens on chain 1 and 2 on chain 2.

Total library size and runtime grow substantially with oxidation limits. The oxidation policy actually applied in a run is printed on the dashboard's Lipid Library page, so you can verify exactly which variants were considered.

## Rarity

Not every formally valid lipid is equally likely to be real. LipidOracle assigns each library entry a **rarity index**, built as the sum of:

- Rarity of the headgroup (e.g. cardiolipin is rarer than phosphatidylcholine)
- Rarity of the chain lengths (very long chains, very short chains, or uneven chain pairs contribute)
- Modifications present, such as hydroxyl or epoxy groups

The resulting rarity is an integer. Examples:

- Rarity `0` is a common lipid: `PC 34:1` or `PE 36:2`
- Rarity `1` is uncommon: `PC 40:0` or `PE O-40:6`
- Rarity `2` or `3` marks genuinely rare lipids: `CL 72:8`, `Cer d18:1/24:0`, or unusual oxidised forms

You control rarity in two ways, and they behave very differently:

**A priori filtering: `PARAM.rarity_max` (default `3`).** Entries above the configured maximum are excluded from the library *before* matching begins. Rare lipids are never considered at all. Setting `rarity_max: 0` keeps every entry regardless of rarity; setting `rarity_max: 1` removes all entries with rarity 2+ from the library. This approach is stricter and more conservative. Use it when you want to exclude unlikely lipids entirely.

**A posteriori scoring: `PARAM.rarity_penalty` (default `0.15`).** Leaving `rarity_max` at 2 or 3 keeps rare forms in the library, but the penalty lowers their matching score. A rare lipid survives into the annotation only when its `score_L2` is high enough to overcome the penalty. This approach lets the data decide: the evidence is there, but weak matches on rare lipids are easier to filter out. This is usually the better choice: you keep all evidence and let score protect against noise rather than deciding in advance which lipids are worth considering.

Both parameters are independent. You can use both (keep all entries in the library with `rarity_max: 0`, but apply a penalty with `rarity_penalty`), or just one.

## Where the library shows up in the output

Every annotation in `annotation.csv` carries two columns that record its library source:

- **`lib_src`**: which library this match came from (e.g. `lipidoracle`, `lipidex`, `lipidblast`, or the name of your custom CSV)
- **`lib_id`**: the identifier or row index of the matched library entry

The diagnostic output includes two full library snapshots:

- **`diag/lib_idlevel1.csv`**: the exact MS1 library actually used for precursor matching in this run
- **`diag/lib_idlevel2.csv`**: the exact MS2 library with fragments used for MS2 scoring

These files are the fastest way to check whether a species you expected was in scope at all. If it does not appear in the snapshot, it was not considered for this run — either excluded by rarity filtering, outside the carbon/double-bond limits, not present in any active library source, or built for a different ion polarity.

## Related

- [Parameters](PARAMETERS.html) for the full configuration reference
- [Retention-time consistency](RT_CHECK.html) for the filter applied after matching
- [C=C and oxidation by EAD](EAD.html) for what happens to matched lipids afterwards

## Questions and problems

Report issues or ask questions at [github.com/zamboni-lab/lipidoracle/issues](https://github.com/zamboni-lab/lipidoracle/issues).
