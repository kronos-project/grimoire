# Framing

At its core, a custom data framing format is used to exchange messages between two peers
maintaining a connection. No difference is made between TCP-based and UDP-based
connections.

Frames generally start with a [header](#header), followed by the [body](#body) holding the
frame opcode and the encapsulated message payload. Assume all data to be encoded in
little-endian byteorder unless stated otherwise.

## Header

Depending on the header type being used, the offsets of data in the following body
may be shifted.

### Small

For [data messages](./data.md) where the DML message payload is below `0x8000` bytes
in size, the following header is used:

| Offset | Type       | Description                               |
|--------|------------|-------------------------------------------|
| 0x0    | uint16     | The constant magic `0xF00D`               |
| 0x2    | uint16     | The length of the following [body](#body) |

## Large

For [data messages](./data.md) above `0x7FFF` bytes in length, a different header
encoding strategy is employed:

| Offset | Type       | Description                               |
|--------|------------|-------------------------------------------|
| 0x0    | uint16     | The constant magic `0xF00D`               |
| 0x2    | uint16     | The constant `0x8000`                     |
| 0x4    | uint32     | The real size of this "large" frame       |

A deserializer should thus determine the size of the frame based on whether the
second header field value is `>= 0x8000`.

## Body

The body provides metadata of the contained message payload that is needed to decode it.

Messages are either [control messages](./control.md) or [data messages](./data.md) which
are further detailed in their respective sections.

| Offset | Type    | Description                                                          |
|--------|---------|----------------------------------------------------------------------|
| 0x0    | bool    | Whether the frame is a [control message](./control.md)               |
| 0x1    | uint8   | The message [opcode](./control.md#opcode); only for control messages |
| 0x2    | uint8   | Reserved; always zero                                                |
| 0x3    | uint8   | Reserved; always zero                                                |
| 0x4    | uint8[] | The contained message data                                           |
