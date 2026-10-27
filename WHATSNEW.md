# What's new

## v1.1.7

### Stage-2 simulation engines (`WORKFLOW.stage2`: `False`, `legacy`, `qc-cid`, `qc-ead`) and legacy Stage-2 fixes

`WORKFLOW.stage2` selects the MS2 fragment simulator used by Stage 2 when `WORKFLOW.lipidoracle: True`:

- **`legacy` (or `True`, default)**: empirical rule-based Stage-2 headgroup (`data/diag_cid_hg.json`) and acyl/alkyl/sphingoid cleavage simulator, now with 5 chemistry fixes:
  1. Ether-linked `PE`/`PS` (`O-`/`P-`) no longer emit acylium (`[RC=O]+`) or `[RC=O-H2O]+` ions from alkyl/alkenyl chains (`!has_ether` gate).
  2. `deduplicate_fragments()` sums weights of coincident fragments from identical chains in symmetric `TG`/`DG` species (conserving reaction-channel degeneracy).
  3. `SM`, `HexCer`, `Hex2Cer`, and `LacCer [M+H]+` include sphingoid dehydration ions (`[Sph+H-H2O]+`, `[Sph+H-2H2O]+`) and `SM [M+H]+` amide ketene loss (`[M+H-Ketene(FA)]+`).
  4. `PC [M+Na]+` includes sodiated ketene loss (`[M+Na-Ketene(FA)]+`).
  5. Negative-mode `FA [M-H]-` (`[M-H-H2O]-`, `[M-H-CO2]-`) and `[M+Ac-H]-` / `[M+CH3COO]-` (`PC`, `LPC`, `SM`, `Cer`, `HexCer`, `Hex2Cer`, `LacCer`) headgroup and acyl/sphingoid rules are supported.
