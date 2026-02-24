# Specification: Native Histogram Trim Operators (Formal)

## 1. Definitions

### 1.1 Native Histogram

A native histogram **H** is a tuple:

```
H = (S, Count, Sum, Buckets, ZeroBucket)
```

where:

- **S** is the schema, determining bucket boundary computation and whether
  the histogram uses exponential or custom (NHCB) buckets.
- **Count** ∈ ℝ≥0 is the total number of observations.
- **Sum** ∈ ℝ is the sum of all observed values.
- **Buckets** is a finite sequence of buckets B₁, B₂, ..., Bₙ. Each bucket
  Bᵢ = (lᵢ, uᵢ, cᵢ) has lower bound lᵢ ∈ ℝ ∪ {-∞}, upper bound
  uᵢ ∈ ℝ ∪ {+∞}, and count cᵢ ∈ ℝ≥0, with lᵢ < uᵢ.
- **ZeroBucket** = (l₀, u₀, c₀, w) exists only for exponential histograms,
  where w > 0 is the zero threshold, l₀ = -w, u₀ = +w, and c₀ ∈ ℝ≥0.

For exponential histograms, **Buckets** is partitioned into positive buckets
(both bounds > 0) and negative buckets (both bounds < 0). The ZeroBucket
bridges the gap around zero.

For NHCB histograms, buckets are ordered by their bounds. The first bucket
may have l₁ = -∞ and the last bucket always has uₙ = +∞.

### 1.2 Trim Operators

The trim operators are binary operators with a histogram on the left and a
scalar threshold T ∈ ℝ ∪ {-∞, +∞} on the right:

- **TRIM_UPPER** (`</`): `H </ T` — produces a histogram retaining only
  observations with values below T.
- **TRIM_LOWER** (`>/`): `H >/ T` — produces a histogram retaining only
  observations with values above T.

The operators also accept vector operands with standard PromQL label
matching. The `bool` modifier is not supported.

## 2. Result Histogram

Given `H' = H </ T` (or `H' = H >/ T`), the result H' has:

- The same schema, zero threshold, and custom bucket values as H.
- The same bucket structure (boundaries), after compaction (Section 7).
- Modified counts, sum, and total count, defined below.

## 3. Retained Count per Bucket

For each bucket Bᵢ = (lᵢ, uᵢ, cᵢ), define the **retained count** c'ᵢ:

### 3.1 TRIM_UPPER (`</`)

```
         ⎧ cᵢ                          if uᵢ ≤ T
c'ᵢ  =   ⎨ cᵢ · φ(Bᵢ, T)              if lᵢ < T < uᵢ
         ⎩ 0                            if lᵢ ≥ T
```

### 3.2 TRIM_LOWER (`>/`)

```
         ⎧ cᵢ                          if lᵢ ≥ T
c'ᵢ  =   ⎨ cᵢ · (1 - φ(Bᵢ, T))        if lᵢ < T < uᵢ
         ⎩ 0                            if uᵢ ≤ T
```

where φ(Bᵢ, T) ∈ [0, 1] is the **under-fraction**: the estimated proportion
of observations in Bᵢ that have value ≤ T. Its definition depends on bucket
type and is given in Section 4.

### 3.3 Zero Bucket

For the zero bucket of exponential histograms, the retained count is computed
identically, but using adjusted effective bounds [L, U] as defined in
Section 5.

## 4. Under-Fraction φ(B, T)

The under-fraction estimates the proportion of observations in bucket
B = (l, u, c) with value ≤ T, assuming a distributional model that depends
on the bucket type.

### 4.1 Finite-Bound Buckets

#### Exponential histogram, positive bucket (0 < l < u, both finite):

Assumes log-uniform distribution:

```
φ(B, T) = (log₂ T - log₂ l) / (log₂ u - log₂ l)
```

#### Exponential histogram, negative bucket (l < u < 0, both finite):

Assumes log-uniform distribution over absolute values, mirrored:

```
φ(B, T) = 1 - (log₂|T| - log₂|u|) / (log₂|l| - log₂|u|)
```

#### NHCB bucket (both bounds finite):

Assumes uniform distribution:

```
φ(B, T) = (T - l) / (u - l)
```

#### Zero bucket (effective bounds [L, U], see Section 5):

Assumes uniform distribution:

```
φ(B, T) = (T - L) / (U - L)
```

### 4.2 Infinity-Bound Buckets

When a bucket has one or both infinite bounds, the under-fraction cannot in
general be estimated from the bucket count alone. The following rules apply.

**Core assumption**: Observations in an infinity bucket are considered to be
**at the infinite bound** — at -∞ for buckets with l = -∞, at +∞ for
buckets with u = +∞. Consequently:

- All observations in a [-∞, u] bucket are considered to have value -∞ (and
  are therefore below any finite T).
- All observations in a [l, +∞] bucket are considered to have value +∞ (and
  are therefore above any finite T).

This leads to the following rules:

#### Bucket with l = -∞:

**Case A** — u > 0, u finite, and T > 0 (NHCB special case):

Treat the bucket as [0, u] and apply uniform interpolation:

