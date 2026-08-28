# Outputs and dashboard

A LipidOracle run produces an output folder that is intended to be self-contained: it includes a copy of the input MGF, the parameters used, compact annotations, diagnostic tables, and an interactive dashboard. Preserve the whole folder with an analysis or publication record.

## Output layout

```text
results/
├── sample.mgf                 # copied input
├── lipidoracle.yaml           # effective configuration used by the run
├── annotation.csv             # compact downstream-facing annotation table
├── summary.csv                # run-level counts and summary values
├── dashboard.html             # interactive dashboard
└── diag/
    ├── annotation_full.csv
    ├── annotation_stage1.csv
    ├── annotation_stage2.csv
    ├── annotation_stage3.csv  # only when stage 3 is enabled
    ├── annotation_stage4.csv  # only when stage 3 is enabled
    ├── input_spectra.csv
    ├── lib_stage1.csv
    ├── lib_stage2.csv
    ├── lib_stage3.csv
    ├── s2_chain_resolvability.csv
    ├── s3_resolvability.csv   # when stage 3 produces applicable diagnostics
    ├── summary_by_feature.csv
    └── RT diagnostic CSVs     # depend on selected RT engine
```

The exact diagnostics depend on the enabled workflow and available evidence. An absent stage-3 or RT file usually means that stage was disabled, no compatible input/parent existed, or no model was generated. For example, a LiRI run can execute but omit `diag/rt_liri.csv` when no local RT model can be fitted. Check the progress log and YAML before interpreting absence as a negative scientific result.

## Main result files

### `annotation.csv`

This is the stable compact export intended for downstream tables, statistics, and feature-level reporting. It contains the accepted annotations after configured filtering and cleanup. Typical fields include:

| Field | Meaning |
| --- | --- |
| feature/spectrum identifiers | Links back to `FEATURE_ID`, `INDEX`, or `TITLE` in the input MGF. |
| precursor *m/z*, RT, polarity, adduct | Acquisition and ion identity context. |
| `name`, `species`, `class`, `formula` | Lipid annotation and chemical metadata. Names use Goslin Shorthand2020. |
| `stage`, `idlevel` | Evidence and specificity of the result. |
| `score_s1`, `score_s2`, `score_s3`, `score_s3_raw` | Stage-specific evidence values. Interpret them within their stage and engine. |
| `score_modcos` | Reference cosine-style MS2 value. It is not the selection score. |
| `rival_resolvability` | Whether observed exclusive fragments separated the call from its best disagreeing rival. |
| `rt_deviation` | Final generic formula-model RT deviation, not necessarily the selected RT engine's own error. |
| library/source metadata | Lets a result be traced to the internal, extra, LipiDex, LipidBlast, or stage-3 source. |
| structural depiction fields | SMILES/CXSMILES where a valid representation can be generated. |

A compact row does not include every candidate considered by the pipeline. Use the stage-specific diagnostics to investigate ambiguity, rejected alternatives, and score construction.

### `summary.csv`

Use this as the run-level overview. It aggregates counts and metadata suitable for comparing processing outcomes across runs. Use the per-feature and stage tables rather than summary totals when diagnosing a discrepancy.

### The copied MGF and YAML

The copied input MGF and effective `lipidoracle.yaml` are part of the result. A CSV cannot be reproduced reliably without its input, search-library version, adduct/threshold choices, and workflow selection.

## Diagnostic files

### Stage-specific annotations

| File | Review question |
| --- | --- |
| `diag/annotation_full.csv` | What was retained across stages, with detailed fields? |
| `diag/annotation_stage1.csv` | Which precursor/sum-composition candidates passed accurate-mass matching? |
| `diag/annotation_stage2.csv` | Which MS2 candidates, chain arrangements, and class assignments survived scoring? |
| `diag/annotation_stage3.csv` | Which individual positional hypotheses were emitted? |
| `diag/annotation_stage4.csv` | What positional consensus was reported? |

