---
name: lipidoracle
description: Run LipidOracle to annotate lipids from LC-MS/MS data. Use whenever the task involves lipid annotation, lipidomics identification, an .mgf file of MS1/MS2 spectra, locating C=C or oxidation positions in acyl chains (EAD/UVPD/OAD/OzID), retention-time filtering of lipid IDs, or reading LipidOracle's annotation.csv / summary.csv / dashboard.html outputs. Also use when the user mentions "lipidoracle", "zambonilab/lipidoracle", or asks to configure lipidoracle.yaml.
---

# LipidOracle

LipidOracle annotates lipids from MS1 and MS2 spectra: accurate-mass matching,
MS2 fragment scoring, retention-time validation, and — on EAD, UVPD, OAD, or
OzID spectra — C=C and oxidation positions within acyl chains.

A run has three **stages** (`s1` MS1, `s2` MS2, `s3` acyl-chain positions) and
parameters are named after the stage that owns them (`s2_lo_score_cutoff`,
`s3_min_score`). Stages are steps; **idlevel** (1–4) is a different axis
describing how much detail a result carries. `ms1_*`/`ms2_*` parameters refer
to the *MS level* of the data, not a stage.

It is distributed **only as a Docker image**. There is no pip package, no
standalone binary, and no public source build. Every run happens through
`docker run`.

## The whole workflow

LipidOracle takes **one input folder** containing a single `.mgf` file (MS1 and
MS2 scans together) and **one output folder**. It is a batch tool: one command,
no interactive prompts, no network calls.

### 0. Getting the MGF

LipidOracle does not read vendor formats or mzML, and does not do peak picking.
Feature detection and MGF export happen upstream, in **MASSter** or **MZmine**.

**MASSter** (Zamboni lab, same authors — the smoothest path, and it can import
the results back):

```python
import masster

sample = masster.Sample("data.mzML")   # or a vendor file, on Windows
sample.find_features()
sample.find_ms2()
sample.export_mgf("data.mgf")
```

Install with
`uv pip install --index https://zamboni-lab.github.io/masster-dist/simple masster`.
For a multi-sample experiment, run the `Study` workflow (`align()` → `merge()`)
and export the consensus spectra with `study.export_mgf("consensus.mgf")`.

Useful `export_mgf()` arguments: `selection="best"` for one representative
spectrum per feature, `selection="all", merge=True` to build consensus spectra,
plus `clean`, `deisotope`, `inty_min`, and `precursor_trim` to control which
peaks survive. Those choices materially change annotation performance — export
settings are part of the LipidOracle tuning surface, not a preprocessing detail.

MASSter's agent guide:
<https://github.com/zamboni-lab/masster-dist/blob/main/docs/agent/masster-agent-guide.md>

**MZmine** also works: process to a feature list and export MS1 and MS2 to a
single MGF.

### The MGF dialect LipidOracle expects

Both MS1 and MS2 scans go in **one file**, distinguished by `MSLEVEL`:

```
BEGIN IONS
TITLE=fid:76, rt:24.24, mz:103.6902
FEATURE_ID=76
CHARGE=0
PEPMASS=103.69015060912331
RTINSECONDS=24.237
MSLEVEL=1
100.07555 7479
...
END IONS
```

- `FEATURE_ID` groups the MS1 and MS2 scans of one precursor; it becomes the
  `mgf_feat` column in the output and is what lets you join results back to the
  upstream feature table. Without it, that link is lost.
- `MSLEVEL` — **absent means 2**. An MGF with no `MSLEVEL` lines is read as
  MS2-only, and the MS1 annotation stage silently finds nothing.
- `RTINSECONDS` — seconds, not minutes. MASSter's own tables use minutes, so
  don't hand-convert; let `export_mgf()` do it.
- `CHARGE` drives polarity inference. Override with `--polarity` if it's absent
  or unreliable.
- MS2 blocks with no peaks are skipped.

The two datasets in `testdata/` are MASSter exports and are the reference for
what a well-formed input looks like.

