# LipidOracle

Lipid annotation from MS1 and MS2 spectra: accurate-mass matching, MS2 scoring, retention-time validation, C=C / oxidation localisation from EAD or UVPD data, etc.

**Preprint: [LipidOracle: Ambiguity-aware lipid annotation across fragmentation methods](https://chemrxiv.org/doi/full/10.26434/chemrxiv.15009711/v1)**

**Documentation: <https://zamboni-lab.github.io/lipidoracle/>**

This repository holds the documentation site and the test data. 

The software ships as the Docker image `zambonilab/lipidoracle` as described in [Run LipidOracle](docs/running-lipidoracle.md).

## Guides

- [Run LipidOracle](docs/running-lipidoracle.md) - installation, MGF preparation, a minimum configuration, and custom libraries.
- [YAML configuration reference](docs/yaml-configuration.md) - start with the `WORKFLOW` section, then tune `PARAM` only when needed.
- [Built-in library](docs/built-in-library.md) - internal library scope, rarity, adducts, and optional external libraries.
- [MS1 and MS2 matching](docs/ms1-ms2-matching.md) - annotation levels, scoring, thresholds, and chain resolvability.
- [Stage 3 with EAD](docs/stage3-ead.md) - C=C and oxygen-site localization, candidate selection, and confidence reporting.
- [Oxidation analysis](docs/oxidation.md) - what MS1, MS2, and EAD2 can and cannot establish for oxygenated lipids.
- [Retention-time checking](docs/retention-time.md) - LiRI, HYDRA, legacy filtering, and RT reference files.
- [Outputs and dashboard](docs/outputs-dashboard.md) - CSV exports, diagnostic files, and interactive review.

Using an AI coding agent? See [AGENTS.md](AGENTS.md) — the repository carries a
LipidOracle skill that teaches agents the full Docker workflow, the parameter
file, and how to read the outputs.

## Run

Input is a single `.mgf` carrying both MS1 and MS2 scans, stored in a local folder `<INPUT-FOLDER>`. 

Ensure Docker is installed locally. Refer to [Docker Desktop](https://docs.docker.com/desktop/) for installation instructions.

```bash
docker pull zambonilab/lipidoracle

docker run --rm \
  -v <INPUT-FOLDER>:/input \
  -v <OUTPUT-FOLDER>:/output \
  zambonilab/lipidoracle
```

If no `lipidoracle.yaml` is found in either input or output folder, LipidOracle writes a default one to the output folder and stops, so you
can review it before the real run. The defaults use the internal library and the LiRI retention-time check. See
[Parameters](https://zamboni-lab.github.io/lipidoracle/params.html) for the
full reference.

## Test data

Two ready-to-run datasets in `testdata/`:

| Folder | Spectra | Size | Notes |
| --- | ---: | ---: | --- |
| `cid/` | 9,948 | 32 MB | CID run, annotation to idlevel 1–2. Completes in seconds. |
| `ead/` | 38,255 | 12 MB | EAD run. Set `stage3: ead1` to localise C=C. |

## Paper data

`paperdata/` holds the input files behind the four benchmarks of the LipidOracle paper, ready to
re-run: the spectra each benchmark was acquired on, with the configuration it ran under - human
plasma by CID, the EAD lipid standards in two acquisitions, the EAD mouse liver, and the UVPD
standards. Nothing else is in there; the results and the scoring references live with the paper's
data record.

Each folder is flat and self-contained, so a configuration finds the files it names when the
engine is run from that folder. `paperdata/README.md` gives the docker command, the one line per
benchmark that changes, and where the spectra come from.

```bash
docker run --rm \
  -v <INPUT-FOLDER>:/input \
  -v <OUTPUT-FOLDER>:/output \
  zambonilab/lipidoracle:1.0.267
```

## License

LipidOracle is licensed under the [PolyForm Noncommercial License 1.0.0](LICENSE). It may be used, modified, and redistributed for non-commercial purposes, subject to the license terms. Commercial use requires separate permission from the copyright holder.
