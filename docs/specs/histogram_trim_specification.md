# Specification: Native Histogram Trim Operators

## 1. Overview

PromQL provides two binary operators for trimming native histograms:

| Operator | Name | Syntax | Semantics |
|----------|------|--------|-----------|
| `</` | TRIM_UPPER | `histogram </ threshold` | Keep observations **below** the threshold; remove observations **above** it |
| `>/` | TRIM_LOWER | `histogram >/ threshold` | Keep observations **above** the threshold; remove observations **below** it |

The left-hand side (LHS) must be a native histogram (either exponential or
NHCB). The right-hand side (RHS) must be a float scalar or float vector value
serving as the trim threshold. The reverse form (`float </ histogram` or
`float >/ histogram`) is not supported.

The operators work with both instant vector/scalar pairs and vector/vector
pairs (with standard PromQL label matching via `on`/`ignoring`).

The `bool` modifier is not supported with trim operators.

## 2. High-Level Algorithm

Given a histogram `H` and a threshold `T`:

1. **Copy** the histogram to avoid mutating the original.
2. **Iterate** over all positive buckets, negative buckets, and the zero
   bucket.
3. For each bucket, determine one of three cases:
   - **Fully retained**: the entire bucket falls within the kept range.
   - **Fully discarded**: the entire bucket falls outside the kept range (set
     its count to 0).
   - **Partially trimmed**: the threshold falls inside the bucket; interpolate
     to estimate the fraction of observations to keep.
4. **Recalculate** the histogram's `Count` and `Sum` from the surviving
   (possibly interpolated) bucket populations.
5. **Compact** the histogram: remove zero-count buckets and merge spans.

If no bucket was modified (the trim is a no-op), `Count` and `Sum` are
preserved exactly from the original histogram.

## 3. Bucket Classification

### 3.1 TRIM_UPPER (`</`)

For each bucket with bounds `[lower, upper]`:

| Condition | Action |
|-----------|--------|
| `upper <= T` | Fully retained |
| `lower < T < upper` | Partially trimmed (interpolate) |
| `lower >= T` | Fully discarded |

### 3.2 TRIM_LOWER (`>/`)

For each bucket with bounds `[lower, upper]`:

| Condition | Action |
|-----------|--------|
| `lower >= T` | Fully retained |
| `lower < T < upper` | Partially trimmed (interpolate) |
| `upper <= T` | Fully discarded |

## 4. Interpolation

When the threshold `T` falls strictly inside a bucket `[lower, upper]` with
count `C`, the implementation estimates how many observations fall below `T`
(the "under-count") and how many fall above. The interpolation method depends
on the histogram type.

### 4.1 Exponential Histograms

Exponential histogram buckets grow geometrically, so observations are assumed
to be **log-uniformly** distributed within each bucket. The fraction of
observations at or below `T` is:

**Positive buckets** (both bounds positive):

```
fraction = (log2(T) - log2(lower)) / (log2(upper) - log2(lower))
```

**Negative buckets** (both bounds negative):

```
fraction = 1 - (log2(|T|) - log2(|upper|)) / (log2(|lower|) - log2(|upper|))
```

The under-count is `C * fraction`.

### 4.2 Custom Bucket Histograms (NHCB)

NHCB buckets have arbitrary boundaries, so observations are assumed to be
**uniformly** (linearly) distributed within each bucket:

```
fraction = (T - lower) / (upper - lower)
```

The under-count is `C * fraction`.

### 4.3 Zero Bucket (Exponential Histograms Only)

The zero bucket spans `[-ZeroThreshold, +ZeroThreshold]` and always uses
**linear** interpolation, but its effective bounds are adjusted depending on
which other buckets exist in the histogram.

**Important**: The presence of "positive buckets" and "negative buckets" is
determined from the **original** (pre-trim) histogram — specifically, whether
any non-zero-count positive or negative buckets exist before trimming begins.
This means that even if all positive buckets are trimmed away, the zero bucket
still uses the bias from the original histogram. This is intentional: the
distributional assumption used for trimming should remain consistent with the
assumption used for sum estimation.