### Getting results back into MASSter

The round trip is supported — after a LipidOracle run:

```python
study.import_oracle("path/to/oracle_output")
```

Then `study.get_id()`, `study.id_select(score=(0.8, 1.0))`, and
`study.id_filter(...)` work on the annotations alongside MASSter's own.

### 1. Pull the image

```bash
docker pull zambonilab/lipidoracle
```

Version 1.0+ is a Rust rewrite. The old Python implementation is
`zambonilab/lipidoracle:v0.9` and is **not** backward compatible — do not mix
its configs with 1.0.

### 2. First run — generates the config, then stops

```bash
docker run --rm \
  -v /abs/path/to/input:/input \
  -v /abs/path/to/output:/output \
  zambonilab/lipidoracle
```

If no `lipidoracle.yaml` exists in the input or output folder, LipidOracle
writes a fully commented default to `<output>/lipidoracle.yaml` and **exits
without annotating anything**. This is expected behaviour, not an error.

Read that file, edit it for the dataset, then re-run the identical command.

### 3. Second run — the real annotation

Same command. Now that `<output>/lipidoracle.yaml` exists, the pipeline runs:
load spectra → build libraries → match precursor m/z (stage 1, idlevel 1) →
score MS2 fragments (stage 2, idlevel 2) → retention-time filter → optional
acyl-chain analysis (stage 3, idlevel 3) → consolidate isomers (idlevel 4) →
write CSVs and dashboard.

CID datasets finish in seconds. EAD datasets take up to a few minutes.

## Rules for driving it

- **Mount absolute paths.** Relative paths silently create empty Docker volumes.
  On Windows use `-v D:/data/input:/input` (forward slashes) or `${PWD}` in
  PowerShell.
- **Exactly one `.mgf` in the input folder.** The container auto-detects
  `/input/*.mgf`; several files make the choice ambiguous.
- **Never edit config inside the container.** Edit
  `<output>/lipidoracle.yaml` on the host — it is bind-mounted, so the next run
  picks it up.
- **A first run that only prints "Created default parameters" is success.**
  Report it as "config generated, review then re-run", not as a failure.
- **Don't hand-write `lipidoracle.yaml` from scratch.** Let the first run emit
  it and edit the keys you need. Unknown keys are rejected, and the emitted file
  carries the comments explaining every setting.
- **Output folder gets a copy of the input `.mgf` and the used
  `lipidoracle.yaml`,** so each output folder is a self-contained record. Use a
  fresh output folder per run rather than overwriting.

### Passing extra CLI flags

The entrypoint forwards arguments to the binary. Useful ones:

```bash
docker run --rm -v /abs/in:/input -v /abs/out:/output \
  zambonilab/lipidoracle --polarity neg --quiet
```

| Flag | Meaning |
| --- | --- |
| `-p`, `--params <file>` | Use a specific YAML instead of the auto-discovered one. Path must be inside a mounted volume. Supplying it also skips the "write default and stop" step. |
| `--polarity pos\|neg` | Force polarity. Default: inferred from the MGF `CHARGE` field. |
| `-d`, `--debug` | Verbose diagnostic logging. |
| `-q`, `--quiet` | Suppress progress messages. |
| `--no-dashboard-single-file` | Skip the portable single-file dashboard export. |

`-i`/`--input` and `-o`/`--output` are already set by the entrypoint; don't pass
them.

## Configuring a run

The decisions that actually matter live under `WORKFLOW:`. Everything else in
the file has a sensible default.

```yaml
WORKFLOW:
  lipidoracle: True      # internal MS1/MS2 engine — keep on
  stage3: False           # False | ead1 | ead2 | uvpd | oad | ozid
  rt_check: liri         # False | liri | hydra | legacy
  library_extra: [ ]     # optional extra CSV libraries
  is_peaks: ''            # optional internal-standard CSV
```

