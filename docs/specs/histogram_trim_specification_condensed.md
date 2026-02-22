# Specification: Native Histogram Trim Operators (Condensed)

## 1. Overview

| Operator | Syntax | Semantics |
|----------|--------|-----------|
| `</` (TRIM_UPPER) | `histogram </ threshold` | Keep observations **below** threshold |
| `>/` (TRIM_LOWER) | `histogram >/ threshold` | Keep observations **above** threshold |

LHS must be a native histogram (exponential or NHCB). RHS must be a float.
The reverse (`float </ histogram`) is not supported. The `bool` modifier is
not supported. Standard vector matching (`on`/`ignoring`/`group_left`/
`group_right`) applies for vector/vector operands.

## 2. Algorithm

1. Copy the histogram.
2. Classify each bucket as fully retained, fully discarded, or partially
   trimmed (threshold inside bucket — interpolate).
3. Recompute `Count` and `Sum` from surviving bucket populations.
4. Compact: remove zero-count buckets, merge spans.

If no bucket count changed, original `Count` and `Sum` are preserved exactly.

## 3. Bucket Classification

**TRIM_UPPER** (`</`): `upper <= T` → retain; `lower < T < upper` →
interpolate; `lower >= T` → discard.

**TRIM_LOWER** (`>/`): `lower >= T` → retain; `lower < T < upper` →
interpolate; `upper <= T` → discard.

At exact boundaries: `upper == T` retained / `lower == T` discarded for
TRIM_UPPER (and vice versa for TRIM_LOWER). At `T = ±Inf`: `</ +Inf` and
`>/ -Inf` are no-ops; `</ -Inf` and `>/ +Inf` discard everything.

## 4. Interpolation

When `T` falls strictly inside a bucket `[lower, upper]` with count `C`,
the fraction of observations below `T` (the "under-count") is estimated.
For TRIM_UPPER, `kept = under-count`; for TRIM_LOWER, `kept = C - under-count`.

### 4.1 Exponential Histograms (log-uniform assumption)

Positive buckets:
`fraction = (log2(T) - log2(lower)) / (log2(upper) - log2(lower))`

Negative buckets:
`fraction = 1 - (log2(|T|) - log2(|upper|)) / (log2(|lower|) - log2(|upper|))`

### 4.2 NHCB (uniform/linear assumption)

`fraction = (T - lower) / (upper - lower)`

### 4.3 Zero Bucket (Exponential Only)

Always uses **linear** interpolation. Effective bounds are biased based on
which non-zero-count buckets exist in the **original** (pre-trim) histogram:

| Original histogram has... | Effective bounds |
|---|---|
| Both positive and negative buckets | `[-ZT, +ZT]` |
| Only positive buckets | `[0, +ZT]` |
| Only negative buckets | `[-ZT, 0]` |
| Neither | `[-ZT, +ZT]` |

(ZT = ZeroThreshold. Input buckets with count 0 are ignored for this
determination.)

With effective bounds `[L, U]`, TRIM_UPPER keeps
`ZeroCount * (T - L) / (U - L)` when `L < T < U` (midpoint `(L+T)/2`),
all when `T >= U`, none when `T <= L`. TRIM_LOWER is symmetric.

## 5. Infinity Bucket Handling

Infinity handling applies only to the **partially trimmed** case (`lower < T
< upper`). The main loop's retain/discard classifications take precedence.

**Guiding principle**: observations in infinity buckets are considered
**infinitely far away** — at `-Inf` for `[-Inf, X]` buckets, at `+Inf` for
`[X, +Inf]` buckets. If distribution cannot be estimated, the bucket is
conservatively discarded.

### 5.1 `[-Inf, X]` Buckets

Conditions evaluated in order (first match wins):

**TRIM_UPPER** (`</`):

1. `T >= upper` — Keep all. (Handled by main loop in practice.)
2. `upper > 0`, `T > 0`, `upper` finite — **Special case**: treat lower as
   `0`, interpolate linearly: kept = `C * T / upper`, midpoint = `T / 2`.
3. `upper <= 0` — Keep all; midpoint = `T`. (Observations at `-Inf` are
   below any `T`.)
4. Otherwise — Discard.

**TRIM_LOWER** (`>/`):

1. `T >= 0`, `upper > T`, `upper` finite — **Special case**: treat lower as
   `0`, interpolate linearly: kept = `C * (1 - T/upper)`, midpoint =
   `(T + upper) / 2`.
2. Otherwise — Discard. (Observations at `-Inf` are below `T`.)

**Special case rationale**: NHCB `[-Inf, X]` with `X > 0` conventionally
represents `[0, X]`, consistent with `histogram_quantile`.

**`X <= 0` summary**: TRIM_UPPER always keeps (observations at `-Inf` are
below any `T`); TRIM_LOWER always discards.

### 5.2 `[X, +Inf]` Buckets

**TRIM_UPPER**: Always discard. (Observations at `+Inf` are above any `T`.)

**TRIM_LOWER**: Always keep (for the `lower < T < +Inf` case that reaches
the handler); midpoint = `T`.

### 5.3 `[-Inf, +Inf]` (Single-Bucket NHCB)

No interpolation possible. `</ +Inf` and `>/ -Inf` keep; any finite
threshold or `</ -Inf` or `>/ +Inf` discards.

## 6. Sum Recalculation

When any bucket is trimmed, `Sum` is recomputed as `Σ (kept_count *
midpoint)` across all surviving buckets. The original `Sum` is discarded.

### Midpoint rules

**Finite bounds**: NHCB uses arithmetic mean `(lower + upper) / 2`.
Exponential uses geometric mean: `sqrt(|lower * upper|)` for positive
buckets, `-sqrt(|lower * upper|)` for negative buckets.

**Infinite bounds**: Both infinite → `0`. Lower `-Inf`, upper > 0 →
`upper / 2`. Lower `-Inf`, upper <= 0 → `upper`. Upper `+Inf` → `lower`.

**Partially trimmed buckets**: The surviving interval is `[lower, T]`
(TRIM_UPPER) or `[T, upper]` (TRIM_LOWER); midpoint is computed for this
narrowed interval. Zero bucket always uses arithmetic mean of its effective
(biased) bounds.

## 7. Output Properties

Preserved: Schema, ZeroThreshold, CustomValues. Recomputed: Count (sum of
surviving bucket counts), Sum (see Section 6), bucket counts, ZeroCount,
Spans (compacted to remove zero-count buckets).

## 8. Summary Table

| Context | Interpolation | Midpoint |
|---------|--------------|----------|
| Exponential, positive bucket | Exponential (log2) | Geometric mean |
| Exponential, negative bucket | Exponential (log2) | Negative geometric mean |
| Exponential, zero bucket | Linear | Arithmetic mean |
| NHCB, regular bucket | Linear | Arithmetic mean |
| NHCB, `[-Inf, X]` (X > 0) | Linear (lower treated as 0) | Arithmetic mean |
| `[X, +Inf]`, TRIM_UPPER | No interpolation — discard | n/a |
| `[X, +Inf]`, TRIM_LOWER | No interpolation — keep | `T` |
| `[-Inf, X]` (X <= 0), TRIM_UPPER | No interpolation — keep | `T` |
| `[-Inf, X]` (X <= 0), TRIM_LOWER | No interpolation — discard | n/a |