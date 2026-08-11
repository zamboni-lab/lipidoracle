---
name: lipidoracle
description: Run LipidOracle to annotate lipids from LC-MS/MS data. Use whenever the task involves lipid annotation, lipidomics identification, an .mgf file of MS1/MS2 spectra, locating C=C or oxidation positions in acyl chains (EAD/UVPD), retention-time filtering of lipid IDs, or reading LipidOracle's annotation.csv / summary.csv / dashboard.html outputs. Also use when the user mentions "lipidoracle", "zambonilab/lipidoracle", or asks to configure lipidoracle.yaml.
---

# LipidOracle

LipidOracle annotates lipids from MS1 and MS2 spectra: accurate-mass matching,
MS2 fragment scoring, retention-time validation, and — on EAD or UVPD spectra —
C=C and oxidation positions within acyl chains.

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
load spectra → build libraries → match precursor m/z (idlevel 1) → score MS2
fragments (idlevel 2) → retention-time filter → optional acyl-chain analysis
(idlevel 3) → consolidate isomers (idlevel 4) → write CSVs and dashboard.

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

The four decisions that actually matter live under `WORKFLOW:`. Everything else
in the file has a sensible default.

```yaml
WORKFLOW:
  lipidoracle: True      # internal MS1/MS2 engine — keep on
  acyl_analysis: False   # False | ead1 | ead2 | uvpd — C=C / oxidation localisation
  rt_check: liri         # False | liri | hydra | legacy
  library_extra: [ ]     # optional extra CSV libraries
```

**`acyl_analysis`** — leave `False` for CID data; it needs EAD or UVPD spectra.
Set `ead1` for the chain-ladder engine (double bonds, no oxygen support at all —
`;O` species are dropped), `ead2` for anything oxidised, or `uvpd` for 193/213 nm
photodissociation data. Accepts `v1`/`v2` as aliases. With `ead2`, also raise
`PARAM.l3_max_candidates` to 30000 — see the caution below.

**`rt_check`** — `liri` (default) fits a per-class retention index from the
run's own confident MS2 hits; safe but leaves classes without MS2 evidence
unfiltered. `hydra` predicts across classes from chemistry features and can
flag classes with zero MS2 confirmation, but is stricter and needs tuning.
`legacy` exists only for rollback comparison. Both engines are followed by a
generic RT filter that removes in-source fragments.

If you miss true positives, relax thresholds gradually; if you get too many
false positives, tighten the score and RT cutoffs. Change one group at a time
and diff the outputs.

### Acyl-chain parameters (`l3_*`)

Two of these are worth knowing before a run rather than after.

**`l3_max_candidates`** (10000) caps enumeration per feature; beyond it the
candidate list is stride-sampled. A single oxidised token such as `20:4;O`
enumerates **~22,300** candidates, so on oxidised work the default silently
discards the true structure *before scoring ever runs*. The output is
indistinguishable from a scoring failure. **Raise it to 30000 whenever
`acyl_analysis: ead2` is set on `;O` species.**

**`l3_selection`** (`posterior`) decides which positional candidates get
reported and when idlevel 4 commits to a position. `posterior` keeps the
smallest candidate set holding 95% of the posterior mass and names a position
only when one placement carries `l3_post_min_mass` (0.5). `legacy` is the old
relative-score cutoff — it names more positions and is right less often
(measured: 84/96 features asserted at 65% correct, versus 55/96 at 73%). Use
`legacy` to reproduce pre-1.0.173 output, not to get more calls.

`l3_post_temperature` and `l3_min_margin` are **fitted against reference sets
with known positions**, not chosen by taste. Do not tune them to make output
look better; changing them invalidates the calibration.

> **Renamed in v1.0.173.** Every idlevel-3/4 parameter moved from `ead_*` to
> `l3_*` (`ead_max_candidates` → `l3_max_candidates`, `ead_chain_cutoff` →
> `l3_legacy_cutoff`, `include_ms1_monochain` → `l3_include_ms1_monochain`, and
> so on). The old spellings still load as aliases, so **do not "fix" an existing
> config** that uses them — but write new configs with `l3_*`.

The complete annotated parameter file, with every key and its default, is in
[reference/lipidoracle.default.yaml](reference/lipidoracle.default.yaml). Read
it before changing anything outside `WORKFLOW:`.

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
| 2 | MS2 fragments matched, scored by `score_L2` | Class confirmed, often individual acyl chains |
| 3 | EAD/UVPD isomer scoring, `score_L3` | C=C and oxidation positions along chains |
| 4 | Consolidation | One row per spectrum merging all idlevel-3 candidates, with a confidence tail |

