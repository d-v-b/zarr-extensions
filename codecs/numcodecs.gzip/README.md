# numcodecs.gzip codec

Defines a `bytes -> bytes` codec that compresses its input with DEFLATE
([RFC 1951][RFC1951]) and wraps the result in the gzip container
([RFC 1952][RFC1952]).

## Background

`numcodecs.gzip` is the name that zarr-python writes into Zarr V3 array metadata
for the `GZip` codec of the [numcodecs](https://github.com/zarr-developers/numcodecs)
library (`zarr.codecs.numcodecs.GZip`, previously `numcodecs.zarr3.GZip`). It is
registered here so that data already written under this name has a
specification that does not depend on the numcodecs source code. The
registered [`gzip`](../gzip/README.md) codec uses the same encoded format, so
new data should use `gzip` instead.

## Codec name

The value of the `name` member in the codec object must be `"numcodecs.gzip"`.

## Configuration parameters

Writers must include the `configuration` member, as a JSON object that may be
empty. Readers may treat a missing `configuration` as `{}`. zarr-python
writes only the parameters that the user passed explicitly, so readers must
accept a configuration with any optional field omitted and apply the default
listed below. Readers must accept and ignore an `id` field equal to
`"gzip"`, which numcodecs 0.14.0–0.15.0 wrote. Writers must not write it. No
other fields are permitted.

| field   | type    | required | default | notes                           |
| ------- | ------- | -------- | ------- | ------------------------------- |
| `level` | integer | no       | `1`     | -1 to 9. Affects encoding only. |

### `level`

The DEFLATE compression level. `0` stores the data without compression in
DEFLATE "stored" blocks. Levels 1 to 9 trade speed for compression ratio, from
fastest to strongest. `-1` selects the zlib library default, which is
equivalent to level 6. The level has no effect on decoding.

## Format

This is a `bytes -> bytes` codec. The encoded representation is a gzip file as
defined in [RFC 1952][RFC1952]: a series of one or more gzip members. Each
member consists of a header, DEFLATE-compressed data ([RFC 1951][RFC1951]), the
CRC-32 of the uncompressed data (4 bytes, little-endian), and the uncompressed
size modulo 2^32 (`ISIZE`, 4 bytes, little-endian).

### Encoding

The encoder compresses the whole input into **exactly one** gzip member and
writes nothing before or after it. The member header written by numcodecs (via
Python's `gzip` module) is 10 bytes long:

| offset | size | field   | value written by numcodecs                                         |
| ------ | ---- | ------- | ------------------------------------------------------------------ |
| 0      | 2    | ID1 ID2 | `1f 8b`                                                            |
| 2      | 1    | CM      | `08` (deflate)                                                     |
| 3      | 1    | FLG     | `00` (no optional fields)                                          |
| 4      | 4    | MTIME   | time of encoding, Unix seconds, little-endian                      |
| 8      | 1    | XFL     | depends on the Python version; Python 3.12 writes `02` if `level` is 9, `04` if `level` is 1, otherwise `00` |
| 9      | 1    | OS      | `ff` (unknown)                                                     |

Because `MTIME` records the time of encoding, numcodecs does not encode the
same input to the same bytes in different seconds. The DEFLATE bit stream also
depends on the encoder implementation and version.

### Decoding

Decoders must accept any data that conforms to RFC 1952:

- one or more members, whose decompressed contents are concatenated in order;
- any combination of the optional header fields `FEXTRA`, `FNAME`, `FCOMMENT`
  and `FHCRC`;
- any value of `MTIME`, `XFL` and `OS`, which decoders ignore.

Decoders must verify the CRC-32 and `ISIZE` of each member and fail on a
mismatch. numcodecs additionally ignores zero bytes that follow the last member.

Data in the zlib format ([RFC 1950][RFC1950]) is **not** a valid encoding for
this codec. numcodecs rejects it, and so does raw DEFLATE data.

### Worked example

The 16-byte input `00 00 01 00 02 00 03 00 04 00 05 00 06 00 07 00` (the
`uint16` values 0 to 7 in little-endian order) is encoded with the default
configuration (level 1) as the 36 bytes

```
1f 8b 08 00 8b 45 fb 6a 04 ff                            header (MTIME = 0x6AFB458B)
63 60 60 64 60 62 60 66 60 61 60 65 60 63 60 67 00 00    DEFLATE data
ed d9 15 c8                                              CRC-32 = 0xC815D9ED
10 00 00 00                                              ISIZE = 16
```

## Example

Array metadata written by zarr-python for
`compressors=[zarr.codecs.numcodecs.GZip()]`:

```json
{
  "data_type": "uint16",
  "codecs": [
    { "name": "bytes", "configuration": { "endian": "little" } },
    { "name": "numcodecs.gzip", "configuration": {} }
  ]
}
```

## Relationship to `gzip`

The registered [`gzip`](../gzip/README.md) codec, specified in the Zarr V3
core specification, uses the same encoded format: gzip data per RFC 1952.
zarr-python implements both codecs with the same numcodecs `GZip` encoder, so
apart from the `MTIME` field both produce the same bytes for the same level.

The configurations differ as follows:

| `numcodecs.gzip`                    | `gzip`                    |
| ----------------------------------- | ------------------------- |
| `level` optional, default `1`       | `level` required          |
| `level` from -1 to 9                | `level` from 0 to 9       |

An array's metadata can therefore be rewritten from `numcodecs.gzip` to `gzip`
without re-encoding any chunks:

```
{"name": "numcodecs.gzip", "configuration": C}
  -> {"name": "gzip", "configuration": {"level": L}}
     where L = C.level if present, else 1; and L = 6 if C.level is -1
```

Writers should prefer `gzip` for new data.

## Implementations

- zarr-python: [`zarr.codecs.numcodecs.GZip`](https://zarr.readthedocs.io/en/stable/api/zarr/codecs/numcodecs/)
- numcodecs: [`numcodecs.GZip`](https://numcodecs.readthedocs.io/en/stable/compression/gzip.html)
  (the codec implementation that zarr-python delegates to)

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)

[RFC1950]: https://www.rfc-editor.org/rfc/rfc1950
[RFC1951]: https://www.rfc-editor.org/rfc/rfc1951
[RFC1952]: https://www.rfc-editor.org/rfc/rfc1952
