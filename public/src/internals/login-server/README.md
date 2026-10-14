# Login Server

This chapter describes the architecture and functionality of the _Login Server_.

Its purpose is to authenticate players, perform _Game Server_ load balancing,
and manage in-game characters for the account.

The primary data exchange protocol is described in [DML](../dml/protocols.md) as
`LoginMessages.xml`, Service ID `7`. It is recommended to keep this open for
reference while reading through the following pages.