`score_L3` is normalised against each spectrum's own best candidate, so **the top
candidate always reads 1.0** whether the evidence is decisive or absent. It ranks
within a spectrum; it does not measure confidence, and it is not comparable
across engines. What separates a determined position from an undetermined one is
whether idlevel 4 named it at all.

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

**Do not report a species-level idlevel-4 name as a failure.** Since v1.0.173
LipidOracle writes positions into the name only when the evidence supports one
placement, so `FA 20:4` at idlevel 4 means the positions were *not determined* —
which is a result, not a missing one. `diag/l3_resolvability.csv` says how
uncertain it was.

Column-by-column output reference: [reference/outputs.md](reference/outputs.md).

## Trying it without your own data

The repository ships two ready-to-run datasets under `testdata/`:

| Folder | Spectra | Notes |
| --- | ---: | --- |
| `testdata/cid/` | 9,948 | CID run, annotation to idlevel 1–2. Seconds. |
| `testdata/ead/` | 38,255 | EAD run. Set `acyl_analysis: ead1` to localise C=C. |

## Troubleshooting

| Symptom | Cause |
| --- | --- |
| Runs and exits immediately, only writes `lipidoracle.yaml` | Expected first-run behaviour. Re-run the same command. |
| "Cannot find input MGF" | No `.mgf` under `/input`, or the volume mount used a relative path. |
| `summary.csv` reports 0 MS1 features | The MGF has no `MSLEVEL=1` blocks, or no `MSLEVEL` at all (defaults to 2). Re-export from MASSter/MZmine including MS1. |
| All RT values look ~60x too small | The MGF used minutes in `RTINSECONDS`. Re-export rather than patching the file. |
| Results can't be joined to the upstream feature table | The MGF was exported without `FEATURE_ID`, so `mgf_feat` is empty. |
| Config changes have no effect | Edited a file outside the mounted output folder, or a stray `*.yaml` in the output folder is being picked up first. |
| `WORKFLOW.acyl_analysis: unrecognized value` | Use `False`, `ead1`/`v1`, `ead2`/`v2`, or `uvpd`. |
| No idlevel 3 rows | `acyl_analysis` is `False`, or the data is CID and carries no EAD/UVPD fragments. |
| idlevel 3 has candidates but idlevel 4 names no positions | Expected under `l3_selection: posterior` when the spectrum did not determine one. Check `top_prob` and `cred95` in `diag/l3_resolvability.csv` before changing anything. |
| Oxidised lipids localise poorly | `acyl_analysis: ead1` ignores `;O` species entirely — use `ead2`. If already on `ead2`, `l3_max_candidates` is likely too low; raise it to 30000. |
| Names changed shape vs an older run | v1.0.173 emits Shorthand2020 (`;5OH`, not `;OH(5)`) and drops `ctu:`/`f:`/`RG:` from CXSMILES. Expected, not a regression. |
| Too many IDs removed | `rt_check: hydra` is strict; try `liri`, or `False` to see the unfiltered set. |
| Dashboard plots are blank | Offline machine — the CDN libraries could not load. The CSVs are unaffected. |

## Reference

Documentation site: <https://zamboni-lab.github.io/lipidoracle/>

| Page | Covers |
| --- | --- |
| [params](https://zamboni-lab.github.io/lipidoracle/params.html) | every `lipidoracle.yaml` setting |
| [idlevel3](https://zamboni-lab.github.io/lipidoracle/idlevel3.html) | the EAD engines, candidate selection, every `l3_*` parameter |
| [nomenclature](https://zamboni-lab.github.io/lipidoracle/nomenclature.html) | Shorthand2020, CXSMILES ambiguity encoding, the confidence tail |
| [rt](https://zamboni-lab.github.io/lipidoracle/rt.html) | the retention-time engines |
| [library](https://zamboni-lab.github.io/lipidoracle/library.html) | class coverage, extra libraries, rarity |

What changed in the current release: [WHATSNEW.md](../../../WHATSNEW.md).

Method: Wu et al., *Analyst*, 2025, [10.1039/d5an00567a](https://doi.org/10.1039/d5an00567a)

Licensed PolyForm Noncommercial 1.0.0 — non-commercial use only, including
educational and public research organisations. Commercial use needs separate
permission from the Zamboni lab, ETH Zurich.