- **`qc-cid`**: first-principles RRKM/TS-2D QC-CID Stage-2 simulator ($E^* = 46\text{ kcal/mol}$, $s_{\text{eff}} = 8.5$), filtering high-barrier ($>50\text{ kcal/mol}$) headgroup artifacts and adding `TG`/`DG [M+Na]+` $\text{Na}^+$-retained `[M+Na-RCOOH]+` ions, `PE`/`PS [M+H]+` ketene losses, `PC [M+H]+` minor *sn*-1 ketene/FA losses, and PUFA ($\text{db} \ge 4$) retro-ene decarboxylation `[RCOO-CO2]-` anions.
- **`qc-ead`**: first-principles 10–20 eV QC-EAD Stage-2 simulator, extending `qc-cid` thermal channels with EAD electronic/radical headgroup and glycerol-backbone diagnostics (`PC`/`LPC [M+H]+` `GPC+H-CH3OH` $m/z\ 226.0839$ and `GPC+H` $m/z\ 258.1101$, `SM [M+H]+` C2–C3 phosphocholine cleavages $m/z\ 225.0999 / 253.0948$, homolytic ester C–O acyloxyl radical loss `[M+Adduct-RCOO•]+`, Baba's regioisomeric *sn*-1 glycerol C1–C2 radical loss `[M+Adduct-R1COOCH2•]+`, and sphingoid C2–C3 backbone ions).
- **`False`**: skips Stage 2 and reports Stage-1 accurate-mass annotations only.

---

### UVPD2: Whole-chain quantum-chemistry multi-wavelength UVPD engine (`193`, `213` [default], `266`, `355 nm`)

`WORKFLOW.stage3` now accepts **`uvpd2`** (and wavelength shorthands `uvpd193`,
`uvpd213`, `uvpd266`, `uvpd355`), backed by `src/stage3_uvpd2.rs` and
`PARAM.s3_uvpd_wavelength_nm: 213.0` (`193`, `213` [default], `266`, and `355 nm`).

- **Wavelength-selective chromophore physics**: models $\pi \to \pi^*$ and
  $n \to \sigma^* / n \to \pi^*$ absorption cross-sections $\sigma(\lambda)$ across
  isolated alkenes, bis-allylic PUFAs, conjugated dienes ($\Delta p = 2$),
  conjugated enones, and divinyl-carbinols, while suppressing the high-BDE
  central $C(sp^2)-C(sp^2)$ single bond inside conjugated dienes.
- **Oxidized (`-OH`, `=O`) + unoxidized (`C=C`) acyl-chain localization**:
  unlike baseline `uvpd` (which skips `;O` chains), `uvpd2` enumerates and
  localizes joint `C=C`, `-OH`, and `=O` positions using capto-dative
  $\alpha$-OH and Norrish Type I/II $\alpha$-ketone photolysis.
- **Full technical writeup**: see [`dev/uvpd2.md`](dev/uvpd2.md).

---

### Formate adducts `[M+HCOO]-` for PC, LPC, SM, PE, LPE, Cer, HexCer

All PC/LPC/SM/Cer/HexCer classes (incl. ether/plasmalogen variants) now consider
`[M+HCOO]-` as the primary negative adduct; PE/LPE list it as a minor channel
(score 0.5, rarity 1). New MS/MS profiles in `data/diag_cid_hg.json`:
SM ([M-CH3]-, m/z 168, PO3-), LPE, Cer and HexCer (loss of HCOOH, then H2O,
CH2O or hexose). The PE formate profile now loses HCOOH instead of the PC-only
methyl formate. Fixes: PE formate chain fragments were 1.008 Da off; SM and
HexCer no longer predict bogus [RCOO]- ions from amide/sphingoid chains in
negative mode (HexCer uses the Cer chain rule).

### Stage-2 evidence gate: `s2_min_matched_rel`

A stage-2 candidate is now discarded unless its most intense matched peak is at
least `PARAM.s2_min_matched_rel` (default `0.01`) of the spectrum's base peak.
The hybrid score is mostly the fraction of *predicted* fragments matched, so a
single noise peak could previously carry a one-fragment prediction (e.g. DG 7:0_7:0
matched to a peak at 0.16% of the base peak). `0` disables the gate. The shipped
`data/lipidoracle.yaml` also sets `ms2_noise_prc` back to 0.3.

On `testdata/cid` this removes 16% of idlevel-2 features; stage-3 benchmarks and
the Martinez benchmark are unchanged.

---


## v1.0.264

### The reported ID level follows the reported name

`idlevel` now describes what the reported name claims, and `stage` describes
which stage derived it. The two used to be the same number, which made the
species-level collapses report more specificity than their names carried:

- `s2_compress_undefined_chains_to_species` (headgroup-only evidence) and
  `S2_unresolvable_chains_to_species` (the resolvability audit found no
  exclusive fragment) rewrite the name to the sum composition. Those rows are
  now `idlevel 1` instead of 2, with `stage` still 2.
- `s2_unresolvable_sn_to_unknown` is unaffected: the chain composition is
  still claimed, so the row stays at `idlevel 2`.

The audit verdict (`s2_resolvability`, `diag/s2_chain_resolvability.csv`) is
written whichever way the options are set, so what changes is the reported
level, never the measurement.

Internally, every decision that means "this spectrum was identified from its
MS2 data" (MS1 fallback suppression, RT training and protection, stage-3
parent admission) now keys on `stage`, so runs that collapse names behave
exactly as before apart from the reported level.

### Confidence tails and resolvability diagnostics are byte-reproducible

Two outputs depended on thread scheduling and hash order, so the same input and
config produced different bytes from run to run (found on liver-ead;
`dev/benchmarks/adduct_fragment_audit.md` has the measurements):

- The idlevel-4 confidence tail (`FA 20:3 [C=C: Δ3 65%, Δ6 65%, ...]`) is built
  from per-position marginals summed over a `HashMap` of position-tuples. Rust
  seeds that hasher per process, so the same f64s were added in a different
  order, and a last-bit difference is enough to swap two entries that print the
  same percentage. Every aggregating engine (ead1, ead2, ead3, had, uvpd) now
  sorts the tuple map before anything is summed over it, and the renderer orders
  printed entries on the printed percentage, ties falling to position number.
- `diag/s3_resolvability.csv` was written in worker-completion order. It is now
  sorted by `spec_uid` and then by its rendered cells -- `spec_uid` alone is not
  unique, because one parent spectrum contributes a row per candidate name.

