# Framing

At its core, a custom data framing format is used to exchange messages between
two peers maintaining a connection. The reviewed codecs establish the TCP
format. Equivalent UDP framing and session behaviour have not been verified.

Frames generally start with a [header](#header), followed by the [body](#body)
holding the frame opcode and the encapsulated message payload. Assume all data
to be encoded in little-endian byteorder unless stated otherwise. Layouts use
the book's [binary notation](../../notation.md).

## Header

Depending on the header type being used, the offsets of data in the following
body may be shifted.

### Small

For bodies below `0x8000` bytes in size, the following header is used:

| Offset | Type  | Field       | Description                               |
| ------ | ----- | ----------- | ----------------------------------------- |
| `0x00` | `u16` | magic       | Constant `0xF00D`                         |
| `0x02` | `u16` | body_length | Byte count of the following [body](#body) |

### Large

For bodies above `0x7FFF` bytes in length, a different header encoding strategy
is employed:

| Offset | Type  | Field         | Description                      |
| ------ | ----- | ------------- | -------------------------------- |
| `0x00` | `u16` | magic         | Constant `0xF00D`                |
| `0x02` | `u16` | length_marker | Constant `0x8000`                |
| `0x04` | `u32` | body_length   | Byte count of the following body |

A deserializer should thus determine the size of the frame based on whether the
second header field value is `>= 0x8000`.

Both lengths exclude the outer header. Small headers occupy four bytes; large
headers occupy eight. Merlin selects the header using the full body length.
kronos-old instead checks the DML payload length, which disagrees near the
boundary; see [verification gaps](../../verification.md#transport).

TCP reads are not frame boundaries. Buffer partial headers and bodies, and
consume exactly one complete frame before parsing the next. A single read may
contain multiple frames. These lengths describe the plaintext framing; encrypted
connections first pass through the
[transport encryption layer](../encryption/handshake.md).

## Body

The body provides metadata of the contained message payload that is needed to
decode it.

Messages are either [control messages](./control.md) or
[data messages](./data.md) which are further detailed in their respective
sections.

Let `B` be `body_length` from the outer header. Offsets below start at the body,
immediately after that header.

| Offset | Type        | Field        | Description                                               |
| ------ | ----------- | ------------ | --------------------------------------------------------- |
| `0x00` | `u8`        | is_control   | `1` for a control message; `0` for data                   |
| `0x01` | `u8`        | opcode       | Control opcode; `0` for data                              |
| `0x02` | `u8`        | reserved_0   | Always `0`                                                |
| `0x03` | `u8`        | reserved_1   | Always `0`                                                |
| `0x04` | `u8[B - 4]` | message_data | Contained message, including its trailing zero if present |
