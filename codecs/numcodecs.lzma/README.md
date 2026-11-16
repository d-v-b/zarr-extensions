# numcodecs.lzma codec

Defines a `bytes -> bytes` codec that compresses its input with the LZMA family
of algorithms, using the API of the liblzma library (part of
[XZ Utils](https://tukaani.org/xz/format.html)) as exposed by Python's
[`lzma`](https://docs.python.org/3/library/lzma.html) module. Depending on
`format`, the encoded representation is an `.xz` stream, a legacy `.lzma`
stream, or a raw filter-chain stream with no container.

## Background

`numcodecs.lzma` is the name that zarr-python writes into Zarr V3 array metadata
for the `LZMA` codec of the [numcodecs](https://github.com/zarr-developers/numcodecs)
library (`zarr.codecs.numcodecs.LZMA`, previously `numcodecs.zarr3.LZMA`). It is
registered here so that data already written under this name has a
specification that does not depend on the numcodecs source code. No registered
codec without the `numcodecs.` prefix uses LZMA.

## Codec name

The value of the `name` member in the codec object must be `"numcodecs.lzma"`.

## Configuration parameters

Writers must include the `configuration` member, as a JSON object that may be
empty. Readers may treat a missing `configuration` as `{}`. zarr-python
writes only the parameters that the user passed explicitly, so readers must
accept a configuration with any optional field omitted and apply the default
listed below. Readers must accept and ignore an `id` field equal to
`"lzma"`, which numcodecs 0.14.0–0.15.0 wrote. Writers must not write it. No
other fields are permitted.

| field     | type                     | required | default | notes                                                          |
| --------- | ------------------------ | -------- | ------- | -------------------------------------------------------------- |
| `format`  | integer                  | no       | `1`     | Container format: `1` = `.xz`, `2` = `.lzma`, `3` = raw. `0` can be used only for decoding. |
| `check`   | integer                  | no       | `-1`    | Integrity check type for `.xz`. Affects encoding only.         |
| `preset`  | integer or `null`        | no       | `null`  | Compression preset. Affects encoding only.                     |
| `filters` | array of objects or `null` | no     | `null`  | Filter chain. Used for decoding only when `format` is `3`.     |

### `format`

| value | liblzma name   | encoded representation                                         |
| ----- | -------------- | -------------------------------------------------------------- |
| `0`   | `FORMAT_AUTO`  | decoding only: the decoder accepts either `.xz` or `.lzma`     |
| `1`   | `FORMAT_XZ`    | one `.xz` stream ([.xz file format][xz-format])                |
| `2`   | `FORMAT_ALONE` | one legacy `.lzma` stream ([.lzma file format][lzma-alone])    |
| `3`   | `FORMAT_RAW`   | raw output of the filter chain, with no header or trailer      |

numcodecs cannot encode with `format` `0`, so arrays configured with it can be
read but not written. Writers must not write `0`.

### `check`

The integrity check stored in an `.xz` stream:

| value | meaning                                                                  |
| ----- | ------------------------------------------------------------------------ |
| `-1`  | the default: CRC64 for `format` `1`; no check for `format` `2` and `3`   |
| `0`   | no check (`.xz` Check ID `0x00`)                                         |
| `1`   | CRC32 (`.xz` Check ID `0x01`)                                            |
| `4`   | CRC64 (`.xz` Check ID `0x04`)                                            |
| `10`  | SHA-256 (`.xz` Check ID `0x0A`)                                          |

If `format` is `2` or `3`, `check` must be `-1` or `0`. The `.xz` stream records
its check type, so decoders do not use this parameter.

### `preset`

An integer from 0 to 9 that selects the compression preset, optionally combined
by bitwise OR with the "extreme" flag `0x80000000` (2147483648); for example,
`2147483654` is preset 6 with the extreme flag. `null` (or omission) selects the
default preset 6. `preset` must be `null` or omitted when `filters` is not
`null`. Presets correspond to LZMA1/LZMA2 encoder options defined by liblzma.
They affect only the encoder, because the format records the options a decoder
needs.

### `filters`

`null` (or omission) means a single LZMA2 filter (or, for `format` `2`, a single
LZMA1 filter) configured by `preset`. Otherwise, `filters` is an array of 1 to 4
filter objects that describes the encoder's filter chain, in order. Each object
has an integer `id` and filter-specific integer options, with the names used by
Python's `lzma` module:

| `id`                    | filter        | options                                                                          |
| ----------------------- | ------------- | -------------------------------------------------------------------------------- |
| `4611686018427387905` (`0x4000000000000001`) | LZMA1 | `preset`, `dict_size`, `lc`, `lp`, `pb`, `mode`, `nice_len`, `mf`, `depth` |
| `33` (`0x21`)           | LZMA2         | same as LZMA1                                                                    |
| `3`                     | Delta         | `dist` (1 to 256)                                                                |
| `4`, `5`, `6`, `7`, `8`, `9` | BCJ: x86, PowerPC, IA-64, ARM, ARM-Thumb, SPARC | `start_offset`                         |

Option values: `mode` is `1` (fast) or `2` (normal), and `mf` is `3` (hc3), `4`
(hc4), `18` (bt2), `19` (bt3) or `20` (bt4). The meaning of each option is that
of the corresponding field of liblzma's `lzma_options_lzma`,
`lzma_options_delta` or `lzma_options_bcj` structure, as documented for
Python's [`lzma`](https://docs.python.org/3/library/lzma.html#specifying-custom-filter-chains)
module. The last filter in the chain must be LZMA1 or LZMA2. For `format` `2`,
the chain must be a single LZMA1 filter, and for `format` `1` LZMA1 is not
allowed.

The LZMA1 filter ID is larger than 2^53, so JSON parsers that read numbers as
IEEE 754 doubles cannot represent it exactly. Readers that need this value must
parse it as a 64-bit integer.

## Format

This is a `bytes -> bytes` codec. Encoders compress the whole input in a
single call and write exactly one stream of the selected format.

### `format` 1: `.xz`

The encoded bytes are exactly one `.xz` Stream as defined by the
[.xz file format][xz-format] (version 1.x). The Stream contains:

- the Stream Header, whose Stream Flags give the check type from `check`;
- one Block, whose Block Header lists the filter chain (`filters`, or LZMA2
  configured by `preset`), followed by the compressed data and the check;
- the Index and the Stream Footer.

An empty input produces a Stream with no Blocks. Decoders must accept any valid
`.xz` Stream and must verify its integrity check (unless the check is `0`). All
the information needed for decoding is recorded in the Stream, so decoders must
ignore `check`, `preset` and `filters` when `format` is `1`.
numcodecs (through Python's `lzma` module) also accepts several concatenated
Streams. It does not accept Stream Padding between or after Streams.

**Known defect.** numcodecs 0.17 passes `filters` to the decoder for every
format. When `format` is `1` or `2` and `filters` is not `null`, numcodecs
therefore fails to decode, so zarr-python cannot read back data that it wrote
with such a configuration
([numcodecs#188](https://github.com/zarr-developers/numcodecs/issues/188)).
The encoded bytes are still valid `.xz` or `.lzma` streams, and decoders that
follow this specification decode them.

### `format` 2: `.lzma` (LZMA_Alone)

The encoded bytes are exactly one legacy `.lzma` stream as described in the
[.lzma file format][lzma-alone]:

| offset | size | content                                                                    |
| ------ | ---- | -------------------------------------------------------------------------- |
| 0      | 1    | LZMA1 properties byte: `(pb × 5 + lp) × 9 + lc`                            |
| 1      | 4    | dictionary size, unsigned little-endian                                    |
| 5      | 8    | uncompressed size, unsigned little-endian; `0xFFFFFFFFFFFFFFFF` = unknown  |
| 13     | ...  | LZMA1 compressed data                                                      |

numcodecs always writes the uncompressed size as unknown (`ff` × 8), so the
LZMA1 data ends with an end-of-payload marker. Decoders must also accept
streams with a known uncompressed size. The LZMA1 coding itself is specified
by the LZMA SDK's `lzma-specification.txt` (part of the
[LZMA SDK](https://www.7-zip.org/sdk.html)).

### `format` 3: raw

The encoded bytes are the output of the liblzma raw encoder for the filter
chain `filters`, with no header, trailer or integrity check. `filters` must not
be `null`, and the decoder must use the same chain, because the stream does not
record it. If the last filter is LZMA2, the data is a sequence of LZMA2 chunks
that ends with a `00` byte. If it is LZMA1, the data is LZMA1 data that ends
with an end-of-payload marker. In both cases the data is the same as the
compressed data of an `.xz` Block or `.lzma` stream using the same filter
chain. No published specification exists for the LZMA2 chunk format apart from
the liblzma implementation.

### Worked examples

The 16-byte input `00 00 01 00 02 00 03 00 04 00 05 00 06 00 07 00` (the
`uint16` values 0 to 7 in little-endian order) is encoded with the default
configuration (`.xz`, CRC64, preset 6) as the 72 bytes

```
fd 37 7a 58 5a 00 00 04 e6 d6 b4 46              Stream Header: magic, flags (CRC64), CRC32
02 00 21 01 16 00 00 00 74 2f e5 a3              Block Header: size, flags, LZMA2 filter (props 0x16), padding, CRC32
01 00 0f                                         LZMA2 chunk: uncompressed, dictionary reset, 16 bytes
00 00 01 00 02 00 03 00 04 00 05 00 06 00 07 00  chunk data
00                                               LZMA2 end marker
e0 1d 9b f2 f7 e1 c8 10                          CRC64 of the uncompressed data
00 01 28 10 e5 0b 6c 60                          Index: indicator, 1 record (unpadded 40, uncompressed 16), CRC32
1f b6 f3 7d 01 00 00 00 00 04 59 5a              Stream Footer: CRC32, backward size, flags, magic
```

With `{"format": 3, "filters": [{"id": 33, "preset": 1}]}`, the same input
encodes to just the 20-byte LZMA2 data above:
`01 00 0f 00 00 01 00 … 07 00 00`.

With `{"format": 2}`, the encoding starts with the 13-byte header
`5d 00 00 80 00 ff ff ff ff ff ff ff ff`. This header gives `lc = 3`, `lp = 0`,
`pb = 2`, a dictionary size of 8 MiB, and an unknown uncompressed size.

## Example

Array metadata written by zarr-python for
`compressors=[zarr.codecs.numcodecs.LZMA(preset=6, check=0)]`:

```json
{
  "data_type": "uint16",
  "codecs": [
    { "name": "bytes", "configuration": { "endian": "little" } },
    { "name": "numcodecs.lzma", "configuration": { "preset": 6, "check": 0 } }
  ]
}
```

A raw LZMA2 stream with a delta pre-filter:

```json
{
  "name": "numcodecs.lzma",
  "configuration": {
    "format": 3,
    "filters": [
      { "id": 3, "dist": 2 },
      { "id": 33, "preset": 1 }
    ]
  }
}
```

## Implementations

- zarr-python: [`zarr.codecs.numcodecs.LZMA`](https://zarr.readthedocs.io/en/stable/api/zarr/codecs/numcodecs/)
- numcodecs: [`numcodecs.LZMA`](https://numcodecs.readthedocs.io/en/stable/compression/lzma.html)
  (the codec implementation that zarr-python delegates to)

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)

[xz-format]: https://tukaani.org/xz/xz-file-format.txt
[lzma-alone]: https://github.com/tukaani-project/xz/blob/master/doc/lzma-file-format.txt
