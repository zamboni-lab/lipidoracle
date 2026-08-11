# LipidOracle

Lipid annotation from MS1 and MS2 spectra: accurate-mass matching, MS2 scoring,
retention-time validation, and C=C / oxidation localisation from EAD or UVPD data.

**Documentation: <https://zamboni-lab.github.io/lipidoracle/>**

This repository holds the documentation site and the test data. The engine ships
as a Docker image.

**Latest release: v1.0.173** — Shorthand 2020 nomenclature throughout, fixed
CXSMILES support, and posterior thresholding for candidate retention. See
[WHATSNEW.md](WHATSNEW.md).

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
| `ead/` | 38,255 | 12 MB | EAD run. Set `acyl_analysis: ead1` to localise C=C. |

## License

LipidOracle is licensed under the [PolyForm Noncommercial License 1.0.0](LICENSE). It may be used, modified, and redistributed for non-commercial purposes, subject to the license terms. Commercial use requires separate permission from the copyright holder.
