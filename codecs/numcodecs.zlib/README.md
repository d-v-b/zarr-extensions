# numcodecs.zlib codec

Defines a `bytes -> bytes` codec that compresses its input with DEFLATE
([RFC 1951][RFC1951]) and wraps the result in the zlib container
([RFC 1950][RFC1950]). This container uses a 2-byte header and an Adler-32
trailer. It is not the gzip container ([RFC 1952][RFC1952]) that the
[`gzip`](../gzip/README.md) and [`numcodecs.gzip`](../numcodecs.gzip/README.md)
codecs use.

## Background

`numcodecs.zlib` is the name that zarr-python writes into Zarr V3 array metadata
for the `Zlib` codec of the [numcodecs](https://github.com/zarr-developers/numcodecs)
library (`zarr.codecs.numcodecs.Zlib`, previously `numcodecs.zarr3.Zlib`). It is
registered here so that data already written under this name has a
specification that does not depend on the numcodecs source code. No registered
codec without the `numcodecs.` prefix uses the zlib container.

## Codec name

The value of the `name` member in the codec object must be `"numcodecs.zlib"`.

## Configuration parameters

Writers must include the `configuration` member, as a JSON object that may be
empty. Readers may treat a missing `configuration` as `{}`. zarr-python
writes only the parameters that the user passed explicitly, so readers must
accept a configuration with any optional field omitted and apply the default
listed below. Readers must accept and ignore an `id` field equal to
`"zlib"`, which numcodecs 0.14.0–0.15.0 wrote. Writers must not write it. No
other fields are permitted.

| field   | type    | required | default | notes                           |
| ------- | ------- | -------- | ------- | ------------------------------- |
| `level` | integer | no       | `1`     | -1 to 9. Affects encoding only. |

### `level`

The zlib compression level. `0` stores the data without compression in DEFLATE
"stored" blocks. Levels 1 to 9 trade speed for compression ratio, from fastest
to strongest. `-1` selects the zlib library default, which is equivalent to
level 6. The level has no effect on decoding.

## Format

This is a `bytes -> bytes` codec. The encoded representation is **exactly one**
zlib stream as defined in [RFC 1950][RFC1950]:

| size (bytes) | content                                                                          |
| ------------ | -------------------------------------------------------------------------------- |
| 1            | `CMF`: `CM = 8` (deflate), `CINFO = 7` (32 KiB window) as written by numcodecs   |
| 1            | `FLG`: `FDICT = 0` (no preset dictionary), `FLEVEL`, `FCHECK`                    |
| variable     | compressed data in the DEFLATE format ([RFC 1951][RFC1951])                      |
| 4            | Adler-32 checksum of the decoded data, **big-endian**                            |

### Encoding

Compress the whole input as one zlib stream at the configured `level`. Write
nothing before or after the stream. The `FLEVEL` bits record the level class:
`0` for level 0 or 1, `1` for levels 2 to 5, `2` for level 6 (and -1), and `3`
for levels 7 to 9. The exact DEFLATE bit stream depends on the encoder
implementation and version, so encoded bytes are not guaranteed to be identical
across implementations.

### Decoding

Decoders must accept any zlib stream that conforms to RFC 1950 and has
`FDICT = 0`. They must verify the Adler-32 checksum and fail on a mismatch. The
decoded output is the decompressed content of the stream. Data in the gzip
format ([RFC 1952][RFC1952]) and raw DEFLATE data without the zlib header are
not valid encodings for this codec, and numcodecs rejects both. numcodecs
ignores any bytes after the end of the zlib stream, but writers must not
produce such bytes.

### Worked example

The 16-byte input `00 00 01 00 02 00 03 00 04 00 05 00 06 00 07 00` (the
`uint16` values 0 to 7 in little-endian order) is encoded with the default
configuration (level 1) as the 24 bytes

```
78 01                                                    CMF, FLG (FLEVEL = 0)
63 60 60 64 60 62 60 66 60 61 60 65 60 63 60 67 00 00    DEFLATE data
00 b8 00 1d                                              Adler-32 = 0x00B8001D
```

## Example

Array metadata written by zarr-python for
`compressors=[zarr.codecs.numcodecs.Zlib(level=9)]`:

```json
{
  "data_type": "uint16",
  "codecs": [
    { "name": "bytes", "configuration": { "endian": "little" } },
    { "name": "numcodecs.zlib", "configuration": { "level": 9 } }
  ]
}
```

## Implementations

- zarr-python: [`zarr.codecs.numcodecs.Zlib`](https://zarr.readthedocs.io/en/stable/api/zarr/codecs/numcodecs/)
- numcodecs: [`numcodecs.Zlib`](https://numcodecs.readthedocs.io/en/stable/compression/zlib.html)
  (the codec implementation that zarr-python delegates to)
- zarrs: [`zlib`](https://docs.rs/zarrs/latest/zarrs/array/codec/bytes_to_bytes/zlib/index.html)
  (reads and writes `numcodecs.zlib`)

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)

[RFC1950]: https://www.rfc-editor.org/rfc/rfc1950
[RFC1951]: https://www.rfc-editor.org/rfc/rfc1951
[RFC1952]: https://www.rfc-editor.org/rfc/rfc1952
