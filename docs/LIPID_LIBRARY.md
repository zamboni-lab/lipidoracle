---
title: The built-in MS2 library
---

# The built-in MS2 library

LipidOracle ships with an internal lipid library that is the primary source for both MS1 and MS2 annotation. LipiDex, LipidBlast, and any libraries you supply through `WORKFLOW.library_extra` are optional additional sources.

Every run records which libraries were actually active. Open `diag/dashboard.html` and go to **Documentation > Lipid Library** to see the exact class and adduct coverage for that run, generated from the library definition in the version you ran. This page describes how the library is built and how to control it; the dashboard tells you what a specific run used.

## What the library covers

The internal library covers the major lipid classes implemented in LipidOracle. For each class it defines:

- the class name and abbreviation
- the relevant adducts, and the highest annotation level reachable for each
- carbon-number and double-bond limits
- the number of unique species generated under the current settings

Two coverage notes worth knowing before you interpret results:

- **Ether-linked species** are represented by their own distinct library keys, not as a modifier on the diacyl entry.
- **CL** has more limited structural resolution than the simpler classes.

Exact class and adduct combinations are defined by the library code and evolve with it. The dashboard table is the authoritative list for any given run.

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

The library can generate oxidised variants of its entries. Two separate limits control how far that goes, because the MS1 and MS2 libraries are built independently:

| Parameter | Default | Controls |
|---|---|---|
| `PARAM.ms1_maxO` | `0` | Extra side-chain oxygens when building the MS1 library |
| `PARAM.ms2_maxO` | `0` | Extra side-chain oxygens during MS2 isomer generation |

Both default to `0`, which means no oxidised variants are generated at all. Raise them if you are looking for oxidised lipids, and expect the library and the runtime to grow accordingly. The oxidation policy actually applied in a run is printed on the dashboard's Lipid Library page.

## Rarity

Not every formally valid lipid is equally likely to be real. LipidOracle scores each library entry with a **rarity index**, summed from the rarity of the headgroup, the chain lengths (very long, very short, or odd-numbered chains all contribute), and the modifications present such as hydroxyl or epoxy groups.

- Rarity `0` is a common lipid, for example `PC 34:1`.
- Rarity `2` or `3` marks lipids that are genuinely rare.

You can act on rarity in two different ways, and they behave very differently.

**Filter before matching**, with `PARAM.rarity_max` (default `3`). Entries above the configured maximum are excluded from the library entirely, so rare lipids are never considered. Setting it to `0` keeps everything. Lowering it makes the run stricter and more conservative.

**Penalise during scoring**, with `PARAM.rarity_penalty` (default `0.15`). Leaving `rarity_max` at 2 or 3 keeps rare forms in play, and the penalty lowers their score instead of removing them. A rare lipid then survives into the annotation only when its `score_L2` is high enough to overcome the penalty. This is usually the better choice: you keep the evidence and let the data decide, rather than deciding in advance.

## Where the library shows up in the output

The `lib_src` and `lib_id` columns in `annotation.csv` record which library an annotation came from and which entry matched. `diag/lib_idlevel1.csv` and `diag/lib_idlevel2.csv` contain the library snapshots actually used for MS1 and MS2 matching in that run, which is the fastest way to check whether a species you expected was in scope at all.

## Related

- [Parameters](PARAMETERS.html) for the full configuration reference
- [Retention-time consistency](RT_CHECK.html) for the filter applied after matching
- [C=C and oxidation by EAD](EAD.html) for what happens to matched lipids afterwards

## Questions and problems

Report issues or ask questions at [github.com/zamboni-lab/lipidoracle/issues](https://github.com/zamboni-lab/lipidoracle/issues).