```
φ(B, T) = T / u
```

Rationale: the first NHCB bucket [-∞, u] with u > 0 conventionally
represents observations in [0, u], consistent with `histogram_quantile`.

**Case B** — u ≤ 0:

```
φ(B, T) = 1     (all observations at -∞ are below T)
```

**Case C** — otherwise (e.g. T ≤ 0 with u > 0, or u = +∞):

```
φ(B, T) = 0     (conservatively discard — distribution unknown)
```

#### Bucket with u = +∞ (and l finite):

```
φ(B, T) = 0     (conservatively discard — observations at +∞ are above T)
```

Note: for TRIM_LOWER, the retained count is `c · (1 - φ) = c · 1 = c`, so
the bucket is fully retained when the partially-trimmed case applies
(l < T < +∞).

#### Bucket with l = -∞ and u = +∞:

```
φ(B, T) = 0     (no estimation possible — fully discard for any finite T)
```

## 5. Zero Bucket Effective Bounds

The zero bucket of an exponential histogram has physical bounds [-w, +w]
where w is the zero threshold. For interpolation and sum estimation, the
**effective bounds** [L, U] are adjusted based on the presence of other
non-empty buckets in the **original** (pre-trim) histogram:

```
         ⎧ [-w, +w]    if both positive and negative buckets have c > 0
[L, U] = ⎨ [ 0, +w]    if only positive buckets have c > 0
         ⎨ [-w,  0]    if only negative buckets have c > 0
         ⎩ [-w, +w]    if no positive or negative buckets have c > 0
```

The rationale is: if only positive (resp. negative) regular buckets exist,
observations in the zero bucket are assumed to be non-negative (resp.
non-positive). With no regular buckets, symmetry is assumed.

The bias is determined from the original histogram, not the trimmed result.
This ensures the distributional assumption used for interpolation is
consistent with the one used for sum estimation.

## 6. Sum Estimation

### 6.1 No-op case

If c'ᵢ = cᵢ for every bucket Bᵢ (and the zero bucket, if present), the
trim is a no-op: Sum' = Sum, Count' = Count. No recomputation occurs.

### 6.2 Recomputation

If any c'ᵢ ≠ cᵢ, the new count and sum are:

```
Count' = Σᵢ c'ᵢ

Sum'   = Σᵢ c'ᵢ · μᵢ
```

where μᵢ is the **midpoint** of the surviving interval for bucket Bᵢ,
defined below.

### 6.3 Surviving Interval

For a bucket Bᵢ = (lᵢ, uᵢ, cᵢ):

- If fully retained: the surviving interval is [lᵢ, uᵢ].
- If partially trimmed by TRIM_UPPER at T: the surviving interval is
  [lᵢ, T].
- If partially trimmed by TRIM_LOWER at T: the surviving interval is
  [T, uᵢ].
- If fully discarded: no contribution to Sum'.

For the zero bucket, lᵢ and uᵢ are replaced by the effective bounds L, U.

### 6.4 Midpoint Function μ(a, b)

Given a surviving interval [a, b], the midpoint is:

#### NHCB or zero bucket (linear model):

```
μ(a, b) = (a + b) / 2
```

#### Exponential histogram, positive bucket:

```
μ(a, b) = √(|a · b|)        (geometric mean, always ≥ 0)
```

#### Exponential histogram, negative bucket:

```
μ(a, b) = -√(|a · b|)       (negative geometric mean, always ≤ 0)
```

#### Infinite bounds:

When one or both bounds of the surviving interval are infinite:

```
         ⎧ 0          if a = -∞ and b = +∞
         ⎨ b/2        if a = -∞ and b > 0
μ(a,b) = ⎨ b          if a = -∞ and b ≤ 0
         ⎨ a          if b = +∞
         ⎩ (defined above for finite a, b)
```

## 7. Compaction

After computing all c'ᵢ, buckets with c'ᵢ = 0 are removed from the bucket
structure. Adjacent spans are merged and offsets adjusted to produce a
minimal representation. The zero bucket's count is updated in place (not
removed even if zero).

## 8. Formal Properties

The following properties hold for the trim operators:

**P1 (Identity)**:
`H </ +∞ = H` and `H >/ -∞ = H`.

**P2 (Annihilation)**:
`H </ -∞` and `H >/ +∞` produce an empty histogram (all counts zero).

**P3 (Monotonicity of count)**:
For all T: `Count(H </ T) ≤ Count(H)` and `Count(H >/ T) ≤ Count(H)`.

**P4 (Complementarity of count)**:
For a given T, let H₁ = H </ T and H₂ = H >/ T. Then:
`Count(H₁) + Count(H₂) = Count(H)`.

Note: complementarity does **not** hold for Sum in general, because the sum
is re-estimated from bucket midpoints rather than partitioned from the
original sum.

**P5 (Idempotence)**:
`(H </ T₁) </ T₂ = H </ min(T₁, T₂)` in terms of retained counts (though
the sum may differ due to re-estimation at each step).

**P6 (Schema preservation)**:
The output histogram has the same schema, bucket boundaries, zero threshold,
and custom values as the input. Only counts, sum, and total count change.