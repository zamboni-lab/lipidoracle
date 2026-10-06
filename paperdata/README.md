# Files used in LipidOracle's paper

The paper includes four benchmarks. This repository includes the the spectra as MGF, the configuration, and the couple of files a configuration
reads. 

```
01_plasma_cid_srm1950/   masster_pos_consensus.mgf, lipidoracle.yaml
02_ead_standards/        light_all.mgf, 7600_all.mgf, targets.csv, lipidoracle.yaml
03_ead_liver/            liver_15m_8600.mgf, lipidoracle.yaml
04_uvpd/                 uvpd-gt-pos.mgf, equisplash.csv, lightsplash.csv, ultimatesplash.csv,
                         lipidoracle.yaml
```

Each folder is flat. Keep it that way: a configuration finds the files it names, such as `02`'s
`targets.csv` or `04`'s SPLASH libraries, beside it. Every configuration is the one its benchmark
ran with, and it carries every setting that run used. One spectrum per folder, since a folder's
spectra are run in alphabetical order until one is named.

## Re-running a benchmark with Docker

Mount the folder at `/input` and an empty folder of its own at `/output`. The image finds the
`lipidoracle.yaml` in `/input`, runs the spectrum beside it, and writes everything into `runs/`:

```bash
cd 02_ead_standards
mkdir -p runs
docker run --rm -v "$PWD":/input -v "$PWD/runs":/output zambonilab/lipidoracle:1.0.267

That is the whole command: no `-e`, no `-p`, no `-w`. Any file the configuration names, such as
`02`'s `targets.csv` or `04`'s SPLASH libraries, has to sit beside it in the folder.

| benchmark | folder | time |
|---|---|---|
| 01 human plasma by CID | `masster_pos_consensus.mgf` + `lipidoracle.yaml` | about a minute |
| 02 EAD lipid standards | `light_all.mgf` in one folder, `7600_all.mgf` in another, each with `targets.csv` | seconds |
| 03 mouse liver by EAD | `liver_15m_8600.mgf` + `lipidoracle.yaml` | about ten minutes |
| 04 UVPD lipid standards | `uvpd-gt-pos.mgf`, the three SPLASH CSVs and `lipidoracle.yaml` | seconds |

Four things about the command:

* `/output` must be a folder of its own, `runs` here, **not** the folder mounted at `/input`. The
  run writes its own configuration into `/output`, so mounting one folder at both destroys the
  benchmark's `lipidoracle.yaml` and the run then falls back to its default workflow and finds no
  identifications. That is what
  `docker run --rm -v <folder>:/input -v <folder>:/output <image>` does, on 1.0.263, 1.0.266 and
  1.0.267 alike: every idlevel count comes out 0.
* **One spectrum per folder.** With two present the run takes the first alphabetically: the EAD
  folder holds `light_all.mgf` and `7600_all.mgf`, and with both there the run reads
  `light_all.mgf`. Split them into two folders, or name the file with
  `-e INPUT=/input/7600_all.mgf`.
* `runs/` should exist first, as above; Docker would otherwise create it owned by root.
* The configuration is the `lipidoracle.yaml` in the folder. A differently named one needs
  `-p /input/<name>.yaml`.

`runs/` then holds the engine's own output: the reported table, its diagnostics, its log, and the
configuration it ran under. A local build of the engine can be used instead of the image with the
same arguments and no container.

## Where the inputs come from

* `01`'s spectra are the MASSter consensus features of Metabolomics Workbench study ST003514
  (project DOI 10.21228/M8JK0Q), released CC BY.
* `02`'s spectra are from Wu et al., ETH Research Collection, Data Collection DOI
  10.3929/ethz-b-000734594: the standards at twelve accumulation times on a ZenoTOF 8600, and a
  second acquisition of the same standards on a ZenoTOF 7600.
* `03`'s spectrum is a mouse-liver extract on the same 8600.
* `04`'s spectrum is the positive-mode UVPD ground-truth set.
