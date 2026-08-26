[Top level](../README.md)

# Alt SSIM Tuning: Unbounded Fixed-Reference Lambda Scaling (Design)

## Table of Contents
__[TOC]__

## Overview

`--alt-ssim-tuning 1` (tune 2 only) enables an alternative SSIM rate-distortion
pathway in SVT-AV1-HDR: it replaces the plain 8x8 variance activity measure
with a mid-tone-weighted perceptual variance computed over 4x4, 8x8 and 16x16
blocks, and it reduces the SSIM full-cost level from `SSIM_LVL_3` to
`SSIM_LVL_1` in the mode decision stage.

The per-block lambda scaling at the core of the tune is **unbounded**: the
scaling factors are mapped to lambda through a **fixed reference constant**
instead of a per-superblock geometric-mean normalization. Absolute block
activity - not just activity relative to a superblock mean - drives per-block
lambda, so quality can be reallocated freely across the whole frame and across
frames, while a symmetric log-domain clamp bounds worst-case lambda swings.

The design follows the unbounded concept introduced for the SSIMULACRA2 tune
(see `Docs/Appendix-SSIMULACRA2-Tune.md`): a fixed reference point, a signed
log-domain curve, and no frame-derived normalization.

This document covers the current behavior, the problem with the old
normalization, the design space, the chosen design, its properties, and the
implementation status.

## Current Behavior

### Factor generation

`aom_av1_set_mb_ssim_rdmult_scaling()` (Source/Lib/Codec/src_ops_process.c)
computes, once per frame in the source-based operations (SBO) stage, a scaling
factor for every 16x16 block of the frame:

1. **Activity measure.** Without the alt tune, the per-16x16 variance is the
   mean of the four contained 8x8 per-pixel variances
   (`svt_aom_get_perpixel_variance`). With the alt tune, a perceptual variance
   (`svt_aom_get_perceptual_perpixel_variance`) is used instead, computed on
   4x4, 8x8 and 16x16 blocks and combined with the fixed weights
   $\tfrac{1}{16} : \tfrac{1}{8} : \tfrac{1}{4}$:
   a mid-grey (mean 128) parabolic weight boosts variance of mid-tone content,
   making the encoder treat mid-tone texture as more complex. Both measures
   operate on the encoder's 8-bit analysis picture (`enhanced_pic`).
2. **Exponential curve.** The activity $`v`$ is mapped through

   $`f(v) = a\,(1 - e^{b v}) + c, \quad a = 67.035434,\; b = -0.0021489,\; c = 17.492222`$

   giving $`f \in [17.49, 84.53]`$ (asserted in code). High activity maps to a
   high factor; high factors raise lambda, spending fewer bits where
   compression artifacts are masked by texture.
3. **Mapping** (the subject of this document):
   - default tune 2: each factor is divided by the frame geometric mean of the
     raw factors (per-frame normalization, unchanged);
   - alt tune: the raw curve output is mapped through a **fixed-reference
     log-domain curve** (below). No per-frame or per-superblock normalization
     is applied.

### The fixed-reference mapping (alt tune)

The raw curve output $`f`$ is normalized by the fixed curve constants - not by
any frame statistics - to the unit interval, then mapped to a signed log2
lambda factor anchored at the curve midpoint:

$`f_{norm} = \frac{f - c}{a} \in [0, 1]`$

$`\log_2(\mathrm{factor}) = 2.0 \cdot (f_{norm} - 0.5)`$

$`\log_2(\mathrm{factor}) = \mathrm{CLIP}(-2.0,\, 2.0,\; \log_2(\mathrm{factor}))`$

$`\mathrm{factor} = 2^{\log_2(\mathrm{factor})} \in [0.25, 4.0]`$

The reference point 0.5 corresponds to the curve midpoint $`c + a/2 \approx
51.0`$: mid-activity content maps to factor 1, flat blocks to factors below 1
(more bits - artifacts are visible), busy blocks to factors above 1 (fewer
bits - artifacts are masked). The reference is a constant, deliberately never
derived from the current frame, so the frame-level average factor is free to
deviate from 1 and the encoder can move bits across superblock boundaries and
across frames.

### Consumption

The factors live in `pa_me_data->ssim_rdmult_scaling_factors[]`
(PictureParentControlSet). At encode time, `aom_av1_set_ssim_rdmult()`
(Source/Lib/Codec/mode_decision.c) aggregates the 16x16 factors covered by the
current coding block with a geometric mean and multiplies the block lambda
(`full_lambda_md`, `fast_lambda_md`) by it. It is invoked from:

- `md_encode_block()` (product_coding_loop.c), the default-lambda path;
- the tail of `svt_aom_set_tuned_blk_lambda()` (mode_decision.c), the
  TPL-lambda path;
- the coding loop (coding_loop.c).

