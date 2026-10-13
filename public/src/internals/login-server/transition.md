# Game Transition

This is a continuation of the [Authentication Chapter](./auth.md).

After successful exchange of `MSG_USER_AUTHEN_V3` and `MSG_USER_AUTHEN_RSP`
messages, the graphical client is now up and running in possession of the
player's `UserID`, username and `ClientKey2`.

The next step is the transition from Patch Client to WizardGraphicalClient.

## PassKey3

PassKey3, or PK3, is derived from ClientKey2 by the client and will be sent to
the server. It uses the same algorithm as [ClientKey1](./auth.md#clientkey1):

```py
from base64 import b64encode
from hashlib import sha512

salt = f"{sid}{time_secs}{time_millis}".encode("ascii")
seed = clientkey2.encode("ascii")
passkey3 = b64encode(sha512(seed + salt).digest()).decode("ascii")
```

Hash the base64 text of CK2, without decoding it or applying the initial
password hash step. Use the current connection's Session Offer values; these may
differ from those used by the Patch Client for authentication. The derivation
agrees with kronos-old's shared passkey helper, but its PK3 validation handler
is unfinished.

## Identity Validation

### Request

The game client first sends `MSG_USER_VALIDATE` with the previously obtained
`UserID` and `PassKey3`.

The same `MachineID` and `PatchClientID` that were already used in
`MSG_USER_AUTHEN_V3` should be sent to reduce the possibility of account theft
if machines were switched in-between authentication and validation. Both
messages also carry `Locale`. The fields are established by the schema; the
exact checks performed by KI have not been verified.

### Response

The server is then supposed to respond with `MSG_USER_VALIDATE_RSP`. This once
again contains an echo of the `UserID`, `PayingUser` and a potential error code.
In the reviewed schema, its field order is `Error` (`INT`), `Reason` (`STR`),
`UserID` (`GID`), `TimeStamp` (`STR`), `PayingUser` (`INT`), and `Flags`
(`INT`).

### Errors

The error codes are once again [string IDs](../string-id.md) with known values
listed below:

| String             | When to send                                  |
| ------------------ | --------------------------------------------- |
| `""`               | No error                                      |
| `"AccountBanned"`  | User's account is banned                      |
| `"MachineBanned"`  | User's machine is banned                      |
| `"ValidateFailed"` | Invalid PK3 for user or internal server error |
| `"Timeout"`        | Timeout while trying to process the request   |
| `"FtpCapped"`      | ???                                           |
| `"ErrorNoLock"`    | ???                                           |
| `"FailedUpload"`   | ???                                           |

## Load Balancing

### Busy Queue

After successful `MSG_USER_VALIDATE_RSP`, the server follows up with
`MSG_USER_ADMIT_IND`.

This is a load balancing mechanism with the _Game Server_ to ensure not too many
users are trying to enter the game at the same time.

The message carries `Status` (`INT`) followed by `PositionInQueue` (`UINT`).
Queue scheduling, including FIFO order or admitting one user at a time, is not
established by the wire format or the reviewed handlers.

When the queue positions of users change, they are supposed to be informed with
`MSG_USER_ADMIT_IND` every time. The user that was removed from the queue is
position `0`.

#### Status

| Value | Meaning                                 |
| ----- | --------------------------------------- |
| `1`   | Success                                 |
| `2`   | Busy; the client must wait in the queue |

Status `2` is recorded in the existing
[magic numbers notes](../dml/magic-numbers.md), but the reviewed handler only
emits status `1`. A versioned capture is still needed to confirm the busy path.

### Afk Handling

The login protocol includes inactivity notifications. The interval and
disconnect policy are not established by the reviewed wire schema.

The client sends `MSG_LOGIN_NOT_AFK` with `BadgeNameID` (`UINT`) to indicate
activity. The exact UI actions that trigger it have not been established.

When the interval expires, the server should free all resources, send a
user-facing notification, and close the connection. The reviewed login schema
provides `MSG_DISCONNECT_LOGIN_AFK` (order `17`) with a `Warning` (`BYT`) field.
The warning values and timing remain unverified.
