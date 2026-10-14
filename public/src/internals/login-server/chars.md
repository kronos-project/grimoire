# Character Management

After successful authentication, validation and admission to the game, players
will gain access to the character management scene.

## WizardCharacterCreationInfo

`WizardCharacterCreationInfo` is a
[`PropertyClass`](../object-property/property-classes.md) used to represent
character data throughout early character selection.

As explained in the [ObjectProperty chapter](../object-property/), readers
should use wizwalker to obtain a dump of all types (including this one).

`m_schoolOfFocus` is a [string ID](../string-id.md) of the school's literal name
and `m_nameIndices` is a concatenations of three bytes for each index with the
MSB always being `0`. Everything else is self-explanatory.

It is serialized with no serializer configuration flags (value `0`) and a
property mask of `0x18`.

## Requesting the List

The Character List must be directly requested from the server when initially
entering the game but also when altering it, e.g. creating new characters.

The client sends the empty `MSG_REQUESTCHARACTERLIST` for that.

The server then follows up with `MSG_STARTCHARACTERLIST` holding a
human-readable name of the _Login Server_ instance and the number of
additionally purchased character slots (to the default 6).

Next, `MSG_CHARACTERINFO` will be sent a finite number of times for each
character there is to this account. It carries a serialized
[`WizardCharacterCreationInfo`](#wizardcharactercreationinfo) object.

Finally, the sequence will be terminated with `MSG_CHARACTERLIST` holding an
error code of `0` (empty string id). This message may be sent prematurely if an
error occurs while still trying to stream the Character List.

## Creating new Characters

The client sends `MSG_CREATE_CHARACTER` with the serialized
[`WizardCharacterCreationInfo`](#wizardcharactercreationinfo) object.

The server then responds with `MSG_CREATE_CHARACTER_RESPONSE`, potentially
holding an error or empty [string ID](../string-id.md) on success.

Errors may occur when the creation info object is malformed or the character
limit for the account is exhausted.

> TODO: Brute force string for `1745079928`.

### Creation Logs

The client logs various stages of the character creation in
`MSG_LOGIN_LOG_CHARACTER_CREATION`. The following explains the `Stage`s and
`Parameter`s in use there.

| Stage | Description                | Parameter     |
| ----- | -------------------------- | ------------- |
| `0`   | Prologue                   | `0` (nothing) |
| `1`   | Consulting Book of Secrets | `0` (nothing) |
| `2`   | School chosen arbitrarily  | School ID     |
| `3`   | School assigned by Book    | School ID     |
| `4`   | Default appearance taken   | Template ID   |
| `5`   | Custom appearance chosen   | Template ID   |
| `6`   | Choosing a name            | `0` (nothing) |

#### School ID

| Value | School  |
| ----- | ------- |
| `0`   | Fire    |
| `1`   | Ice     |
| `2`   | Storm   |
| `3`   | Myth    |
| `4`   | Life    |
| `5`   | Death   |
| `6`   | Balance |

## Deleting a Character

The client sends `MSG_DELETE_CHARACTER` with the unique `CharID` of the
character.

The server is expected to respond with `MSG_DELETE_CHARACTER_RESPONSE`,
potentially holding an error or empty [string ID](../string-id.md) on success.

Errors may occur when `CharID` does not exist or references a character that
does not belong to the currently authenticated account.

## Entering the Game

When the player selected its character of choice and hits **Play**, the client
will send `MSG_SELECTCHARACTER` with the selected character ID.

The server will then confirm the selection with `MSG_CHARACTERSELECTED` with IP
and port of the _Game Server_, a random Key that will be sent back for
validation later, player and char IDs, the current zone and position, and a
potential error.

New Wizards start at `WizardCity/Tutorial_Exterior`, with zone ID
`191965934121493239` and location `"START"`.

From this point onwards, the _Login Server_ connection is closed and all client
communication will happen through the _Game Server_.
