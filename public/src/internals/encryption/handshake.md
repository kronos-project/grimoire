# Handshake

## Overview

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
| 0x0    | uint16  | The proposed to agree on                                        |
| 0x2    | int32   | The high 4 bytes of the UNIX timestamp the message was sent at  |
| 0x6    | int32   | The low 4 bytes of the UNIX timestamp the message was sent at   |
| 0xA    | uint32  | The milliseconds into the second this message was sent at       |
| 0xE    | uint32  | Length prefix for the following SignedMessage                   |
| 0x12   | uint8[] | A message suffixed with an RSA signature for the message        |
| -      | uint8   | Reserved; always zero                                           |

The structure of the signed message is as follows:

| Offset | Type   | Name            | Description                                       |
|--------|--------|-----------------|---------------------------------------------------|
| 0x0    | byte   | Flags           |                                                   |
| 0x1    | byte   | KeySlot         | Key slot selected by the server                   |
| 0x2    | byte   | KeyMask         | Key XOR mask used for decoding the RSA public key |
| 0x3    | byte   | ChallengeLength | Length of challenge bytes                         |
| 0x4    | byte[] | Challenge       |                                                   |
| -      | uint32 | Nonce           | One-time value to be echoed back to the server    |

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

| Algorithm            | Value |
|----------------------|-------|
| RSA/OAEP-MGF1(SHA-1) | 0x1   |


## Unencrypted mode

The client can opt-out of encryption by setting the serverbound EncryptedMessage field to a single null byte in the SessionAccept message. However, the live server will only respond to a subset of the LOGIN protocol in this mode. For example, a request for the character list will be ignored. It is believed that this mode exists for backwards compatibility in the launcher.

The server can also opt-out of encryption by omitting the SignedMessage field. This is a vulnerability because it makes man-in-the-middle attacks possible in the handshake phase by terminating the server's encryption and forwarding unencrypted packets to the client. Kingsisle have unfortunately relied on security-by-obscurity and overlooked what is a fairly amateur flaw. Overall, the risk to players is low so this has not been reported.