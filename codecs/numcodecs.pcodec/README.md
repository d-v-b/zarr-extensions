# numcodecs.pcodec codec

Defines an `array -> bytes` codec that losslessly compresses a chunk of numbers
with [Pcodec] (pco). The chunk elements are taken in C (row-major) order and
stored as a single pco *standalone* file.

## Background

`numcodecs.pcodec` is the name that zarr-python writes for the numcodecs
`PCodec` codec. It is registered here so that existing data written with this
name has a specification. No registered codec without the `numcodecs.` prefix
implements pco.

## Codec name

The value of the `name` member in the codec object must be
`"numcodecs.pcodec"`.

## Configuration parameters

| Field                  | Type            | Required | Default               | Notes                                                                        |
| ---------------------- | --------------- | -------- | --------------------- | ---------------------------------------------------------------------------- |
| `level`                | integer         | no       | `8`                   | compression level, `0` to `12`                                               |
| `mode_spec`            | string          | no       | `"auto"`              | `"auto"` or `"classic"`                                                      |
| `delta_spec`           | string          | no       | `"auto"`              | `"auto"`, `"no_op"`, `"none"`, `"try_consecutive"` or `"try_lookback"`       |
| `paging_spec`          | string          | no       | `"equal_pages_up_to"` | must be `"equal_pages_up_to"`                                                |
| `delta_encoding_order` | integer or null | no       | `null`                | `0` to `7`; see [below](#delta_spec-and-delta_encoding_order)                |
| `equal_pages_up_to`    | integer         | no       | `262144`              | maximum number of elements per pco chunk, at least `1`                       |

zarr-python writes only the parameters that the user passed, so
`configuration` is often `{}`. Readers must accept a configuration with
omitted optional fields and apply the defaults above. Any other field makes the
configuration invalid, with one exception: `numcodecs.zarr3` in numcodecs
0.14.0 to 0.15.0 also wrote `"id": "pcodec"` inside `configuration`. Readers
must accept and ignore that member; writers must not write it.

Writers must include the `configuration` member, even when it is `{}`:
zarr-python rejects `numcodecs.*` codec metadata without it. Readers may treat
a missing `configuration` as `{}`.

None of the parameters are needed to decode a chunk. They only steer the
encoder, and the decoder reads everything it needs from the encoded bytes.

### `level`

Controls how hard the encoder searches for a compact representation. Higher
values are slower and usually compress better. pco rejects values above `12`.

### `mode_spec`

`"auto"` lets the encoder pick the pco mode (Classic, IntMult, FloatMult or
FloatQuant) that it estimates compresses best. `"classic"` forces Classic mode.
numcodecs does not expose pco's other mode specs (for example Dict mode).

### `delta_spec` and `delta_encoding_order`

`delta_spec` selects the delta encoding search:

- `"auto"`: the encoder chooses among no delta, consecutive deltas and
  lookback deltas.
- `"no_op"`: no delta encoding. `"none"` is a deprecated alias for `"no_op"`
  that numcodecs still accepts (with a warning); writers should write
  `"no_op"`.
- `"try_consecutive"`: consecutive deltas of order `delta_encoding_order`.
- `"try_lookback"`: try lookback delta encoding.

`delta_encoding_order` is an integer from `0` to `7` or `null`. These
combinations are valid:

| `delta_spec`                  | `delta_encoding_order` | Effect                                                                 |
| ----------------------------- | ---------------------- | ---------------------------------------------------------------------- |
| `"try_consecutive"`           | `0` to `7`             | consecutive deltas of that order (`0` is the same as no delta)          |
| `"auto"`                      | `0` to `7`             | same as `"try_consecutive"` with that order (kept for older metadata)   |
| any value                     | `null` or omitted      | as described by `delta_spec`                                           |

Every other combination is invalid: a non-null `delta_encoding_order` with
`"no_op"`, `"none"` or `"try_lookback"`, and `"try_consecutive"` without a
non-null `delta_encoding_order`. zarr-python accepts invalid combinations
when it creates an array, but numcodecs 0.17.1 raises an error when it encodes
the first chunk.

### `paging_spec` and `equal_pages_up_to`

The only paging spec is `"equal_pages_up_to"`. A Zarr chunk of `N` elements is
split into `P = ceil(N / equal_pages_up_to)` pco chunks of nearly equal size:
the first `N mod P` pco chunks hold `floor(N / P) + 1` elements and the rest
hold `floor(N / P)`. The pco format limits a pco chunk to `2^24` (16777216)
elements, so encoding fails if `equal_pages_up_to` is larger than `2^24` and
the Zarr chunk has more than `2^24` elements. Writers should not choose
`equal_pages_up_to` larger than `2^24`.

## Supported data types

| Zarr data type | pco number type | pco number type byte |
| -------------- | --------------- | -------------------- |
| `uint16`       | u16             | `7`                  |
| `uint32`       | u32             | `1`                  |
| `uint64`       | u64             | `2`                  |
| `int16`        | i16             | `8`                  |
| `int32`        | i32             | `3`                  |
| `int64`        | i64             | `4`                  |
| `float16`      | f16             | `9`                  |
| `float32`      | f32             | `5`                  |
| `float64`      | f64             | `6`                  |

Other data types are not supported. numcodecs 0.17.1 rejects `int8` and
`uint8` (pco can encode 8-bit types only if a flag is set that numcodecs does
not expose), and rejects `bool` and complex types. zarr-python also encodes the
`numpy.datetime64` and `numpy.timedelta64` data types, which have 64-bit
integer storage, as pco `i64` (number type byte `4`) holding the stored integer
values.

## Format

This is an `array -> bytes` codec. The encoded chunk is a pco standalone file
as defined in the [pco format specification] (the "Standalone Format" section,
which embeds the "Wrapped Format Components"). The rest of this section states
which standalone file a chunk maps to; it does not repeat the pco spec.

### Encoding

Given a chunk of shape `[s_0, ..., s_{n-1}]` (with `n >= 0`) and one of the
supported data types:

1. Let `N = s_0 * ... * s_{n-1}` (`N = 1` for a zero-dimensional chunk). Take
   the `N` chunk elements in C order: the last dimension varies fastest.
2. Write a pco standalone file that holds this sequence of `N` numbers with
   the pco number type from the table above, split into pco chunks as
   described for [`equal_pages_up_to`](#paging_spec-and-equal_pages_up_to),
   each with exactly one page.

The file pcodec 1.x (the Python pco binding used by numcodecs 0.17.1) writes
has this layout. Fields are packed least-significant bit first, and multi-byte
fields are little-endian:

| Bits / bytes             | Content                                                                                 |
| ------------------------ | --------------------------------------------------------------------------------------- |
| 4 bytes                  | magic `70 63 6f 21` (ASCII `pco!`)                                                      |
| 1 byte                   | standalone version, `3`                                                                 |
| 1 byte                   | uniform number type, `0` (none declared)                                                |
| 6 bits                   | `n_hint_log2 - 1`                                                                       |
| `n_hint_log2` bits       | `n_hint = N`, followed by zero bits up to the next byte boundary                        |
| 2 bytes                  | wrapped format version: major `4`, minor `1`                                            |
| per pco chunk: 1 byte    | pco number type byte                                                                    |
| per pco chunk: 3 bytes   | number of elements in the pco chunk minus 1, little-endian                              |
| per pco chunk            | wrapped chunk metadata, then one wrapped page holding all elements of the pco chunk     |
| 1 byte                   | termination byte `00`                                                                   |

Encoders may make any choice the pco format allows (mode, delta encoding,
binning, standalone and format version), so the encoded bytes are not unique.
The configuration only influences these choices. Releases of pcodec before
1.0 wrote earlier format versions (standalone version 2 or 3, wrapped format
version 3 or lower). Under pco's compatibility guarantee, current decoders
read those too.

### Decoding

1. Parse the bytes as a pco standalone file following the
   [pco format specification], and concatenate the numbers of all its pco
   chunks in order.
2. Each pco chunk's number type byte (and the uniform number type byte if it
   is not `0`) must be the one the table above gives for the array's data
   type, and the total number of elements must equal `N`. Otherwise the chunk
   is invalid.
3. Arrange the `N` numbers in C order into the chunk shape.

numcodecs 0.17.1 decodes with `pcodec.standalone.simple_decompress`, which
returns a one-dimensional array of whatever number type the file declares.
zarr-python then reshapes it to the chunk shape, but does not check the number
type.

### Worked example

A `uint32` chunk of shape `[3]` holding `[1, 2, 3]`, encoded with
`{"level": 0, "mode_spec": "classic", "delta_spec": "no_op"}`, is these 24
bytes (as written by zarr-python 3.4.2 with numcodecs 0.17.1 and pcodec 1.1.0):

```
70 63 6f 21 03 00 c1 04 01 01 02 00 00 00 10 00 08 00 00 00 10 00 24 00
```

| Bytes                 | Meaning                                                                                                                  |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `70 63 6f 21`         | magic `pco!`                                                                                                             |
| `03`                  | standalone version 3                                                                                                     |
| `00`                  | no uniform number type                                                                                                   |
| `c1`                  | low 6 bits `000001`: `n_hint_log2 = 2`; next 2 bits `11`: `n_hint = 3`                                                   |
| `04 01`               | wrapped format version 4.1                                                                                               |
| `01`                  | pco chunk number type `u32`                                                                                              |
| `02 00 00`            | 3 elements in this pco chunk                                                                                             |
| `00`                  | mode 0 (Classic) in 4 bits, delta encoding 0 (none) in 4 bits                                                            |
| `10 00 08 00 00 00 10 00` | one latent variable: `ans_size_log = 0`, 1 bin, lower bound `1`, 2 offset bits, then zero bits to the byte boundary |
| `24`                  | page: no tANS state bits, then 2-bit offsets `0`, `1`, `2` (bits `00`, `10`, `01`, least significant bit first)          |
| `00`                  | termination byte                                                                                                         |

Decoding adds each offset to the bin's lower bound: `[1, 2, 3]`.

## Example

Array metadata `codecs` written by zarr-python for
`serializer=zarr.codecs.numcodecs.PCodec(mode_spec="classic", delta_spec="try_consecutive", delta_encoding_order=2, equal_pages_up_to=1000)`
and no compressors:

```json
{
  "codecs": [
    {
      "name": "numcodecs.pcodec",
      "configuration": {
        "mode_spec": "classic",
        "delta_spec": "try_consecutive",
        "delta_encoding_order": 2,
        "equal_pages_up_to": 1000
      }
    }
  ]
}
```

With `PCodec()` the entry is `{ "name": "numcodecs.pcodec", "configuration": {} }`.

## Implementations

- [zarr-python](https://github.com/zarr-developers/zarr-python):
  `zarr.codecs.numcodecs.PCodec`
- [numcodecs](https://github.com/zarr-developers/numcodecs): `numcodecs.PCodec`
  (uses the [pcodec](https://pypi.org/project/pcodec/) Python package)
- [zarrs](https://docs.rs/zarrs/latest/zarrs/array/codec/array_to_bytes/pcodec/index.html)
  (experimental)

## References

- [Pcodec]: the pco repository, with the Rust reference implementation.
- [pco format specification]: `docs/format.md` in the pco repository,
  pinned here to release v1.1.0.

[Pcodec]: https://github.com/pcodec/pcodec
[pco format specification]: https://github.com/pcodec/pcodec/blob/v1.1.0/docs/format.md

## Change log

No changes yet.

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)
