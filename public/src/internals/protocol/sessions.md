# Sessions

Sessions between a client and a server are an important aspect of the network communication
as they carry connection state used by later protocol operations. Establishing
a transport session does not authenticate the user's account; that is a
separate [login exchange](../login-server/auth.md).

## Handshake

To establish a new sessions between two peers, either directly after a client opened a
connection or after the [invalidation of a session](#refreshing-a-session), a handshake
must be performed.

Clients should listen for [Session Offer](./control.md#session-offer) messages at any time,
and respond with the appropriate [Session Accept](./control.md#session-accept) after
receiving it.

When the session IDs match and the server does not respond with yet another
[Session Offer](./control.md#session-offer) message, the session should be considered
established.

Both parties are expected to cache the relevant information from the handshake, such as
server uptime values, the timestamp the messages were sent, and obviously also the ID.
Various algorithms may rely on these data as unique variables individually assigned to
every client.

## Refreshing a session

The behaviour below is retained from the original notes. The reviewed
repositories do not establish KI's coordination between servers or the
scope of session ID uniqueness. Confirm refresh and retry behaviour with
the target client/server revision before relying on it.

During the lifetime of a TCP or UDP connection with the server, a very common event to
expect is such sessions actually being dropped in the middle of operations.

This act of dropping sessions is generally taking place when a client connects to
several servers at the same time and establishes sessions with them while already maintaining
a session with one of the servers. The server in question then tells all other servers
(including the ones the client is still connected to) to drop any sessions with that specific
client.

But instead of closing the underlying socket, the session will be refreshed. Since servers
are assuming that active exchange of messages only happens with one of them at the same
time, regardless of how many a client is actually connected to, a server will wait for the
first message from a client after the session invalidation, ignore it, and instead initiate
the [handshaking process](#handshake) with a different ID.

When the process is completed, the client is expected to re-send its last message prior to
receiving a new [Session Offer](./control.md#session-offer) message as the server ignored
it up to that point.

## Session ID

The session ID identifies an active session and is represented as an
unsigned 16-bit integer. The reviewed code does not establish whether its
uniqueness extends beyond the server that issued it.

A value of `0` is reserved to the server to indicate that a
[Keep Alive](./control.md#keep-alive) message is server-initiated. It should never be
assigned to clients in order to avoid confusion of frame types.

## Heartbeat

When the [handshake](#handshake) was successfully completed and the session is up and
running, the heartbeating process must begin in order to keep the session alive.

This step involves exchanging [Keep Alive](./control.md#keep-alive) and corresponding
[Keep Alive Rsp](./control.md#keep-alive-rsp) messages at fixed intervals and waiting
for the response of the peer.

The reviewed kronos-old frame helpers specify client requests every
**10 seconds** and server requests every **60 seconds**. These are reference
intervals; exact KI timeout and retry policies are not verified.

The response echoes the request's six payload bytes, changing the opcode
from `0x3` to `0x4`. A response's layout is determined by the initiating
request, rather than by which peer sends the response. How many missed
responses cause a disconnect remains unverified.

## Access Level

Access Levels are numeric values denoting a specific set of permissions associated with
the session between an individual client and the server. Generally speaking, the higher
the value, the higher the permissions it grants.

Every DML message may have an access level value specified as a minimum threshold that
must be met in order for a peer to be allowed to process the data.

A non-exhaustive list of known values and their meanings is:

| Value | Description                                                         |
|-------|---------------------------------------------------------------------|
|  `0`  | No special requirements; can be used anytime                        |
|  `1`  | A valid session must be established in order to process the message |