| Original histogram has... | Effective lower bound | Effective upper bound |
|---|---|---|
| Both positive and negative non-zero-count buckets | `-ZeroThreshold` | `+ZeroThreshold` |
| Only positive non-zero-count buckets | `0` | `+ZeroThreshold` |
| Only negative non-zero-count buckets | `-ZeroThreshold` | `0` |
| No non-zero-count positive or negative buckets | `-ZeroThreshold` | `+ZeroThreshold` |

The rationale: if only positive (resp. negative) buckets exist, all
observations in the zero bucket are assumed to be non-negative
(resp. non-positive). If neither exists, symmetry is assumed.

Note: Buckets with count exactly 0 in the input are skipped and do not count
as "existing" for the purpose of this bias determination.

With effective bounds `[L, U]`:

**TRIM_UPPER** (`</`):

| Condition | Kept count | Midpoint for sum |
|-----------|-----------|-----------------|
| `T <= L` | 0 | n/a |
| `T >= U` | `ZeroCount` | `(L + U) / 2` |
| `L < T < U` | `ZeroCount * (T - L) / (U - L)` | `(L + T) / 2` |

**TRIM_LOWER** (`>/`):

| Condition | Kept count | Midpoint for sum |
|-----------|-----------|-----------------|
| `T <= L` | `ZeroCount` | `(L + U) / 2` |
| `T >= U` | 0 | n/a |
| `L < T < U` | `ZeroCount * (U - T) / (U - L)` | `(T + U) / 2` |

## 5. Infinity Bucket Handling

Infinity buckets arise in NHCB histograms (the first bucket may have lower
bound `-Inf`; the last bucket always has upper bound `+Inf`). They also arise
as overflow buckets in exponential histograms, though these are rarely
populated in practice.

The general principle is **conservative**: if the distribution within an
infinity bucket cannot be estimated, the entire bucket is discarded rather
than guessed at.

**Important architectural note**: The infinity bucket handling described in
this section only applies to the **"partially trimmed"** case — i.e., when
the main loop (Section 3) determines that `lower < T < upper`. The main
loop's "fully retained" and "fully discarded" classifications take precedence
and are evaluated first, so the infinity handler never sees those cases.

### 5.1 Bucket with Lower Bound `-Inf`

The guiding principle for `-Inf`-lower-bound buckets is that observations in
an infinity bucket are considered **infinitely far away** from any finite
value. For a `[-Inf, X]` bucket, observations are assumed to be at `-Inf`.

The conditions below are evaluated **in order** (first match wins):

#### TRIM_UPPER (`</`):

1. `T >= upper` — Keep entire bucket. All observations are at `-Inf`, which
   is below any finite `T`. (In practice, this case is handled by the main
   loop's "fully retained" classification before infinity handling is
   reached.)
2. `upper > 0` and `T > 0` and `upper` is finite — **Special case**: treat
   lower bound as `0`; interpolate linearly: kept count = `C * T / upper`,
   midpoint = `T / 2`.
3. `upper <= 0` — Keep entire bucket; midpoint = `T`. Since observations are
   at `-Inf`, they are below any finite `T`, so all are retained. The
   midpoint `T` represents the upper end of the surviving interval
   `[-Inf, T]`.
4. Otherwise — Discard entire bucket (cannot estimate distribution).

#### TRIM_LOWER (`>/`):

1. `T >= 0` and `upper > T` and `upper` is finite — **Special case**: treat
   lower bound as `0`; interpolate linearly: kept count =
   `C * (1 - T/upper)`, midpoint = `(T + upper) / 2`.
2. Otherwise — Discard entire bucket. Since observations are at `-Inf`, they
   are below any finite `T`, and TRIM_LOWER removes observations below `T`.

**Rationale for the special case**: The first NHCB bucket `[-Inf, X]` where
`X > 0` conventionally represents observations in `[0, X]`. This is
consistent with `histogram_quantile`'s treatment of this bucket. The effective
lower bound of `0` is used for both count interpolation and sum estimation.

### 5.2 Bucket with Upper Bound `+Inf`

Observations in a `[X, +Inf]` bucket are considered to be at `+Inf`.

#### TRIM_UPPER (`</`):