Nothing asserted changes: idlevel-1/2/3 rows, idlevel-4 structural names and the
idlevel counts are identical across runs both before and after. What changes is
that `annotation.csv` and the diagnostic are now byte-identical across runs,
which is what makes a version-to-version diff readable.
`dev/check_run_reproducibility.sh` runs the check.

### The unresolvable-chain collapse is off by default

`S2_unresolvable_chains_to_species: True` rewrites an arrangement to its sum
composition when the resolvability audit finds no exclusive fragment separating
it from its best rival. The audit is right that the split is not proven, but the
withheld split is generally the correct one: on light-ead the chain assignment
was right in 58 further spectra that the collapse discarded, and stage 3, given
an unresolved composition instead of those chains, asserted nothing further
(`dev/benchmarks/benchmark2/README.md`, §5.3 and Tables 1.1-1.2). The shipped
`data/lipidoracle.yaml` therefore sets it `False`.

The default now reports the arrangement at `idlevel 2` with the *sn* claim
dropped (`s2_unresolvable_sn_to_unknown: True`), and the option remains
available for runs that must not name a split the data do not prove. Nothing
else changes: the verdict still lands in `s2_resolvability` and
`diag/s2_chain_resolvability.csv`, so the abstention stays visible and
measurable. The internal default in `src/config.rs` was already `false`; this
aligns the shipped YAML with it.

## v1.0.243

Stage 3's scoring and selection are now harmonized across all five acyl-chain
engines (EAD1, EAD2, OAD, UVPD, OzID). Two things change what a run produces
if it isn't already on the shipped defaults.

### 1. `score_s3` reports the actual posterior probability

Under the default `s3_selection: posterior`, `score_s3` used to be the old
per-spectrum log/MAD rescale: the top candidate was always pinned to exactly
`1.0`, and every candidate at or below a robust floor clamped flat to `0.0`,
regardless of how much evidence actually separated them. That column now
carries the candidate's posterior probability directly — the same quantity
`s3_post_credible_level`, `s3_post_odds_floor`, and `s3_post_min_mass` are
defined against. A decisive call now reads `0.98`; a three-way tie reads
`0.34`, `0.33`, `0.33` — instead of `1.0`, `1.0`, `1.0` for all three.

Emitted rows sum to the credible mass, not to `1`, since only the credible
set (plus a few context rows) gets written; mass pruned by the odds floor
stays visibly absent rather than being redistributed onto the survivors.
Because probabilities span many orders of magnitude, the column is now
formatted adaptively (up to six decimals, or scientific notation below
`1e-4`) instead of a fixed three decimals, which used to flatten a
meaningful fraction of idlevel-3 rows to an indistinguishable `0.000`.

The old rescale behavior is unchanged under `s3_selection: legacy`.

### 2. `s3_intensity_scaling` and `s3_score_rescaling` now apply to every engine

Both keys used to be inconsistent across engines:

- **`s3_intensity_scaling`** was read by EAD2 only; EAD1 and OAD silently
  hardcoded `sqrt` regardless of the setting. It now applies uniformly to
  EAD1, EAD2, OAD, and UVPD (OzID has no intensity-weighted score term to
  scale). EAD1's own fitted value is `sqrt`, but the shared default is
  `none` — set `s3_intensity_scaling: 'sqrt'` explicitly if you were relying
  on EAD1's old built-in behavior.
- **`s3_score_rescaling`** used to run unconditionally, even under
  `posterior` selection where its output was immediately discarded, and
  only on EAD1, EAD2, and OAD; UVPD and OzID ignored it entirely. It is now
  confined to `s3_selection: legacy` and applies to all five engines, and
  its default changed from `'mad'` to `'none'` — matching the Python
  reference's own unnormalized output. This only affects `legacy`
  selection; `posterior`, the default, was already unaffected by this key.

Both keys still accept the same values as before; only what they apply to,
and what happens when neither is set, has changed.

Nothing needs to be rewritten to upgrade for most users: `s3_selection:
posterior` is already the default, and idlevel-4 names are unchanged (a
softmax is monotone in the raw score, so ranking and selection are
unaffected). Only EAD1 users depending on its old implicit `sqrt` intensity
weighting, or anyone running `s3_selection: legacy` with a non-default
`s3_score_rescaling`, need to check their config.

