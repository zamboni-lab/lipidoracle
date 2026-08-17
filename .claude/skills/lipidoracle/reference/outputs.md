# LipidOracle output reference

Everything below lands in the folder mounted at `/output`.

## Layout

```
<output>/
├── annotation.csv        # one row per accepted annotation — the main result
├── summary.csv           # run statistics, key/value
├── dashboard.html        # self-contained interactive report
├── data.mgf              # copy of the input, for provenance
├── lipidoracle.yaml      # copy of the parameters actually used
└── diag/
    ├── annotation_full.csv
    ├── annotation_stage1.csv
    ├── annotation_stage2.csv
    ├── annotation_idlevel3.csv     # only when stage3 is on
    ├── annotation_idlevel4.csv     # only when stage3 is on
    ├── s3_resolvability.csv        # only when stage3 is on
    ├── summary_by_feature.csv
    ├── input_spectra.csv
    ├── lib_stage1.csv
    ├── lib_stage2.csv
    └── rt_*.csv
```

The input `.mgf` and the effective YAML are copied in, so an output folder is a
complete, reproducible record of one run. Use a fresh output folder per run.

## annotation.csv

The compact result table. One row per surviving annotation; a spectrum can have
several rows when candidates tie.

| Column | Meaning |
| --- | --- |
| `mgf_index` | Position of the scan in the input MGF |
| `mgf_feat` | Feature id grouping the MS1 and MS2 scans of one precursor |
| `mgf_title` | `TITLE` line from the MGF |
| `prec_uid` | Precursor identifier |
| `mz` | Observed precursor m/z |
| `rt` | Observed retention time (seconds) |
| `rt_deviation` | Observed minus model-predicted RT; large values were candidates for removal |
| `idlevel` | Evidence level 1–4 (see below) |
| `formula` | Neutral molecular formula |
| `ion` | Adduct, e.g. `[M+H]+` |
| `name` | Most specific name the evidence supports — a strict Shorthand2020 name, plus a bracketed confidence tail at idlevel 4 |
| `species` | Species-level (shorthand) name |
| `class` | Lipid class, e.g. `PC`, `TG`, `FA` |
| `mod_str` | Modifications, e.g. `;O`, `;5OH` |
| `rarity` | Rarity index of the library entry; higher = less common, penalised in scoring |
| `score_s2` | MS2 match score (idlevel 2+) |
| `score_s3` | Positional localisation score (idlevel 3/4). Not comparable across stage-3 engines. |
| `ms2_tic` | Total ion current of the filtered MS2 spectrum |
| `ms2_evidence` | Which structural features the matched fragments support |
| `ms2_match_n` / `ms2_missed_n` | Library fragments found / not found |
| `lib_id`, `lib_src` | Library entry and which library it came from |
| `smiles` | CXSMILES structure, with ambiguity blocks where positions are unresolved |

### Identification levels

| idlevel | Evidence | Detail reached |
| --- | --- | --- |
| 1 | Precursor m/z only | Class, total carbons, total double bonds |
| 2 | MS2 fragments matched (`score_s2`) | Class confirmed, often individual chains |
| 3 | EAD/UVPD/OAD/OzID isomer scoring (`score_s3`) | C=C and oxidation positions |
| 4 | Isomer consolidation | One row per spectrum merging idlevel-3 candidates |

Filter to `idlevel >= 2` for MS2-confirmed calls. idlevel 1 rows are mass-only
and carry the most false positives — the RT engines exist mainly to prune them.

### Reading names

Names are **Shorthand2020** (Liebisch et al. 2020), enforced throughout; other
spellings in input libraries are normalised on load.

`/` between chains = sn-positions known. `_` = composition known, positions not.
No separator (`PC 34:1`) = species-level shorthand only.

Modifications follow a semicolon with **the position in front of the
abbreviation**: `;3OH,5OH` hydroxyls, `;5oxo` ketone, `;8COOH` extra carboxyl,
`;5Ep` epoxide, `;[11-13cy3:0]` cyclopropane. Double-bond geometry when known:
`18:1(9Z)`, `18:1(9E)`.

#### The idlevel-4 confidence tail

At idlevel 4 the `name` carries a bracketed tail giving the support behind each
localised position:

```
FA 20:4;OH [DB sn1: Δ5 100%, Δ8 100%, Δ12 100%, Δ14 50%, Δ15 50%; OH sn1: 11 100%]
```

Aspects (`DB`, `OH`, `oxo`) and chains are separated by `;`, positions within an
aspect by `,`. **Strip the tail before parsing the name** — everything from
` [` to the closing `]` is the extension, and the part before it is the strict
Shorthand2020 core.

Note that a group's position can appear in the tail while the name body leaves
it unpositioned (`FA 20:4;OH`, not `FA 20:4;11OH`). That is deliberate: a group
positioned mid-chain alongside unlocalised double bonds does not say how those
bonds divide around it, so the body would be unrepresentable.

What the percentage means depends on `PARAM.s3_selection`: under `posterior`
(the default) it is that position's marginal probability; under `legacy` it is
the fraction of retained candidates supporting it — candidate support, not a
calibrated probability.

#### CXSMILES

