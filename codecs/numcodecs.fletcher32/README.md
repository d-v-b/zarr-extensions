# numcodecs.fletcher32 codec

Defines a `bytes -> bytes` codec that appends a 32-bit Fletcher checksum of the
input to the unchanged input, and verifies the checksum on decoding. The
checksum is the variant used by HDF5 (`H5_checksum_fletcher32`). Other
published Fletcher-32 variants differ from it in the byte order of the 16-bit
words and in how a sum that is a multiple of 65535 is represented, so
implementations must follow the definition below exactly.

## Background

`numcodecs.fletcher32` is the name that zarr-python writes for the numcodecs
`Fletcher32` codec. It is registered here so that existing data written with
this name has a specification. No registered codec without the `numcodecs.`
prefix computes this checksum; for new data that only needs a checksum,
[`crc32c`](../crc32c/README.md) is the more widely supported choice.

## Codec name

The value of the `name` member in the codec object must be
`"numcodecs.fletcher32"`.

## Configuration parameters

This codec has no configuration parameters. The `configuration` member must be
the empty object `{}`; any field in it other than `id` (see below) makes the
configuration invalid.

Readers must accept and ignore an `id` field equal to `"fletcher32"`, which
numcodecs 0.14.0–0.15.0 wrote; writers must not write it.

Writers must include the `configuration` member: zarr-python rejects
`numcodecs.*` codec metadata without it. Readers may treat a missing
`configuration` as `{}`.

## Format

This is a `bytes -> bytes` codec. The encoded output is always exactly 4 bytes
longer than the input, and the checksum is always placed at the end.

### Checksum algorithm

Given an input byte sequence `D` of length `n`:

1. If `n` is odd, append one zero byte to `D`. Let `m = ceil(n / 2)`.
2. Split the result into `m` 16-bit unsigned words, each read in
   **big-endian** order: `w[i] = 256 * D[2i] + D[2i + 1]` for
   `i = 0 .. m - 1`. (So a trailing odd byte `x` becomes the word `x * 256`.)
3. Using unbounded integers, compute

   ```
   S1 = w[0] + w[1] + ... + w[m-1]
   S2 = m * w[0] + (m - 1) * w[1] + ... + 1 * w[m-1]
   ```

   `S2` is the sum of the successive values taken by the running sum `S1`
   (that is, after adding each word to `S1`, add `S1` to `S2`).
4. Reduce each sum to 16 bits with

   ```
   R(x) = 0                        if x == 0
   R(x) = 1 + ((x - 1) mod 65535)  if x > 0
   ```

   `R(x)` is congruent to `x` modulo 65535 but represents a non-zero multiple
   of 65535 as `0xFFFF`, never as `0`. `R(x)` is `0` only when every word is
   zero.
5. The checksum is `C = R(S2) * 65536 + R(S1)`.

For the empty input (`n = 0`) the checksum is `0`.

The following incremental procedure (the one used by HDF5 and numcodecs, with
32-bit unsigned accumulators and end-around-carry folding) produces the same
value and is given for information. HDF5 processes blocks of at most 360 words
so that the 32-bit accumulators cannot overflow.

```
sum1 = 0; sum2 = 0; k = 0
words = floor(n / 2)
while words > 0:
    t = min(words, 360); words = words - t
    repeat t times:
        sum1 = sum1 + (D[k] << 8 | D[k+1]); k = k + 2
        sum2 = sum2 + sum1
    sum1 = (sum1 & 0xFFFF) + (sum1 >> 16)
    sum2 = (sum2 & 0xFFFF) + (sum2 >> 16)
if n is odd:
    sum1 = sum1 + (D[n-1] << 8)
    sum2 = sum2 + sum1
    sum1 = (sum1 & 0xFFFF) + (sum1 >> 16)
    sum2 = (sum2 & 0xFFFF) + (sum2 >> 16)
sum1 = (sum1 & 0xFFFF) + (sum1 >> 16)
sum2 = (sum2 & 0xFFFF) + (sum2 >> 16)
C = (sum2 << 16) | sum1
```

### Encoding

1. Compute the checksum `C` of all `n` bytes of the input `D`.
2. The output is `D` followed by `C` as a 4-byte unsigned integer in
   **little-endian** byte order. The 4 stored bytes are therefore: low byte of
   `R(S1)`, high byte of `R(S1)`, low byte of `R(S2)`, high byte of `R(S2)`.

### Decoding

1. If the input is shorter than 4 bytes, decoding must fail with an error.
2. Let `D` be all but the last 4 bytes and `S` the last 4 bytes.
3. Compute the checksum of `D` and compare it to `S` read as a little-endian
   32-bit unsigned integer. If they differ, decoding must fail with an error and
   must not return `D`. (numcodecs raises `RuntimeError`.)
4. Otherwise the output is `D`.

### Worked examples

| Input `D` (hex)                | Words                              | `R(S1)`  | `R(S2)`  | Encoded output (hex)                             |
| ------------------------------ | ---------------------------------- | -------- | -------- | ------------------------------------------------ |
| `31 32 33 34 35 36 37 38 39`   | `3132 3334 3536 3738 3900`         | `0x09D5` | `0x09DF` | `31 32 33 34 35 36 37 38 39 d5 09 df 09`         |
| `01 02 03`                     | `0102 0300`                        | `0x0402` | `0x0504` | `01 02 03 02 04 04 05`                           |
| `01`                           | `0100`                             | `0x0100` | `0x0100` | `01 00 01 00 01`                                 |
| `ff ff`                        | `FFFF`                             | `0xFFFF` | `0xFFFF` | `ff ff ff ff ff ff`                              |
| `00 00 01 00 02 00 03 00`      | `0000 0100 0200 0300`              | `0x0600` | `0x0A00` | `00 00 01 00 02 00 03 00 00 06 00 0a`            |

In the first row, `S1 = 0x109D4 = 68052` and `S2 = 0x309DC = 199132`, giving
`R(S1) = 0x09D5` and `R(S2) = 0x09DF`. The `ff ff` row shows that a sum of
65535 is stored as `0xFFFF`, not `0`. The last row is a `uint16` chunk
`[0, 1, 2, 3]` after the `bytes` codec (little-endian), as zarr-python stores
it.

## Example

Array metadata `codecs` written by zarr-python for
`compressors=[zarr.codecs.numcodecs.Fletcher32()]`:

```json
{
  "codecs": [
    { "name": "bytes", "configuration": { "endian": "little" } },
    { "name": "numcodecs.fletcher32", "configuration": {} }
  ]
}
```

## Implementations

- [zarr-python](https://github.com/zarr-developers/zarr-python):
  `zarr.codecs.numcodecs.Fletcher32`
- [numcodecs](https://github.com/zarr-developers/numcodecs):
  `numcodecs.Fletcher32`. numcodecs 0.17.1 cannot encode an empty input, nor
  decode a 4-byte input (it raises `IndexError`); zarr-python never passes an
  empty chunk to this codec.
- [zarrs](https://github.com/zarrs/zarrs): experimental, behind the
  `fletcher32` feature flag. zarrs 0.19 to 0.23 wrote non-conformant checksums
  for odd-length input (the last byte was omitted); later versions accept those
  checksums when reading.

## References

- [H5checksum.c]: HDF5 source, function `H5_checksum_fletcher32`, the
  reference from which the numcodecs implementation was converted.

[H5checksum.c]: https://github.com/HDFGroup/hdf5/blob/577c192518598c7e2945683655feffcdbdf5a91b/src/H5checksum.c#L99-L141

## Change log

No changes yet.

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)