## v1.0.195

### OAD and OzID stage-3 engines

LipidOracle can now localise acyl-chain double bonds from OAD and OzID spectra
in addition to EAD and UVPD.

- **OAD** implements the MS-DIAL `PCOadSpectrumGenerator` and common acyl OAD
  rules for unoxidised diacyl PC `[M+H]+` spectra. OAD02 and OAD15 are required
  anchors; the remaining OAD ions provide intensity-weighted ranking evidence.
  Unsupported classes, adducts, oxidised lipids, ether chains, and sphingoid
  bases fail closed.
- **OzID** matches formula-aware aldehyde/Criegee product pairs and supports
  the validated PC/PE `[M+H]+`, PI `[M+Na]+`, and DG/TG `[M+NH4]+` profiles.
  Complete diagnostic pairs drive localisation, with methyl-end notation and
  the PUFA double-bond-index correction handled automatically.
- Both engines use the shared posterior candidate selection and report their
  localisation uncertainty in the stage-3 diagnostics. They are selected with
  `WORKFLOW.stage3: oad` or `WORKFLOW.stage3: ozid`.

## v1.0.180

### 1. First implementation of Lipid Data Quality Scoring

The LDQ implements the scoring proposed by Hofmann et al. (*J Lipid Res.* 2025; 66(8):100817).
Scores are reported in the `ldq_*` columns.

## v1.0.173

Three changes matter to anyone reading LipidOracle output: names now follow
Shorthand 2020 exactly, the CXSMILES encoding of structural ambiguity was
repaired, and positional candidates are retained by posterior mass rather than
by a score ratio.

Nothing needs to be rewritten to upgrade. Every renamed parameter and every
legacy name spelling is still accepted, so existing `lipidoracle.yaml` files and
existing custom libraries load unchanged.

---

### 1. Shorthand 2020 nomenclature

The canonical `name` column is now a strict **Shorthand2020** name as defined by
Liebisch et al. (*J Lipid Res.* 2020;61(12):1539–1555).

- **Positional modifiers moved in front of the abbreviation**: `;11OH` and
  `;5oxo`, not the legacy `;OH(11)` / `;oxo(5)`.
- **Legacy spellings are normalised at every input boundary.** `;OH(11)`,
  `;hydroxy(11)`, `;oxo(5)`, `;ep(5)` and `;cyc(3)` all convert on load, so
  older libraries and project files keep working.
- **Full Table 1A coverage** — hydroxyl, ketone, methyl, ethyl, methoxy,
  hydroperoxy, amino, thiol, nitrile, nitro, extra carboxyl, phosphate,
  sulfate, halogens — plus Table 1B rings (`;5Ep`, `;[11-13cy3:0]`).
- **Names that cannot be read unambiguously are refused rather than guessed at.**
  A group on C1 of an acyl chain names the linkage, not a substituent. A
  localised group combined with unlocalised double bonds (`FA 18:2;5OH`) never
  says how the bonds divide across the split, so it returns nothing; give the
  positions and it resolves.

At idlevel 4, localisation confidence is appended to the name as a bracketed
tail rather than forced into the name body:

```
FA 20:4;OH [DB sn1: Δ5 100%, Δ8 100%, Δ12 100%, Δ14 50%, Δ15 50%; OH sn1: 11 100%]
```

The hydroxyl position lives in the tail because writing it into the body
(`FA 20:4;11OH`) would make the name unrepresentable — a group positioned
mid-chain alongside unlocalised bonds does not say how those bonds divide around
it.

### 2. CXSMILES support fixed

Extended CXSMILES is the canonical structure output, and its encoding of
ambiguity was reworked and validated against **CDK**, the reference
implementation for this format.

**Removed:**

- **`ctu:` blocks.** `ctu:` is a ChemAxon *query* feature meaning "match either
  configuration", intended for substructure search. Most toolkits already treat
  a bare `C=C` that way. Emitting it turned every partially determined structure
  into a query without adding any information. A double bond of undetermined
  geometry is now simply written `C=C`.
