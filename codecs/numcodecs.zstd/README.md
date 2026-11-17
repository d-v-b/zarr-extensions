---
status: superseded
superseded_by: [zstd]
---

# numcodecs.zstd codec

Defines a `bytes -> bytes` codec that compresses its input with
[Zstandard][RFC8878]. The encoded representation is a Zstandard frame with no
additional header or trailer.

## Background

`numcodecs.zstd` is the name that zarr-python writes into Zarr V3 array metadata
for the `Zstd` codec of the [numcodecs](https://github.com/zarr-developers/numcodecs)
library (`zarr.codecs.numcodecs.Zstd`, previously `numcodecs.zarr3.Zstd`). It is
registered here so that data already written under this name has a
specification that does not depend on the numcodecs source code. The
registered [`zstd`](../zstd/README.md) codec produces the same encoded bytes;
new data should use `zstd` instead.

## Codec name

The value of the `name` member in the codec object must be `"numcodecs.zstd"`.

## Configuration parameters

Writers must include the `configuration` member, as a JSON object that may be
empty. Readers may treat a missing `configuration` as `{}`. zarr-python
writes only the parameters that the user passed explicitly, so readers must
accept a configuration with any optional field omitted and apply the default
listed below. Readers must accept and ignore an `id` field equal to
`"zstd"`, which numcodecs 0.14.0–0.15.0 wrote. Writers must not write it. No
other fields are permitted.

| field      | type    | required | default | notes                                        |
| ---------- | ------- | -------- | ------- | -------------------------------------------- |
| `level`    | integer | no       | `0`     | -131072 to 22. Affects encoding only.        |
| `checksum` | boolean | no       | `false` | Whether the encoder writes a content checksum. |

### `level`

The Zstandard compression level. `0` selects the default level of the zstd
library (currently 3). Negative values select the faster levels, and higher
positive values compress more slowly and more strongly. numcodecs lowers values
above 22 to 22, and the zstd library raises values below -131072 to -131072.
Writers should only write values in the range -131072 to 22. The level has no
effect on decoding.

### `checksum`

If `true`, each frame written by the encoder has its `Content_Checksum_flag`
set and ends with a content checksum ([RFC 8878, section 3.1.1][RFC8878]).
If `false`, the encoder does not write a checksum. Decoders do not use this
parameter, because each frame header records whether a checksum is present.

## Format

This is a `bytes -> bytes` codec.

### Encoding

The encoder compresses the whole input into **exactly one** Zstandard frame as
defined in [RFC 8878, section 3.1.1][RFC8878], and writes nothing else. The
frame produced by numcodecs has these properties:

- `Frame_Content_Size` is present and equals the input length. In the frame
  header descriptor, `Frame_Content_Size_flag` is non-zero, or
  `Single_Segment_flag` is set, which also implies that the field is present.
- `Dictionary_ID_flag` is 0. No dictionary is used.
- `Content_Checksum_flag` equals `checksum`. When it is set, the frame ends with
  the low 32 bits of the XXH64 hash (seed 0) of the decoded content, in
  little-endian order.

### Decoding

The decoder must accept a sequence of one or more frames, as described in
[RFC 8878, section 3.1][RFC8878]. The sequence may contain:

- Zstandard frames, with or without `Frame_Content_Size`, and with or without a
  content checksum;
- skippable frames (magic numbers `0x184D2A50` to `0x184D2A5F`), which the
  decoder ignores.

The decoded output is the concatenation of the content of all Zstandard frames,
in order. If a frame has a content checksum, decoders should verify it and fail
on a mismatch. Decoding fails if any bytes remain that do not form a complete
frame. Frames that need a dictionary are not supported.

This is what numcodecs does. Writers should still emit a single frame with
`Frame_Content_Size` present, as described under [Encoding](#encoding),
because of the following limitation. When every Zstandard frame in the input
declares a content size, numcodecs 0.17 rejects input whose declared sizes add
up to 0. So it cannot decode its own encoding of an empty input
([numcodecs#831](https://github.com/zarr-developers/numcodecs/issues/831)).
Decoders should return an empty output in that case.

### Worked example

The 16-byte input `00 00 01 00 02 00 03 00 04 00 05 00 06 00 07 00` (the
`uint16` values 0 to 7 in little-endian order) is encoded with
`{"checksum": true}` as the 29 bytes

```
28 b5 2f fd                                       magic number 0xFD2FB528
24                                                frame header descriptor: Single_Segment_flag=1, Content_Checksum_flag=1
10                                                Frame_Content_Size = 16 (1-byte field)
81 00 00                                          block header: Last_Block=1, Block_Type=Raw, Block_Size=16
00 00 01 00 02 00 03 00 04 00 05 00 06 00 07 00   raw block content
88 49 49 01                                       content checksum
```

With `{}` (no checksum), the frame header descriptor is `20` and the last 4
bytes are absent.

## Example

Array metadata written by zarr-python for
`compressors=[zarr.codecs.numcodecs.Zstd(level=3, checksum=True)]`:

```json
{
  "data_type": "uint16",
  "codecs": [
    { "name": "bytes", "configuration": { "endian": "little" } },
    {
      "name": "numcodecs.zstd",
      "configuration": { "level": 3, "checksum": true }
    }
  ]
}
```

## Relationship to `zstd`

The registered [`zstd`](../zstd/README.md) codec uses the same encoded format,
which is a Zstandard frame with no extra header. zarr-python implements both
codecs with the same numcodecs `Zstd` encoder and passes it the same `level`
and `checksum`. For the same parameters and zstd library version, the two
codecs therefore produce byte-identical output.

The configurations differ as follows:

| `numcodecs.zstd`                           | `zstd`                                         |
| ------------------------------------------ | ---------------------------------------------- |
| `level` optional, default `0`              | `level` required                               |
| `checksum` optional, default `false`       | `checksum` optional; omit if `false`           |

An array's metadata can therefore be rewritten from `numcodecs.zstd` to `zstd`
without re-encoding any chunks, using this mapping:

```
{"name": "numcodecs.zstd", "configuration": C}
  -> {"name": "zstd", "configuration": {"level": C.level or 0, "checksum": true}}   if C.checksum is true
  -> {"name": "zstd", "configuration": {"level": C.level or 0}}                     otherwise
```

Writers should prefer `zstd` for new data.

## Implementations

- zarr-python: [`zarr.codecs.numcodecs.Zstd`](https://zarr.readthedocs.io/en/stable/api/zarr/codecs/numcodecs/)
- numcodecs: [`numcodecs.Zstd`](https://numcodecs.readthedocs.io/en/stable/compression/zstd.html)
  (the codec implementation that zarr-python delegates to)

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)

[RFC8878]: https://www.rfc-editor.org/rfc/rfc8878
