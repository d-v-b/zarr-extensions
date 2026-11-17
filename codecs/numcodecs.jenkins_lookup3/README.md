# numcodecs.jenkins_lookup3 codec

Defines a `bytes -> bytes` codec that appends a 32-bit hash of the input,
computed with Bob Jenkins' lookup3 `hashlittle` function, to the unchanged
input, and verifies the hash on decoding. This is the same function HDF5 uses
for its metadata checksums (`H5_checksum_lookup3`).

## Background

`numcodecs.jenkins_lookup3` is the name that zarr-python writes for the
numcodecs `JenkinsLookup3` codec. It is registered here so that existing data
written with this name has a specification. No registered codec without the
`numcodecs.` prefix computes this hash; for new data that only needs a
checksum, [`crc32c`](../crc32c/README.md) is the more widely supported choice.

## Codec name

The value of the `name` member in the codec object must be
`"numcodecs.jenkins_lookup3"`.

## Configuration parameters

| Field     | Type    | Required | Default | Notes                                       |
| --------- | ------- | -------- | ------- | ------------------------------------------- |
| `initval` | integer | no       | `0`     | hash seed, `0` to `4294967295` (`2^32 - 1`) |
| `prefix`  | null    | no       | `null`  | must be `null` if present; see below        |

zarr-python writes only the parameters that the user passed, so
`configuration` is often `{}`. numcodecs' own `get_config()` always emits both
fields (`{"initval": 0, "prefix": null}` by default), so configurations copied
from it carry both. Readers must accept a configuration with omitted optional
fields and apply the defaults above. Any other field makes the configuration
invalid.

Readers must accept and ignore an `id` field equal to `"jenkins_lookup3"`,
which numcodecs 0.14.0–0.15.0 wrote; writers must not write it.

Writers must include the `configuration` member, even when it is `{}`:
zarr-python rejects `numcodecs.*` codec metadata without it. Readers may treat
a missing `configuration` as `{}`.

### `initval`

The initial value (seed) passed to `hashlittle`. Different seeds give
unrelated hashes, so a reader must use the same `initval` as the writer.

### `prefix`

In numcodecs, `prefix` is an optional byte string that is hashed in front of
the data (the hash is computed over `prefix` followed by the data); the prefix
itself is not stored in the encoded output. numcodecs and zarr-python have no
JSON representation for a non-null prefix: zarr-python fails to write such
metadata, and numcodecs cannot read a prefix from a JSON list or string. This
specification therefore only allows `prefix` to be absent or `null`, meaning
an empty prefix. Readers must reject any other value.

## Format

This is a `bytes -> bytes` codec. The encoded output is always exactly 4 bytes
longer than the input, and the hash is always placed at the end.

### Hash algorithm

The hash is `hashlittle(key, length, initval)` from Bob Jenkins' public-domain
[lookup3.c] (2006). The algorithm below is equivalent and is normative. All
arithmetic is on unsigned 32-bit integers, modulo `2^32`; `rot(x, k)` is a left
rotation of the 32-bit value `x` by `k` bits, `(x << k) | (x >> (32 - k))`.

```
mix(a, b, c):
    a -= c;  a ^= rot(c, 4);   c += b
    b -= a;  b ^= rot(a, 6);   a += c
    c -= b;  c ^= rot(b, 8);   b += a
    a -= c;  a ^= rot(c, 16);  c += b
    b -= a;  b ^= rot(a, 19);  a += c
    c -= b;  c ^= rot(b, 4);   b += a

final(a, b, c):
    c ^= b;  c -= rot(b, 14)
    a ^= c;  a -= rot(c, 11)
    b ^= a;  b -= rot(a, 25)
    c ^= b;  c -= rot(b, 16)
    a ^= c;  a -= rot(c, 4)
    b ^= a;  b -= rot(a, 14)
    c ^= b;  c -= rot(b, 24)

hash(K, initval):            # K is a byte sequence of length L
    a = b = c = 0xDEADBEEF + L + initval    # modulo 2^32
    if L == 0:
        return c                            # no mixing for empty input
    while more than 12 bytes of K remain:
        a += next 4 bytes of K, read as a little-endian uint32
        b += next 4 bytes of K, read as a little-endian uint32
        c += next 4 bytes of K, read as a little-endian uint32
        mix(a, b, c)
    pad the remaining 1 to 12 bytes with zero bytes to 12 bytes
    a += bytes 0..3 of the padded block, read as a little-endian uint32
    b += bytes 4..7 of the padded block, read as a little-endian uint32
    c += bytes 8..11 of the padded block, read as a little-endian uint32
    final(a, b, c)
    return c
```