Because the factors are $`2^{l}`$ with $`l`$ the per-block log2 factor, the
block-level geometric mean is exactly the arithmetic mean of the covered log2
factors - a local summary, not a normalization. It cannot re-bind the frame, so
consumption is unchanged by this design.

The tune gates (enc_handle.c) restrict `alt_ssim_tuning` to tune 2; the SSIM
full-cost level is selected in product_coding_loop.c
($`SSIM\_LVL\_1`$ for the alt tune, $`SSIM\_LVL\_3`$ otherwise).

## The Problem (with the previous design)

The previous alt-tune implementation normalized each factor by the geometric
mean of its own superblock, forcing every superblock's factors to average 1.
It discarded the **absolute** activity level of each superblock: a flat sky
superblock and a busy texture superblock both got a relative factor
distribution centered on 1, so the encoder could not move bits from busy
regions to flat regions *across* superblock boundaries, and flat frames and
busy frames were treated identically at the frame level. Only within-superblock
relative variation survived.

The intended behavior is the opposite: a flat block should get a *lower* lambda
(more bits - artifacts are visible), a busy block a *higher* lambda (fewer
bits - artifacts are masked), with the magnitudes set by the absolute activity
measured on the source, free to deviate anywhere in the frame and across
frames.

## Design Space

Any dimensionless per-block multiplier needs a **reference** that maps the
activity measure to "factor = 1": a monotone map from variance to a multiplier
has no natural origin. The design space is therefore "where does the reference
come from". Options considered:

| Reference | Mechanism | Per-frame freedom | Long-run bias | Verdict |
|---|---|---|---|---|
| Fixed constant | `f` anchored at the curve midpoint | kept | content-class dependent by design | **chosen** (mirrors the SSIMULACRA2 tune) |
| Adaptive window + scene-cut reset | ring of last K frames, cleared on scene change | kept | 0 by construction | superseded (see Notes) |
| Exponential moving average (EMA) | `a += α·(m − a)` | kept | 0 by construction | rejected: slow + eternal tail |
| Rank/quantile ladder | factor by activity rank | within-frame only | 0 per frame | **rejected**: GM = 1 per frame in disguise |
| Rate-control compensation | nudge frame QP by factor GM | none | 0 | **rejected**: mathematically identical to per-frame GM normalization |
| Curve re-parameterization | retune `a`, `b`, `c` so output is centered near 1 | kept | depends on corpus coverage | rejected: bit-identical to dividing by a constant; fragile |

## Chosen Design: Fixed Reference, Log-Domain Curve

### Rationale

The fixed-reference design is stateless and structurally cannot re-normalize
the frame:

- **No per-frame normalization**: the reference is the constant 0.5 (curve
  midpoint). A frame's factors are never centered by that same frame's
  statistics; the frame GM deviates from 1 freely, which is the point of the
  feature.
- **Bounded**: the factor range is hard-bounded to $`[0.25, 4.0]`$ (one
  log2 factor of $`\pm 2`$), so worst-case lambda swings are bounded
  regardless of content - no transients to manage, no cold-start phase.
- **Stateless**: no fields on `SequenceControlSet`, no scene-cut reset, no
  cross-channel contamination, no superres-recode double-counting (recodes
  skip the SBO kernel entirely). Scene changes need no special handling
  because the reference is immune to content statistics.
- **Deterministic**: a pure function of the current frame's activity.

### Behavior

- **Flat block** ($`f_{norm} < 0.5`$): factor < 1, lambda down, bits up.
- **Mid-activity block** ($`f_{norm} \approx 0.5`$): factor ≈ 1.
- **Busy block** ($`f_{norm} > 0.5`$): factor > 1, lambda up, bits down.
- **Frame level**: because the alt perceptual variance boosts mid-tone
  activity (up to ~2x, src_ops_process.c), curve inputs are inflated at
  mid-tones and $`f`$ clusters near the curve ceiling, so $`f_{norm}`$ skews
  above 0.5 and the average factor is typically above 1. The alt tune
  therefore tends to *reduce* per-frame bitrate at fixed CRF relative to a
  normalized reference; rate control absorbs the per-frame deviation at the QP
  level, and the clamp bounds the systematic deviation to `[0.25, 4.0]`.

## Interaction with Rate Control

The alt tune deliberately allows the *frame-level* factor mean to deviate from
1, so per-frame bitrate moves with content (flat scenes gain bits, busy scenes
lose them). Rate control absorbs this at the QP level, as it does for any
lambda change. The clamped factor range keeps the deviation bounded so RC never
fights a systematic drift larger than `[0.25, 4.0]`. A resulting file-size
change at fixed CRF versus the previous per-superblock-normalized alt tune is
expected and is the point of the feature.

## Implementation Status

Implemented on the `unbounded_ssim` branch:

1. **src_ops_process.c** (`aom_av1_set_mb_ssim_rdmult_scaling`, alt branch):
   the per-superblock geometric-mean normalization was replaced by the
   fixed-reference log-domain mapping above. The superblock variables
   (`num_rows_sb`, `num_cols_sb`, `num_blk_w`, `num_blk_h`) were deleted. The
   frame-mean path (default tune 2), the activity measures, the curve, and the
   pre-transform `assert(var > 17.0 && var < 85.0)` are unchanged.