Always discard the entire bucket. Since observations are at `+Inf`, they are
above any finite threshold `T`, and TRIM_UPPER removes observations above
`T`.

Note: When `T = +Inf`, the main loop classifies the bucket as "fully
retained" (`upper <= T` is `+Inf <= +Inf` = true) before the infinity
handler is ever reached.

#### TRIM_LOWER (`>/`):

For the "partially trimmed" case (where `lower < T`), the infinity handler
keeps the entire bucket with midpoint = `T`. Since observations are at `+Inf`,
they are above any finite `T`, so all are retained. The midpoint `T`
represents the lower end of the surviving interval `[T, +Inf]`.

Note: When `T <= lower`, the main loop classifies the bucket as "fully
retained" (`lower >= T`) before the infinity handler is reached. When
`T >= +Inf`, the main loop classifies it as "fully discarded". So the
infinity handler only sees the case `lower < T < +Inf`, where it always
keeps the bucket.

### 5.3 Bucket `[-Inf, +Inf]` (Single-Bucket NHCB)

This is a degenerate case where a custom bucket histogram has a single bucket
spanning `[-Inf, +Inf]`. Since neither bound is finite, no interpolation is
possible:

- `</ +Inf`: Keep (the `T >= upper` path triggers)
- `>/ -Inf`: Keep (the `T <= lower` path triggers)
- Any finite threshold or `</ -Inf` or `>/ +Inf`: Discard

## 6. Sum Recalculation

When any bucket is trimmed (partially or fully), the histogram's `Sum` is
**recomputed from scratch** based on the surviving observations. The original
`Sum` is not used.

For each surviving bucket (or surviving portion of a partially trimmed
bucket), the sum contribution is:

```
contribution = kept_count * midpoint
```

where `midpoint` is the representative value for the surviving interval.

### 6.1 Midpoint Calculation

The midpoint depends on the histogram type and the surviving interval
`[lower, upper]`:

#### Finite bounds:

| Histogram type | Midpoint |
|---------------|----------|
| NHCB (linear) | `(lower + upper) / 2` (arithmetic mean) |
| Exponential, positive bucket | `sqrt(abs(lower * upper))` (geometric mean, always positive) |
| Exponential, negative bucket | `-sqrt(abs(lower * upper))` (geometric mean, always negative) |

#### Infinite bounds:

| Bounds | Midpoint |
|--------|----------|
| Both `-Inf` and `+Inf` | `0` |
| Lower is `-Inf`, upper > 0 | `upper / 2` |
| Lower is `-Inf`, upper <= 0 | `upper` |
| Upper is `+Inf` | `lower` |

#### Partially trimmed buckets:

When a bucket is partially trimmed, the "surviving interval" is
`[bucket.lower, T]` for TRIM_UPPER or `[T, bucket.upper]` for TRIM_LOWER.
The midpoint is computed for this narrowed interval using the same rules
above.

### 6.2 Zero bucket midpoint

The zero bucket midpoint is always computed using the arithmetic mean of the
effective bounds (after the bias adjustment described in Section 4.3), using
the narrowed interval if partially trimmed.

### 6.3 No-op optimization

If no bucket was modified at all, the original `Count` and `Sum` are
preserved exactly. The sum is only recomputed when at least one bucket was
trimmed.

## 7. Boundary Thresholds

### 7.1 Threshold on a Bucket Boundary

When `T` exactly equals a bucket boundary, no interpolation is needed:

- **TRIM_UPPER** (`</`): A bucket with `upper == T` is fully retained; a
  bucket with `lower == T` is fully discarded.
- **TRIM_LOWER** (`>/`): A bucket with `lower == T` is fully retained; a
  bucket with `upper == T` is fully discarded.

### 7.2 Threshold at +/-Inf

| Expression | Result |
|-----------|--------|
| `H </ +Inf` | No-op (keep everything) |
| `H >/ -Inf` | No-op (keep everything) |
| `H </ -Inf` | Empty histogram (discard everything) |
| `H >/ +Inf` | Empty histogram (discard everything) |

## 8. Output Histogram Properties

The output histogram preserves the following from the input:

