# numcodecs.quantize codec

Defines an `array -> array` codec that rounds floating-point values to the
nearest multiple of a power of two chosen from a number of decimal digits, so
that the values become more compressible. The codec is lossy; decoding only
converts the data type.

## Background

`numcodecs.quantize` is the name that zarr-python writes for the numcodecs
[`Quantize`](https://numcodecs.readthedocs.io/en/stable/filter/quantize.html)
codec. It is registered here so that existing data written with that name has a
specification. No other codec in this registry performs the same rounding.

## Codec name

The value of the `name` member in the codec object must be `"numcodecs.quantize"`.

## Configuration parameters

| field | type | required | default | notes |
| - | - | - | - | - |
| `digits` | integer, `-308` to `307` | yes | | Number of decimal digits after the decimal point to preserve. See [`digits`](#digits). |
| `dtype` | string | no | the codec input data type | Data type of the codec input (decoded data). See [Data type identifiers](#data-type-identifiers). |
| `astype` | string | no | value of `dtype` | Data type of the codec output (encoded data). |

The `configuration` member is required: writers must include it. Readers may
treat a missing `configuration` member as `{}` (which is invalid for this codec,
since `digits` is required). Readers must accept a configuration that omits
`dtype` or `astype` and apply the defaults above.

Readers must accept and ignore an `id` field equal to `"quantize"`, which
numcodecs 0.14.0–0.15.0 wrote; writers must not write it.

### `digits`

The quantization step is `q = 2^-b`, where `b` is the smallest integer such
that `2^b >= 10^digits`, i.e. `b = ceil(digits * log2(10))`. For example:

| `digits` | -3 | -2 | -1 | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
| - | - | - | - | - | - | - | - | - | - | - | - | - |
| `b` | -9 | -6 | -3 | 0 | 4 | 7 | 10 | 14 | 17 | 20 | 24 | 27 |

`q` is therefore the largest power of two not greater than `10^-digits`. A
negative `digits` rounds to multiples of a power of two greater than one.

numcodecs computes `b` as
`ceil(log2(10.0 ** -e))` with `e = floor(log10(10.0 ** -digits))` (or `ceil`
for non-negative `e`) in binary64 arithmetic; this equals the exact definition
above for every integer `digits` in the permitted range (verified for
`-30 <= digits <= 59`, and at the range limits). Outside `-308 <= digits <= 307`,
numcodecs raises an error. numcodecs also accepts a non-integer `digits` (e.g. `1.5`
behaves like `2`, and `-0.5` like `-1`); writers must write an integer.

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
See the note on zarr-python under [Implementations](#implementations): writers
should omit `astype` or set it equal to `dtype`.

### Data type identifiers

`dtype` and `astype` hold a string naming a floating-point type. Readers must
accept

- the Zarr V3 core data type names `"float16"`, `"float32"`, `"float64"`, and
- NumPy [array-protocol type strings](https://numpy.org/doc/stable/reference/arrays.interface.html)
  for the same types, i.e. strings matching `^[<>|=]?f[248]$`, such as `"<f8"`
  or `"f4"`.

Writers should write the Zarr V3 name. All accepted spellings of the same type
denote the same type. Other data types are not supported (numcodecs raises an
error).

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

### Encoding

Let `s = 2^b` (see [`digits`](#digits)). For each input element `x`, with
IEEE 754 arithmetic in `dtype` and rounding to nearest, ties to even:

```
y = convert(rint(x * s) / s, astype)
```

where `rint` rounds to the nearest integer with ties to even
([`numpy.around`](https://numpy.org/doc/stable/reference/generated/numpy.around.html)),
and `convert` rounds to nearest, ties to even, from `dtype` to `astype` (values
whose rounded magnitude exceeds the largest finite value of `astype` become
infinity of the same sign).

Because `s` is a power of two, the multiplication and division are exact
unless they overflow or underflow, so the result is `x` rounded to the nearest
multiple of `2^-b`, with ties to the even multiple. The sign of zero is
preserved, NaN remains NaN and infinities are unchanged.

If `|x| * s` exceeds the largest finite value of `dtype`, `x * s` overflows and
the encoded value is infinity of the same sign as `x`. For example, with
`dtype` `float16` and `digits` `3` (`s = 1024`), every value with magnitude of
`64` or more encodes as infinity. Writers should choose `digits` so that this
does not occur.

### Decoding

```
x = convert(y, dtype)
```

The rounding is not undone.

### Worked example

`digits` `1` (`b = 4`, so values are rounded to multiples of `1/16`),
`dtype` `"float64"`, input
`[0.0, 0.1111, 0.3333, 0.03125, 0.09375, -2.71828]`: the encoded array is
`[0.0, 0.125, 0.3125, 0.0, 0.125, -2.6875]`. `0.03125 * 16 = 0.5` and
`0.09375 * 16 = 1.5` are ties, rounded to the even integers `0` and `2`. A
little-endian `bytes` codec stores `0.125` as `00 00 00 00 00 00 c0 3f`.

## Example

Array metadata produced by zarr-python 3.4.2 for `filters=[Quantize(digits=3)]`
on a `float64` array (zarr-python filled in `dtype`):

```json
{
    "data_type": "float64",
    "codecs": [
        {
            "name": "numcodecs.quantize",
            "configuration": {"digits": 3, "dtype": "float64"}
        },
        {"name": "bytes", "configuration": {"endian": "little"}}
    ]
}
```

## Implementations

- zarr-python: `zarr.codecs.numcodecs.Quantize`
- numcodecs: `numcodecs.Quantize` (for Zarr V2 `"id": "quantize"`)

Notes on zarr-python:

- zarr-python (3.4.2 and the main branch as of November 2026) does not report
  `astype` as the output data type of this codec to the following codecs. With
  `astype` different from `dtype`, writing succeeds but the array cannot be
  read back by zarr-python. Data with such a configuration written by
  zarr-python may contain chunks whose elements were serialized as `astype`
  while later codecs were configured for `dtype`.
- zarr-python 3.4.2 and earlier pass the numcodecs output to the next codec as
  a one-dimensional array instead of an array with the chunk shape
  ([zarr-python#4553](https://github.com/zarr-developers/zarr-python/issues/4553)).
  With the `bytes` codec next, the stored bytes are unaffected.

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)
