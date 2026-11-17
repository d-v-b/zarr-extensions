# numcodecs.adler32 codec

Defines a `bytes -> bytes` codec that stores a 32-bit Adler-32 checksum of the
input next to the unchanged input, and verifies the checksum on decoding.

## Background

`numcodecs.adler32` is the name that zarr-python writes for the numcodecs
`Adler32` codec. It is registered here so that existing data written with this
name has a specification. No registered codec without the `numcodecs.` prefix
computes this checksum; for new data that only needs a checksum,
[`crc32c`](../crc32c/README.md) is the more widely supported choice.

## Codec name

The value of the `name` member in the codec object must be
`"numcodecs.adler32"`.

## Configuration parameters

| Field      | Type   | Required | Default   | Notes                                         |
| ---------- | ------ | -------- | --------- | --------------------------------------------- |
| `location` | string | no       | `"start"` | `"start"` or `"end"`: where the checksum goes |

zarr-python writes only the parameters that the user passed, so
`configuration` is often `{}`. Readers must accept a configuration with
omitted optional fields and apply the defaults above. Any other field makes the
configuration invalid.

Readers must accept and ignore an `id` field equal to `"adler32"`, which
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

The checksum is Adler-32 as defined in [RFC 1950] sections 8.2 and 9 (the
checksum used in the zlib format):

```
s1 = 1
s2 = 0
for each byte b of the input:
    s1 = (s1 + b) mod 65521
    s2 = (s2 + s1) mod 65521
checksum = s2 * 65536 + s1
```

The checksum of the empty byte string is `0x00000001`.

### Encoding

Given an input byte sequence `D` of length `n` (`n >= 0`):

1. Compute `C`, the Adler-32 of all `n` bytes of `D`.
2. Serialize `C` as a 4-byte unsigned integer in **little-endian** byte order
   (least significant byte first). Call this `S`. This is the opposite of the
   zlib format, which stores Adler-32 most significant byte first.
3. If `location` is `"start"`, the output is `S` followed by `D`. If `location`
   is `"end"`, the output is `D` followed by `S`.

### Decoding

1. If the input is shorter than 4 bytes, decoding must fail with an error.
2. If `location` is `"start"`, `S` is the first 4 bytes and `D` is the rest; if
   `location` is `"end"`, `S` is the last 4 bytes and `D` is everything before.
3. Compute the Adler-32 of `D` and compare it to `S` read as a little-endian
   32-bit unsigned integer. If they differ, decoding must fail with an error and
   must not return `D`. (numcodecs raises `RuntimeError`.)
4. Otherwise the output is `D`.

### Worked examples

Input: the ASCII bytes `31 32 33 34 35 36 37 38 39` (`"123456789"`). The
Adler-32 is `0x091E01DE`, serialized as `de 01 1e 09`.

- `location` `"start"` (default): `de 01 1e 09 31 32 33 34 35 36 37 38 39`
- `location` `"end"`: `31 32 33 34 35 36 37 38 39 de 01 1e 09`

An empty input encodes as `01 00 00 00`.

A `uint16` chunk `[0, 1, 2, 3]` after the `bytes` codec (little-endian) is
`00 00 01 00 02 00 03 00`. Its Adler-32 is `0x001C0007`, so with the default
configuration the stored chunk is `07 00 1c 00 00 00 01 00 02 00 03 00`.

## Example

Array metadata `codecs` written by zarr-python for
`compressors=[zarr.codecs.numcodecs.Adler32()]`:

```json
{
  "codecs": [
    { "name": "bytes", "configuration": { "endian": "little" } },
    { "name": "numcodecs.adler32", "configuration": {} }
  ]
}
```

With `Adler32(location="end")` the second entry is
`{ "name": "numcodecs.adler32", "configuration": { "location": "end" } }`.

## Implementations

- [zarr-python](https://github.com/zarr-developers/zarr-python):
  `zarr.codecs.numcodecs.Adler32`
- [numcodecs](https://github.com/zarr-developers/numcodecs): `numcodecs.Adler32`
  (computes the checksum with Python's `zlib.adler32`)
- [zarrs](https://github.com/zarrs/zarrs): experimental, behind the `adler32`
  feature flag

## References

- [RFC 1950] P. Deutsch, J-L. Gailly. ZLIB Compressed Data Format
  Specification version 3.3. May 1996. Section 8.2 describes Adler-32 and
  section 9 gives sample code.

[RFC 1950]: https://www.rfc-editor.org/rfc/rfc1950

## Change log

No changes yet.

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)