The complete annotated parameter file, with every key and its default, is in
[reference/lipidoracle.default.yaml](reference/lipidoracle.default.yaml). Read
it before changing anything outside `WORKFLOW:`.

### Choosing the `stage3` engine

Leave `False` for CID data; every engine needs a position-sensitive
dissociation method.

| Value | Use for |
| --- | --- |
| `ead1` | Chain-ladder engine for double bonds. No oxygen support at all — `;O` species are dropped. |
| `ead2` | Exact-formula cleavage engine; required for anything oxidised (`;O`). |
| `uvpd` | 193/213 nm photodissociation data. |
| `oad` | Unoxidised diacyl PC `[M+H]+` only (MS-DIAL OAD rules). Other classes, adducts, oxidised lipids, ether chains, and sphingoid bases are skipped, not approximated. |
| `ozid` | Aldehyde/Criegee diagnostic pairs on the validated PC/PE `[M+H]+`, PI `[M+Na]+`, and DG/TG `[M+NH4]+` profiles. Everything else fails closed. |

With `ead2` on oxidised species, also raise `PARAM.s3_max_candidates` to
30000 — see below.

### Choosing the RT model

`rt_check` selects at most one retention-time engine, run after MS1/MS2
matching:

| Value | Behaviour |
| --- | --- |
| `liri` (default) | Fits a per-class retention index from this run's own confident MS2 hits. Safe, but leaves classes with no MS2 evidence (e.g. idlevel-1 FA/CE hits) unfiltered. |
| `hydra` | Cross-class hydrophobicity model sharing information via headgroup fragments; can also flag idlevel-1 hits with zero MS2 confirmation. Stricter, needs tuning. |
| `legacy` | The original two-regression model (generic formula + per-class), always run together. |
| `False` | No RT filtering. |

Every engine works from this run's own annotations alone. Point `rt_ref` at a
CSV of known retention times (columns `hg`, `c`, `db`, `mod`, `rt`) to sharpen
the fit further — it is optional; an example lives at `/data/rt_ref_2min.csv`
in the image. A generic formula-based check runs afterward regardless of
engine, governed by `PARAM.rt_deviation_cutoff` (fraction of the RT axis when
≤ 1, seconds when > 1). If you get too many false positives, tighten
`rt_check` (try `hydra`) or lower `rt_deviation_cutoff`; if you lose true
positives, relax it or drop to `liri` or `False`.

### Adding libraries

`WORKFLOW.library_extra` takes a list of extra CSV (or CSV.GZ) libraries
merged into the search, as absolute container paths or local relative paths:

```yaml
WORKFLOW:
  library_extra: [ '/data/my_library.csv' ]
```

A custom CSV can be a plain MS1 list or carry MS2 fragments. Recognised
columns: `name`, `species`, `formula`, `chains` (defaults to 0), `hg_class`,
`adduct`, `mod`, and the optional fragment set `frag_mz`, `frag_w`,
`frag_label`, `frag_type`.

For internal standards, point `WORKFLOW.is_peaks` at a CSV instead — the image
ships `/data/equisplash.csv`, `/data/lightsplash.csv`, and
`/data/ultimatesplash.csv` (the EquiSPLASH, LightSPLASH, and UltimateSPLASH
mixes) ready to use directly; a custom file needs columns `name`, `mass`, `c`,
`db`, `mod`, `hg`. Leave `is_peaks` `''` to skip IS matching.

### Matching tolerances

The core knobs, all under `PARAM:`:

| Parameter | Default | Controls |
| --- | --- | --- |
| `ms1_tolerance` | 0.01 Da | Precursor m/z window for candidate lookup. |
| `ms2_tolerance` | 0.02 Da | Fragment m/z window for MS2 peak matching. |
| `ms1_ppm_penalty` | 0.03 | Score penalty per absolute ppm of MS1 mass error. |
| `s1_lo_score_cutoff` / `s1_score_top_cutoff` | 0.2 / 0.5 | Absolute and relative MS1 score floors. |
| `s2_lo_score_cutoff` / `s2_score_top_cutoff` | 0.2 / 0.9 | Absolute and relative MS2 score floors. |
| `rarity_max` | 3 | Excludes library entries above this rarity index before matching. `0` keeps only the most common species. |
| `rarity_penalty` | 0.15 | Score penalty per rarity unit, applied instead of excluding rare entries outright. |

