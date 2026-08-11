# What's new

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

`score_L3` is normalised against each spectrum's own best candidate, so the top
hit always reads 1.0 whether the evidence is overwhelming or absent. There was
no way for the output to say **"this position is not determined"**.

Thresholding on the score *ratio* — the old `ead_chain_cutoff` — inherits that
blindness. Two near-ties and two hundred near-ties both pass, and the number of
survivors says nothing about how well localised the answer is. Measured against
a null of chemically impossible (allene) placements, structures the enumerator
excludes precisely because they cannot occur, **an impossible candidate landed
within 1% of the best real one in 11.4% of features.** No value of the ratio
cutoff separates real from impossible.

The new default, `l3_selection: posterior`, converts raw scores into a
probability distribution with a softmax and keeps the smallest set of candidates
whose probabilities sum to `l3_post_credible_level`. The **size of that set is
the confidence statement**: decisive evidence yields one candidate, an
uninformative spectrum yields many. Positions enter the idlevel-4 name only when
one placement carries at least `l3_post_min_mass`; otherwise the chain is named
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

Set `l3_selection: legacy` to reproduce earlier results.

New diagnostic: `diag/l3_resolvability.csv` carries the posterior over
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
is gone, and `l3_min_score` applies after normalisation, where an absolute floor
on a 0–1 scale is meaningful.

| Reference set | Top-1 before | Top-1 after | idlevel-4 asserted | of those, correct |
| --- | ---: | ---: | ---: | ---: |
| `ara-ead` (114 spectra) | 28% | **80%** | 84 / 114 | 98% |
| `light-ead` (163 spectra) | 53% | **75%** | 100 / 163 | 93% |

Two new gates decide whether idlevel 4 claims a position at all, both fitted
against known positions: `l3_min_margin` (0.05) on the relative gap between the
best two candidates, read on the raw scale because normalisation pins the winner
at 1.0 by construction; and `l3_min_matched` (3), because a relative margin can
look decisive when the whole comparison rests on one peak.

The hand-tuned distance-weight array and its ×6 spike at distance 2 were
**kept** — flattening either collapses accuracy to 11–20%, so the published
method's empirical claim about EAD cleavage is load-bearing and correct. Only
its scoring was broken.

### 5. Parameter renames

Every idlevel-3/4 parameter is now prefixed `l3_`. **The old spellings are still
accepted as aliases**, so no config file needs editing.

| Old | New | Note |
| --- | --- | --- |
| `ead_max_unsaturated` | `l3_max_unsaturated` | |
| `ead_max_oxygen` | `l3_max_oxygen` | |
| `ead_max_candidates` | `l3_max_candidates` | |
| `ead_max_output` | `l3_max_output` | |
| `ead_min_score` | `l3_min_score` | now applied after normalisation |
| `ead_intensity_scaling` | `l3_intensity_scaling` | `ead2` only; `ead1` always applies `sqrt` |
| `include_ms1_monochain` | `l3_include_ms1_monochain` | |
| `ead_chain_cutoff` | `l3_legacy_cutoff` | now used only when `l3_selection: legacy` |
| `ead_chain_corr_weight` | `l3_chain_corr_weight` | no longer used by any engine |

The published parameter reference also had drifted from the shipped
`lipidoracle.yaml` and has been corrected: `l3_max_unsaturated` is 6 (documented
as 2), `l3_max_candidates` is 10000 (documented as 30000), and
`l3_legacy_cutoff` is 0.99 (documented as 0.95). These are documentation fixes,
not behaviour changes.

New in this release: `l3_selection`, `l3_post_credible_level`,
`l3_post_min_mass`, `l3_post_temperature`, `l3_min_margin`, `l3_min_matched`,
`l3_null_decoys`.

`WORKFLOW.acyl_analysis` now prefers the engine names `ead1` / `ead2` / `uvpd`.
`True`, `v1` and `v2` remain accepted.

---

## Documentation

- [Acyl-chain analysis](https://zamboni-lab.github.io/lipidoracle/idlevel3.html)
  — the two engines, candidate selection, and every `l3_` parameter.
- [Nomenclature and CXSMILES](https://zamboni-lab.github.io/lipidoracle/nomenclature.html)
  — Shorthand 2020, how ambiguity is encoded, and what the confidence tail means.
- [Parameters](https://zamboni-lab.github.io/lipidoracle/params.html)
  — every setting in `lipidoracle.yaml`.
