# endian codec (legacy name of `bytes`)

Defines `endian`, the name that drafts of the Zarr V3 specification gave to the
[`bytes`](https://zarr-specs.readthedocs.io/en/latest/v3/codecs/bytes/index.html)
codec. It is an `array -> bytes` codec. This name is registered so that
existing data written with it has a specification. It is read-only: writers
must not write it.

## Background

The Zarr V3 specification defined this codec as `endian` until
[commit 267043d](https://github.com/zarr-developers/zarr-specs/commit/267043d2a40e7fad02d77ca08854adbcd4b3e890)
(August 2023) renamed it to `bytes` without changing its configuration or
encoding. The
[last version of the `endian` specification](https://github.com/zarr-developers/zarr-specs/blob/8741619ef4a651621b26d1390bf1233df39014db/docs/v3/codecs/endian/v1.0.rst)
describes the same format as the `bytes` codec.

[zarrita](https://pypi.org/project/zarrita/) releases 0.1.0a0 to 0.1.0a18
(March to August 2023) wrote this name, for example
`{"name": "endian", "configuration": {"endian": "little"}}`, in the `codecs`
of arrays and in the `codecs` and `index_codecs` of `sharding_indexed` codecs.
zarrita 0.1.0a19 and later write `bytes`, and read `endian` as `bytes`.

## Codec name

The value of the `name` member in the codec object is `"endian"`.

## Configuration parameters

| field    | type   | required | notes                         |
| -------- | ------ | -------- | ----------------------------- |
| `endian` | string | yes      | `"little"` or `"big"`         |

No other fields are permitted.

## Format

An `endian` codec with a given configuration encodes and decodes exactly as the
[`bytes`](https://zarr-specs.readthedocs.io/en/latest/v3/codecs/bytes/index.html)
codec with the same configuration. Readers must treat
`{"name": "endian", "configuration": C}` as `{"name": "bytes", "configuration": C}`
wherever a codec may appear, including inside the `codecs` and `index_codecs`
of a `sharding_indexed` codec.

Implementations may rewrite `endian` to `bytes` in stored metadata without
changing any chunk data.

## Writing

Writers must not write this codec name. They must write `bytes` instead.

## Example

Array metadata `codecs` written by zarrita 0.1.0a18 for a `uint16` array:

```json
{
  "codecs": [
    { "name": "endian", "configuration": { "endian": "little" } }
  ]
}
```

This is read as:

```json
{
  "codecs": [
    { "name": "bytes", "configuration": { "endian": "little" } }
  ]
}
```

## Implementations

- [zarrita](https://pypi.org/project/zarrita/) 0.1.0a0 to 0.1.0a18 write this
  name; 0.1.0a19 and later read it as `bytes`.
- [zarr-python](https://github.com/zarr-developers/zarr-python) registers the
  name as an alias of `BytesCodec`, but up to and including 3.4 fails to open
  arrays that use it.

## Change log

No changes yet.

## Current maintainers

* [zarr-python core development team](https://github.com/orgs/zarr-developers/teams/python-core-devs)
