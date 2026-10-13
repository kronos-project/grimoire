# Groundwork

This discusses the foundations and prerequirements for a Login Server
implementation without going into detail on concrete functionality yet.

## Client Connectivity

In practical terms, the timeframe of communication between game client and
*Login Server* starts with opening the Patch Client/WizardGraphicalClient
and ends when hitting the **Play** button in character selection.

The Patch Server will source the server IP and port from `PatchConfig.xml`
and forward the information to the WizardGraphicalClient via the `-L` CLI
argument.

> When trying to make the Patch Client connect to a custom Login Server, it
> is mandatory to bypass KingsIsle's *Patch Server* or the `PatchConfig.xml`
> change will be reverted before being able to make it into the game.

## Secondary Message Protocols

The Login Server fundamentally needs to support some basic functionality
aside from its primary `LoginMessages` protocol.

### Pings

Every so often, the client may send `MSG_PING` from `BaseMessages.xml`
(Service ID `1`) to the server to check its responsiveness. The server
should respond with `MSG_PING_RSP` from the same protocol upon receiving.
Both these messages don't carry any data payloads.

### Client Disconnection

When clients shut down gracefully, they send `MSG_CLIENT_DISCONNECT` from
`GameMessages.xml` (Service ID `5`). In such an event, the server should
close the open client connection and free up occupied resources.

Note that a variety of problems on the user end can summon unexpected
connection aborts. The implementation should still detect these events and
free up its resources accordingly.

## Client Sessions

When a client initially connects to the *Login Server*, a new [session](../protocol/sessions.md)
must be established before being able to process any `LoginMessages`.

A server implementation should cache **Session ID** and **Session Offer timestamp**
per client as they will be needed later.
