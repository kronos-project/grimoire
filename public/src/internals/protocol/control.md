# Control Messages

Control messages are transmitted to signal events related to the creation and maintenance of
[sessions](./sessions.md) and connections in general.

The following explains the purpose and structure of various types of control messages.

*TODO: Figure out opcode 0x1 and 0x2.*

## Opcode

To identify a control message in the first place, the [frame body](./framing.md#body) encodes
the message opcode. Opcodes are unique values that map to exactly one type of frame and tell
the parser how the message data should be interpreted.

For concrete values, see the following message types.

## Session Offer

> Opcode: `0x0`

When a new client connects to the server, it is greeted with this type of frame. Since most
of the game data exchange requires an active session, it must first be established through
the client completing the handshake by responding with a [Session Accept](#session-accept)
message.

| Offset | Type    | Description                                                     |
|--------|---------|-----------------------------------------------------------------|
| 0x0    | uint16  | The proposed [session ID](./sessions.md#session-id) to agree on |
| 0x2    | int32   | The high 4 bytes of the UNIX timestamp the message was sent at  |
| 0x6    | int32   | The low 4 bytes of the UNIX timestamp the message was sent at   |
| 0xA    | uint32  | The milliseconds into the second this message was sent at       |
| 0xE    | uint32  | Length prefix for unknown field                                 |
| 0x12   | uint8[] | Unknown                                                         |
| -      | uint8   | Reserved; always zero                                           |

## Keep Alive

> Opcode: `0x3`

A bi-directional opcode that may be initiated independently by both, client and server.
These messages are exchanged at fixed intervals and a [Keep Alive Rsp](#keep-alive-rsp)
from the other peer ensures the session is still alive.

The structure of this message (and the response to it) varies depending on the party
sending it:

**Client-initiated Keep Alive:**

| Offset | Type   | Description                                                 |
|--------|--------|-------------------------------------------------------------|
| 0x0    | uint16 | The corresponding [session ID](./sessions.md#session-id)    |
| 0x2    | uint16 | The milliseconds into the second the message was sent at    |
| 0x4    | uint16 | How many minutes have elapsed since the session was started |

**Server-initiated Keep Alive:**

| Offset | Type   | Description                                               |
|--------|--------|-----------------------------------------------------------|
| 0x0    | uint16 | [Invalid session ID value](./sessions.md#session-id)      |
| 0x2    | uint32 | The number of milliseconds since the server was started   |

## Keep Alive Rsp

> Opcode: `0x4`

This message is used to acknowledge a received [Keep Alive](#keep-alive) message and
confirm that the other end of the connection is still listening. Either peer should
respond to this using the structure of the preceding [Keep Alive](#keep-alive) message.

## Session Accept

> Opcode: `0x5`

Sent from the client to the server as a response to a [Session Offer](#session-offer)
message. This completes the handshake and confirms the creation of a new session bound
to the proposed ID.

From that point onwards, the session must be kept alive by
[heartbeating](./sessions.md#heartbeat).

| Offset | Type    | Description                                                    |
|--------|---------|----------------------------------------------------------------|
| 0x0    | uint16  | Reserved; always zero                                          |
| 0x2    | int32   | The high 4 bytes of the UNIX timestamp the message was sent at |
| 0x6    | int32   | The low 4 bytes of the UNIX timestamp the message was sent at  |
| 0xA    | uint32  | The milliseconds into the second the message was sent at       |
| 0xE    | uint16  | The proposed session ID from [Session Offer](#session-offer)   |
| 0x10   | uint32  | Length prefix for unknown field                                |
| 0x14   | uint8[] | Unknown                                                        |
| -      | uint8   | Reserved; always zero                                          |

