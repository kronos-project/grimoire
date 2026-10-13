# Handshake

## Overview

The detailed additions below are tied to Merlin's reviewed 2023 fixtures.
The current live game's key selection, rotation, and negotiation rules
have not been independently verified; see the
[evidence record](../../verification.md#encryption).

Wizard101 uses AES-GCM, an authenticated symmetric cipher, for encrypting traffic. The AES key and IV are generated clientside and exchanged with the server by encrypting them using public key cryptography, or more specifically RSA.

## Exchange

Five RSA public keys are embedded in each `WizardGraphicalClient.exe` binary. These are rotated semi-regularly, usually when a new version is released. When a handshake is initiated, the server indicates one of the keys to use at random and the client uses it to wrap an AES key and IV used for subsequent immediate communication, similar in principle to TLS.

The embedded keys have multiple layers of obfuscation which can make them tricky to locate. This is described in more detail [here](./obfuscation.md#pubkey-blob).

After the handshake, a new IV is sent periodically, and the GCM context is reinitialised (reset.) Every 4096 (0x1000) blocks of clientbound traffic, the server transmits a 16 byte GCM authentication tag for the preceding traffic, and 16 byte IV for the client to use for decrypting traffic that follows. Similarly, every 256 (0x100) blocks of serverbound traffic, the client transmits an authentication tag and new IV. The key remains static for the duration of the session. Note that an AES block is 16 bytes.

<div class="warning">
This usage of AES-GCM is insecure and is not supported by all cryptography libraries. Data decrypted with AES-GCM should only be used if it can be authenticated. However, Wizard101 ignores this and uses the data anyway, only authenticating it periodically when a tag is received.

Most cryptography libraries mandate that an authentication tag follows the ciphertext for succcessful decryption and do not support encryption without immediate tagging. Wizard101 uses Crypto++ which appears to have neither of these requirements.
</div>

## Handshake

The handshake begins with a SessionOffer packet from the server. The signed message is followed by 256 bytes of an RSA signature.

| Offset | Type    | Description                                                     |
|--------|---------|-----------------------------------------------------------------|
| 0x0    | uint16  | The proposed session ID                                         |
| 0x2    | int32   | The high 4 bytes of the UNIX timestamp the message was sent at  |
| 0x6    | int32   | The low 4 bytes of the UNIX timestamp the message was sent at   |
| 0xA    | uint32  | The milliseconds into the second this message was sent at       |
| 0xE    | uint32  | Length prefix for the following SignedMessage                   |
| 0x12   | uint8[] | A message suffixed with an RSA signature for the message        |
| -      | uint8   | Reserved; always zero                                           |

The signed portion starts as follows, with offsets relative to the
start of the length-prefixed blob. Merlin gives more detail inside the
challenge region than the original notes:

| Offset | Type   | Name            | Description                                       |
|--------|--------|-----------------|---------------------------------------------------|
| 0x0    | byte   | Flags           |                                                   |
| 0x1    | byte   | KeySlot         | Key slot selected by the server                   |
| 0x2    | byte   | KeyMask         | Key XOR mask used for decoding the RSA public key |
| 0x3    | byte   | ChallengeLength | Length of region beginning at `0x4`               |
| 0x4    | uint16 | FNVOffset       | Offset into the raw keyring blob                  |
| 0x6    | uint16 | FNVLength       | Length of keyring bytes to hash                   |
| 0x8    | byte   | ChallengeType   | Challenge algorithm selector                      |
| 0x9    | byte[] | Challenge       | Remaining `ChallengeLength - 5` challenge bytes   |
| -      | uint32 | Nonce           | One-time value to be echoed back to the server    |

The nonce is at `0x4 + ChallengeLength`. After it come 256 signature
bytes in the reviewed capture. The signature covers the signed portion
from `Flags` through `Nonce`, excluding the signature itself, length
prefix, and outer session fields. Merlin verifies it with
RSA PKCS#1 v1.5 and SHA-1. The key-slot selector addresses one of five
keys, indexed `0` through `4`.

Merlin's offer fixture has `ChallengeLength = 17`, `FNVOffset = 1007`,
`FNVLength = 473`, and a nonce at offset `21`. It supports the region
layout above. The flag bits and challenge algorithm are still unknown;
the reviewed proxy forwards the client's challenge response.

The client will then decode the key slot selected by the server and use the resulting public key to verify the signature. It will also compute the challenge and assemble a SessionAccept packet to return to the server:

| Offset | Type    | Description                                                    |
|--------|---------|----------------------------------------------------------------|
| 0x0    | uint16  | Reserved; always zero                                          |
| 0x2    | int32   | The high 4 bytes of the UNIX timestamp the message was sent at |
| 0x6    | int32   | The low 4 bytes of the UNIX timestamp the message was sent at  |
| 0xA    | uint32  | The milliseconds into the second the message was sent at       |
| 0xE    | uint16  | The proposed session ID from SessionOffer                      |
| 0x10   | uint32  | Length prefix for the following EncryptedMessage               |
| 0x14   | uint8   | A byte denoting the asymmetric encryption algorithm            |
| 0x15   | uint8[] | An encrypted string which contains the session key and IV      |
| -      | uint8   | Trailing zero, outside the EncryptedMessage length             |

| Algorithm            | Value |
|----------------------|-------|
| RSA/OAEP-MGF1(SHA-1) | 0x1   |

RSA encryption uses OAEP with SHA-1, MGF1(SHA-1), and an empty label.
The length includes the algorithm byte and RSA ciphertext. With a
2048-bit key it is `257`, followed by the separate trailing zero.

### Encrypted confirmation plaintext

Merlin's reviewed confirmation fixture establishes this layout, before
RSA encryption. All integers are little-endian.

| Offset | Type | Meaning |
|--------|------|---------|
| 0x0 | uint8 | Flags; meanings unverified |
| 0x1 | uint32 | FNV-1a-32 result for the offered keyring region |
| 0x5 | uint8[4] | Challenge response in the reviewed fixture |
| 0x9 | uint32 | Echo of the offer nonce |
| 0xD | uint32 | Client epoch seconds |
| 0x11 | uint8[16] | AES-128 key |
| 0x21 | uint8[16] | Initial GCM IV |

The fixture is 49 bytes. Four-byte challenge responses are an assumption
in Merlin's parser, not an established rule for every challenge type.
FNV-1a-32 starts at `0x811C9DC5`; for each selected raw keyring byte,
XOR the byte and multiply by `0x01000193`, wrapping to 32 bits. Hash the
raw blob region, rather than the decoded ASN.1 key.

### Encrypted stream boundaries

The offer and accept are exchanged as plaintext frames. Merlin switches
to AES-GCM after the accept, wrapping the entire subsequent framed byte
stream, including frame headers. There are independent cipher states and
byte counts in each direction, initially using the exchanged key and IV.

The periodic boundaries are `65536` ciphertext bytes towards the client
and `4096` towards the server. At each boundary, a 16-byte tag is followed
by a 16-byte replacement IV. These 32 bytes belong to the encryption
layer, have no `0xF00D` header, and do not count towards the next ciphertext
interval. A boundary may interrupt a frame or fall between frames; remove
the tag/IV bytes before passing decrypted bytes to the frame parser.

The reviewed Merlin stream code does not verify received tags and has
incomplete boundary accounting. It supports the intended record layout,
but cannot validate tag correctness or arbitrary stream splits. Those
remain [verification gaps](../../verification.md#encryption).


## Unencrypted mode

The original notes describe the client opting out by sending a single
zero algorithm byte as `EncryptedMessage`, with a length of `1`. This is
separate from the trailing frame zero. Restrictions on which login
messages KI accepts in this mode have not been verified for a current
server revision.

The original notes also describe a server opting out by omitting the
signed message. Whether this means a zero-length extended blob or the
older layout without a length prefix is not verified. The reviewed
repositories do not establish whether a current client accepts this
downgrade.