- **Schema**: unchanged
- **ZeroThreshold**: unchanged
- **CustomValues**: unchanged (for NHCB)

The following are recomputed:

- **Count**: sum of all surviving bucket counts
- **Sum**: recomputed from surviving bucket midpoints (see Section 6)
- **PositiveBuckets / NegativeBuckets**: individual counts modified or zeroed
- **ZeroCount**: modified if the zero bucket was (partially) trimmed
- **Spans**: compacted to remove zero-count buckets

## 9. Post-Processing: Compaction

After trimming, `Compact(0)` is called on the histogram. This:

1. Removes buckets whose count is 0.
2. Merges adjacent spans where gaps have been eliminated.
3. Adjusts span offsets accordingly.

This ensures the output histogram has a minimal, clean representation with
no unnecessary zero-count buckets.

## 10. Vector Matching

Trim operators support standard PromQL vector matching semantics:

- **vector / scalar**: `histogram_vector </ 2.5` trims each histogram in
  the vector at threshold 2.5.
- **vector / vector**: `histogram_vector >/ on (label) float_vector` matches
  histogram and float series by labels, using each matched float as the
  threshold for the corresponding histogram.

Standard `on()` / `ignoring()` / `group_left()` / `group_right()` modifiers
apply.

## 11. Detailed Edge Case Examples

### 11.1 Zero Bucket Only (No Positive or Negative Buckets)

Given: `{schema:0, count:5, sum:0, z_bucket:5, z_bucket_w:0.1}`

The zero bucket covers `[-0.1, +0.1]`. With no other buckets, symmetry is
assumed (effective bounds remain `[-0.1, +0.1]`).

| Expression | Effective range kept | Kept count | Sum |
|-----------|---------------------|-----------|-----|
| `>/ 0` | `[0, 0.1]` | 2.5 | 0.125 |
| `</ 0` | `[-0.1, 0]` | 2.5 | -0.125 |
| `>/ 0.05` | `[0.05, 0.1]` | 1.25 | 0.09375 |
| `</ -0.05` | `[-0.1, -0.05]` | 1.25 | -0.09375 |
| `>/ 0.1` | empty | 0 | 0 |
| `</ -0.1` | empty | 0 | 0 |

### 11.2 Positive-Biased Zero Bucket

Given: `{schema:0, sum:8.02..., count:12, z_bucket:2, z_bucket_w:0.5, buckets:[10]}`

Only positive buckets exist, so the zero bucket effective bounds are `[0, 0.5]`.

| Expression | Effect |
|-----------|--------|
| `>/ 0` | No-op (everything is >= 0) |
| `</ 0` | Empty (nothing is < 0) |
| `>/ 0.1` | Zero bucket trimmed: keep `2 * (0.5-0.1)/(0.5-0) = 1.6` |
| `</ 0.5` | Positive buckets discarded, zero bucket fully kept (count=2) |

### 11.3 NHCB First Bucket `[-Inf, X]` with `X > 0`

Given: `{schema:-53, sum:33, count:101, custom_values:[5], buckets:[1 100]}`

Buckets are `[-Inf, 5]` (count=1) and `(5, +Inf]` (count=100).

The first bucket `[-Inf, 5]` has its lower bound treated as `0` since
`upper > 0` (per Section 5.1):

| Expression | First bucket `[-Inf, 5]` | Second bucket `(5, +Inf]` | Total result |
|-----------|-------------------------|--------------------------|-------------|
| `</ 2.0` | Interpolate in `[0, 5]`: keep `0.4`, midpoint `1.0` | Discarded (infinity bucket, TRIM_UPPER) | count=0.4, sum=0.4 |
| `>/ 2.0` | Interpolate in `[0, 5]`: keep `0.6`, midpoint `3.5` | Fully retained (lower=5 >= T=2), midpoint=5 | count=100.6, sum=502.1 |
| `>/ -10` | Discarded (`T < 0`, special case condition `T >= 0` fails) | Fully retained (`lower >= T`: 5 >= -10), midpoint=5 | count=100, sum=500 |
| `>/ 0` | Interpolated: keep `1*(1-0/5)=1`, midpoint=2.5 | Fully retained | count=101, sum=33 (no-op) |

