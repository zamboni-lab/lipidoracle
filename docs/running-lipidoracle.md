# Run LipidOracle

This guide takes an experiment from an exported MGF file to an annotation folder. For interpretation of the generated files, see [Outputs and dashboard](outputs-dashboard.md).

## 1. Install

### Docker

Docker is the simplest reproducible option. Pull the current image:

```bash
docker pull zambonilab/lipidoracle
```

Make separate local input and output folders. Put one `.mgf` file in the input folder. The container wrapper uses `/input` and `/output` by default. You can use any two folders, even the same one, and simply map the local folder to the expected mounts for the linux system embedded in the Docker container with `-v`:

```bash
docker run --rm \
  -v "$PWD/<local-input-folder>:/input" \
  -v "$PWD/<local-result-folder>:/output" \
  zambonilab/lipidoracle
```

On the first execution, the wrapper writes `lipidoracle.yaml` to the output folder and stops so it can be reviewed. Run the same command again after editing the file. To use different mounted locations, set `INPUT` and `OUTPUT` environment variables for the wrapper instead of passing duplicate `--input` or `--output` flags:

```bash
docker run --rm \
  -e INPUT=/work/input -e OUTPUT=/work/results \
  -v "$PWD:/work" \
  zambonilab/lipidoracle
```

The image includes runtime assets under `/data`, including the default YAML and library template. A custom library or RT-reference file must be available in the mounted filesystem, for example `/work/my_library.csv`.

### Build from source

The project requires a Rust toolchain compatible with the checked-in `rust-toolchain.toml`.

```bash
git clone https://github.com/zamboni-lab/lipidoracle-rs.git
cd lipidoracle-rs
cargo build --release
./target/release/lipidoracle --help
```

Run a local build with:

```bash
./target/release/lipidoracle \
  --input /path/to/sample.mgf \
  --params /path/to/lipidoracle.yaml \
  --output /path/to/results
```

`cargo run -- --input ... --output ...` is useful during development. `cargo test` runs the test suite.

## 2. Prepare the input data

LipidOracle reads **Mascot Generic Format (MGF)**. One `BEGIN IONS` to `END IONS` block is one spectrum. The parser accepts these fields case-insensitively:

| Field | Required | Purpose |
| --- | --- | --- |
| `PEPMASS` | Yes | Precursor *m/z*. If a value follows the *m/z*, it is ignored. |
| `RTINSECONDS` or `RTINMINUTES` | Yes in practice | Retention time. Minutes are converted to seconds. |
| `MSLEVEL` | Recommended | `1` for MS1 and `2` for MS2. Missing means MS2. |
| `CHARGE` | Recommended | Determines polarity, for example `1+` or `1-`. Use `--polarity pos` or `--polarity neg` to override inference. |
| `FEATURE_ID`, `INDEX`, `TITLE` | Optional | Preserved identifiers for output and dashboard review. |
| `m/z intensity` lines | MS2 | Positive, finite fragment peaks. Empty MS2 blocks are discarded. |

Example MS2 block:

```text
BEGIN IONS
TITLE=feature_124 scan_890
FEATURE_ID=124
PEPMASS=760.5850
RTINSECONDS=312.45
MSLEVEL=2
CHARGE=1+
184.0733 55000
522.3550 12100
760.5850 8000
END IONS
```

### Export checklist

1. Export centroided MS/MS spectra in MGF, retaining precursor *m/z*, polarity, and retention time.
2. Keep the experiment to one polarity per run when possible. LipidOracle infers the dominant polarity from `CHARGE`.
3. Supply meaningful feature identifiers so identified spectra can be traced back to the feature table and raw data.
4. Do not pre-delete diagnostic low-abundance peaks solely to reduce file size. The pipeline has configurable cleanup (`ms2_noise_abs`, `ms2_noise_prc`, `ms2_mztrim`).
5. For stage 3, preserve the fragmentation spectra produced by the intended acquisition mode. EAD2, OAD, OzID, and UVPD are not interchangeable analysis modes.

Precursor records within ±0.002 Da and ±0.5 seconds are deduplicated during import. If distinct acquisition events are being merged unexpectedly, inspect precursor *m/z* and RT in the exported MGF.

## 3. Create a minimum configuration

Save the following as `lipidoracle.yaml` next to the MGF. This is a minimal routine MS1/MS2 run. `PARAM: {}` is intentional: it activates the individual documented parameter defaults while keeping the example minimal.

```yaml
VERSION: 1.0.246

WORKFLOW:
  lipidoracle: true
  stage3: false
  rt_check: false

PARAM: {}
```

The first run can omit `--params`: LipidOracle copies a full default configuration to the output folder, prints its location, and stops so it can be reviewed. The CLI otherwise first looks for `<output>/lipidoracle.yaml`, then a YAML file in the output or input directory.

Run the example:

```bash
lipidoracle -i sample.mgf -p lipidoracle.yaml -o results
```