- **`f:` component groups and `RG:` R-groups** for unresolved sn-position. Both
  were unreadable to some common tooling, and the `RG:` form could not coexist
  with `Sg:` at all — CDK rejects a nested CXSMILES block inside an R-group
  definition, so a name carrying both kinds of ambiguity had to abandon one. An
  R-group also stated the wrong cardinality: read ordinarily it admits
  `PC 16:0/16:0` as well, and 27 assignments for a TG where the name allows 6.

**In their place**, an unresolved sn-assignment is carried by `$snN$` atom
labels on a **complete, connected molecule**, plus a `swappable(sn1,sn2)` token:

```
PC 16:0_18:1(9)
↓
C(COP(=O)([O-])OCC[N+](C)(C)C)(OC(=O)CCCCCCCC=CCCCCCCCC)COC(=O)CCCCCCCCCCCCCCC
|$;;;;;;;;;;;;;sn2;;;;;;;;;;;;;;;;;;;;;sn1$| swappable(sn1,sn2)
```

Every toolkit reads the structure, and the different kinds of ambiguity now
compose freely on the same molecule.

**Also corrected:** the position-variation (`m:`) block now points at a dummy
atom, as the format requires. An unlocalised hydroxyl is written `*O` — the `*`
takes the variable bond, the `O` is the group it carries. CDK silently ignores
an `m:` block pointing anywhere else, and other toolkits reject the string
outright.

**Confidence in the trailer.** Idlevel-4 percentages ride along as
`dbPos(...)` / `mPos(...)` tokens anchored to the same labels, so a name
converts to CXSMILES and back with its percentages intact:

```
FA 18:2 [DB sn1: Δ9 92%, Δ12 88%]
↓
OC(=O)CC=CC=CC |$;sn1$,Sg:n:3:a:ht,Sg:n:5:b:ht,Sg:n:8:c:ht|
constrain(a+b+c=14);dbPos(sn1:9@92,12@88)
```

Note that a **fully determined structure now carries no `|...|` tail at all** —
the presence of a tail is itself the signal that something was left open.

Two caveats if you consume the SMILES column: a renderer that ignores `Sg:`
fails *silently*, drawing a chain many carbons short, so expand before drawing;
and the trailer does not survive a round trip through a toolkit, because
`constrain(...)` and `swappable(...)` live in the title field and get dropped.

### 3. Posterior thresholding for candidate retention

`score_s3` is normalised against each spectrum's own best candidate, so the top
hit always reads 1.0 whether the evidence is overwhelming or absent. There was
no way for the output to say **"this position is not determined"**.

Thresholding on the score *ratio* — the old `ead_chain_cutoff` — inherits that
blindness. Two near-ties and two hundred near-ties both pass, and the number of
survivors says nothing about how well localised the answer is. Measured against
a null of chemically impossible (allene) placements, structures the enumerator
excludes precisely because they cannot occur, **an impossible candidate landed
within 1% of the best real one in 11.4% of features.** No value of the ratio
cutoff separates real from impossible.

The new default, `s3_selection: posterior`, converts raw scores into a
probability distribution with a softmax and keeps the smallest set of candidates
whose probabilities sum to `s3_post_credible_level`. The **size of that set is
the confidence statement**: decisive evidence yields one candidate, an
uninformative spectrum yields many. Positions enter the idlevel-4 name only when
one placement carries at least `s3_post_min_mass`; otherwise the chain is named
at species level and the tail still lists what is known.

The temperature is **fitted, not chosen** — swept against known positions on two
reference sets, where 0.01 maximises discrimination while keeping calibration
error low. Score gaps are taken relative to the top score, so one temperature is
meaningful for both engines despite their raw scores differing by orders of
magnitude.

The trade is coverage against correctness. On a mono-hydroxy reference set with
`ead2`:

| Selection | Positions asserted | Correct |
| --- | ---: | ---: |
| `legacy` (old behaviour) | 84 / 96 | 55 (65%) |
| `posterior` (new default) | 55 / 96 | 40 (73%) |

Set `s3_selection: legacy` to reproduce earlier results.

