# Data Management Layer

The "Data Management Layer", often referred to as "DML", is a proprietary data
serialization system developed by KingsIsle for use in their games. At its core,
it follows a
[remote procedure call](https://en.wikipedia.org/wiki/Remote_procedure_call)
model as it is primarily used for data exchange over the network which triggers
the execution of designated handlers based on message types.

Protocols and data messages are described by a custom specification format in
[XML](https://de.wikipedia.org/wiki/Extensible_Markup_Language).
Correspondingly, such specification files are often seen as regular
`*Messages.xml` files. Their structure and composition will be explained on the
fly when discussing protocols and messages.