Widen tolerances or lower score cutoffs to recover more candidates; tighten
them to cut false positives. Change one parameter at a time and diff the
output before changing another.

### Stage-3 parameters (`s3_*`)

Two of these are worth knowing before a run rather than after.

**`s3_max_candidates`** (10000) caps enumeration per feature; beyond it the
candidate list is stride-sampled. A single oxidised token such as `20:4;O`
enumerates **~22,300** candidates, so on oxidised work the default silently
discards the true structure *before scoring ever runs*. The output is
indistinguishable from a scoring failure. **Raise it to 30000 whenever
`stage3: ead2` is set on `;O` species.**

**`s3_selection`** (`posterior`) decides which positional candidates get
reported and when idlevel 4 commits to a position. `posterior` keeps the
smallest candidate set holding 95% of the posterior mass and names a position
only when one placement carries `s3_post_min_mass` (0.5). `legacy` is the
relative-score cutoff — it names more positions and is right less often
(measured: 84/96 features asserted at 65% correct, versus 55/96 at 73%).

`s3_post_temperature` and `s3_min_margin` are **fitted against reference sets
with known positions**, not chosen by taste. Do not tune them to make output
look better; changing them invalidates the calibration. OAD and OzID reuse the
same EAD-fitted temperature and remain provisional until a larger known-position
panel calibrates them separately.

## Reading the results

Start with `dashboard.html` — a self-contained page with the run summary, an
RT/mz scatter, and clickable MS2 match panels. It renders offline except for
three CDN libraries (Plotly, Tabulator, RDKit).

Then `summary.csv` (key-value run statistics: feature counts per idlevel,
average scores) and `annotation.csv` (one row per accepted annotation).

Identification levels encode how much evidence backs each call:

| idlevel | Evidence | Detail reached |
| --- | --- | --- |
| 1 | Precursor m/z only | Shorthand name: class, total carbons, total double bonds |
| 2 | MS2 fragments matched, scored by `score_s2` | Class confirmed, often individual acyl chains |
| 3 | EAD/UVPD/OAD/OzID isomer scoring, `score_s3` | C=C and oxidation positions along chains |
| 4 | Consolidation | One row per spectrum merging all idlevel-3 candidates, with a confidence tail |

`score_s3` is normalised against each spectrum's own best candidate, so **the top
candidate always reads 1.0** whether the evidence is decisive or absent. It ranks
within a spectrum; it does not measure confidence, and it is not comparable
across engines — equal `score_s3` values from different stage-3 engines are not
equivalent evidence. What separates a determined position from an undetermined
one is whether idlevel 4 named it at all.

Names are strict **Shorthand2020** (Liebisch et al.): modifications carry their
position in front of the abbreviation, `;5OH` not `;OH(5)`. A `/` between chains
means sn-positions are known, `_` means they are not.

Ambiguity is reported, not resolved. Candidate structures are extended CXSMILES:
`Sg:` marks a chain run holding an unlocalised double bond, `m:` a modification
that could sit on any of several carbons, and `$snN$` labels plus a
`swappable(...)` token an unresolved sn-assignment. **A fully determined
structure has no `|...|` tail at all**, so the presence of a tail is itself the
signal that something was left open.

At idlevel 4 the `name` also carries a bracketed confidence tail —
`FA 18:2 [DB sn1: Δ9 92%, Δ12 88%]`. Strip it before parsing the name; the part
before ` [` is the strict Shorthand2020 core.

**Do not report a species-level idlevel-4 name as a failure.** LipidOracle
writes positions into the name only when the evidence supports one placement,
so `FA 20:4` at idlevel 4 means the positions were *not determined* — which is
a result, not a missing one. `diag/s3_resolvability.csv` says how uncertain it
was.

