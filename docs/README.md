# LipidOracle documentation

LipidOracle is an MS-based lipid annotation engine. It uses a staged workflow: precursor accurate-mass matching (stage 1), MS2 library and fragment matching (stage 2), and optional within-chain localization (stage 3). It reports only the structural detail supported by the available evidence.

## Guides

1. [Run LipidOracle](running-lipidoracle.md) - installation, MGF preparation, a minimum configuration, and custom libraries.
2. [YAML configuration reference](yaml-configuration.md) - start with the `WORKFLOW` section, then tune `PARAM` only when needed.
3. [Built-in library](built-in-library.md) - internal library scope, rarity, adducts, and optional external libraries.
4. [MS1 and MS2 matching](ms1-ms2-matching.md) - annotation levels, scoring, thresholds, and chain resolvability.
5. [Stage 3 with EAD](stage3-ead.md) - C=C and oxygen-site localization, candidate selection, and confidence reporting.
6. [Oxidation analysis](oxidation.md) - what MS1, MS2, and EAD2 can and cannot establish for oxygenated lipids.
7. [Retention-time checking](retention-time.md) - LiRI, HYDRA, legacy filtering, and RT reference files.
8. [Outputs and dashboard](outputs-dashboard.md) - CSV exports, diagnostic files, and interactive review.
