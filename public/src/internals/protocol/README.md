# Network Protocol

The game makes use of a custom application-level protocol. The reviewed sources
cover TCP; UDP behaviour is a [verification gap](../../verification.md).

This section goes into great detail of said protocol regarding its creation and
management of user sessions and the custom framing protocol it uses to exchange
data.

> **Note:** Readers should be familiar with the
> [Data Management Layer](../dml/index.html) system prior to working through
> this.