The words are assembled from bytes in little-endian order regardless of the
host's byte order.

Test vectors (all confirmed with numcodecs; the last two are the `hashlittle`
vectors listed in lookup3.c):

| Key                                      | `initval`    | Hash         |
| ---------------------------------------- | ------------ | ------------ |
| empty                                    | `0`          | `0xDEADBEEF` |
| empty                                    | `0xDEADBEEF` | `0xBD5B7DDE` |
| ASCII `"Four score and seven years ago"` | `0`          | `0x17770551` |
| ASCII `"Four score and seven years ago"` | `1`          | `0xCD628161` |

### Encoding

Given an input byte sequence `D` of length `n` (`n >= 0`):

1. Compute `H = hash(D, initval)`.
2. The output is `D` followed by `H` as a 4-byte unsigned integer in
   little-endian byte order (least significant byte first).

### Decoding

1. If the input is shorter than 4 bytes, decoding must fail with an error.
2. Let `D` be all but the last 4 bytes and `S` the last 4 bytes.
3. Compute `hash(D, initval)` and compare it to `S` read as a little-endian
   32-bit unsigned integer. If they differ, decoding must fail with an error and
   must not return `D`. (numcodecs raises `RuntimeError`.)
4. Otherwise the output is `D`.

### Worked examples

| Input `D` (hex)                  | `initval` | Hash         | Encoded output (hex)                               |
| -------------------------------- | --------- | ------------ | -------------------------------------------------- |
| empty                            | `0`       | `0xDEADBEEF` | `ef be ad de`                                      |
| `31 32 33 34 35 36 37 38 39`     | `0`       | `0x845D9A96` | `31 32 33 34 35 36 37 38 39 96 9a 5d 84`           |
| `31 32 33 34 35 36 37 38 39`     | `1`       | `0x476CE774` | `31 32 33 34 35 36 37 38 39 74 e7 6c 47`           |
| `00 00 01 00 02 00 03 00`        | `0`       | `0xFD5EB3BF` | `00 00 01 00 02 00 03 00 bf b3 5e fd`              |
| `00 00 01 00 02 00 03 00`        | `5`       | `0xD2593CAA` | `00 00 01 00 02 00 03 00 aa 3c 59 d2`              |

The last two rows are a `uint16` chunk `[0, 1, 2, 3]` after the `bytes` codec
(little-endian), as zarr-python stores it.

## Example

Array metadata `codecs` written by zarr-python for
`compressors=[zarr.codecs.numcodecs.JenkinsLookup3(initval=5)]`:

```json
{
  "codecs": [
    { "name": "bytes", "configuration": { "endian": "little" } },
    { "name": "numcodecs.jenkins_lookup3", "configuration": { "initval": 5 } }
  ]
}
```

With `JenkinsLookup3()` the second entry is
`{ "name": "numcodecs.jenkins_lookup3", "configuration": {} }`.

## Implementations

- [zarr-python](https://github.com/zarr-developers/zarr-python):
  `zarr.codecs.numcodecs.JenkinsLookup3`
- [numcodecs](https://github.com/zarr-developers/numcodecs):
  `numcodecs.JenkinsLookup3`

## References

- [lookup3.c]: Bob Jenkins, May 2006, public domain. The function
  `hashlittle()`.
- [H5checksum.c]: HDF5 source, function `H5_checksum_lookup3`, the code from
  which the numcodecs implementation was converted.

[lookup3.c]: https://www.burtleburtle.net/bob/c/lookup3.c
[H5checksum.c]: https://github.com/HDFGroup/hdf5/blob/577c192518598c7e2945683655feffcdbdf5a91b/src/H5checksum.c#L378-L472

## Change log

No changes yet.

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)