Useful CLI flags are `-d` or `--debug` for verbose logs, `-q` or `--quiet` for minimal progress output, and `--no-dashboard-single-file` to generate a split dashboard instead of the default self-contained `dashboard.html`.

## 4. Routine run variants

**Enable conservative in-run RT screening:**

```yaml
WORKFLOW:
  lipidoracle: true
  stage3: false
  rt_check: liri
  rt_ref: ''
```

**Run EAD2 for C=C and supported oxygen-site localization:**

```yaml
WORKFLOW:
  lipidoracle: true
  stage3: ead2
  rt_check: liri

PARAM:
  s3_selection: posterior
  s3_max_candidates: 100000
  s3_max_oxygen: 2
```

Read [YAML configuration](yaml-configuration.md) before changing thresholds, and [Stage 3 with EAD](stage3-ead.md) before interpreting position calls.

## 5. Add an extra library

`WORKFLOW.library_extra` accepts one or more `.csv` or `.csv.gz` files. They are merged into the search library, and MS1 matching is enabled automatically when the list is non-empty.

```yaml
WORKFLOW:
  lipidoracle: true
  library_extra:
    - /work/my_reference_library.csv
```

### Required CSV schema

The bundled [`data/library_extra_template.csv`](../data/library_extra_template.csv) is the starting point. It uses:

```csv
lib_id,name,shorthand,formula,mass,adducts,score,rarity,chains,smiles
EX001,Sterol,ST 27:1;O,C27H46O,386.3549,"[M+H]+|[M+H-H2O]+",0.5,1,0,
```

| Column | Meaning |
| --- | --- |
| `lib_id` | A stable, unique identifier. |
| `name` | Display name. Use a valid lipid shorthand name where structural conversion is desired. |
| `shorthand` | Goslin-style shorthand. Keep it consistent with `name`. |
| `formula` | Neutral molecular formula. |
| `mass` | Neutral monoisotopic mass. Provide a value consistent with `formula`. |
| `adducts` | Supported adducts, separated by `|`, for example `"[M+H]+|[M-H]-"`. |
| `score` | Library prior or reference score. |
| `rarity` | Non-negative rarity index used by the configured rarity filter and penalty. |
| `chains` | Number of chains, for example `0` for the sterol template. |
| `smiles` | Optional molecular SMILES. Leave empty if unavailable. |

Use UTF-8 CSV, a header row, and literal adduct syntax used by the configuration. Test a small library before large production additions.

### Optional MS2 reference spectra

An extra library can include predicted/reference MS2 fragments in the same CSV. Add aligned list fields, quoted because commas are CSV delimiters:

```csv
lib_id,name,shorthand,formula,adducts,frag_mz,frag_label,frag_w,frag_type
EX002,Example lipid,PC 16:0_18:1,C42H82NO8P,[M+H]+,"184.0733,496.3390","headgroup,NL 16:0","100,30","diag,acyl"
```

| Field | Accepted aliases | Meaning |
| --- | --- | --- |
| fragment *m/z* | `frag_mz`, `ms2_mz`, `pred_mz` | A list of predicted fragment *m/z* values. Invalid or nonpositive values are discarded. |
| fragment labels | `frag_label`, `ms2_label`, `pred_label` | Labels in the same order as fragment *m/z*. Missing labels are blank. |
| fragment weights | `frag_w`, `ms2_w`, `pred_w` | Relative scoring weights in the same order. Missing values default to `20.0`. |
| fragment types | `frag_type`, `ms2_type`, `pred_type` | Types in the same order. Missing values default to `0`. |

Lists accept commas, semicolons, or pipes, with optional square brackets. All four lists are aligned by index. The MS2 spectrum is keyed by canonical `species|adduct`; when duplicate records are supplied, the spectrum with more valid fragments wins.

Practical recommendations:

- The current generic parser requires `formula`. `mass` is accepted for compatibility but neutral mass is recalculated from formula.
- Use `class` and `adducts` for optional fields. `adducts` accepts comma, semicolon, or pipe separation. Blank adducts use `PARAM.adducts` and are filtered by the configured adducts, likelihoods, and run polarity.
- Add **MS1-only** records first when only formula and precursor evidence are known. A library row without `frag_mz` still contributes to MS1 but does not activate MS2 scoring by itself.
- Add MS2 fragments only when precursor/adduct identity, polarity, and fragmentation mode are matched to the input data. A mismatched spectrum can increase false positives more than it improves coverage.
- Include diagnostic fragments and meaningful relative weights. Avoid spectra dominated by precursor carryover or background.
- Keep a changelog and version the library file. The output records library provenance, making a run reproducible only when the same library version is retained.
- Review extra-library hits in the dashboard and compare `score_s2`, matched fragments, RT behavior, and ambiguity with internal-library results.

See [Built-in library](built-in-library.md) for how extra libraries interact with the internal library, LipiDex, LipidBlast, rarity, and adduct settings.