The `smiles` column is extended CXSMILES, validated against CDK. Blocks inside
the `|...|` delimiter: `Sg:` variable-length chain segment (unlocalised double
bond), `m:` position-variation bond (unlocalised modification), `$snN$` atom
labels (unresolved sn-assignment). LipidOracle extensions live *after* the
closing `|`, in the title field: `constrain(a+b=15)` gives the `Sg:` run
lengths, `swappable(sn1,sn2)` permits the labelled positions to permute, and
`dbPos(...)` / `mPos(...)` carry the idlevel-4 percentages.

**A fully determined structure has no `|...|` tail at all** — the presence of a
tail is itself the signal that something was left open.

Two traps if you consume this column:

- **A renderer that ignores `Sg:` fails silently**, drawing a chain many carbons
  short with no warning. Expand before depicting.
- **The trailer does not survive a round trip** through a toolkit: the `|...|`
  block gets renumbered correctly, but `constrain(...)`, `swappable(...)`,
  `dbPos(...)` and `mPos(...)` live in the title field and are dropped. Preserve
  it yourself if something rewrites these strings.

`ctu:` blocks and the `f:` / `RG:` encodings of unresolved sn-assignment do not
appear in this format — do not write parsers expecting them. Undetermined
double-bond geometry is a bare `C=C`; unresolved sn-assignment uses `$snN$`
labels on a complete molecule instead.

## summary.csv

Two columns, key and value. Input file, version, MS1 and MS2 feature counts,
how many features got at least one ID per idlevel, and average scores per level.
Read this first to sanity-check a run: if "features with at least one ID" is
near zero, the polarity, adduct list, or tolerances are wrong.

## dashboard.html

Self-contained; open directly in a browser, no server. RT/mz scatter colourable
by class, rarity, or score and filterable by idlevel; Kendrick mass defect and
chain-distribution views; clickable MS2 match panels tracing each peak to the
fragment it matched; structures rendered on the fly from the CXSMILES.

Plotly, Tabulator, and RDKit load from CDNs — on an offline machine the plots
stay blank while the CSVs remain fully valid.

## diag/ files

| File | Contents |
| --- | --- |
| `annotation_full.csv` | Every column the engine tracks: `ppm`, `dmz`, per-metric scores (`score_s2_metric`, `score_s2_modcos`), matched and missed fragment lists, `ms2_top10`, `rank`, `compressed`. Use this to explain *why* a call scored the way it did. |
| `annotation_stage1.csv`, `annotation_stage2.csv` | Per-stage snapshots before later filtering |
| `annotation_idlevel3.csv` | Retained positional candidates, up to `s3_max_output` per feature. Broader than what reaches `annotation.csv`. |
| `annotation_idlevel4.csv` | The per-feature positional consensus |
| `s3_resolvability.csv` | Why a position was or was not called: `n_candidates`, `top_prob` (posterior mass on the winner), `cred95` (size of the 95% credible set), `db_marginals` (per-position marginals, strongest first), `margin` and `matched_n` (what the idlevel-4 gates read), and `null_rel` when `s3_null_decoys` is on. For `ozid`, resolvability is pair-specific: the winner needs more complete diagnostic pairs than its best structurally distinct rival. One row per scored feature. **Reports only — it gates nothing.** |
| `summary_by_feature.csv` | One row per MS1 feature, aggregating its spectra, peaks, entropy, and all competing IDs |
| `input_spectra.csv` | Parsed spectra as the engine saw them — first stop when the MGF may be malformed |
| `lib_stage1.csv` | Generated MS1 library: masses, adducts, chains, rarity |
| `lib_stage2.csv` | Generated MS2 library: expected fragments with labels, weights, types |
| `rt_liri.csv` / `rt_hydra.csv` | Per-candidate RT model output: `rt_pred`, `sigma`, `z`, `tier`, `verdict` — shows exactly which IDs the engine rejected |
| `rt_generic.csv` | The generic post-filter: `rt_pred`, `err`, `cutoff`, `protected`, `severe`, `verdict` |

To trace why an expected lipid is missing, walk backwards: `lib_stage1.csv`
(was it in the library?) → `annotation_stage1.csv` (did the mass match?) →
`annotation_stage2.csv` (did MS2 score above cutoff?) → `rt_*.csv` (did the RT
filter reject it?).

To trace why a **position** was not called, the walk is different — the
annotation is there, it just names the chain at species level. Check
`annotation_idlevel3.csv` first (were candidates enumerated at all? if the file
is empty, `stage3` is off or the data is not EAD/UVPD/OAD/OzID), then
`s3_resolvability.csv`:

- `n_candidates` at or near `s3_max_candidates` means the search was truncated
  by stride-sampling. A single oxidised token like `20:4;O` enumerates ~22,300
  candidates; below that cap the true structure can be sampled out *before
  scoring runs*, and the output looks identical to a scoring failure. Raise
  `s3_max_candidates` to 30000 for oxidised work.
- `top_prob` below `s3_post_min_mass` (default 0.5) is the ordinary case: the
  spectrum genuinely did not determine the position, and the credible set in
  `cred95` says how uncertain it was. This is the intended behaviour of
  `s3_selection: posterior`, not a defect.
- `margin` below `s3_min_margin` or `matched_n` below `s3_min_matched` means
  `ead1`'s or `oad`'s gates abstained.
- For `ozid`, an incomplete diagnostic pair (missing aldehyde or Criegee
  partner) blocks localisation by default (`ozid_require_complete_pair: True`).
- Setting `s3_selection: legacy` will name more positions, at a measurably lower
  hit rate.