Note on `>/ -10`: The `[-Inf, 5]` bucket is discarded because the
TRIM_LOWER special-case condition (`T >= 0`) fails when `T = -10`. The
`(5, +Inf]` bucket is **not** subject to infinity bucket handling in this
case — the main loop classifies it as "fully retained" (`lower >= T`, i.e.,
`5 >= -10`) before the infinity handler is ever invoked. The infinity handler
is only reached for buckets in the "partially trimmed" case (`lower < T <
upper`).

### 11.4 NHCB with `[-Inf, X]` where `X <= 0`

When the first NHCB bucket has `upper <= 0`, the special `lower = 0`
treatment does NOT apply (that only triggers when `upper > 0`). The behavior
follows directly from the "observations at `-Inf`" principle:

- **TRIM_UPPER** (`</`): The bucket is **always kept**. When `T >= upper`,
  the main loop classifies it as "fully retained." When `T < upper <= 0`,
  the infinity handler's condition 3 (`upper <= 0`) triggers, keeping the
  entire bucket with midpoint = `T`. In both cases, observations at `-Inf`
  are below `T`.
- **TRIM_LOWER** (`>/`): The bucket is **always discarded**. Observations at
  `-Inf` are below any finite `T`, so they are removed. When `T >= upper`,
  the main loop discards it directly. When `T < upper`, the infinity handler
  falls through to the discard case (the special-case condition `T >= 0`
  either fails because `T < 0`, or `upper > T` fails because `upper <= 0 <=
  T`).

### 11.5 Single NHCB Bucket `[-Inf, +Inf]`

Given: `{schema:-53, sum:100, count:100, buckets:[100]}`

| Expression | Result |
|-----------|--------|
| `</ +Inf` | Keep (count=100, sum=100) |
| `>/ -Inf` | Keep (count=100, sum=100) |
| `</ 10` | Discard (count=0, sum=0) — lower is -Inf, conditions for interpolation not met, then upper is +Inf |
| `>/ 10` | Discard (count=0, sum=0) — same reasoning |

This is the most pathological case. The bucket's infinite extent in both
directions makes any estimation impossible, so trimming at any finite point
discards the entire bucket.

### 11.6 Exponential Histogram Trimmed at Geometric Midpoint

Given: `{schema:0, buckets:[2 4 8 16], ...}` where bucket boundaries are at
powers of 2: `(0.5, 1], (1, 2], (2, 4], (4, 8]` (for schema 0).

Trimming at `sqrt(2) ≈ 1.4142` (the geometric midpoint of `(1, 2]`):
- `</ 1.4142`: The `(1, 2]` bucket (count=4) is split in half by exponential
  interpolation, keeping count=2.
- `>/ 1.4142`: The same bucket keeps the other half, count=2.

## 12. Summary of Interpolation Methods by Context

| Context | Interpolation | Midpoint |
|---------|--------------|----------|
| Exponential histogram, positive bucket | Exponential (log2) | Geometric mean |
| Exponential histogram, negative bucket | Exponential (log2) | Negative geometric mean |
| Exponential histogram, zero bucket | Linear | Arithmetic mean |
| NHCB, regular bucket | Linear | Arithmetic mean |
| NHCB, `[-Inf, X]` bucket (X > 0) | Linear (treating lower as 0) | Arithmetic mean |
| NHCB, `[X, +Inf]` bucket, TRIM_UPPER | No interpolation — always discard | n/a |
| NHCB, `[X, +Inf]` bucket, TRIM_LOWER | No interpolation — always keep (see Note) | `T` |
| Any histogram, `[-Inf, X]` (X <= 0), TRIM_UPPER | No interpolation — always keep | `T` |
| Any histogram, `[-Inf, X]` (X <= 0), TRIM_LOWER | No interpolation — always discard | n/a |

Note: For `[X, +Inf]` TRIM_LOWER, the infinity handler only sees cases where
`lower < T < +Inf`, and always keeps the bucket. When `T <= lower`, the main
loop handles it as "fully retained" without invoking the infinity handler.