2. **Consumption**: unchanged. `aom_av1_set_ssim_rdmult()` still aggregates
   covered factors with a geometric mean; with $`\mathrm{factor} = 2^{l}`$
   this is the arithmetic mean of the log2 factors, and no normalization is
   reintroduced at the block level.
3. **Docs**: README.md and Docs/Parameters.md updated to describe the
   unbounded behavior.

No new configuration parameters were added; the alt tune is still enabled with
`--alt-ssim-tuning 1` alone. The mapping constants (gain 2.0, reference 0.5,
clamp [0.25, 4.0]) are tuned with the curve; see the optional companion knobs
below for the extension point if tuning experiments call for them.

## Safety: Lambda Clamp

The per-block factor is clamped in the log domain in
`aom_av1_set_mb_ssim_rdmult_scaling` after the fixed-reference mapping:

```c
log2_factor = CLIP3(-2.0, 2.0, log2_factor);
```

Because the consumption-side aggregation is a geometric mean of clamped
factors, the aggregated block multiplier is also bounded by `[0.25, 4.0]`;
no separate clamp is needed at the consumption site. The clamp engages only
for extreme activity (flat content at the curve floor, or texture/grain at the
ceiling) and bounds worst-case lambda swings.

## Optional Companion Knobs (not implemented)

Two orthogonal knobs were considered but intentionally not added:

- a **strength** exponent applied to the aggregated block multiplier -
  $`\mathrm{factor}^\gamma`$ - scaling the contrast of the quality
  adjustments (how much a busy block gives up versus a flat block);
- a **global bias** shifting the log2 factor uniformly - equivalent to
  retuning the reference constant.

Both are single-line extensions of the mapping
($`\log_2 = \mathrm{bias} + \gamma \cdot 2.0 \cdot (f_{norm} - 0.5)`$) and can
be layered in later if tuning experiments call for them.

## Notes and Limitations

- **Superseded design**: an earlier version of this document proposed an
  adaptive reference window (geometric mean of the last K frames' activity
  statistics, cleared on scene changes) as the replacement for the
  per-superblock normalization. The fixed-reference design was chosen instead
  for its statelessness, its bounded behavior with no transients, and its
  symmetry with the SSIMULACRA2 tune. The window design remains a viable
  alternative if content-class-dependent average bitrate is ever undesired.
- **Content-class dependence**: with a fixed reference, a permanently-flat
  movie keeps below-1 factors forever (more bits per frame), and a
  permanently-busy movie the opposite. This is the intended unbounded
  behavior; rate control absorbs it at the QP level, and the clamp bounds it.
- The perceptual variance path operates on the encoder's 8-bit analysis
  picture, as before; high-bit-depth behavior is unchanged by this design.
- Superres recodes skip the SBO kernel early (src_ops_process.c), so a
  recoded frame's factors are computed once, on the coded (non-recode)
  pass.
- The gain (2.0), reference (0.5) and clamp ([0.25, 4.0]) constants are
  heuristics mirroring the SSIMULACRA2 tune; they are subject to tuning
  experiments.

## Validation Plan

1. Build (release) and encode tune 2 `--alt-ssim-tuning 1` before/after on:
   - a flat-heavy clip (sky/faces) - expect a quality uplift and a bitrate
     increase on flat regions;
   - a busy clip (texture/film grain) - expect bit savings on busy regions;
   - a multi-scene clip with hard cuts - verify no factor anomaly after cuts
     (the design is stateless, so none is expected);
   - a 10-bit HDR sample - verify no assert trips and no lambda anomalies.
2. Confirm `assert(17.0 < var && var < 85.0)` (src_ops_process.c) still
   holds (it is evaluated pre-mapping, so it does).
3. Confirm per-frame SSIM and total-bitrate deltas are in the expected
   direction and that per-frame factor means deviate from 1 on a single-scene
   clip (a debug print of the factors across frames, via the `do_print`
   diagnostics in `aom_av1_set_mb_ssim_rdmult_scaling`).
4. Confirm default tune 2 (`--alt-ssim-tuning 0`) output is bit-identical
   (its frame-normalization path is untouched).
5. Run `SvtAv1E2ETests` as an API regression smoke test (the SBO factor path
   has no unit-test coverage).

## References

- `aom_av1_set_mb_ssim_rdmult_scaling` / `aom_av1_set_ssim_rdmult`:
  Source/Lib/Codec/src_ops_process.c, Source/Lib/Codec/mode_decision.c
- `--alt-ssim-tuning` introduction: README.md, commit 05c38c878
- The unbounded fixed-reference concept is shared with the SSIMULACRA2 tune:
  Docs/Appendix-SSIMULACRA2-Tune.md
