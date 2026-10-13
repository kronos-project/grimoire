# String ID

In many places throughout the game, strings are hashed into 32-bit integer values
which are more compact to handle and convey similar intent.

Such places are *ObjectProperty* serialization, implementation-defined error codes
in the network protocol, and enums in code.

## Algorithm

The following presents our flavor of the algorithm which differs from KI's but
produces matching results.

```py
def sign_extend(value: int) -> int:
    return (value & 0x7FFFFFFF) - (value & 0x80000000)


def make_string_id(string: str) -> int:
    result = 0

    for index, value in enumerate(string.encode()):
        value -= 32
        shift = 5 * index % 32

        result ^= sign_extend(value << shift)
        if shift > 24:
            result ^= sign_extend(value >> (32 - shift))

    return abs(result) & 0xFFFFFFFF
```
