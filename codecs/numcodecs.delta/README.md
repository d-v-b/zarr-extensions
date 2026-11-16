# numcodecs.delta codec

Defines an `array -> array` codec that stores the first element of a chunk
followed by the differences between consecutive elements, taken in
lexicographical (C) order over the whole chunk. Optionally, the stored values
are converted to a different data type.

## Background

`numcodecs.delta` is the name that zarr-python writes for the numcodecs
[`Delta`](https://numcodecs.readthedocs.io/en/stable/filter/delta.html) codec.
It is registered here so that existing data written with that name has a
specification. No other codec in this registry performs delta encoding.

## Codec name

The value of the `name` member in the codec object must be `"numcodecs.delta"`.

## Configuration parameters

| field | type | required | default | notes |
| - | - | - | - | - |
| `dtype` | string | yes | | Data type of the codec input (decoded data). See [Data type identifiers](#data-type-identifiers). |
| `astype` | string | no | value of `dtype` | Data type of the codec output (encoded data). |

The `configuration` member is required: writers must include it. Readers may
treat a missing `configuration` member as `{}` (which is invalid for this codec,
since `dtype` is required). Readers must accept a configuration that omits
`astype` and must then use the value of `dtype`.

Readers must accept and ignore an `id` field equal to `"delta"`, which
numcodecs 0.14.0–0.15.0 wrote; writers must not write it.

### `dtype`

The data type of the input to this codec, i.e. the array data type if this is
the first array-to-array codec, otherwise the output data type of the preceding
array-to-array codec. If `dtype` does not denote that data type, the
configuration is invalid and readers must reject it. (numcodecs reinterprets the
input element bytes as `dtype` without checking, so a mismatch silently
corrupts data.)

Unlike most numcodecs filters wrapped by zarr-python, `dtype` is not filled in
from the array metadata; it must be present.

### `astype`

The data type of the output of this codec. If omitted, it is equal to `dtype`.

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
input using `dtype` (and of its encoded input using `astype`) in the host's
byte order, so a `>` in `dtype` makes numcodecs misread the data on
little-endian hosts. Writers must not write `>`, and readers may reject it. In
NumPy, `=`, `|` and no character mean the host's byte order. zarr-python is in
practice used on little-endian hosts, where these spellings are equivalent to
`<`.

The `bool` type (`"bool"`, `"b1"`) and complex types are not supported by this
codec. numcodecs accepts `bool`, but computes differences with logical XOR and
sums with logical OR, so the round trip is not lossless; readers should reject
a configuration whose `dtype` or `astype` is `bool`.

## Format

This is an `array -> array` codec.

### Shape and element order

The output array has the same shape as the input array and data type `astype`.
Let `n` be the number of elements of the chunk and let `x[0], ..., x[n-1]` be
the input elements in lexicographical (C) order, i.e. the last dimension varies
fastest. The encoded elements `y[0], ..., y[n-1]` are placed in the output
array in the same lexicographical order. Differences are taken across the
boundaries of the last dimension: the chunk is treated as a single
one-dimensional sequence.

### Arithmetic and conversion

"Arithmetic in type `T`" means:

- for an integer type with `N` bits: exact integer arithmetic followed by
  reduction modulo `2^N` into the range of `T` (two's complement wraparound);
- for a floating-point type: IEEE 754 binary16, binary32 or binary64 arithmetic
  with rounding to nearest, ties to even, after converting each operand to `T`.

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

```
y[0] = convert(x[0], astype)
y[i] = convert(x[i] - x[i-1], astype)     for 1 <= i < n
```

where `x[i] - x[i-1]` is computed with arithmetic in `dtype`.

If `x[0]` is not representable in `astype`, the result is
implementation-defined; encoders should fail. (numcodecs with NumPy 2 raises an
error in some such cases, e.g. integer to a narrower integer type, and applies
`convert` in others.)

When both types are integer types, `astype` must be wide enough to hold
`x[0]`; differences that do not fit wrap around (rule 1) and the original data
can then be recovered only if `astype` has the same width as `dtype`.

When `dtype` is a floating-point type, the differences are rounded, so the
round trip is in general not lossless even if `astype` is equal to `dtype`.

### Decoding

Decoding computes running sums of the encoded elements in an accumulation type
`A` and converts each running sum to `dtype`:

```
s[0] = convert(y[0], A)
s[i] = s[i-1] + convert(y[i], A)          for 1 <= i < n   (arithmetic in A)
x[i] = convert(s[i], dtype)
```

`A` is determined from `dtype` and `astype` (this is NumPy's type promotion,
`numpy.result_type(dtype, astype)`):

| `dtype` and `astype` | `A` |
| - | - |
| both integer types, except the case in the next row | `dtype` (equivalently, any integer type at least as wide; the result is `(y[0] + ... + y[i]) mod 2^N`) |
| one is `uint64` and the other is a signed integer type | `float64` (lossy; see below) |
| both floating-point types | the wider of the two |
| `float64` and any integer type | `float64` |
| `float32` and an 8- or 16-bit integer type | `float32` |
| `float32` and a 32- or 64-bit integer type | `float64` |
| `float16` and an 8-bit integer type | `float16` |
| `float16` and a 16-bit integer type | `float32` |
| `float16` and a 32- or 64-bit integer type | `float64` |

With `uint64` paired with a signed integer type, the running sums are rounded
to `float64` and the round trip is lossy; writers must not use such a
configuration.

Decoding is the exact inverse of encoding when `dtype` and `astype` are
integer types of equal width (e.g. `int16`/`int16` or `uint32`/`int32`, but not
`uint64`/`int64`), and for narrower integer `astype` when all differences and
`x[0]` fit in `astype`.

### Worked example

`dtype` `"int16"`, `astype` `"int8"`, input `[100, 102, 104, 103]`: the encoded
`int8` array is `[100, 2, 2, -1]`, stored by a little-endian `bytes` codec as
the bytes `64 02 02 ff`. With `astype` omitted, the encoded `int16` array is the
same values, stored as `64 00 02 00 02 00 ff ff`.

`dtype` `"uint8"`, input of shape `(2, 3)` `[[1, 2, 3], [10, 8, 9]]`: the
encoded array is `[[1, 1, 1], [7, 254, 1]]` (`8 - 10` wraps to `254`).

## Example

Array metadata produced by zarr-python 3.4.2 for
`filters=[Delta(dtype="int16", astype="int8")]` on an `int16` array:

```json
{
    "data_type": "int16",
    "codecs": [
        {
            "name": "numcodecs.delta",
            "configuration": {"dtype": "int16", "astype": "int8"}
        },
        {"name": "bytes"}
    ]
}
```

## Implementations

- zarr-python: `zarr.codecs.numcodecs.Delta`
- numcodecs: `numcodecs.Delta` (for Zarr V2 `"id": "delta"`)

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
- When an array-to-array codec that changes the memory layout (e.g.
  `transpose`) follows this codec, zarr-python before
  [zarr-python#4558](https://github.com/zarr-developers/zarr-python/pull/4558)
  decodes elements in the wrong order.

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)
