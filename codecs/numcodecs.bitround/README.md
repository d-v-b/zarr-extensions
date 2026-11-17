---
status: superseded
superseded_by: [bitround]
---

# numcodecs.bitround codec

Defines an `array -> array` codec that rounds the significand (mantissa) of
IEEE 754 floating-point values to a given number of bits, with ties to even, by
integer arithmetic on their binary representation. The trailing significand
bits become zero, which makes the data more compressible. The codec is lossy;
decoding is the identity.

## Background

`numcodecs.bitround` is the name that zarr-python writes for the numcodecs
[`BitRound`](https://numcodecs.readthedocs.io/en/stable/filter/bitround.html)
codec. It is registered here so that existing data written with that name has a
specification. The [`bitround`](../bitround/README.md) codec covers the same
algorithm for more data types and lists `numcodecs.bitround` as a deprecated
alias; new data should use `bitround`. See
[Relationship to `bitround`](#relationship-to-bitround).

## Codec name

The value of the `name` member in the codec object must be `"numcodecs.bitround"`.

## Configuration parameters

| field | type | required | default | notes |
| - | - | - | - | - |
| `keepbits` | integer, `>= 0` | yes | | Number of explicit significand bits to keep. At most `10` for `float16`, `23` for `float32`, `52` for `float64`. |

The `configuration` member is required: writers must include it. Readers may
treat a missing `configuration` member as `{}` (which is invalid for this codec,
since `keepbits` is required).

Readers must accept and ignore an `id` field equal to `"bitround"`, which
numcodecs 0.14.0–0.15.0 wrote; writers must not write it.

A `keepbits` greater than the number of explicit significand bits `M` of the
input data type is invalid (numcodecs raises an error when encoding). A
`keepbits` equal to `M` makes the codec the identity.

## Supported data types

The input data type must be `float16`, `float32` or `float64` (IEEE 754
binary16, binary32 and binary64). The output data type is the same as the input
data type, and the output shape is the same as the input shape.

## Format

This is an `array -> array` codec. Each output element depends only on the
input element at the same position.

### Encoding

Let `W` be the bit width of the data type (`16`, `32`, `64`) and `M` the number
of explicit significand bits (`10`, `23`, `52`). If `keepbits == M`, the output
equals the input. Otherwise let `k = M - keepbits` (so `1 <= k <= M`). For each
element, let `u` be its IEEE 754 binary representation interpreted as an
unsigned `W`-bit integer. The encoded representation is

```
u' = (u + ((u >> k) & 1) + 2^(k-1) - 1) mod 2^W
v  = u' & ~(2^k - 1)
```

i.e. the low `k` bits are rounded off with round-half-to-even on the kept bits,
and the addition wraps around modulo `2^W`. A carry out of the significand
increments the exponent, so values can round up to the next power of two, and
the largest finite values round to infinity.

Consequences of operating on the representation:

- Zeros and infinities are unchanged.
- NaNs are not guaranteed to be preserved: depending on its significand bits,
  a NaN may become an infinity (when the rounded significand is zero) or a zero
  (when the rounding carries through the exponent into, or out of, the sign
  bit), as determined by the integer arithmetic above. For example, with
  `keepbits == 0` the default quiet NaN (`0x7fc00000` for `float32`) becomes
  `-0.0`, and with `keepbits == 3` the `float16` NaN `0x7c01` becomes positive
  infinity.

### Decoding

Decoding is the identity: the decoded element has the same binary
representation as the encoded element.

### Worked example

`float32` input `[1.2345679, -8.358719, 98765.43]` with `keepbits` `3` (`k = 20`)
has representations `3f9e0652`, `c105bd50`, `47c0e6b7` (hexadecimal). For the
first, `(u >> 20) & 1 = 1`, so `u' = 3f9e0652 + 1 + 7ffff = 3fa60652` and
`v = 3fa00000` (`1.25`). The encoded values are `[1.25, -8.0, 98304.0]`, stored
by a little-endian `bytes` codec as `00 00 a0 3f 00 00 00 c1 00 00 c0 47`.

## Example

Array metadata produced by zarr-python 3.4.2 for `filters=[BitRound(keepbits=5)]`
on a `float32` array:

```json
{
    "data_type": "float32",
    "codecs": [
        {"name": "numcodecs.bitround", "configuration": {"keepbits": 5}},
        {"name": "bytes", "configuration": {"endian": "little"}}
    ]
}
```

## Relationship to `bitround`

The [`bitround`](../bitround/README.md) codec has the same configuration
(`keepbits`) and, for `float16`, `float32` and `float64`, the same algorithm.
Stored bytes are identical for all values except negative NaNs whose
significand bits at positions `k - 1` through `M - 1` are all one, for which
the addition above exceeds `2^W - 1`: numcodecs wraps around and stores `+0.0`,
whereas the zarrs implementation of `bitround` saturates the addition and
stores a negative NaN (or negative infinity if `keepbits` is `0`). The
`bitround` specification does not define this case.

Metadata may be rewritten from `numcodecs.bitround` to `bitround` with the same
`keepbits` without re-encoding: decoding is the identity for both codecs.

`bitround` additionally supports integer and complex data types, which
`numcodecs.bitround` rejects.

## Implementations

- zarr-python: `zarr.codecs.numcodecs.BitRound`
- numcodecs: `numcodecs.BitRound` (for Zarr V2 `"id": "bitround"`)
- [zarrs](https://github.com/zarrs/zarrs): reads `numcodecs.bitround` as an
  alias of `bitround` (see the difference for negative NaNs above).

Note on zarr-python: the numcodecs encoder returns the rounded data viewed as a
signed integer array of the same width, and zarr-python passes that array to
the next codec although it declares the output data type to be the input data
type. With the `bytes` codec next, the stored bytes are unaffected.

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)
