# numcodecs.bz2 codec

Defines a `bytes -> bytes` codec that compresses its input with bzip2. The
encoded representation is a bzip2 stream (the contents of a `.bz2` file) with no
additional header or trailer.

## Background

`numcodecs.bz2` is the name that zarr-python writes into Zarr V3 array metadata
for the `BZ2` codec of the [numcodecs](https://github.com/zarr-developers/numcodecs)
library (`zarr.codecs.numcodecs.BZ2`, previously `numcodecs.zarr3.BZ2`). It is
registered here so that data already written under this name has a
specification that does not depend on the numcodecs source code. No registered
codec without the `numcodecs.` prefix uses bzip2.

## Codec name

The value of the `name` member in the codec object must be `"numcodecs.bz2"`.

## Configuration parameters

Writers must include the `configuration` member, as a JSON object that may be
empty. Readers may treat a missing `configuration` as `{}`. zarr-python
writes only the parameters that the user passed explicitly, so readers must
accept a configuration with any optional field omitted and apply the default
listed below. Readers must accept and ignore an `id` field equal to
`"bz2"`, which numcodecs 0.14.0–0.15.0 wrote. Writers must not write it. No
other fields are permitted.

| field   | type    | required | default | notes                          |
| ------- | ------- | -------- | ------- | ------------------------------ |
| `level` | integer | no       | `1`     | 1 to 9. Affects encoding only. |

### `level`

The bzip2 block size, in units of 100 000 bytes. The encoder splits its input
into blocks of at most `level × 100 000` bytes before the Burrows–Wheeler
transform. Larger values usually compress better and use more memory. The
encoder records the level in the stream header, so decoders do not use this
parameter.

## Format

This is a `bytes -> bytes` codec.

bzip2 has no formal standard. The format is defined by its reference
implementation, [libbzip2](https://sourceware.org/bzip2/) (version 1.0.x;
see also its [manual](https://sourceware.org/bzip2/manual/manual.html)).
Joe Tsai has written an independent, unofficial
[description of the bitstream](https://github.com/dsnet/compress/blob/master/doc/bzip2-format.pdf).
A bzip2 stream consists of:

1. the 4-byte header `42 5a 68` (`"BZh"`) followed by one ASCII digit `'1'` to
   `'9'` (`0x31` to `0x39`) giving the block size level;
2. zero or more compressed blocks, each beginning with the 48-bit block magic
   `0x314159265359` and a 32-bit block CRC;
3. the 48-bit end-of-stream magic `0x177245385090`, a 32-bit combined stream
   CRC, and zero to seven padding bits up to the next byte boundary.

The block and end-of-stream fields are packed MSB-first and are not
byte-aligned after the first block.

### Encoding

The encoder compresses the whole input into **exactly one** bzip2 stream with
block size level `level` and writes nothing before or after it. For an empty
input, the stream has no blocks.

### Decoding

Decoders must accept one bzip2 stream with any block size level and decode it
to its content. They must verify the block CRCs and the combined stream CRC,
and fail on a mismatch. numcodecs (through Python's `bz2` module) also accepts
several concatenated streams and returns their concatenated contents. It
ignores trailing bytes after the first complete stream if those bytes do not
begin a valid stream. Writers must not produce either case.

### Worked example

The 16-byte input `00 00 01 00 02 00 03 00 04 00 05 00 06 00 07 00` (the
`uint16` values 0 to 7 in little-endian order) is encoded with the default
configuration (level 1) as 45 bytes, which begin with

```
42 5a 68 31          "BZh1": block size level 1
31 41 59 26 53 59    block magic
8d 56 64 45          block CRC
...
```

## Example

Array metadata written by zarr-python for
`compressors=[zarr.codecs.numcodecs.BZ2(level=9)]`:

```json
{
  "data_type": "uint16",
  "codecs": [
    { "name": "bytes", "configuration": { "endian": "little" } },
    { "name": "numcodecs.bz2", "configuration": { "level": 9 } }
  ]
}
```

## Implementations

- zarr-python: [`zarr.codecs.numcodecs.BZ2`](https://zarr.readthedocs.io/en/stable/api/zarr/codecs/numcodecs/)
- numcodecs: [`numcodecs.BZ2`](https://numcodecs.readthedocs.io/en/stable/compression/bz2.html)
  (the codec implementation that zarr-python delegates to)
- zarrs: [`bz2`](https://docs.rs/zarrs/latest/zarrs/array/codec/bytes_to_bytes/bz2/index.html)
  (reads and writes `numcodecs.bz2`)

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)
