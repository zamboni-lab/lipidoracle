# Files used in LipidOracle's paper

The paper includes four benchmarks. This repository includes the the spectra as MGF, the configuration, and the couple of files a configuration
reads. 

| benchmark | folder | time |
|---|---|---|
| 01 human plasma by CID | `masster_pos_consensus.mgf` + `lipidoracle.yaml` | about a minute |
| 02 EAD lipid standards | `light_all.mgf` in one folder, `7600_all.mgf` in another, each with `targets.csv` | seconds |
| 03 mouse liver by EAD | `liver_15m_8600.mgf` + `lipidoracle.yaml` | about ten minutes |
| 04 UVPD lipid standards | `uvpd-gt-pos.mgf`, the three SPLASH CSVs and `lipidoracle.yaml` | seconds |

## Re-running a benchmark with Docker

Mount the folder at `/input` and an empty folder of its own at `/output`. The image finds the
`lipidoracle.yaml` in `/input`, runs the spectrum beside it, and writes everything into `runs/`:

```bash
cd 02_ead_standards
mkdir -p runs
docker run --rm -v "$PWD":/input -v "$PWD/runs":/output zambonilab/lipidoracle:1.0.267
```

## Where the inputs come from

* `01`'s spectra are the MASSter consensus features of Metabolomics Workbench study ST003514
  (project DOI 10.21228/M8JK0Q), released CC BY.
* `02`'s spectra are from Wu et al., ETH Research Collection, Data Collection DOI
  10.3929/ethz-b-000734594: the standards at twelve accumulation times on a ZenoTOF 8600, and a
  second acquisition of the same standards on a ZenoTOF 7600.
* `03`'s spectrum is a mouse-liver extract on the same 8600.
* `04`'s spectrum is the positive-mode UVPD ground-truth set.
