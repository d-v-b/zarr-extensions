# numcodecs.crc32c codec

Defines a `bytes -> bytes` codec that stores a 32-bit CRC-32C (Castagnoli)
checksum of the input next to the unchanged input, and verifies the checksum
on decoding.

## Background

`numcodecs.crc32c` is the name that zarr-python writes for the numcodecs
`CRC32C` codec. It is registered here so that existing data written with this
name has a specification. The registered [`crc32c`](../crc32c/README.md) codec
computes the same checksum, and with the default `location` the encoded bytes
are identical (see [Relationship to `crc32c`](#relationship-to-crc32c)). New
data should use `crc32c`.

## Codec name

The value of the `name` member in the codec object must be
`"numcodecs.crc32c"`.

## Configuration parameters

| Field      | Type   | Required | Default | Notes                                         |
| ---------- | ------ | -------- | ------- | --------------------------------------------- |
| `location` | string | no       | `"end"` | `"start"` or `"end"`: where the checksum goes |

zarr-python writes only the parameters that the user passed, so
`configuration` is often `{}`. Readers must accept a configuration with
omitted optional fields and apply the defaults above. Any other field makes the
configuration invalid.

Readers must accept and ignore an `id` field equal to `"crc32c"`, which
numcodecs 0.14.0–0.15.0 wrote; writers must not write it.

Writers must include the `configuration` member, even when it is `{}`:
zarr-python rejects `numcodecs.*` codec metadata without it. Readers may treat
a missing `configuration` as `{}`.

### `location`

`"start"` places the 4-byte checksum before the data; `"end"` places it after
the data. The default is `"end"`. Note that the default differs from
[`numcodecs.crc32`](../numcodecs.crc32/README.md) and
[`numcodecs.adler32`](../numcodecs.adler32/README.md), whose default is
`"start"`.

## Format

This is a `bytes -> bytes` codec. The encoded output is always exactly 4 bytes
longer than the input.

### Checksum algorithm

The checksum is CRC-32C as defined in [RFC 3720] section 12.1 and appendix B.4
(named `CRC-32/ISCSI` in the [CRC catalogue]), with these parameters:

| Parameter                         | Value                                           |
| --------------------------------- | ----------------------------------------------- |
| width                             | 32 bits                                         |
| polynomial                        | `0x1EDC6F41` (bit-reflected: `0x82F63B78`)      |
| initial register value            | `0xFFFFFFFF`                                    |
| input reflected                   | yes (bits of each byte are processed LSB first) |
| output reflected                  | yes                                             |
| final XOR                         | `0xFFFFFFFF`                                    |
| checksum of ASCII `"123456789"`   | `0xE3069283`                                    |
| checksum of the empty byte string | `0x00000000`                                    |

Equivalently, in terms of the bit-reflected polynomial:

```
crc = 0xFFFFFFFF
for each byte b of the input:
    crc = crc XOR b
    repeat 8 times:
        if crc AND 1: crc = (crc >> 1) XOR 0x82F63B78
        else:         crc = crc >> 1
checksum = crc XOR 0xFFFFFFFF
```

### Encoding

Given an input byte sequence `D` of length `n` (`n >= 0`):

1. Compute `C`, the CRC-32C of all `n` bytes of `D`.
2. Serialize `C` as a 4-byte unsigned integer in little-endian byte order
   (least significant byte first). Call this `S`.
3. If `location` is `"end"`, the output is `D` followed by `S`. If `location`
   is `"start"`, the output is `S` followed by `D`.

### Decoding

1. If the input is shorter than 4 bytes, decoding must fail with an error.
2. If `location` is `"end"`, `S` is the last 4 bytes and `D` is everything
   before; if `location` is `"start"`, `S` is the first 4 bytes and `D` is the
   rest.
3. Compute the CRC-32C of `D` and compare it to `S` read as a little-endian
   32-bit unsigned integer. If they differ, decoding must fail with an error and
   must not return `D`. (numcodecs raises `RuntimeError`.)
4. Otherwise the output is `D`.

### Worked examples

Input: the ASCII bytes `31 32 33 34 35 36 37 38 39` (`"123456789"`). The
CRC-32C is `0xE3069283`, serialized as `83 92 06 e3`.

- `location` `"end"` (default): `31 32 33 34 35 36 37 38 39 83 92 06 e3`
- `location` `"start"`: `83 92 06 e3 31 32 33 34 35 36 37 38 39`

A `uint16` chunk `[0, 1, 2, 3]` after the `bytes` codec (little-endian) is
`00 00 01 00 02 00 03 00`. Its CRC-32C is `0xE898528B`, so with the default
configuration the stored chunk is `00 00 01 00 02 00 03 00 8b 52 98 e8`.

## Example

Array metadata `codecs` written by zarr-python for
`compressors=[zarr.codecs.numcodecs.CRC32C()]`:

```json
{
  "codecs": [
    { "name": "bytes", "configuration": { "endian": "little" } },
    { "name": "numcodecs.crc32c", "configuration": {} }
  ]
}
```

With `CRC32C(location="start")` the second entry is
`{ "name": "numcodecs.crc32c", "configuration": { "location": "start" } }`.

## Relationship to `crc32c`

The registered [`crc32c`](../crc32c/README.md) codec computes the same RFC 3720
CRC-32C and appends it to the data as a little-endian 32-bit unsigned integer.
It has no configuration parameters.

- With `location` `"end"` (explicit or omitted), `numcodecs.crc32c` produces
  byte-for-byte the same encoded chunks as `crc32c`. Metadata can be rewritten
  from `numcodecs.crc32c` to `{"name": "crc32c"}` (and back) without
  re-encoding any chunk.
- With `location` `"start"` the checksum precedes the data, so the stored
  chunks are not valid `crc32c` chunks. Converting such data to `crc32c`
  requires rewriting every chunk (moving the first 4 bytes to the end); the
  checksum value itself does not change.

## Implementations

- [zarr-python](https://github.com/zarr-developers/zarr-python):
  `zarr.codecs.numcodecs.CRC32C`
- [numcodecs](https://github.com/zarr-developers/numcodecs): `numcodecs.CRC32C`
  (only defined when the `google-crc32c` or `crc32c` Python package is
  installed)

## References

- [RFC 3720] J. Satran et al. Internet Small Computer Systems Interface
  (iSCSI). April 2004. Section 12.1 and appendix B.4 define CRC-32C.
- [CRC catalogue]: Greg Cook, Catalogue of parametrised CRC algorithms,
  entry `CRC-32/ISCSI`.

[RFC 3720]: https://www.rfc-editor.org/rfc/rfc3720
[CRC catalogue]: https://reveng.sourceforge.io/crc-catalogue/17plus.htm

## Change log

No changes yet.

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)