New diagnostic: `diag/s3_resolvability.csv` carries the posterior over
candidates, the per-position marginals, and the `margin` / `matched_n` columns
behind the idlevel-4 gates. It reports only; it gates nothing.

### 4. EAD engine `ead1` is now intensity-aware

Not headline news, but it changes results more than anything else in this
release.

`ead1`'s score was *matched weight / total predicted weight* — the fraction of
its own ladder that found any peak at all. Because the engine emits three
neutral-loss variants at every bond, that ladder covers nearly every nominal
mass in the window, so roughly 50 of 57 bins matched for **every** hypothesis
alike. On a typical spectrum the entire spread across chemically unrelated
structures was 0.880–0.890: the ranking inside that 1% band was noise, and a
0.99 relative cutoff kept all of it.

The score is now `Σ(weight × √intensity)` over matched peaks, mapped back into
`[0,1]` by the same log/MAD transform `ead2` uses. The Pearson correlation term
is gone, and `s3_min_score` applies after normalisation, where an absolute floor
on a 0–1 scale is meaningful.

| Reference set | Top-1 before | Top-1 after | idlevel-4 asserted | of those, correct |
| --- | ---: | ---: | ---: | ---: |
| `ara-ead` (114 spectra) | 28% | **80%** | 84 / 114 | 98% |
| `light-ead` (163 spectra) | 53% | **75%** | 100 / 163 | 93% |

Two new gates decide whether idlevel 4 claims a position at all, both fitted
against known positions: `s3_min_margin` (0.05) on the relative gap between the
best two candidates, read on the raw scale because normalisation pins the winner
at 1.0 by construction; and `s3_min_matched` (3), because a relative margin can
look decisive when the whole comparison rests on one peak.

The hand-tuned distance-weight array and its ×6 spike at distance 2 were
**kept** — flattening either collapses accuracy to 11–20%, so the published
method's empirical claim about EAD cleavage is load-bearing and correct. Only
its scoring was broken.

### 5. Parameter renames

Every idlevel-3/4 parameter is now prefixed `s3_`. **The old spellings are still
accepted as aliases**, so no config file needs editing.

| Old | New | Note |
| --- | --- | --- |
| `ead_max_unsaturated` | `s3_max_unsaturated` | |
| `ead_max_oxygen` | `s3_max_oxygen` | |
| `ead_max_candidates` | `s3_max_candidates` | |
| `ead_max_output` | `s3_max_output` | |
| `ead_min_score` | `s3_min_score` | now applied after normalisation |
| `ead_intensity_scaling` | `s3_intensity_scaling` | `ead2` only; `ead1` always applies `sqrt` |
| `include_ms1_monochain` | `s3_include_ms1_monochain` | |
| `ead_chain_cutoff` | `s3_legacy_cutoff` | now used only when `s3_selection: legacy` |
| `ead_chain_corr_weight` | `s3_chain_corr_weight` | no longer used by any engine |

The published parameter reference also had drifted from the shipped
`lipidoracle.yaml` and has been corrected: `s3_max_unsaturated` is 6 (documented
as 2), `s3_max_candidates` is 10000 (documented as 30000), and
`s3_legacy_cutoff` is 0.99 (documented as 0.95). These are documentation fixes,
not behaviour changes.

New in this release: `s3_selection`, `s3_post_credible_level`,
`s3_post_min_mass`, `s3_post_temperature`, `s3_min_margin`, `s3_min_matched`,
`s3_null_decoys`.

`WORKFLOW.stage3` now prefers the engine names `ead1` / `ead2` / `uvpd`.
`True`, `v1` and `v2` remain accepted.

---

## Documentation

- [Acyl-chain analysis](https://zamboni-lab.github.io/lipidoracle/stage3.html)
  — the two engines, candidate selection, and every `s3_` parameter.
- [Nomenclature and CXSMILES](https://zamboni-lab.github.io/lipidoracle/nomenclature.html)
  — Shorthand 2020, how ambiguity is encoded, and what the confidence tail means.
- [Parameters](https://zamboni-lab.github.io/lipidoracle/params.html)
  — every setting in `lipidoracle.yaml`.
