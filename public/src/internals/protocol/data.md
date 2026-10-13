# Data Messages

Data messages carry application-specific data defined by a [DML Protocol](../dml/protocols.md).
However, DML as a serialization system is not inherently related to the framing protocol
itself, and should be considered a separate entity which is chosen due to its flexibility.

Along with the encoded DML payload, a small header provides the metadata needed for the
peer to understand the message.

The "message" terminology can either stand for a data message as part of the framing
protocol, which is documented here, or a DML message as a data structure encoded inside
a data message. These are not to be confused with each other, see
[Data Management Layer](../dml/index.html) for more details on the serialization system.

## Structure

An encoded message always follows a fixed structure. The opcode in the
[frame body](./framing.md#body) is always set to `0` for data messages.

| Offset | Type       | Description                                                      |
|--------|------------|------------------------------------------------------------------|
| 0x0    | uint8      | The service ID the message belongs to                            |
| 0x1    | uint8      | The order number within the protocol                             |
| 0x2    | uint16     | The length of the entire DML message data, including this header |
| 0x4    | uint8[len - 4] | The serialized DML message data                              |
| `len`  | uint8      | Trailing null byte; not considered by the length field           |

Here `len` is the value of the `uint16` length field and must be at least
`4`. If the serialized DML payload has `N` bytes, `len = N + 4` and the
frame body has `N + 9` bytes: four bytes of frame metadata, four bytes of
DML header, the payload, and one trailing zero. The outer frame length
includes this zero; the inner DML length excludes it.

For an empty `MSG_PING` in the reviewed BaseMessages definition, the full
small frame is:

```text
0D F0 09 00 00 00 00 00 01 01 04 00 00
```

The large outer frame header does not widen this inner 16-bit length.
Payloads above `65531` bytes cannot be represented by this layout without
an additional mechanism, which has not been verified.
