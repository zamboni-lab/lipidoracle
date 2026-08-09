# Notes for coding agents

This repository holds the LipidOracle documentation site and test data. The
annotation engine itself is **not** in this repo — it ships only as a Docker
image, `zambonilab/lipidoracle`. There is nothing here to build, compile, or
install.

## If you are asked to run LipidOracle

Read [.claude/skills/lipidoracle/SKILL.md](.claude/skills/lipidoracle/SKILL.md)
first. It is the complete operating guide: the Docker invocation, the two-step
first run, how to configure `lipidoracle.yaml`, and how to interpret the
outputs. Claude Code loads it automatically as a skill; other agents should read
it as a plain document.

Supporting references, both linked from the skill:

- [.claude/skills/lipidoracle/reference/lipidoracle.default.yaml](.claude/skills/lipidoracle/reference/lipidoracle.default.yaml)
  — every parameter with its default and an inline explanation.
- [.claude/skills/lipidoracle/reference/outputs.md](.claude/skills/lipidoracle/reference/outputs.md)
  — every output file and column.

## The one-paragraph version

LipidOracle starts from an `.mgf`; it does not read vendor formats or mzML and
does not do peak picking. Produce the MGF upstream with
[MASSter](https://github.com/zamboni-lab/masster-dist)
(`sample.find_features()` → `sample.find_ms2()` → `sample.export_mgf(...)`, and
`study.import_oracle(...)` to pull the results back) or with MZmine. Both MS1
and MS2 scans go in one file, tagged with `MSLEVEL`.

Then put that single `.mgf` in a folder, pick an output folder, and run:

```bash
docker run --rm -v /abs/in:/input -v /abs/out:/output zambonilab/lipidoracle
```

The first run writes a commented `lipidoracle.yaml` into the output folder and
exits without annotating — that is expected. Review the file, then run the exact
same command again to get `annotation.csv`, `summary.csv`, `dashboard.html`, and
`diag/`.

## Things that commonly go wrong

- Relative `-v` paths create empty Docker volumes. Always mount absolute paths.
- More than one `.mgf` in the input folder makes auto-detection ambiguous.
- An MGF without `MSLEVEL` lines is read as MS2-only — the MS1 stage then finds
  nothing. `RTINSECONDS` must be seconds, and `FEATURE_ID` is what links results
  back to the upstream feature table.
- Editing a config that is not inside the mounted output folder has no effect.
- A run that only prints "Created default parameters" succeeded; re-run it.

## Repository conventions

- `docs/` is the published GitHub Pages site (plain HTML, no build step).
- `testdata/cid/` and `testdata/ead/` are ready-to-run example datasets.
- Licensed PolyForm Noncommercial 1.0.0. Non-commercial use only.
