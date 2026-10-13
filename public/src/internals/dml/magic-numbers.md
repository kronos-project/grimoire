# Magic Numbers

Many messages use magic numbers, or numbers that don't describe
anything about what they mean, to relay information. In most cases, these magic numbers are actually the KI generic hash of a string. While this can be frustrating, they're not truly magic given they're self describing. Below are the currently understood magic numbers that have no hash correspondence and just *are*.

## DML

### MSG_USER_ADMIT_IND

Status
| Value | Meaning |
| ----- | ------- |
| 1     | Servers are ready to admit the client into the main game |
| 2     | Servers are busy and the client must queue before entry |


### MSG_CHARACTERSELECTED

PrepPhase
| Value | Meaning |
| ----- | ------- |
| 0     | Server admits the client into the game. Provided fields are valid. |
| 1     | Server has accepted the client into the game, but has not yet provided all necessary information. Wait for another message with updated fields. |

These values are retained from the original research notes. The reviewed
login handler confirms admission status `1` with queue position `0`.
Busy status `2` and both `PrepPhase` values require versioned captures;
the reviewed repositories do not implement those paths completely.
