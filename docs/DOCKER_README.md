---
title: Installation and usage
---

# LipidOracle

This is the annotation engine for lipids by MS1/MS2 data developed by the Zamboni lab at ETH Zurich. It includes libraries for CID, EAD, and UVPD, C=C position and oxidation identification in acyl chains, RT checking, export of CXSMILES structure generation for annotated lipids, etc. 

<b>With v1.0, we fully redesigned and rewrote the codebase in Rust.</b> This led to a massive increase in performance and an expansion of functionalities.

<b>We don't guarantee backward compatibility with the old Python-based version, whicch remains available at zambonilab/lipidoracle:v0.9</b>

Reference: Wu et al, Optimisation of electron-induced dissociation parameters for molecular annotation of glycerides and phospholipids in fast LC-MS, Analyst, 2025, DOI: [10.1039/d5an00567a](https://doi.org/10.1039/d5an00567a)

## Installation
LipidOracle runs as a Docker container. To install Docker, follow [Docker Desktop installation](https://www.docker.com/products/docker-desktop/). Pull the image from Docker Hub:

```bash
docker pull zambonilab/lipidoracle
```

## Input data

MS2 and optionally MS1 data must be provided in a single `.mgf` file located in a local folder (`<INPUT-FOLDER>`).

The processing steps and key parameters are defined in a YAML file named `lipidoracle.yaml` in the output folder (`<OUTPUT-FOLDER>`). If missing in the designated output and input folders, a default file is created automatically in the output folder and the program stops so you can review and rerun.

## Description of the workflow

LipidOracle performs annotation of lipids analyzed by tandem MS using CID, EAD, or UVPD. Workflow order:

1. Load MS2 and MS1 data from an MGF file.
   - Targets: A list of m/z targets can be provided to reduce the number of features to be processed. This is useful for focused validation runs and troubleshooting.
   - Internal standards: internal standards (incl. deuterated forms) can be included to ensure that they are found regardless of libraries.
2. Library generation: Creates lookup libraries for identifying precursors. It supports the internal LipidOracle library, LipiDex, and LipidBlast. Additional libraries can be provided as *.csv. These are specified in `lipidoracle.yaml`. Libraries can contain MS2 data.
3. MS1 matching: Identify features by accurate mass using the available libraries. This step produces `idlevel 1` matches.
4. MS2 matching: For all `idlevel 1` matches that also provide MS2 data, proceed to MS2 matching. This step produces `idlevel 2` matches.
5. Retention-time validation based on a class-specific model. Two model exists (liri and hydra), plut the legacy method ported from the Python codebase.
   - External RT anchors can be provided to refine training (optional).
6. Acyl chain analysis (requires EAD or UVPD spectra, `WORKFLOW.acyl_analysis`). If enabled, it will attempt to identify C=C and oxidation position in acyl chains. The result are `idlevel 3` matches, which focuses on the similarity of simulated acyl chain MS2 spectra for individual isomers. The results are aggregated into `idlevel 4` matches - one per spectrum.
    - The maximum number of isomers tested is 10'000. 
8. Export results, diagnostic files, and a compact `dashboard.html` to visualize key results.


## How to run

1. Create a local input folder (`<INPUT-FOLDER>`) and place the `.mgf` file there. Optionally, you can also add a previous `lipidoracle.yaml`. 
2. Run:
```bash
docker run --rm -v <INPUT-FOLDER>:/input -v <OUTPUT-FOLDER>:/output  zambonilab/lipidoracle
```
3. If no params file is found in the in put or output folder, a default `lipidoracle.yaml` is created and the container exits. It will stop and ask to review the file.
4. If needed, edit the `.yaml` file to adjust the workflow to your needs. Or simply run it and have a look at the results.
5. Run again.

## Parameters and defaults

The default `lipidoracle.yaml` is created when nothing is provided.

### Important paraneters of the YAML file

```yaml
VERSION: x.y.z

WORKFLOW:
  mz_target: ''           # allows to focus on specific m/z values
  lipidoracle: True       # main switch for the internal MS1/MS2 annotation engine
  lipidex: False          # included LipiDex (for testing, less functionalities)      
  lipidblast: False       # includes LIpidBlast (for testing, less functionalities)
  library_extra: [ ]      # specified additional libraries in csv format
  acyl_analysis: False    # run EAD/UVPD analysis of acyl chains
  rt_check: liri          # False | 'legacy' | 'liri' (default) | 'hydra' -- see docs/RT_CHECK.md

PARAM:
  rarity_max: 3                                 # max rarity index kept (lower = stricter, more conservative IDs)
  ...
  ms2_maxO: 0                                   # max extra side-chain oxygens in MS2 isomer generation
  ms1_maxO: 0                                   # max extra side-chain oxygens in MS1 library generation
  ...
  ms1_tolerance: 0.01                           # precursor m/z tolerance (Da)
  ...
  ms2_noise_abs: 10                             # absolute MS2 peak intensity floor
  ...
  ead_max_unsaturated: 6                        # max double bonds accepted for exhaustive EAD/UVPD position search
  ead_max_oxygen: 2                             # max oxygen sites accepted for exhaustive EAD/UVPD position search
  ead_max_candidates: 30000                     # cap on enumerated (db, oh/ket) candidates per feature
  ...
  ead_chain_cutoff: 0.98                        # relative EAD/UVPD score cutoff vs top candidate
```

## Retention time

### RT Reference data
To reduce spurious identifications, we recommend building a retention time library for each gradient on the basis of standards or manually validated IDs. It can be indicated in the `lipidoracle.yml` as `rt_ref: '/output/rt_ref_2min.csv'`

The CSV file must have these columns:

- `hg`: lipid class
- `c`: total number of carbons
- `db`: total number of double bonds
- `mod`: modifications separated by spaces
- `rt`: retention time in seconds

Example:

```csv
hg,c,db,mod,rt
SM,30,0,O2,53.46
SM,40,0,O2,90.3
SM,41,0,O2,93
SM,42,0,O2,96.54
SM,32,0,O2,61.5
```

### RT checks

`WORKFLOW.rt_check` selects exactly one RT-plausibility engine, applied after MS1/MS2 matching:

```yaml
WORKFLOW:
  rt_check: liri   # False | 'legacy' | 'liri' (default) | 'hydra'
```

- **`liri`** (default) — builds a retention index (`h = C - a_db*DB - a_o*O`) and fits a monotone `h -> RT`
  map independently per `(class, ether)` group, using only that group's own anchors. Safe (never wrong for
  lack of evidence) but silent on classes with too few anchors of their own. In other words, it won't filter
  MS1-only hits of classes that are rare or for which there aren't good MS2 libraries like FAs, and CEs.
- **`hydra`** — shares one calibration curve across classes via a chemistry-informed feature vector, so it can
  flag implausible hits even in classes with zero anchors of their own, at the cost of being more aggressive
  (particularly on idlevel-1 hits).
- **`legacy`** — the original two linear regressions (a pooled "generic" formula-based fit and a per-class
  fit). Kept for rollback/comparison; measured removal precision is close to a coin flip and is superseded by
  `liri` on every metric measured.
- **`False`** — disables RT checking entirely.

## LipidOracle outputs

### Identification Levels
The output reports levels extended to include EAD/UVPD analysis of acyl chains:
- **idlevel 1**: identification based on m/z
- **idlevel 2**: identification based on MS2 matching
- **idlevel 3**: feature analyzed for double-bond position in acyl chains
- **idlevel 4**: aggregate of idlevel 3 candidates reporting consolidated C=C and OH information

### Scores
- `score_L2` is an hybrid score that we use for idlevel 2 matching, which combines two elements: (i) how well the weighted MS2 library was matched, (ii) how well the the spectra correlate.
- `score_L2_modcos` it's a modified cosine score. We don't use for ranking, but it's reported for information.
- `score_L3` score of C=C/OH analysis for EAD spectra. The score differs betwen ead_v1 (for Lipids, no oxidation) and ead_v2 (for oxidized PUFAs).

### Evidence
- For **idlevel 1** rows, MS2-specific fields are often empty.
- For **idlevel 2** rows, `score_L2`, `score_L2_modcos`, `ms2_matched*`, and `ms2_missed*` are usually the main diagnostic fields.
- For **idlevel 3/4** rows, `score_L3`, `score_L3_data`, and `ms2_evidence` become important.

### Files

Primary files:

- `annotation.csv`
- `summary.csv`

Useful files in `diag/` include:

- `dashboard.html`
- `annotation_full.csv`
- `summary_by_feature.csv`

Additional files for tracking processing in `diag/` include:

- `input_spectra.csv`
- `annotation_idlevel1.csv`
- `annotation_idlevel2.csv`
- `annotation_idlevel3.csv`
- `annotation_idlevel4.csv`
- `lib_idlevel1.csv`
- `lib_idlevel2.csv`
- `lib_idlevel3.csv`
- `rt_liri.csv`

## The internal LipidOracle MS2 library

The internal library is designed to be broad and practical for routine lipidomics annotation. It includes the internal LipidOracle library plus optional additional sources such as LipiDex, LipidBlast, and user-provided extra CSV libraries. The dashboard's Documentation tab lists the libraries actually used in a given run.

### Rarity

To differentiate common and rare lipids, LipidOracle uses a rarity score.

Common chains such as `16:0`, `18:1`, and `20:4` have low or zero rarity contribution. Uncommon chains increase the score. The rarity of a lipid is the combined rarity across its chain composition and contributes a configurable penalty.

### Classes

The internal library covers the major lipid classes used in the Rust implementation for MS1/MS2 matching. Exact supported class/adduct combinations are implementation-defined and may evolve with the library code. For practical use, the most relevant point is that:

- internal LipidOracle matching is the primary fully supported path
- LipiDex and LipidBlast remain available as optional comparative sources
- CL species remain restricted in structural resolution relative to simpler classes

### Acyl chains and oxidation

The internal library supports a wide range of chains, including oxidation according to the configured oxygen limits:

- `PARAM.ms2_maxO` for MS2 library matching
- `PARAM.ms1_maxO` for MS1 library matching

## Parallel processing of multiple files

Although the internal data structures are optimized, the workflow still performs substantial on-the-fly work for candidate generation, scoring, and reporting.

The Rust implementation does use internal parallelism through `PARAM.ncores` for selected work such as MS2 scoring and EAD analysis. At the same time, running multiple files as separate container jobs can still be a useful operational strategy when processing batches.

For example, you can launch multiple `docker run --rm ...` commands in parallel from a parent workflow if your machine has enough CPU and memory headroom.

## More documentation

- [Parameters](PARAMETERS.html) covers every setting in `lipidoracle.yaml`.
- [The built-in MS2 library](LIPID_LIBRARY.html) covers class coverage, extra libraries, oxidation, and rarity.
- [Retention-time consistency](RT_CHECK.html) covers the three RT engines and which to use.
- [Locating C=C and oxidation by EAD](EAD.html) covers acyl-chain analysis.

## Questions and problems

Report issues or ask questions at [github.com/zamboni-lab/lipidoracle/issues](https://github.com/zamboni-lab/lipidoracle/issues).
