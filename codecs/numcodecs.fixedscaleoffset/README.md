---
status: superseded
superseded_by: [scale_offset, cast_value]
---

# numcodecs.fixedscaleoffset codec

Defines an `array -> array` codec that maps each element `x` to
`round((x - offset) * scale)` and converts the result to a (typically narrower
integer) data type. It is a lossy codec commonly used to store floating-point
data with a fixed absolute precision as integers.

## Background

`numcodecs.fixedscaleoffset` is the name that zarr-python writes for the
numcodecs
[`FixedScaleOffset`](https://numcodecs.readthedocs.io/en/stable/filter/fixedscaleoffset.html)
codec. It is registered here so that existing data written with that name has a
specification. New data should use the [`scale_offset`](../scale_offset/README.md)
codec followed by the [`cast_value`](../cast_value/README.md) codec instead,
whose behavior is fully specified for all inputs.

## Codec name

The value of the `name` member in the codec object must be
`"numcodecs.fixedscaleoffset"`.

## Configuration parameters

| field | type | required | default | notes |
| - | - | - | - | - |
| `offset` | number | yes | | Subtracted from each element during encoding. See [Lexical form of `offset` and `scale`](#lexical-form-of-offset-and-scale). |
| `scale` | number, not zero | yes | | Multiplies each element during encoding. |
| `dtype` | string | no | the codec input data type | Data type of the codec input (decoded data). See [Data type identifiers](#data-type-identifiers). |
| `astype` | string | no | value of `dtype` | Data type of the codec output (encoded data). |

The `configuration` member is required: writers must include it. Readers may
treat a missing `configuration` member as `{}` (which is invalid for this codec,
since `offset` and `scale` are required). Readers must accept a configuration
that omits `dtype` or `astype` and apply the defaults above.

Readers must accept and ignore an `id` field equal to `"fixedscaleoffset"`,
which numcodecs 0.14.0–0.15.0 wrote; writers must not write it.

### `dtype`

The data type of the input to this codec, i.e. the array data type if this is
the first array-to-array codec, otherwise the output data type of the preceding
array-to-array codec. If omitted, it is that data type (zarr-python fills it in
when creating an array, so it is normally present). If present and not equal to
that data type, the configuration is invalid and readers must reject it.
(numcodecs reinterprets the input element bytes as `dtype` without checking, so
a mismatch silently corrupts data.)

### `astype`

The data type of the output of this codec. If omitted, it is equal to `dtype`.

### Lexical form of `offset` and `scale`

zarr-python parses the configuration with Python's `json` module, which yields
an integer for a JSON number written without a fraction or exponent (an
*integer literal*, e.g. `10`) and a floating-point number otherwise (e.g.
`10.0`, `1e1`). When `dtype` is an integer type, the [encoding](#encoding)
depends on this distinction, so implementations must preserve it. Writers
should write `offset` and `scale` as integer literals only when integer
arithmetic in `dtype` is intended.

### Data type identifiers

`dtype` and `astype` hold a string. Readers must accept

- the Zarr V3 core data type names `"int8"`, `"int16"`, `"int32"`, `"int64"`,
  `"uint8"`, `"uint16"`, `"uint32"`, `"uint64"`, `"float16"`, `"float32"`,
  `"float64"`, and
- NumPy [array-protocol type strings](https://numpy.org/doc/stable/reference/arrays.interface.html)
  for the same types, i.e. strings matching
  `^[<>|=]?([iu][1248]|f[248])$`, such as `"<f8"`, `"|u1"` or `"u2"`.

Writers should write the Zarr V3 name. All accepted spellings of the same type
denote the same type.

The byte-order character has no effect on the encoded chunk: this codec
operates on values, and the byte order of the stored representation is set by
the array-to-bytes codec. However, numcodecs reinterprets the bytes of its
input using `dtype` in the host's byte order, so a `>` in `dtype` makes
numcodecs misread the data on little-endian hosts (verified: the array decodes
as zeros). Writers must not write `>`, and readers may reject it. In NumPy,
`=`, `|` and no character mean the host's byte order. zarr-python is in
practice used on little-endian hosts, where these spellings are equivalent to
`<`.

## Format

This is an `array -> array` codec.

The output array has the same shape as the input array and data type `astype`.
Each output element depends only on the input element at the same position.

### Arithmetic and conversion

"Arithmetic in type `T`" means:

- for an integer type with `N` bits: exact integer arithmetic followed by
  reduction modulo `2^N` into the range of `T` (two's complement wraparound);
- for a floating-point type: IEEE 754 binary16, binary32 or binary64 arithmetic
  with rounding to nearest, ties to even, after converting each operand to `T`.

`rint(v)` rounds `v` to the nearest integer, ties to even, keeping the
floating-point type of `v` (NaN and infinities are unchanged); this is
[`numpy.around`](https://numpy.org/doc/stable/reference/generated/numpy.around.html)
with zero decimals.

`convert(v, T)` denotes conversion of the value `v` to type `T`:

1. integer to integer: the value congruent to `v` modulo `2^N` that lies in the
   range of `T`, where `N` is the bit width of `T`;
2. integer or floating-point to floating-point: round to nearest, ties to even;
   a value whose rounded magnitude exceeds the largest finite value of `T`
   becomes infinity of the same sign; NaN remains NaN (payload unspecified);
   the sign of zero is preserved;
3. floating-point to integer: `v` is truncated toward zero; if the truncated
   value is in the range of `T` it is the result. Otherwise (NaN, infinity, or
   out of range), the result is implementation-defined. NumPy delegates this
   case to a C conversion with undefined behavior, and its results differ
   between CPU architectures (e.g. x86-64 and arm64).

### Encoding

For each input element `x`:

If `dtype` is a floating-point type, with arithmetic in `dtype`:

```
y = convert(rint((x - offset) * scale), astype)
```

If `dtype` is an integer type:

```
d = x - offset          arithmetic in dtype     if offset is an integer literal
d = float64(x) - offset arithmetic in float64   otherwise
v = d * scale           arithmetic in dtype     if d is an integer (dtype) value and scale is an integer literal
v = float64(d) * scale  arithmetic in float64   otherwise
y = convert(v, astype)        if v is an integer (dtype) value
y = convert(rint(v), astype)  otherwise
```

If an integer-literal `offset` or `scale` is not representable in an integer
`dtype` (e.g. a negative `offset` with an unsigned `dtype`), the result is
implementation-defined; numcodecs with NumPy 2 raises an error.

Notes:

- Rounding is to nearest with ties to even, applied to the value computed in
  floating-point arithmetic, which may already be inexact: with `dtype`
  `float64`, `(1000.35 - 1000) * 10` is `3.5000000000002274` and rounds to `4`,
  while `(1000.25 - 1000) * 10` is exactly `2.5` and rounds to `2`. With
  `dtype` `float32`, the arithmetic is done in binary32, which can give a
  different result for the same decimal input.
- When `astype` is an integer type, encoded values outside its range, and NaN or
  infinite inputs, fall under rule 3 of `convert` and are
  implementation-defined. Writers should choose `offset`, `scale` and `astype`
  so that this does not occur.

### Decoding

For each encoded element `y`, let `T` be `astype` if `astype` is a
floating-point type, and `float64` otherwise. With arithmetic in `T`:

```
x = convert(convert(y, T) / scale + offset, dtype)
```

Note that when `dtype` is an integer type the result is truncated toward zero,
not rounded, so it may be one less in magnitude than the original value (e.g.
`scale` `3`, encoded `5`: `5 / 3 = 1.67` decodes to `1`). The decoding does not
depend on the lexical form of `offset` and `scale`.

### Worked example

`offset` `1000`, `scale` `10`, `dtype` `"float64"`, `astype` `"uint8"`, input
`[1000.0, 1000.1, 1000.25, 1000.35, 1025.5]`: the values of
`(x - 1000) * 10` are `[0.0, 1.0000000000002274, 2.5, 3.5000000000002274,
255.0]`, the encoded `uint8` array is `[0, 1, 2, 4, 255]` (bytes
`00 01 02 04 ff`), and decoding yields `[1000.0, 1000.1, 1000.2, 1000.4,
1025.5]`.

## Example

Array metadata produced by zarr-python 3.4.2 for
`filters=[FixedScaleOffset(offset=1000, scale=10, astype="uint8")]` on a
`float64` array (zarr-python filled in `dtype`):

```json
{
    "data_type": "float64",
    "codecs": [
        {
            "name": "numcodecs.fixedscaleoffset",
            "configuration": {
                "offset": 1000,
                "scale": 10,
                "astype": "uint8",
                "dtype": "float64"
            }
        },
        {"name": "bytes"}
    ]
}
```

## Relationship to `scale_offset`

The [`scale_offset`](../scale_offset/README.md) codec followed by the
[`cast_value`](../cast_value/README.md) codec expresses the same
transformation, with the difference that its behavior is defined (or an error)
for every input:

```json
[
    {"name": "scale_offset", "configuration": {"offset": 1000, "scale": 10}},
    {
        "name": "cast_value",
        "configuration": {"data_type": "uint8", "rounding": "nearest-even"}
    }
]
```

The encoded bytes of the two forms are identical for every element for which
this codec's output is well-defined, when `dtype` is a floating-point type and
`astype` is an integer type: both compute `(x - offset) * scale` in `dtype` and
round to nearest, ties to even. (When `astype` is a floating-point type, this
codec still rounds to an integer, while `cast_value` does not.) Differences:

- `scale_offset` errors when an intermediate value is not representable, and
  `cast_value` errors on out-of-range values unless `out_of_range` is set;
  this codec wraps (integer `dtype` with integer-literal parameters) or is
  implementation-defined (floating-point to integer). `"out_of_range": "wrap"`
  reproduces numcodecs only for the integer-arithmetic case, not in general for
  floating-point to integer conversion.
- For decoding, this codec computes in `float64` (integer `astype`) or in
  `astype` (floating-point `astype`) and converts to `dtype` last, whereas
  `[scale_offset, cast_value]` first converts to `dtype` and then computes in
  `dtype`.

Consequently, metadata may be rewritten from `numcodecs.fixedscaleoffset` to
`[scale_offset, cast_value]` without re-encoding when `dtype` is `float64` and
`astype` is an integer type or `float64`: decoded values are bit-identical
(verified with zarr-python 3.4.2 on random data). For `dtype` `float32` (one
rounding of a `float64` result versus `float32` arithmetic), floating-point
`astype` narrower than `dtype`, or integer `dtype` (truncation versus integer
division), decoded values may differ.

An earlier version of the [`scale_offset`](../scale_offset/README.md)
specification described this codec in a "Legacy" section; that description is
replaced by this document.

## Implementations

- zarr-python: `zarr.codecs.numcodecs.FixedScaleOffset`
- numcodecs: `numcodecs.FixedScaleOffset` (for Zarr V2 `"id": "fixedscaleoffset"`)
- [zarrs](https://github.com/zarrs/zarrs) (experimental): computes in `float32`
  for `float32` data and `float64` otherwise (including `float16`), rounds
  decoded integers instead of truncating, and saturates out-of-range values, so
  results can differ from this specification in the cases noted above.

Notes on zarr-python:

- zarr-python 3.4.2 and earlier pass the numcodecs output (a one-dimensional
  array) to the next codec instead of an array with the chunk shape; codecs that
  depend on the shape (e.g. `transpose`, `sharding_indexed`) then fail or
  produce non-conforming output
  ([zarr-python#4553](https://github.com/zarr-developers/zarr-python/issues/4553)).
  With the `bytes` codec next, the stored bytes are unaffected.
- zarr-python before 3.3.0 could encode an F-contiguous in-memory chunk in
  column-major order
  ([zarr-python#4116](https://github.com/zarr-developers/zarr-python/pull/4116)),
  which does not conform to this specification.

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)
