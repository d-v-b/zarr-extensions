# numcodecs.blosc codec

Defines a `bytes -> bytes` codec that compresses its input with the Blosc
(version 1) meta-compressor. Blosc splits the input into blocks, optionally
applies a byte-shuffle or bit-shuffle to each block, and compresses each block
with an internal compressor. It stores the result in the
[Blosc chunk format][blosc-format].

## Background

`numcodecs.blosc` is the name that zarr-python writes into Zarr V3 array
metadata for the `Blosc` codec of the
[numcodecs](https://github.com/zarr-developers/numcodecs) library
(`zarr.codecs.numcodecs.Blosc`, previously `numcodecs.zarr3.Blosc`). It is
registered here so that data already written under this name has a
specification that does not depend on the numcodecs source code. The
registered [`blosc`](../blosc/README.md) codec uses the same encoded format.
That codec records the shuffle mode and type size explicitly, so new data
should use `blosc` instead.

## Codec name

The value of the `name` member in the codec object must be `"numcodecs.blosc"`.

## Configuration parameters

Writers must include the `configuration` member, as a JSON object that may be
empty. Readers may treat a missing `configuration` as `{}`. zarr-python writes
only the parameters that the user passed explicitly, so readers must accept a
configuration with any optional field omitted and apply the default listed
below. Readers must accept and ignore an `id` field equal to `"blosc"`, which
numcodecs 0.14.0–0.15.0 wrote. Writers must not write it. No other fields are
permitted.

| field       | type    | required | default | notes                                                         |
| ----------- | ------- | -------- | ------- | ------------------------------------------------------------- |
| `cname`     | string  | no       | `"lz4"` | Internal compressor. Affects encoding only.                   |
| `clevel`    | integer | no       | `5`     | 0 to 9. Affects encoding only.                                |
| `shuffle`   | integer | no       | `1`     | `-1`, `0`, `1` or `2`. Affects encoding only.                 |
| `blocksize` | integer | no       | `0`     | ≥ 0. Affects encoding only.                                   |
| `typesize`  | integer | no       | `1`     | ≥ 1. Affects encoding only. Requires numcodecs ≥ 0.16.        |

The chunk header records every parameter the decoder needs. Decoding therefore
never uses the configuration: a decoder must decode any valid Blosc chunk
whatever the configuration says.

### `cname`

The internal compressor, one of `"blosclz"`, `"lz4"`, `"lz4hc"`, `"zlib"`,
`"zstd"` or `"snappy"`. numcodecs builds of c-blosc do not include Snappy, so
numcodecs cannot write or read `"snappy"` data.

### `clevel`

The compression level, from 0 to 9. A level of `0` stores the input without
compression (see [memcpyed chunks](#memcpyed-chunks)).

### `typesize`

The element size in bytes, `T`, used by the shuffle filters and stored in the
chunk header.

**Default.** If `typesize` is omitted, `T` is **1**. zarr-python passes the
encoder a buffer of bytes (`uint8`), and numcodecs uses that buffer's item
size. The array's data type is not taken into account. This differs from Zarr
V2 and from the [`blosc`](../blosc/README.md) codec, where `T` usually equals
the element size of the array's data type. If `T` is 1, byte-shuffle has no
effect. The chunk header field holds values from 1 to 255. If `typesize` is
greater than 255, c-blosc stores and uses `T = 1`.

### `shuffle`

| value | meaning                                                         |
| ----- | --------------------------------------------------------------- |
| `-1`  | automatic: bit-shuffle if `T` is 1, otherwise byte-shuffle      |
| `0`   | no shuffle                                                      |
| `1`   | byte-shuffle                                                    |
| `2`   | bit-shuffle                                                     |

`T` is the effective type size described above. With zarr-python, `-1`
therefore means bit-shuffle unless `typesize` is set to a value from 2 to 255.

### `blocksize`

The requested block size in bytes. `0` lets the encoder choose. c-blosc raises
a requested size below 128 to 128 and rounds it down to a multiple of `T`. The
block size used is recorded in the chunk header.

## Format

This is a `bytes -> bytes` codec. The encoded representation is one chunk in
version 2 of the Blosc 1.x chunk format, as described in
[README_CHUNK_FORMAT][blosc-format] and implemented by
[c-blosc](https://github.com/Blosc/c-blosc). This section restates that format,
and settles points that the document leaves ambiguous by following c-blosc
1.21.

All integers are little-endian.

### Header

| offset | size | field       | meaning                                                                  |
| ------ | ---- | ----------- | ------------------------------------------------------------------------ |
| 0      | 1    | `version`   | Blosc format version; must be `2`                                        |
| 1      | 1    | `versionlz` | format version of the internal compressor (`1` for all compressors)      |
| 2      | 1    | `flags`     | see below                                                                |
| 3      | 1    | `typesize`  | `T`, 1 to 255                                                            |
| 4      | 4    | `nbytes`    | uncompressed size `N` (uint32)                                           |
| 8      | 4    | `blocksize` | block size `B` (uint32)                                                  |
| 12     | 4    | `cbytes`    | total size of the encoded chunk, including this header (uint32)          |

`flags`:

| bit(s)  | mask   | meaning when set                                                       |
| ------- | ------ | ---------------------------------------------------------------------- |
| 0       | `0x01` | byte-shuffle was applied                                               |
| 1       | `0x02` | memcpyed: the data is stored uncompressed                              |
| 2       | `0x04` | bit-shuffle was applied                                                |
| 3       | `0x08` | reserved; must be 0                                                    |
| 4       | `0x10` | "don't split": blocks are not split into `T` streams                   |
| 5–7     | `0xE0` | compressor code: `0` blosclz, `1` lz4/lz4hc, `2` snappy, `3` zlib, `4` zstd |

`N` must not exceed 2^31 − 1 − 16 bytes.

### Memcpyed chunks

If flag bit 1 is set, `cbytes` must equal `N + 16`, and the `N` bytes after the
header are the decoded data. The shuffle flags have no effect in this case,
because the stored bytes are not shuffled. c-blosc writes a memcpyed chunk when
`clevel` is 0, when `N < 128`, or when compression would not reduce the size.

### Compressed chunks

If flag bit 1 is clear, let `nblocks = ceil(N / B)` and `leftover = N mod B`.
After the header comes `bstarts`: `nblocks` signed 32-bit integers. `bstarts[k]`
is the offset of block `k`'s data, **measured from the start of the chunk**
(byte 0 of the header).

Block `k` decodes to `bsize = B` bytes, except that the last block decodes to
`leftover` bytes when `leftover > 0`. A block's data is a sequence of `nsplits`
splits, where

```
nsplits = T   if flag 0x10 is clear and T <= 16 and floor(B / T) >= 128
                 and block k is not the leftover block
          1   otherwise
```

Each split is a signed 32-bit `csize` followed by `csize` bytes, and decodes to
`neblock = bsize / nsplits` bytes. If `csize == neblock`, the bytes are stored
uncompressed. Otherwise they are one stream of the internal compressor, which
must decode to exactly `neblock` bytes:

| code | compressor  | stream format                                                               |
| ---- | ----------- | --------------------------------------------------------------------------- |
| 0    | blosclz     | BloscLZ; no published specification exists apart from c-blosc's [`blosclz.c`](https://github.com/Blosc/c-blosc/blob/main/blosc/blosclz.c) |
| 1    | lz4, lz4hc  | one [LZ4 block](https://github.com/lz4/lz4/blob/dev/doc/lz4_Block_format.md), with no size prefix |
| 2    | snappy      | one Snappy [raw stream](https://github.com/google/snappy/blob/main/format_description.txt) |
| 3    | zlib        | one zlib stream ([RFC 1950](https://www.rfc-editor.org/rfc/rfc1950))         |
| 4    | zstd        | one Zstandard frame ([RFC 8878](https://www.rfc-editor.org/rfc/rfc8878))     |

Concatenating the decoded splits of block `k` gives the shuffled block `s` of
`bsize` bytes. The decoded block is then

- the inverse byte-shuffle of `s`, if flag bit 0 is set and `T > 1`;
- the inverse bit-shuffle of `s`, if flag bit 2 is set and `bsize >= T`;
- `s` itself, otherwise.

The decoded block occupies bytes `k × B` to `k × B + bsize` of the output.

### Shuffle filters

Both filters work on one block of `bsize` bytes at a time. Let `n = floor(bsize / T)`.

**Byte-shuffle.** The first `n × T` bytes are read as an `n × T` row-major byte
matrix and written transposed. The remaining `bsize − n × T` bytes are copied
unchanged to the end:

```
shuffled[j * n + i] = block[i * T + j]     for 0 <= i < n, 0 <= j < T
```

**Bit-shuffle.** This is the bitshuffle transform of the
[bitshuffle](https://github.com/kiyo-masui/bitshuffle) library. Let
`m = n − (n mod 8)`. The first `m × T` bytes are rewritten as `8 × T` bit planes
of `m / 8` bytes each. Plane `p = 8 × j + b` holds bit `b` (bit 0 is the least
significant) of byte `j` of every element, in element order. Element `i`'s bit
is stored in byte `floor(i / 8)` of the plane, at bit position `i mod 8`
(least-significant first). The remaining `bsize − m × T` bytes are copied
unchanged to the end. The inverse transformations restore the original order.

### Encoding

The encoder must produce a chunk that conforms to the description above, with
`typesize` field `T` and shuffle flag chosen as described under
[Configuration parameters](#configuration-parameters), and the compressor code
of `cname`. The encoder chooses the block size (subject to `blocksize`),
whether to split blocks, and whether to fall back to a memcpyed chunk. These
choices are recorded in the chunk, so different encoders may produce different
bytes for the same input.

### Decoding

1. Read the header. Fail if `version` is not 2, if flag bit 3 is set, or if
   `typesize` is 0.
2. If `N` is 0, the output is empty. numcodecs 0.17 fails here instead
   ([numcodecs#831](https://github.com/zarr-developers/numcodecs/issues/831)),
   so writers should avoid encoding empty inputs.
3. Otherwise, decode as a memcpyed or compressed chunk as described above. The
   output has length `N`.

### Worked example

zarr-python with the default configuration (`{}`) encodes the 16-byte input
`00 00 01 00 02 00 03 00 04 00 05 00 06 00 07 00` (the `uint16` values 0 to 7 in
little-endian order) as this 32-byte memcpyed chunk:

```
02                    version
01                    versionlz
33                    flags: byte-shuffle | memcpyed | don't split | compressor 1 (lz4)
01                    typesize T = 1 (the default; not the item size of uint16)
10 00 00 00           nbytes = 16
10 00 00 00           blocksize = 16
20 00 00 00           cbytes = 32
00 00 01 00 02 00 03 00 04 00 05 00 06 00 07 00
```

With `{"cname": "zstd", "clevel": 3, "shuffle": -1}`, the flags byte is `96`:
bit-shuffle (because `T` is 1), memcpyed, don't split, and compressor 4 (zstd).

## Example

Array metadata written by zarr-python for
`compressors=[zarr.codecs.numcodecs.Blosc(cname="zstd", clevel=3, shuffle=-1)]`:

```json
{
  "data_type": "uint16",
  "codecs": [
    { "name": "bytes", "configuration": { "endian": "little" } },
    {
      "name": "numcodecs.blosc",
      "configuration": { "cname": "zstd", "clevel": 3, "shuffle": -1 }
    }
  ]
}
```

## Relationship to `blosc`

The registered [`blosc`](../blosc/README.md) codec, specified in the Zarr V3
core specification, uses the same Blosc chunk format. Decoding depends only on
the chunk header, so a `blosc` decoder decodes every `numcodecs.blosc` chunk and
vice versa. zarr-python implements both codecs with the same numcodecs `Blosc`
encoder. Given the same `cname`, `clevel`, shuffle mode, type size and
`blocksize`, the two codecs produce identical bytes.

The configurations differ as follows:

| `numcodecs.blosc`                                  | `blosc`                                                 |
| -------------------------------------------------- | ------------------------------------------------------- |
| all fields optional                                | `cname`, `clevel`, `shuffle`, `blocksize` required      |
| `shuffle` is an integer: `-1`, `0`, `1`, `2`       | `shuffle` is `"noshuffle"`, `"shuffle"`, `"bitshuffle"` |
| `typesize` optional, default 1                     | `typesize` required unless `shuffle` is `"noshuffle"`   |

An array's metadata can be rewritten from `numcodecs.blosc` to `blosc` without
re-encoding any chunks. To keep future writes identical to past ones, use the
effective values, not the array's data type:

```
T        = C.typesize if present (and <= 255), else 1
shuffle  = { -1: ("bitshuffle" if T == 1 else "shuffle"),
              0: "noshuffle", 1: "shuffle", 2: "bitshuffle" }[C.shuffle if present, else 1]
{"name": "numcodecs.blosc", "configuration": C}
  -> {"name": "blosc", "configuration": {
        "cname": C.cname or "lz4", "clevel": C.clevel or 5, "shuffle": shuffle,
        "typesize": T, "blocksize": C.blocksize or 0}}
```

(Here `C.clevel or 5` means `C.clevel` if present, else 5. An explicit `0` is
kept.)

This mapping differs from the one zarr-python uses to migrate Zarr V2 arrays
(`zarr migrate v3`, which resolves `-1` from the array's item size). In Zarr V2,
numcodecs received a typed array, so `T` was the item size. With
`numcodecs.blosc` in Zarr V3, it receives bytes.

Writers should prefer `blosc` for new data and set `typesize` to the element
size of the data being shuffled.

## Implementations

- zarr-python: [`zarr.codecs.numcodecs.Blosc`](https://zarr.readthedocs.io/en/stable/api/zarr/codecs/numcodecs/)
- numcodecs: [`numcodecs.Blosc`](https://numcodecs.readthedocs.io/en/stable/compression/blosc.html)
  (the codec implementation that zarr-python delegates to)

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)

[blosc-format]: https://github.com/Blosc/c-blosc/blob/main/README_CHUNK_FORMAT.rst