Column-by-column output reference: [reference/outputs.md](reference/outputs.md).

## Trying it without your own data

The repository ships two ready-to-run datasets under `testdata/`:

| Folder | Spectra | Notes |
| --- | ---: | --- |
| `testdata/cid/` | 9,948 | CID run, annotation to idlevel 1–2. Seconds. |
| `testdata/ead/` | 38,255 | EAD run. Set `stage3: ead1` to localise C=C. |

## Troubleshooting

| Symptom | Cause |
| --- | --- |
| Runs and exits immediately, only writes `lipidoracle.yaml` | Expected first-run behaviour. Re-run the same command. |
| "Cannot find input MGF" | No `.mgf` under `/input`, or the volume mount used a relative path. |
| `summary.csv` reports 0 MS1 features | The MGF has no `MSLEVEL=1` blocks, or no `MSLEVEL` at all (defaults to 2). Re-export from MASSter/MZmine including MS1. |
| All RT values look ~60x too small | The MGF used minutes in `RTINSECONDS`. Re-export rather than patching the file. |
| Results can't be joined to the upstream feature table | The MGF was exported without `FEATURE_ID`, so `mgf_feat` is empty. |
| Config changes have no effect | Edited a file outside the mounted output folder, or a stray `*.yaml` in the output folder is being picked up first. |
| `WORKFLOW.stage3: unrecognized value` | Use `False`, `ead1`/`v1`, `ead2`/`v2`, `uvpd`, `oad`, or `ozid`. |
| No idlevel 3 rows | `stage3` is `False`, or the data is CID and carries no EAD/UVPD/OAD/OzID fragments. |
| idlevel 3 has candidates but idlevel 4 names no positions | Expected under `s3_selection: posterior` when the spectrum did not determine one. Check `top_prob` and `cred95` in `diag/s3_resolvability.csv` before changing anything. |
| Oxidised lipids localise poorly | `stage3: ead1` ignores `;O` species entirely — use `ead2`. If already on `ead2`, `s3_max_candidates` is likely too low; raise it to 30000. |
| Too many IDs removed | `rt_check: hydra` is strict; try `liri`, or `False` to see the unfiltered set. |
| Dashboard plots are blank | Offline machine — the CDN libraries could not load. The CSVs are unaffected. |
| `oad`/`ozid` finds nothing on data you expect it to handle | Both fail closed outside their validated profile: `oad` is unoxidised diacyl PC `[M+H]+` only; `ozid` is PC/PE `[M+H]+`, PI `[M+Na]+`, DG/TG `[M+NH4]+` only. Other classes, adducts, oxidised lipids, ether chains, and sphingoid bases are skipped, not approximated. |

## Reference

Documentation site: <https://zamboni-lab.github.io/lipidoracle/>

| Page | Covers |
| --- | --- |
| [params](https://zamboni-lab.github.io/lipidoracle/params.html) | every `lipidoracle.yaml` setting |
| [stage3](https://zamboni-lab.github.io/lipidoracle/stage3.html) | the EAD/UVPD/OAD/OzID engines, candidate selection, every `s3_*` parameter |
| [nomenclature](https://zamboni-lab.github.io/lipidoracle/nomenclature.html) | Shorthand2020, CXSMILES ambiguity encoding, the confidence tail |
| [rt](https://zamboni-lab.github.io/lipidoracle/rt.html) | the retention-time engines |
| [library](https://zamboni-lab.github.io/lipidoracle/library.html) | class coverage, extra libraries, rarity |

What changed in the current release: [WHATSNEW.md](../../../WHATSNEW.md).

Method: Wu et al., *Analyst*, 2025, [10.1039/d5an00567a](https://doi.org/10.1039/d5an00567a)

Licensed PolyForm Noncommercial 1.0.0 — non-commercial use only, including
educational and public research organisations. Commercial use needs separate
permission from the Zamboni lab, ETH Zurich.
