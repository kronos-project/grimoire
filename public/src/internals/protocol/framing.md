# Framing

At its core, a custom data framing format is used to exchange messages between two peers
maintaining a connection. The reviewed codecs establish the TCP format.
Equivalent UDP framing and session behaviour have not been verified.

Frames generally start with a [header](#header), followed by the [body](#body) holding the
frame opcode and the encapsulated message payload. Assume all data to be encoded in
little-endian byteorder unless stated otherwise.

## Header

Depending on the header type being used, the offsets of data in the following body
may be shifted.

### Small

For bodies below `0x8000` bytes in size, the following header is used:

| Offset | Type       | Description                               |
|--------|------------|-------------------------------------------|
| 0x0    | uint16     | The constant magic `0xF00D`               |
| 0x2    | uint16     | The length of the following [body](#body) |

## Large

For bodies above `0x7FFF` bytes in length, a different header encoding
strategy is employed:

| Offset | Type       | Description                               |
|--------|------------|-------------------------------------------|
| 0x0    | uint16     | The constant magic `0xF00D`               |
| 0x2    | uint16     | The constant `0x8000`                     |
| 0x4    | uint32     | The length of the following body          |

A deserializer should thus determine the size of the frame based on whether the
second header field value is `>= 0x8000`.

Both lengths exclude the outer header. Small headers occupy four bytes;
large headers occupy eight. Merlin selects the header using the full body
length. kronos-old instead checks the DML payload length, which disagrees
near the boundary; see [verification gaps](../../verification.md#transport).

TCP reads are not frame boundaries. Buffer partial headers and bodies, and
consume exactly one complete frame before parsing the next. A single read
may contain multiple frames. These lengths describe the plaintext framing;
encrypted connections first pass through the
[transport encryption layer](../encryption/handshake.md).

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
