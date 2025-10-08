# `anscombe-transform`

Defines an array-array codec that encodes an input array using the [Anscombe transform](https://en.wikipedia.org/wiki/Anscombe_transform) and decodes using the inverse Anscombe transform.

The Anscombe transform is [bijection](https://en.wikipedia.org/wiki/Bijection) from a [Poisson-distributed](https://en.wikipedia.org/wiki/Poisson_distribution) variable to an approximately [Gaussian-distributed](https://en.wikipedia.org/wiki/Normal_distribution) variable with a variance of 1.

This transformation is useful in sensing applications to mitigate [shot noise](https://en.wikipedia.org/wiki/Shot_noise). Shot noise is typically modelled as a Poisson process. The variance of a Poisson-distributed signal scales with its mean. The Anscombe transform maps a Poisson-distributed signal to a Gaussian-distributed signal with a variance near 1. Decoupling the mean of the signal from its variance facilitates noise removal and data compression, the latter of which is the intended application of this codec.

### Codec algorithm

#### Encoding

In addition to the input data (an array), the encoding procedure takes the following parameters:

| name | type | 
| - | - | 
| `conversion_gain` | positive real number |
| `zero_level` | real number |
| `beta` | real number from the inverval `(0, 1]` |

For each element `x` of the input array, an output value `y` is generated via the following procedure:

```python
def anscombe_transform(x, conversion_gain, zero_level, beta):
    # Convert to event-rate units
    event_rate = (x - zero_level) / sensitivity

    zero_slope = 1.0 / (beta * math.sqrt(3.0 / 8.0))
    offset = zero_level * zero_slope / sensitivity

    if value < 0:
        # Linear extrapolation for negative photon rates
        result = offset + value * zero_slope
    else:
        # Anscombe variance-stabilizing transform
        result = offset + (2.0 / beta) * (np.sqrt(value + 3.0 / 8.0) - np.sqrt(3.0 / 8.0))
    y[it.multi_index] = result
    it.iternext()

return y

```



#### Decoding
<TODO>

### Codec metadata

| field | type | required | notes |
| - | - | - | - |
| `"name"` | literal `"anscombe-transform"` | yes | |
| `"configuration"` | [anscombe transform configuration](#configuration-metadata) | yes | |

#### Configuration metadata

| field | type | required | notes |
| - | - | - | - |
| `"zero_level"` | number | yes | The value in the input array that corresponds to 0 detected events.
| `"beta"` | number from the interval `(0, 1]` | yes | <TODO> |
| `"conversion_gain"` | positive number | yes | The magnitude of a single recorded event in the input data |
| `"encoded_dtype"` | Zarr V3 data type metadata or `null` | no | The data type of the output array. If unspecified or `null`, then the output data type matches the input data type. | 

### Supported array data types

This codec is compatible with any array data type that supports the operations required to implement the Anscombe transformation and its inverse. 