With posterior stage-3 selection, `score_s3` is a probability over the generated closed candidate set. Written rows do not necessarily sum to 1 because the export contains the credible set and limited context rows, not the entire enumerated universe. With legacy selection, it is a raw or rescaled within-feature score rather than a probability.

### Candidate and resolvability diagnostics

- `diag/s2_chain_resolvability.csv` documents whether the reported chain arrangement had observed exclusive evidence relative to a rival. `yes`, `no`, and `uncontested` have distinct meanings.
- `diag/s2_chain_candidates.csv` appears only when `s2_chain_dump_candidates: true`; it can be large and is for calibration/debugging.
- `diag/s3_resolvability.csv` reports positional-rival diagnostics. It is evidence of distinguishability, not an engine-independent confidence percentage.
- `diag/summary_by_feature.csv` is useful for feature-level audit and aggregation.
- `diag/input_spectra.csv` provides imported spectrum metadata useful for tracing parsing, RT, precursor, and peak-count issues.
- `diag/lib_stage*.csv` snapshots the generated library inputs used at each stage, supporting reproducibility and library-coverage review.

### Retention-time diagnostics

`diag/rt_liri.csv`, `diag/rt_hydra.csv`, `diag/rt_hydra_anchors.csv`, `diag/rt_class.csv`, and `diag/rt_generic.csv` appear as applicable. The selected engine's table reports its own model and decision. `rt_generic.csv` records the final generic formula check and supplies the `rt_deviation` exported to annotations. Read [Retention time](retention-time.md) before comparing these values.

## Use the dashboard

By default, `dashboard.html` is a portable single-file export with the data, LipidOracle HTML/CSS/JavaScript, and logo embedded. Open it locally in a modern browser:

```bash
xdg-open results/dashboard.html       # Linux
open results/dashboard.html           # macOS
start results\\dashboard.html          # Windows Command Prompt
```

The dashboard still loads Plotly and Tabulator from CDN URLs, so a network connection is required for those external libraries unless a local deployment replaces the URLs. If local browser restrictions affect a large dashboard, serve the result directory over HTTP:

```bash
cd results
python -m http.server 8000
# browse to http://localhost:8000/dashboard.html
```


## Dashboard review workflow

1. **Start with run summary and library information.** Confirm input, polarity, active libraries, workflow stages, and configuration.
2. **Filter the annotation table.** Examine ID level, class, source, score, RT deviation, and rarity before focusing on individual structures.
3. **Open a spectrum.** Compare the observed MS2 peaks with reported/diagnostic fragments. Check precursor, adduct, and RT first.
4. **Inspect ambiguity.** For level-2 calls, read `rival_resolvability`. For level-3/4 calls, inspect the positional candidates, `s3_kept`, posterior/raw score fields, and confidence tail.
5. **Review RT in context.** Treat a large deviation as a question to investigate, not a substitute for fragment inspection.
6. **Export or preserve evidence.** Keep the dashboard, compact export, diagnostics, YAML, input MGF, and any custom library version together.

## Suggested downstream filtering

There is no universal numerical cutoff that turns all lipid annotations into ground truth. A transparent filtering protocol should name the evidence it uses, for example:

- stage/ID-level inclusion rule;
- source-library policy;
- precursor and fragment tolerance values;
- score rules, including stage-2 relative cutoff;
- `rival_resolvability` policy;
- RT engine and rejection review policy;
- stage-3 selection parameters and treatment of ambiguous candidates;
- validation with standards or orthogonal methods for high-impact structural claims.

Do not compare raw `score_s3` values across EAD1, EAD2, OAD, UVPD, or OzID. Their chemistry, candidate sets, raw scores, and normalization differ. For feature-level quantification, keep the relationship between your original feature table and the MGF identifiers explicit.

See [MS1 and MS2 matching](ms1-ms2-matching.md) and [Stage 3 with EAD](stage3-ead.md) for the evidence behind dashboard fields.
