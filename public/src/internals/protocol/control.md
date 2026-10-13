# Control Messages

Control messages are transmitted to signal events related to the creation and
maintenance of [sessions](./sessions.md) and connections in general.

The following explains the purpose and structure of various types of control
messages.

Layouts use the book's [binary notation](../../notation.md). All offsets start
at the control payload (`message_data` in the frame body), excluding the outer
header and the four-byte frame metadata.

_TODO: Figure out opcode 0x1 and 0x2._

## Opcode

To identify a control message in the first place, the
[frame body](./framing.md#body) encodes the message opcode. Opcodes are unique
values that map to exactly one type of frame and tell the parser how the message
data should be interpreted.

For concrete values, see the following message types.

## Session Offer

> Opcode: `0x0`

When a new client connects to the server, it is greeted with this type of frame.
Since most of the game data exchange requires an active session, it must first
be established through the client completing the handshake by responding with a
[Session Accept](#session-accept) message.

Let `L` be `signed_message_length` in this table.

| Offset     | Type    | Field                 | Description                                     |
| ---------- | ------- | --------------------- | ----------------------------------------------- |
| `0x00`     | `u16`   | session_id            | Proposed session ID                             |
| `0x02`     | `i32`   | timestamp_high        | High 32 bits of epoch seconds                   |
| `0x06`     | `i32`   | timestamp_low         | Low 32 bits of epoch seconds                    |
| `0x0A`     | `u32`   | timestamp_millis      | Milliseconds into the second                    |
| `0x0E`     | `u32`   | signed_message_length | Signed blob byte count, including its signature |
| `0x12`     | `u8[L]` | signed_message        | Signed message and RSA signature                |
| `0x12 + L` | `u8`    | terminator            | Always `0`; outside the signed blob             |

The offer's length-prefixed field is `SignedMessage` in the
[encryption handshake](../encryption/handshake.md). Its length includes the
signature but excludes its four-byte prefix and the trailing zero. kronos-old's
plaintext codec omits both the length and the blob, placing the trailing zero at
`0xE`. These are different layouts; an empty blob in the extended layout still
requires a four-byte zero length.

## Keep Alive

> Opcode: `0x3`

A bi-directional opcode that may be initiated independently by both, client and
server. These messages are exchanged at fixed intervals and a
[Keep Alive Rsp](#keep-alive-rsp) from the other peer ensures the session is
still alive.

The structure of this message (and the response to it) varies depending on the
party sending it:

**Client-initiated Keep Alive:**

| Offset | Type  | Field            | Description                       |
| ------ | ----- | ---------------- | --------------------------------- |
| `0x00` | `u16` | session_id       | Corresponding session ID          |
| `0x02` | `u16` | timestamp_millis | Milliseconds into the second      |
| `0x04` | `u16` | elapsed_minutes  | Minutes since the session started |

**Server-initiated Keep Alive:**

| Offset | Type  | Field         | Description                           |
| ------ | ----- | ------------- | ------------------------------------- |
| `0x00` | `u16` | session_id    | Always `0`; server-initiated request  |
| `0x02` | `u32` | uptime_millis | Milliseconds since the server started |

## Keep Alive Rsp

> Opcode: `0x4`

This message is used to acknowledge a received [Keep Alive](#keep-alive) message
and confirm that the other end of the connection is still listening. Either peer
should respond to this using the structure of the preceding
[Keep Alive](#keep-alive) message.

## Session Accept

> Opcode: `0x5`

Sent from the client to the server as a response to a
[Session Offer](#session-offer) message. This completes the handshake and
confirms the creation of a new session bound to the proposed ID.

From that point onwards, the session must be kept alive by
[heartbeating](./sessions.md#heartbeat).

Let `L` be `encrypted_message_length` in this table.

| Offset     | Type    | Field                    | Description                                   |
| ---------- | ------- | ------------------------ | --------------------------------------------- |
| `0x00`     | `u16`   | reserved                 | Always `0`                                    |
| `0x02`     | `i32`   | timestamp_high           | High 32 bits of epoch seconds                 |
| `0x06`     | `i32`   | timestamp_low            | Low 32 bits of epoch seconds                  |
| `0x0A`     | `u32`   | timestamp_millis         | Milliseconds into the second                  |
| `0x0E`     | `u16`   | session_id               | Proposed session ID from the offer            |
| `0x10`     | `u32`   | encrypted_message_length | Blob byte count, including the algorithm byte |
| `0x14`     | `u8[L]` | encrypted_message        | Algorithm byte and encrypted confirmation     |
| `0x14 + L` | `u8`    | terminator               | Always `0`; outside the encrypted blob        |

The accept's length-prefixed field is `EncryptedMessage` in the encryption
handshake. Its length includes the algorithm byte. kronos-old's plaintext codec
instead omits the length and blob, placing the trailing zero at `0x10`. Select
the layout for the target client revision; do not infer it solely from whether
the connection is encrypted.
