# numcodecs.lz4 codec

Defines a `bytes -> bytes` codec that compresses its input as a single
[LZ4 block][lz4-block], prefixed with the length of the uncompressed data as a
4-byte little-endian integer. The output is **not** an LZ4 frame: it has no
frame magic number, no frame descriptor and no checksums.

## Background

`numcodecs.lz4` is the name that zarr-python writes into Zarr V3 array metadata
for the `LZ4` codec of the [numcodecs](https://github.com/zarr-developers/numcodecs)
library (`zarr.codecs.numcodecs.LZ4`, previously `numcodecs.zarr3.LZ4`). It is
registered here so that data already written under this name has a
specification that does not depend on the numcodecs source code.

## Codec name

The value of the `name` member in the codec object must be `"numcodecs.lz4"`.

## Configuration parameters

Writers must include the `configuration` member, as a JSON object that may be
empty. Readers may treat a missing `configuration` as `{}`. zarr-python
writes only the parameters that the user passed explicitly, so readers must
accept a configuration with any optional field omitted and apply the default
listed below. Readers must accept and ignore an `id` field equal to
`"lz4"`, which numcodecs 0.14.0–0.15.0 wrote. Writers must not write it. No
other fields are permitted.

| field          | type    | required | default | notes                       |
| -------------- | ------- | -------- | ------- | --------------------------- |
| `acceleration` | integer | no       | `1`     | Affects encoding only.      |

### `acceleration`

The LZ4 "acceleration" factor passed to the LZ4 block compressor
(`LZ4_compress_fast`). Larger values compress faster and less well. A value less
than or equal to `0` is treated as `1`. This parameter has no effect on the
encoded format or on decoding; any conforming LZ4 block encoder may be used
with any value.

## Format

This is a `bytes -> bytes` codec. Let `N` be the length in bytes of the decoded
(uncompressed) data. The encoded representation is:

| offset | size (bytes) | content                                                   |
| ------ | ------------ | --------------------------------------------------------- |
| 0      | 4            | `N`, as an unsigned 32-bit little-endian integer          |
| 4      | remainder    | exactly one LZ4 block, as defined by the [LZ4 Block Format][lz4-block] |

The LZ4 block occupies every byte after the 4-byte header; no bytes may follow it.
This layout is the same as that produced by `lz4.block.compress(data,
store_size=True)` in the [python-lz4](https://python-lz4.readthedocs.io/en/stable/lz4.block.html)
package.

### Encoding

1. `N` must not exceed `0x7E000000` (2 113 929 216, the LZ4 block format's
   maximum input size). Encoders must reject larger inputs.
2. Write `N` as a 4-byte little-endian unsigned integer.
3. Append the LZ4 block obtained by compressing the whole input as one block.

### Decoding

1. If the input is shorter than 4 bytes, decoding fails.
2. Read `N` from the first 4 bytes (little-endian).
3. Decode the remaining bytes as one LZ4 block. Decoding fails if the block is
   malformed, if it does not consume all remaining bytes, or if it does not
   produce exactly `N` bytes.

Decoders should accept `N = 0` and return an empty output; numcodecs 0.17
rejects it (see [numcodecs#831](https://github.com/zarr-developers/numcodecs/issues/831)),
so writers should avoid encoding empty inputs with this codec. numcodecs also
reads the size field as a signed 32-bit integer, so values of `N` of `2^31` or
more are rejected; this does not matter in practice given the limit in encoding
step 1.

### Worked example

The 16-byte input `00 00 01 00 02 00 03 00 04 00 05 00 06 00 07 00` (the
`uint16` values 0 to 7 in little-endian order) is encoded, with the default
configuration, as the 22 bytes

```
10 00 00 00                                       N = 16
f0 01                                             LZ4 token: 15 + 1 = 16 literals, no match
00 00 01 00 02 00 03 00 04 00 05 00 06 00 07 00   literals
```

## Example

Array metadata written by zarr-python for
`compressors=[zarr.codecs.numcodecs.LZ4(acceleration=5)]`:

```json
{
  "data_type": "uint16",
  "codecs": [
    { "name": "bytes", "configuration": { "endian": "little" } },
    { "name": "numcodecs.lz4", "configuration": { "acceleration": 5 } }
  ]
}
```

With `zarr.codecs.numcodecs.LZ4()`, the codec is written as
`{"name": "numcodecs.lz4", "configuration": {}}`.

## Implementations

- zarr-python: [`zarr.codecs.numcodecs.LZ4`](https://zarr.readthedocs.io/en/stable/api/zarr/codecs/numcodecs/)
- numcodecs: [`numcodecs.LZ4`](https://numcodecs.readthedocs.io/en/stable/compression/lz4.html)
  (the codec implementation that zarr-python delegates to)

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)

[lz4-block]: https://github.com/lz4/lz4/blob/dev/doc/lz4_Block_format.md
