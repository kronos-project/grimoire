# Game Transition

This is a continuation of the [Authentication Chapter](./auth.md).

After successful exchange of `MSG_USER_AUTHEN_V3` and `MSG_USER_AUTHEN_RSP`
messages, the graphical client is now up and running in posession of the
player's `UserID`, username and `ClientKey2`.

The next step is the transition from Patch Client to WizardGraphicalClient.

## PassKey3

PassKey3, or PK3, is derived from ClientKey2 by the client and will be sent
to the server. It uses the same algorithm as [ClientKey1](./auth.md#clientkey1):

```py
from base64 import b64encode as base64
from hashlib import sha512

# Feed ClientKey2 into the hasher.
state = sha512(clientkey2)
# Mix in salt built from currently cached session information.
# NOTE: These may NOT be the same values which were used for authentication.
state.update(f"{sid}{time_secs}{time_millis}")

# Receive PK3 as the base64-encoded hash of *ClientKey2* and *salt*.
passkey3 = base64(state.digest())
```

## Identity Validation

### Request

The game client first sends `MSG_USER_VALIDATE` with the previously obtained
`UserID` and `PassKey3`.

The same `MachineID` and `PatchClientID` that were already used in
`MSG_USER_AUTHEN_V3` should be sent to reduce the possibility of account theft
if machines were switched in-between authentication and validation. Servers
should do the mandatory checks.

### Response

The server is then supposed to respond with `MSG_USER_VALIDATE_RSP`. This once
again contains an echo of the `UserID`, `PayingUser` and a potential error code.

### Errors

The error codes are once again [string IDs](../string-id.md) with known values
listed below:

| String              | When to send                                  |
|---------------------|-----------------------------------------------|
| `""`                | No error                                      |
| `"AccountBanned"`   | User's account is banned                      |
| `"MachineBanned"`   | User's machine is banned                      |
| `"ValidateFailed"`  | Invalid PK3 for user or internal server error |
| `"Timeout"`         | Timeout while trying to process the request   |
| `"FtpCapped"`       | ???                                           |
| `"ErrorNoLock"`     | ???                                           |
| `"FailedUpload"`    | ???                                           |

## Load Balancing

### Busy Queue

Immediately after `MSG_USER_VALIDATE_RSP`, the server is expected to follow up
with `MSG_USER_ADMIT_IND`.

This is a load balancing mechanism with the *Game Server* to ensure not too many
users are trying to enter the game at the same time.

The server is expected to maintain a FIFO queue of users trying to enter the game
and admit them one at a time when the server can currently handle the load.

When the queue positions of users change, they are supposed to be informed with
`MSG_USER_ADMIT_IND` every time. The user that was removed from the queue is
position `0`.

#### Status

| Value | Meaning |
|-------|---------|
| `1`   | Success |

### Afk Handling

The server is expected to kick AFK players at fixed intervals to prevent them from
taking up server resources unnecessarily.

Each time the player clicks a button in the WizardGraphicalClient, a
`MSG_LOGIN_NOT_AFK` message should be sent to reset the kick interval for the
current connection.

When the interval expires, the server should free all resources, send a
user-facing notification, and close the connection.
