# numcodecs.shuffle codec

Defines a `bytes -> bytes` codec that reorders the bytes of its input by
treating the input as a sequence of fixed-size elements and transposing it, so
that the first byte of every element comes first, then the second byte of
every element, and so on. This byte shuffle often makes the output of an
array-to-bytes codec more compressible by a following compression codec. The
output has the same length as the input.

## Background

`numcodecs.shuffle` is the name that zarr-python writes into Zarr V3 array
metadata for the `Shuffle` codec of the
[numcodecs](https://github.com/zarr-developers/numcodecs) library
(`zarr.codecs.numcodecs.Shuffle`, previously `numcodecs.zarr3.Shuffle`). It is
registered here so that data already written under this name has a
specification that does not depend on the numcodecs source code. This codec
operates on bytes. It is a `bytes -> bytes` codec, not an `array -> array`
codec (see [numcodecs#767](https://github.com/zarr-developers/numcodecs/issues/767)).

## Codec name

The value of the `name` member in the codec object must be
`"numcodecs.shuffle"`.

## Configuration parameters

Writers must include the `configuration` member, as a JSON object that may be
empty. Readers may treat a missing `configuration` as `{}`. zarr-python
writes only the parameters that the user passed explicitly, so readers must
accept a configuration with any optional field omitted and apply the default
listed below. Readers must accept and ignore an `id` field equal to
`"shuffle"`, which numcodecs 0.14.0–0.15.0 wrote. Writers must not write it. No
other fields are permitted.

| field         | type    | required | default                     | notes                     |
| ------------- | ------- | -------- | --------------------------- | ------------------------- |
| `elementsize` | integer | no       | see [below](#elementsize)   | Element size in bytes, ≥ 1. |

### `elementsize`

The size in bytes, `E`, of the elements that are transposed.

**Default.** If `elementsize` is omitted, `E` is the size in bytes of one
element of the data type that the codec chain's `array -> bytes` codec
receives. That data type is the array's data type after all `array -> array`
codecs in the chain have been applied. For example, it is 2 for `uint16`, 8 for
`float64`, and 1 for `bool`.

When zarr-python creates or opens an array, it fills in this value and writes
it into the metadata it creates. For example, it writes `"elementsize": 2` for
a `uint16` array even if the user passed no `elementsize`. This default differs
from numcodecs' own default for its `Shuffle` class, which is 4. Readers must
use the default defined here, not 4.

There is no meaningful default for data types without a fixed element size,
such as the variable-length string type. Writers should always write
`elementsize` explicitly.

numcodecs treats any `elementsize` less than or equal to 1, including 0 and
negative values, as the identity transformation. Readers may do the same for
0, but writers must write a positive integer.

## Format

This is a `bytes -> bytes` codec. Let `in` be the input of length `L` bytes and
`E = elementsize`.

If `E = 1`, the output is identical to the input, both when encoding and when
decoding.

Otherwise, `L` must be a multiple of `E`. If it is not, encoding and decoding
both fail. Unlike the shuffle inside [`blosc`](../blosc/README.md), this codec
has no rule for trailing bytes. Let `n = L / E`.

**Encoding** produces `out` of length `L` with

```
out[j * n + i] = in[i * E + j]     for 0 <= i < n, 0 <= j < E
```

that is, the input is read as an `n × E` row-major byte matrix and written out
transposed (`E × n`).

**Decoding** applies the inverse permutation:

```
out[i * E + j] = in[j * n + i]     for 0 <= i < n, 0 <= j < E
```

An empty input encodes and decodes to an empty output.

### Worked example

With `elementsize` 2, the 16-byte input
`00 00 01 00 02 00 03 00 04 00 05 00 06 00 07 00` (the `uint16` values 0 to 7
in little-endian order) is encoded as

```
00 01 02 03 04 05 06 07 00 00 00 00 00 00 00 00
```

With `elementsize` 3, the input `00 01 02 03 04 05 06 07 08` is encoded as
`00 03 06 01 04 07 02 05 08`.

## Example

Array metadata written by zarr-python for a `uint16` array with
`compressors=[zarr.codecs.numcodecs.Shuffle(), zarr.codecs.numcodecs.Zstd()]`.
zarr-python filled in `elementsize` from the data type:

```json
{
  "data_type": "uint16",
  "codecs": [
    { "name": "bytes", "configuration": { "endian": "little" } },
    { "name": "numcodecs.shuffle", "configuration": { "elementsize": 2 } },
    { "name": "numcodecs.zstd", "configuration": {} }
  ]
}
```

## Implementations

- zarr-python: [`zarr.codecs.numcodecs.Shuffle`](https://zarr.readthedocs.io/en/stable/api/zarr/codecs/numcodecs/)
- numcodecs: [`numcodecs.Shuffle`](https://numcodecs.readthedocs.io/en/stable/filter/shuffle.html)
  (the codec implementation that zarr-python delegates to)
- zarrs: [`shuffle`](https://docs.rs/zarrs/latest/zarrs/array/codec/bytes_to_bytes/shuffle/index.html)
  (reads and writes `numcodecs.shuffle`)

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)
