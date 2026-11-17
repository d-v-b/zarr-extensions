---
status: superseded
superseded_by: [packbits]
---

# numcodecs.packbits codec

Defines an `array -> array` codec that packs a `bool` array into a
one-dimensional `uint8` array, eight elements per byte with the first element
in the most significant bit, preceded by a byte that holds the number of
padding bits.

## Background

`numcodecs.packbits` is the name that zarr-python writes for the numcodecs
[`PackBits`](https://numcodecs.readthedocs.io/en/stable/filter/packbits.html)
codec. It is registered here so that existing data written with that name has a
specification. The [`packbits`](../packbits/README.md) codec is a more general
`array -> bytes` codec for packing values into bits; new data should use it.
See [Relationship to `packbits`](#relationship-to-packbits).

## Codec name

The value of the `name` member in the codec object must be `"numcodecs.packbits"`.

## Configuration parameters

This codec has no configuration parameters. The configuration is the empty
object `{}`.

The `configuration` member is required: writers must include it. Readers may
treat a missing `configuration` member as `{}`.

Readers must accept and ignore an `id` field equal to `"packbits"`, which
numcodecs 0.14.0–0.15.0 wrote; writers must not write it.

## Supported data types

The input data type must be `bool`. The output data type is `uint8`.

## Format

This is an `array -> array` codec.

### Encoding

Let `n` be the number of elements of the input chunk and `b[0], ..., b[n-1]`
the elements in lexicographical (C) order (the last dimension varies fastest).
Let `m = ceil(n / 8)` and `p = 8 * m - n` (the number of padding bits,
`0 <= p <= 7`).

The output is a one-dimensional `uint8` array of shape `(1 + m,)`:

- element `0` holds `p`;
- element `1 + j` (for `0 <= j < m`) holds the bits `b[8j], ..., b[8j + 7]`, with
  `b[8j]` in the most significant bit (bit 7) and `b[8j + 7]` in the least
  significant bit (bit 0); a `true` element is a `1` bit. Bit positions beyond
  `b[n-1]` in the last byte are padding and are written as `0`.

This bit order is that of
[`numpy.packbits`](https://numpy.org/doc/stable/reference/generated/numpy.packbits.html)
with its default `bitorder="big"`.

### Decoding

The decoder reads `p` from element `0`, unpacks the remaining elements in the
same bit order, and discards the last `p` bits; element `i` of the result is
`true` if its bit is `1`. The decoded elements are placed in lexicographical
order in an array of the chunk shape. A decoder must reject an encoded array
whose length `L` and header `p` do not satisfy `p <= 7` and `8 * (L - 1) - p = n`.
Decoders must ignore the values of the padding bits.

### Worked example

Input `[true, false, false, true, true]` (`n = 5`, `m = 1`, `p = 3`): the
encoded array is `[3, 152]`, i.e. the bytes `03 98` (`0x98` is `10011000` in
binary).

Input of shape `(3, 3)`
`[[true, false, false], [true, true, true], [false, false, true]]` (`n = 9`,
`m = 2`, `p = 7`): the encoded array is `[7, 156, 128]`, i.e. the bytes
`07 9c 80`.

## Example

Array metadata produced by zarr-python 3.4.2 for `filters=[PackBits()]` on a
`bool` array:

```json
{
    "data_type": "bool",
    "codecs": [
        {"name": "numcodecs.packbits", "configuration": {}},
        {"name": "bytes"}
    ]
}
```

Because the output data type is `uint8`, the following `bytes` codec writes the
encoded array unchanged and needs no `endian` configuration.

## Relationship to `packbits`

The [`packbits`](../packbits/README.md) codec is an `array -> bytes` codec;
this codec is an `array -> array` codec whose `uint8` output is serialized by a
subsequent array-to-bytes codec (typically `bytes`). Applied to `bool` data,
`packbits` with `"padding_encoding": "first_byte"` produces the same layout (a
leading byte holding the number of padding bits, then one bit per element), but
the `packbits` specification does not state the order of bits within a byte,
and the zarrs implementation of `packbits` places the first element in the
least significant bit, the opposite of this codec
([zarr-extensions#85](https://github.com/zarr-developers/zarr-extensions/issues/85)).
Data encoded with this codec therefore must not be relabeled as `packbits`
without re-encoding, unless and until `packbits` defines a most-significant-bit
first order.

## Implementations

- zarr-python: `zarr.codecs.numcodecs.PackBits`
- numcodecs: `numcodecs.PackBits` (for Zarr V2 `"id": "packbits"`)
- [zarrs](https://github.com/zarrs/zarrs): `numcodecs.packbits`

Note on zarr-python: `PackBits.validate` checks the array data type rather than
the data type of the codec input, so zarr-python rejects this codec when it is
preceded by an array-to-array codec that produces `bool` from another data
type.

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)
