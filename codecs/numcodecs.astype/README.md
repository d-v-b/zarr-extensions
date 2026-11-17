---
status: superseded
superseded_by: [cast_value]
---

# numcodecs.astype codec

Defines an `array -> array` codec that converts each element to another data
type when encoding, and back when decoding, by numeric value conversion.

## Background

`numcodecs.astype` is the name that zarr-python writes for the numcodecs
[`AsType`](https://numcodecs.readthedocs.io/en/stable/filter/astype.html)
codec. It is registered here so that existing data written with that name has a
specification. The [`cast_value`](../cast_value/README.md) codec performs value
conversion with explicitly configured rounding and out-of-range handling; new
data should use it. See [Relationship to `cast_value`](#relationship-to-cast_value).

## Codec name

The value of the `name` member in the codec object must be `"numcodecs.astype"`.

## Configuration parameters

| field | type | required | default | notes |
| - | - | - | - | - |
| `encode_dtype` | string | yes | | Data type of the codec output (encoded data). See [Data type identifiers](#data-type-identifiers). |
| `decode_dtype` | string | no | the codec input data type | Data type of the codec input (decoded data). |

The `configuration` member is required: writers must include it. Readers may
treat a missing `configuration` member as `{}` (which is invalid for this codec,
since `encode_dtype` is required). Readers must accept a configuration that
omits `decode_dtype` and apply the default above.

Readers must accept and ignore an `id` field equal to `"astype"`, which
numcodecs 0.14.0–0.15.0 wrote; writers must not write it.

### `encode_dtype`

The data type of the output of this codec.

### `decode_dtype`

The data type of the input to this codec, i.e. the array data type if this is
the first array-to-array codec, otherwise the output data type of the preceding
array-to-array codec. If omitted, it is that data type (zarr-python fills it in
when creating or opening an array, so it is normally present). If present and
not equal to that data type, the configuration is invalid and readers must
reject it. (numcodecs reinterprets the input element bytes as `decode_dtype`
without checking, so a mismatch silently corrupts data.)

### Data type identifiers

`encode_dtype` and `decode_dtype` hold a string. Readers must accept

- the Zarr V3 core data type names `"bool"`, `"int8"`, `"int16"`, `"int32"`,
  `"int64"`, `"uint8"`, `"uint16"`, `"uint32"`, `"uint64"`, `"float16"`,
  `"float32"`, `"float64"`, `"complex64"`, `"complex128"`, and
- NumPy [array-protocol type strings](https://numpy.org/doc/stable/reference/arrays.interface.html)
  for the same types, i.e. strings matching
  `^[<>|=]?(b1|[iu][1248]|f[248]|c(8|16))$`, such as `"<f8"`, `"|u1"` or
  `"c16"`.

Writers should write the Zarr V3 name. All accepted spellings of the same type
denote the same type. Other data types (which numcodecs accepts, e.g.
date/time types) are not covered by this specification.

The byte-order character has no effect on the encoded chunk: this codec
operates on values, and the byte order of the stored representation is set by
the array-to-bytes codec (verified for `encode_dtype` `"<f4"`, `">f4"`, `"=f4"`
and `"f4"`, which store identical bytes). However, numcodecs reinterprets the
bytes of its input using `decode_dtype` in the host's byte order, so a `>` in
`decode_dtype` makes numcodecs misread the data on little-endian hosts
(verified: the array decodes as zeros). Writers must not write `>` in
`decode_dtype`, and readers may reject it. In NumPy, `=`, `|` and no character
mean the host's byte order. zarr-python is in practice used on little-endian
hosts, where these spellings are equivalent to `<`.

## Format

This is an `array -> array` codec.

The output array has the same shape as the input array and data type
`encode_dtype`. Each output element depends only on the input element at the
same position.

### Encoding and decoding

Encoding computes `y = convert(x, encode_dtype)` for each element `x`; decoding
computes `x = convert(y, decode_dtype)` for each element `y`, where
`convert(v, T)` is the following conversion (that of NumPy's
`ndarray.astype` with its default unsafe casting):

1. integer to integer: the value congruent to `v` modulo `2^N` that lies in the
   range of `T`, where `N` is the bit width of `T` (two's complement
   wraparound); e.g. `int32` `128` becomes `int8` `-128`;
2. integer or floating-point to floating-point: round to nearest, ties to even;
   a value whose rounded magnitude exceeds the largest finite value of `T`
   becomes infinity of the same sign; NaN remains NaN (payload unspecified);
   the sign of zero is preserved;
3. floating-point to integer: `v` is truncated toward zero; if the truncated
   value is in the range of `T` it is the result. Otherwise (NaN, infinity, or
   out of range), the result is implementation-defined. NumPy delegates this
   case to a C conversion with undefined behavior, and its results differ
   between CPU architectures (e.g. x86-64 and arm64; on arm64, NumPy converts
   `float64` `NaN` to `0`, and `40000.0` to `int16` `-25536`);
4. `bool` to a numeric type: `false` becomes `0` and `true` becomes `1`;
   a numeric type to `bool`: zero (including `-0.0`) becomes `false`, any other
   value (including NaN) becomes `true`;
5. complex types: complex to complex converts the real and imaginary parts
   separately by rule 2; a real value converted to a complex type has an
   imaginary part of zero; a complex value converted to a real, integer or
   `bool` type discards the imaginary part and converts the real part by
   rules 2–4.

The round trip is lossless if every value of `decode_dtype` that occurs is
exactly representable in `encode_dtype`.

### Worked example

`decode_dtype` `"float64"`, `encode_dtype` `"float32"`, input
`[1.5, 0.1, 1e39]`: the encoded `float32` values are
`[1.5, 0.10000000149011612, inf]`, stored by a little-endian `bytes` codec as
`00 00 c0 3f cd cc cc 3d 00 00 80 7f`, and decoding yields
`[1.5, 0.10000000149011612, inf]`.

## Example

Array metadata produced by zarr-python 3.4.2 for
`filters=[AsType(encode_dtype="float32")]` on a `float64` array (zarr-python
filled in `decode_dtype`):

```json
{
    "data_type": "float64",
    "codecs": [
        {
            "name": "numcodecs.astype",
            "configuration": {"encode_dtype": "float32", "decode_dtype": "float64"}
        },
        {"name": "bytes", "configuration": {"endian": "little"}}
    ]
}
```

## Relationship to `cast_value`

[`cast_value`](../cast_value/README.md) supports integer and floating-point
types only (not `bool` or complex types). Between those types, this codec and
`cast_value` with `"rounding": "nearest-even"` produce the same values for
conversions under rules 1 and 2, except that `cast_value` treats out-of-range
values as an error unless `out_of_range` is set (`"wrap"` reproduces rule 1;
`"clamp"` reproduces rule 2 for floating-point targets). For floating-point to
integer conversion, this codec truncates toward zero (`cast_value` with
`"rounding": "towards-zero"`), and its out-of-range behavior is
implementation-defined.

Metadata may therefore be rewritten from `numcodecs.astype` to `cast_value`
(with `data_type` set to `encode_dtype`) without re-encoding when both data
types are integer or floating-point types, choosing `out_of_range` as above and
`rounding` to match the decoding conversion (`encode_dtype` to
`decode_dtype`): `"nearest-even"` unless it is floating-point to integer, then
`"towards-zero"`. Decoded values agree wherever this codec's decoding is
well-defined.

## Implementations

- zarr-python: `zarr.codecs.numcodecs.AsType`
- numcodecs: `numcodecs.AsType` (for Zarr V2 `"id": "astype"`)

Notes on zarr-python:

- zarr-python 3.4.2 and earlier pass the numcodecs output to the next codec as
  an array of the chunk shape (this codec does not flatten), so they are not
  affected by
  [zarr-python#4553](https://github.com/zarr-developers/zarr-python/issues/4553).
- zarr-python chooses the default array-to-bytes codec from the array data
  type, not from `encode_dtype`; for example an `int8` array with
  `encode_dtype` `"int16"` needs an explicit `bytes` codec with `endian`.

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)
