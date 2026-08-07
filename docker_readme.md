# LipidOracle Docker Guide

LipidOracle is distributed as a Docker container for non-commercial use under the [PolyForm Noncommercial License 1.0.0](LICENSE).

## Installation

Install [Docker Desktop](https://www.docker.com/products/docker-desktop/) or Docker Engine, then pull the image:

```bash
docker pull zambonilab/lipidoracle
```

## Running LipidOracle

Place one `.mgf` input file in a local input folder and provide a local output folder:

```bash
docker run --rm \
  -v <INPUT-FOLDER>:/input \
  -v <OUTPUT-FOLDER>:/output \
  zambonilab/lipidoracle
```

If no `lipidoracle.yaml` is found in either mounted folder, LipidOracle writes a default one to the output folder and stops so that you can review the settings. Edit it as needed, then run the same command again.

See the [online documentation](https://zamboni-lab.github.io/lipidoracle/) for the complete workflow and parameter reference.

## License

LipidOracle is licensed under the [PolyForm Noncommercial License 1.0.0](LICENSE). Commercial use requires separate permission from the copyright holder.
