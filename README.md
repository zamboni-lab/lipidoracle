# LipidOracle

Lipid annotation from MS1 and MS2 spectra: accurate-mass matching, MS2 scoring,
retention-time validation, and C=C / oxidation localisation from EAD or UVPD data.

**Documentation: <https://zamboni-lab.github.io/lipidoracle/>**

This repository holds the documentation site and the test data. The engine ships
as a Docker image.

## Run

Input is a single `.mgf` carrying both MS1 and MS2 scans. Settings live in a
`lipidoracle.yaml`.

Ensure Docker is installed locally. Refer to [Docker Desktop](https://docs.docker.com/desktop/) for installation instructions.

```bash
docker pull zambonilab/lipidoracle

docker run --rm \
  -v <INPUT-FOLDER>:/input \
  -v <OUTPUT-FOLDER>:/output \
  zambonilab/lipidoracle
```

Put the `.mgf` in the input folder. If no `lipidoracle.yaml` is found in either
folder, LipidOracle writes a default one to the output folder and stops, so you
can review it before the real run. The defaults use the internal library and the
LiRI retention-time check. See
[Parameters](https://zamboni-lab.github.io/lipidoracle/PARAMETERS.html) for the
full reference.

## Test data

Two ready-to-run datasets in `testdata/`:

| Folder | Spectra | Size | Notes |
| --- | ---: | ---: | --- |
| `cid/` | 9,948 | 32 MB | CID run, annotation to idlevel 1–2. Completes in seconds. |
| `ead/` | 38,255 | 12 MB | EAD run. Set `acyl_analysis: v1` to localise C=C. |

## License

LipidOracle is licensed under the [PolyForm Noncommercial License 1.0.0](LICENSE). It may be used, modified, and redistributed for non-commercial purposes, subject to the license terms. Commercial use requires separate permission from the copyright holder.
