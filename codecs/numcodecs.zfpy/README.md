# numcodecs.zfpy codec

Defines an `array -> bytes` codec that compresses a chunk with the [zfp]
algorithm. The encoded chunk is a zfp stream that starts with a zfp header
recording the chunk shape, scalar type and compression parameters, so a chunk
can be decoded without the codec configuration.

## Background

`numcodecs.zfpy` is the name that zarr-python writes for the numcodecs `ZFPY`
codec, which calls `zfpy.compress_numpy` from zfp's Python bindings. It is
registered here so that existing data written with this name has a
specification. The registered [`zfp`](../zfp/README.md) codec uses the same
algorithm without the header and supports more data types; new data should
prefer `zfp`. See [Relationship to `zfp`](#relationship-to-zfp).

## Codec name

The value of the `name` member in the codec object must be `"numcodecs.zfpy"`.

## Configuration parameters

| Field       | Type    | Required | Default | Notes                                                    |
| ----------- | ------- | -------- | ------- | -------------------------------------------------------- |
| `mode`      | integer | no       | `4`     | `2` fixed rate, `3` fixed precision, `4` fixed accuracy, `5` reversible |
| `tolerance` | number  | no       | `-1`    | used when `mode` is `4`; a negative value means unset    |
| `rate`      | number  | no       | `-1`    | used when `mode` is `2`; a negative value means unset    |
| `precision` | integer | no       | `-1`    | used when `mode` is `3`; a negative value means unset    |

zarr-python writes only the parameters that the user passed, so
`configuration` is often `{}`. Readers must accept a configuration with
omitted optional fields and apply the defaults above. Any other field makes the
configuration invalid, with one exception: `numcodecs.zarr3` in numcodecs
0.14.0 to 0.15.0 also wrote `"id": "zfpy"` inside `configuration`. Readers
must accept and ignore that member; writers must not write it.

Writers must include the `configuration` member, even when it is `{}`:
zarr-python rejects `numcodecs.*` codec metadata without it. Readers may treat
a missing `configuration` as `{}`.

None of the parameters are needed to decode a chunk: the zfp header in each
chunk carries the compression parameters. They only steer the encoder.

### `mode`, `tolerance`, `rate` and `precision`

`mode` uses the integer values of zfp's `zfp_mode` enumeration (exported by
`zfpy` as `zfpy.mode_fixed_rate`, `zfpy.mode_fixed_precision` and
`zfpy.mode_fixed_accuracy`). For each mode, only the matching parameter is
used; the other two are ignored. Writers should omit the parameters that do
not match `mode`, or set them to `-1`. The encoder sets zfp's four compression
parameters (`minbits`, `maxbits`, `maxprec`, `minexp`, described in the
[zfp compression modes] documentation) as follows, where `d` is the number of
dimensions of the zfp field (see [Format](#format)):

| `mode` | Condition         | zfp mode        | `minbits` | `maxbits` | `maxprec`                                     | `minexp`                                         |
| ------ | ----------------- | --------------- | --------- | --------- | --------------------------------------------- | ------------------------------------------------ |
| `4`    | `tolerance > 0`   | fixed accuracy  | 1         | 16658     | 64                                            | `floor(log2(tolerance))`                         |
| `4`    | `tolerance == 0`  | expert (zfp defaults) | 1         | 16658     | 64                                            | -1074                                            |
| `2`    | `rate >= 0`       | fixed rate      | `b`       | `b`       | 64                                            | -1074                                            |
| `3`    | `precision >= 0`  | fixed precision | 1         | 16658     | `64` if `precision` is 0, else `min(precision, 64)` | -1074                                      |
| `2`, `3`, `4` | matching parameter negative or omitted | reversible | 1 | 16658 | 64                                   | -1075                                            |
| `5`    | (none)            | reversible      | 1         | 16658     | 64                                            | -1075                                            |

Here `b = floor(4^d * rate + 0.5)`, the number of bits per block of `4^d`
values. zfpy computes `b` without the minimum that zfp's own
`zfp_stream_set_rate` applies to floating-point types (9 bits for `float32`,
12 for `float64`). Writers must not choose a rate for which `b` is 0.

So the default configuration (`{}`, or `{"mode": 4}` without `tolerance`)
gives lossless reversible compression, not fixed accuracy.

numcodecs 0.17.1 only encodes with `mode` `2`, `3` or `4`; with `5` (or any
other value) it raises an error when it encodes a chunk, but it decodes such
chunks. zarrs writes `5` for reversible mode.

## Supported data types

| Zarr data type | zfp scalar type   | zfp type code |
| -------------- | ----------------- | ------------- |
| `int32`        | `zfp_type_int32`  | 1             |
| `int64`        | `zfp_type_int64`  | 2             |
| `float32`      | `zfp_type_float`  | 3             |
| `float64`      | `zfp_type_double` | 4             |

Values are compressed as they are; there is no promotion of other types.
numcodecs 0.17.1 rejects every other data type, including `uint32`, `uint64`,
`float16` and the 8- and 16-bit integer types. zarr-python also encodes the
`numpy.datetime64` and `numpy.timedelta64` data types as `zfp_type_int64`,
holding the stored 64-bit integer values.

## Supported chunk shapes

A chunk with `d` dimensions, `1 <= d <= 4`, is a `d`-dimensional zfp field.
The Zarr chunk dimensions map to zfp field sizes in reverse order, as for the
[`zfp`](../zfp/README.md) codec, because zfp's `nx` varies fastest and Zarr
chunks are in C order:

- 1D: `[nx]`
- 2D: `[ny, nx]`
- 3D: `[nz, ny, nx]`
- 4D: `[nw, nz, ny, nx]`

zarr-python passes a zero-dimensional chunk to the codec as a one-element 1D
array, so it is encoded as a 1D field with `nx = 1`. Chunks with more than four
dimensions cannot be encoded.

The header limits the size of each dimension:

| `d` | Limit per dimension |
| --- | ------------------- |
| 1   | `2^32` (the header has 48 bits for `nx - 1`, but zfp 1.0.1 reads only the low 32) |
| 2   | `2^24`              |
| 3   | `2^16`              |
| 4   | `2^12`              |

Encoding a larger chunk fails.

## Format

This is an `array -> bytes` codec. The encoded chunk is a zfp header followed
directly by a zfp compressed stream, as written by zfp's
`zfp_write_header(stream, field, ZFP_HEADER_FULL)` and then `zfp_compress`
([zfp high-level API]).

### Bit order

The encoded chunk is a sequence of bits. Bit `k` of the sequence is bit
`k mod 8` (where bit 0 is the least significant) of byte `floor(k / 8)`. A
field of `n` bits with value `v` stored at bit offset `p` occupies bits `p` to
`p + n - 1`, with the least significant bit of `v` at bit `p`.

zfp reads and writes its bit stream in 64-bit words in the host's byte order.
The definition above matches what zfp writes on little-endian hosts, which is
what zarr-python runs on in practice. A zfp build on a big-endian host with
the default 64-bit word size writes byte-swapped words (see the zfp
[FAQ](https://zfp.readthedocs.io/en/release1.0.1/faq.html)). Such output does
not conform to this specification.

### Header

| Bit offset | Width | Content                                                                    |
| ---------- | ----- | -------------------------------------------------------------------------- |
| 0          | 8     | `0x7A` (`z`)                                                               |
| 8          | 8     | `0x66` (`f`)                                                               |
| 16         | 8     | `0x70` (`p`)                                                               |
| 24         | 8     | zfp codec version, `5` (zfp 0.5.4 through 1.0.1)                           |
| 32         | 52    | field metadata `META`                                                      |
| 84         | 12    | mode word `M`                                                              |
| 96         | 52    | only if `M == 0xFFF`: the rest of the 64-bit long mode word               |

`META` holds these fields, from its least significant bit:

| `META` bits | Content                                                                                   |
| ----------- | ----------------------------------------------------------------------------------------- |
| 0-1         | zfp type code minus 1 (`0` int32, `1` int64, `2` float32, `3` float64)                   |
| 2-3         | `d - 1`                                                                                   |
| 4-51 (d=1)  | `nx - 1` (48 bits)                                                                        |
| 4-51 (d=2)  | `nx - 1`, `ny - 1` (24 bits each)                                                         |
| 4-51 (d=3)  | `nx - 1`, `ny - 1`, `nz - 1` (16 bits each)                                               |
| 4-51 (d=4)  | `nx - 1`, `ny - 1`, `nz - 1`, `nw - 1` (12 bits each)                                     |

`M` is the short (12-bit) mode word when `M <= 4094`:

| `M`          | zfp mode        | `minbits` | `maxbits` | `maxprec`  | `minexp`     |
| ------------ | --------------- | --------- | --------- | ---------- | ------------ |
| 0-2047       | fixed rate      | `M + 1`   | `M + 1`   | 64         | -1074        |
| 2048-2175    | fixed precision | 1         | 16658     | `M - 2047` | -1074        |
| 2176         | reversible      | 1         | 16658     | 64         | -1075        |
| 2177-4094    | fixed accuracy  | 1         | 16658     | 64         | `M - 3251`   |

zfp writes the short form whenever the parameters fit one of these rows. If
they do not, `M` is `0xFFF` (4095) and the header has a 64-bit long mode word
`L` at bit offset 84, whose low 12 bits are `0xFFF`. Then
`minbits = ((L >> 12) & 0x7FFF) + 1`, `maxbits = ((L >> 27) & 0x7FFF) + 1`,
`maxprec = ((L >> 42) & 0x7F) + 1` and `minexp = ((L >> 49) & 0x7FFF) - 16495`.
numcodecs writes the long form when `tolerance` is `0` or `precision` is `0`
or at least `64` (these give zfp's default parameters `1, 16658, 64, -1074`,
which match no short-form row), when `tolerance` is at least `2^844`, and when
the fixed-rate `b` exceeds 2048.

The header is 96 bits (12 bytes) with a short mode word and 148 bits with a
long one.

### Compressed stream

The compressed stream starts at the bit right after the header (bit 96 or 148,
so not necessarily at a byte boundary). It is the output of `zfp_compress` for
the field described by the header, compressed with the header's `minbits`,
`maxbits`, `maxprec` and `minexp`, using zfp codec version 5 and serial
execution. The stream format is defined by the [zfp algorithm] and its
reference implementation. After the last block, zero bits pad the stream to a
multiple of 64 bits, so the encoded chunk length is a multiple of 8 bytes.

### Decoding

1. Read the header. The first four bytes must be `7A 66 70 05`. Recover
   the scalar type, `d`, the field sizes and the compression parameters.
2. The scalar type must match the array's data type (see
   [Supported data types](#supported-data-types)), and the field sizes, listed
   slowest-varying first (`[nx]`, `[ny, nx]`, `[nz, ny, nx]` or
   `[nw, nz, ny, nx]`), must equal the chunk shape (`[1]` for a
   zero-dimensional chunk). Otherwise the chunk is invalid.
3. Decompress `nx * ny * nz * nw` values (absent sizes count as 1) from the
   compressed stream with those parameters, and arrange them in C order into
   the chunk shape.

numcodecs 0.17.1 decodes with `zfpy.decompress_numpy`, which returns an array
shaped from the header. zarr-python then reshapes that array to the chunk
shape, without checking that the header shape or scalar type matches.

### Worked examples

A `float64` chunk of shape `[4]` holding `[1, 2, 3, 4]`, encoded with
`{"mode": 4, "tolerance": 0.01}`, is these 24 bytes:

```
7a 66 70 05 33 00 00 00 00 00 c0 ca 05 88 59 00 00 00 00 00 00 00 00 00
```

- Bytes 0-3: magic `zfp` and codec version 5.
- Byte 4 is `0x33`: type bits `11` (float64), dimension bits `00` (`d = 1`),
  and the low 4 bits of `nx - 1 = 3`. Bytes 5 to 9 and the low half of byte 10
  are the rest of `nx - 1`, all zero.
- High half of byte 10 (`0xc`) and byte 11 (`0xca`): `M = 0xcac = 3244`, fixed
  accuracy with `minexp = 3244 - 3251 = -7`. Since `0.01 = 0.64 * 2^-6`,
  `floor(log2(0.01)) = -7`.
- Bytes 12 onwards: the compressed block, then zero padding to 192 bits.

A `float32` chunk of shape `[2, 3]` holding `[[0, 1, 2], [3, 4, 5]]`, encoded
with `{}` (reversible), starts with this 12-byte header:

```
7a 66 70 05 26 00 00 10 00 00 00 88
```

Here `META = 0x10000026`: type bits `10` (float32), dimension bits `01`
(`d = 2`), `nx - 1 = 2` (the last Zarr dimension, size 3) and `ny - 1 = 1`
(the first Zarr dimension, size 2). `M = 0x880 = 2176` (reversible).

For a `float64` chunk of shape `[2, 3, 4]` encoded with
`{"mode": 4, "tolerance": 0}`, the header has `M = 0xFFF` and the long mode
word `L = 0x787AFE0888000FFF`. This decodes to `minbits = 1`,
`maxbits = 16658`, `maxprec = 64` and `minexp = -1074`.

## Example

Array metadata `codecs` written by zarr-python for
`serializer=zarr.codecs.numcodecs.ZFPY(mode=4, tolerance=0.01)` and no
compressors:

```json
{
  "codecs": [
    {
      "name": "numcodecs.zfpy",
      "configuration": { "mode": 4, "tolerance": 0.01 }
    }
  ]
}
```

With `ZFPY()` the entry is `{ "name": "numcodecs.zfpy", "configuration": {} }`,
which means reversible compression.

## Relationship to `zfp`

The registered [`zfp`](../zfp/README.md) codec and `numcodecs.zfpy` use the
same zfp algorithm and map chunk dimensions to zfp field sizes the same way.
They differ as follows:

- Configuration: `zfp` names modes with strings (`"fixed_rate"`,
  `"fixed_precision"`, `"fixed_accuracy"`, `"reversible"`, `"expert"`), and
  `mode` and its parameter are required. `numcodecs.zfpy` uses integer mode
  codes, a negative value for "unset", and defaults that fall back to
  reversible mode. `numcodecs.zfpy` has no expert mode in its configuration,
  although its long-form headers can hold expert parameters.
- Data types: `zfp` also accepts lower-precision and unsigned types by
  promotion. `numcodecs.zfpy` accepts only `int32`, `int64`, `float32` and
  `float64`.
- Fixed rate: `zfp` uses zfp's `zfp_stream_set_rate`, which applies a minimum
  number of bits per block for floating-point types. `numcodecs.zfpy` does not.
- Encoded bytes: `zfp` writes no header. A `numcodecs.zfpy` chunk is the 96- or
  148-bit header followed by the same compressed bits that a headerless
  encoding with the same compression parameters produces, then zero padding to
  64 bits. The stored bytes are therefore not identical, and array metadata
  cannot be rewritten from one codec to the other without changing the chunks.

A `numcodecs.zfpy` chunk can be converted to a `zfp` chunk without running the
compressor again. Shift the bit sequence down by the header length and drop
trailing zero bytes beyond the 64-bit boundary that `zfp` would produce. Then
use a `zfp` configuration whose mode yields the same `minbits`, `maxbits`,
`maxprec` and `minexp` as the header. For fixed rate this is only possible
when the type-dependent minimum does not change the bits per block. Because
the parameters live in each chunk's header, every chunk of the array must
have the same parameters for a single `zfp` configuration to apply.

## Implementations

- [zarr-python](https://github.com/zarr-developers/zarr-python):
  `zarr.codecs.numcodecs.ZFPY`
- [numcodecs](https://github.com/zarr-developers/numcodecs): `numcodecs.ZFPY`
  (uses [zfpy](https://pypi.org/project/zfpy/), zfp's Python bindings)
- [zarrs](https://docs.rs/zarrs/latest/zarrs/array/codec/array_to_bytes/zfpy/index.html)
  (experimental)

## References

- [zfp]: the zfp repository and reference implementation.
- [zfp algorithm]: description of the zfp compression algorithm (release 1.0.1).
- [zfp compression modes]: definitions of `minbits`, `maxbits`, `maxprec` and
  `minexp`.
- [zfp high-level API]: `zfp_write_header`, `zfp_read_header`, `zfp_compress`.
- `zfp_field_metadata`, `zfp_stream_mode`, `zfp_stream_set_mode` and
  `zfp_write_header` in [src/zfp.c] define the header bit fields.

[zfp]: https://github.com/LLNL/zfp
[zfp algorithm]: https://zfp.readthedocs.io/en/release1.0.1/algorithm.html
[zfp compression modes]: https://zfp.readthedocs.io/en/release1.0.1/modes.html
[zfp high-level API]: https://zfp.readthedocs.io/en/release1.0.1/high-level-api.html
[src/zfp.c]: https://github.com/LLNL/zfp/blob/1.0.1/src/zfp.c

## Change log

No changes yet.

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)
