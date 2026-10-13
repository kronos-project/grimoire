# Character Management

After successful authentication, validation and admission
to the game, players will gain access to the
character management scene.

## WizardCharacterCreationInfo

`WizardCharacterCreationInfo` is a [`PropertyClass`](../object-property/property-classes.md)
used to represent character data throughout early character selection.

As explained in the [ObjectProperty chapter](../object-property/), readers should
use wizwalker to obtain a dump of all types (including this one).

`m_schoolOfFocus` is a [string ID](../string-id.md) of the school's literal name
and `m_nameIndices` is a `uint32` containing three one-byte name indices
and a zero high byte. The order of the three name components still needs
confirmation against a client revision.

It is serialized with no serializer configuration flags (value `0`) and a property mask
of `0x18`, in shallow mode in the reviewed character creation handler.
This selects properties carrying both network transmission flags. Nested
appearance and equipment objects require matching type definitions too.

### Message identifiers

The reviewed LoginMessages schema uses different XML tags and `_MsgName`
aliases for several character messages. The names below denote the same
wire messages; their service ID is `7`.

| Order | XML tag | `_MsgName` alias |
|-------|---------|-----------------|
| 1 | `MSG_CHARACTER_INFO` | `MSG_CHARACTERINFO` |
| 2 | `MSG_END_CHARACTER_LIST` | `MSG_CHARACTERLIST` |
| 4 | `MSG_CREATE_CHARACTER` | `MSG_CREATECHARACTER` |
| 5 | `MSG_CREATE_CHARACTER_RESPONSE` | `MSG_CREATECHARACTERRESPONSE` |
| 6 | `MSG_DELETE_CHARACTER` | `MSG_DELETECHARACTER` |
| 7 | `MSG_DELETE_CHARACTER_RESPONSE` | `MSG_DELETECHARACTERRESPONSE` |
| 8 | `MSG_REQUEST_CHARACTER_LIST` | `MSG_REQUESTCHARACTERLIST` |
| 12 | `MSG_START_CHARACTER_LIST` | `MSG_STARTCHARACTERLIST` |
| 28 | `MSG_LOGIN_LOG_CHARACTER_CREATION` | `MSG_LOGINLOGCHARACTERCREATION` |

These orders are a reference snapshot, not a guarantee for other client
revisions. Use the XML bundled with the target client.

## Requesting the List

The Character List must be directly requested from
the server when initially entering the game but also
when altering it, e.g. creating new characters.

The client sends the empty `MSG_REQUESTCHARACTERLIST`
for that.

The server then follows up with `MSG_STARTCHARACTERLIST` holding a human-readable
name of the *Login Server* instance and the number
of additionally purchased character slots (to the default 6).

Next, `MSG_CHARACTERINFO` will be sent a finite number of times for each character
there is to this account. It carries a serialized [`WizardCharacterCreationInfo`](#wizardcharactercreationinfo)
object.

Finally, the sequence will be terminated with `MSG_CHARACTERLIST` holding an error
code of `0` (empty string id). This message may be sent prematurely if an error
occurs while still trying to stream the Character List.

## Creating new Characters

The client sends `MSG_CREATE_CHARACTER` with the serialized
[`WizardCharacterCreationInfo`](#wizardcharactercreationinfo) object.

The server then responds with `MSG_CREATE_CHARACTER_RESPONSE`, potentially holding
an error or empty [string ID](../string-id.md) on success.

Errors may occur when the creation info object is malformed or the character
limit for the account is exhausted.

The reviewed handler uses the string ID of `TooManyChars` for an exhausted
character limit. Its value is `255337254`. The original notes' separate
error value `1745079928` has not been mapped to a string.

### Creation Logs

The client logs various stages of the character creation in `MSG_LOGIN_LOG_CHARACTER_CREATION`.
The following explains the `Stage`s and `Parameter`s in use there.

| Stage | Description                | Parameter     |
|-------|----------------------------|---------------|
| `0`   | Prologue                   | `0` (nothing) |
| `1`   | Consulting Book of Secrets | `0` (nothing) |
| `2`   | School chosen arbitrarily  | School ID     |
| `3`   | School assigned by Book    | School ID     |
| `4`   | Default appearance taken   | Template ID   |
| `5`   | Custom appearance chosen   | Template ID   |
| `6`   | Choosing a name            | `0` (nothing) |

#### School ID

| Value | School  |
|-------|---------|
| `0`   | Fire    |
| `1`   | Ice     |
| `2`   | Storm   |
| `3`   | Myth    |
| `4`   | Life    |
| `5`   | Death   |
| `6`   | Balance |

## Deleting a Character

The client sends `MSG_DELETE_CHARACTER` with the unique `CharID` of the character.

The server is expected to respond with `MSG_DELETE_CHARACTER_RESPONSE`, potentially
holding an error or empty [string ID](../string-id.md) on success.

Errors may occur when `CharID` does not exist or references a character that does
not belong to the currently authenticated account.

The reviewed handler returns the string ID of `DeletionFailed` in this
case. Both creation and deletion responses carry an `INT` error field;
the character-list terminator instead uses `UINT`.

## Entering the Game

When the player selected its character of choice and hits **Play**, the client
will send `MSG_SELECTCHARACTER` with the selected character ID.

The server will then confirm the selection with `MSG_CHARACTERSELECTED` with IP
and port of the *Game Server*, a random Key that will be sent back for validation
later, player and char IDs, the current zone and position, and a potential error.

The original notes record `WizardCity/Tutorial_Exterior`, zone ID
`191965934121493239`, and location `"START"` for new Wizards. The reviewed
repositories do not verify this initial zone or the selection exchange.

The existing [magic numbers notes](../dml/magic-numbers.md) describe
`PrepPhase = 1` as an incomplete preparation response and `PrepPhase = 0`
as a completed response with usable connection details. Do not enter the
game on the incomplete response. These values still need a versioned
capture for independent confirmation.

The next stage is a connection to the indicated *Game Server*. Exact login
connection closure timing, the game-server validation message, and how the
selection key is checked remain [verification gaps](../../verification.md).
