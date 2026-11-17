# numcodecs.crc32 codec

Defines a `bytes -> bytes` codec that stores a 32-bit CRC-32 checksum of the
input next to the unchanged input, and verifies the checksum on decoding. This
is the CRC-32 used by gzip, zlib and PNG (named `CRC-32/ISO-HDLC` in the
[CRC catalogue]).

## Background

`numcodecs.crc32` is the name that zarr-python writes for the numcodecs
`CRC32` codec. It is registered here so that existing data written with this
name has a specification. No registered codec without the `numcodecs.` prefix
computes this checksum; for new data that only needs a checksum,
[`crc32c`](../crc32c/README.md) is the more widely supported choice.

## Codec name

The value of the `name` member in the codec object must be `"numcodecs.crc32"`.

## Configuration parameters

| Field      | Type   | Required | Default   | Notes                                         |
| ---------- | ------ | -------- | --------- | --------------------------------------------- |
| `location` | string | no       | `"start"` | `"start"` or `"end"`: where the checksum goes |

zarr-python writes only the parameters that the user passed, so
`configuration` is often `{}`. Readers must accept a configuration with
omitted optional fields and apply the defaults above. Any other field makes the
configuration invalid.

Readers must accept and ignore an `id` field equal to `"crc32"`, which
numcodecs 0.14.0–0.15.0 wrote; writers must not write it.

Writers must include the `configuration` member, even when it is `{}`:
zarr-python rejects `numcodecs.*` codec metadata without it. Readers may treat
a missing `configuration` as `{}`.

### `location`

`"start"` places the 4-byte checksum before the data; `"end"` places it after
the data. The default is `"start"`. Note that the default differs from
[`numcodecs.crc32c`](../numcodecs.crc32c/README.md), whose default is `"end"`.

## Format

This is a `bytes -> bytes` codec. The encoded output is always exactly 4 bytes
longer than the input.

### Checksum algorithm

The checksum is the 32-bit cyclic redundancy check defined in
[RFC 1952] section 8 (identical to ISO 3309 / ITU-T V.42 CRC-32), with these
parameters:

| Parameter                         | Value                                        |
| --------------------------------- | -------------------------------------------- |
| width                             | 32 bits                                      |
| polynomial                        | `0x04C11DB7` (bit-reflected: `0xEDB88320`)   |
| initial register value            | `0xFFFFFFFF`                                 |
| input reflected                   | yes (bits of each byte are processed LSB first) |
| output reflected                  | yes                                          |
| final XOR                         | `0xFFFFFFFF`                                 |
| checksum of ASCII `"123456789"`   | `0xCBF43926`                                 |
| checksum of the empty byte string | `0x00000000`                                 |

Equivalently, in terms of the bit-reflected polynomial:

```
crc = 0xFFFFFFFF
for each byte b of the input:
    crc = crc XOR b
    repeat 8 times:
        if crc AND 1: crc = (crc >> 1) XOR 0xEDB88320
        else:         crc = crc >> 1
checksum = crc XOR 0xFFFFFFFF
```

### Encoding

Given an input byte sequence `D` of length `n` (`n >= 0`):

1. Compute `C`, the CRC-32 of all `n` bytes of `D`.
2. Serialize `C` as a 4-byte unsigned integer in little-endian byte order
   (least significant byte first). Call this `S`.
3. If `location` is `"start"`, the output is `S` followed by `D`. If `location`
   is `"end"`, the output is `D` followed by `S`.

### Decoding

1. If the input is shorter than 4 bytes, decoding must fail with an error.
2. If `location` is `"start"`, `S` is the first 4 bytes and `D` is the rest; if
   `location` is `"end"`, `S` is the last 4 bytes and `D` is everything before.
3. Compute the CRC-32 of `D` and compare it to `S` read as a little-endian
   32-bit unsigned integer. If they differ, decoding must fail with an error and
   must not return `D`. (numcodecs raises `RuntimeError`.)
4. Otherwise the output is `D`.

### Worked examples

Input: the ASCII bytes `31 32 33 34 35 36 37 38 39` (`"123456789"`). The CRC-32
is `0xCBF43926`, serialized as `26 39 f4 cb`.

- `location` `"start"` (default): `26 39 f4 cb 31 32 33 34 35 36 37 38 39`
- `location` `"end"`: `31 32 33 34 35 36 37 38 39 26 39 f4 cb`

A `uint16` chunk `[0, 1, 2, 3]` after the `bytes` codec (little-endian) is
`00 00 01 00 02 00 03 00`. Its CRC-32 is `0x2F5A9784`, so with the default
configuration the stored chunk is `84 97 5a 2f 00 00 01 00 02 00 03 00`.

## Example

Array metadata `codecs` written by zarr-python for
`compressors=[zarr.codecs.numcodecs.CRC32(location="end")]`:

```json
{
  "codecs": [
    { "name": "bytes", "configuration": { "endian": "little" } },
    { "name": "numcodecs.crc32", "configuration": { "location": "end" } }
  ]
}
```

With `CRC32()` the second entry is
`{ "name": "numcodecs.crc32", "configuration": {} }`.

## Implementations

- [zarr-python](https://github.com/zarr-developers/zarr-python):
  `zarr.codecs.numcodecs.CRC32`
- [numcodecs](https://github.com/zarr-developers/numcodecs): `numcodecs.CRC32`
  (computes the checksum with Python's `zlib.crc32`)

## References

- [RFC 1952] P. Deutsch. GZIP file format specification version 4.3. May 1996.
  Section 8 defines this CRC and gives sample code.
- [CRC catalogue]: Greg Cook, Catalogue of parametrised CRC algorithms,
  entry `CRC-32/ISO-HDLC`.

[RFC 1952]: https://www.rfc-editor.org/rfc/rfc1952
[CRC catalogue]: https://reveng.sourceforge.io/crc-catalogue/17plus.htm

## Change log

No changes yet.

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)
