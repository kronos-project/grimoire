# Protocols

Protocols are the primary entity of the DML serialization system. Each protocol
is uniquely identified by an ID and groups together several
[messages](./messages.md) together.

One protocol may be described per XML file and the name of the protocol (which
is equal to the specification file name in most cases) may be defined as the
root XML tag:

```xml
<MyAwesomeProtocol>
  <!-- Protocol description here -->
</MyAwesomeProtocol>
```

## Protocol information

Every protocol must have a child XML element named `_ProtocolInfo` assigned to
it. Its record holds the following mandatory message fields:

| Name                | Type | Description                                                |
| ------------------- | ---- | ---------------------------------------------------------- |
| ServiceID           | UBYT | A unique service ID for the protocol service provider      |
| ProtocolType        | STR  | A unique name for the type of protocol being defined       |
| ProtocolVersion     | INT  | The version of the DML system that is used by the protocol |
| ProtocolDescription | STR  | A description of the purpose the protocol serves           |

When serializing protocol messages over the network, the ServiceID identifies
the higher-order protocol of the message to read. As such, implementors will
have to track all accessible `ServiceID`s accordingly for lookup.

For more information on types, records and fields, see the documentation for
[messages](./messages.md).

### Example

```xml
<MyAwesomeProtocol>
  <_ProtocolInfo>
    <RECORD>
      <ServiceID TYPE="UBYT">123</ServiceID>
      <ProtocolType TYPE="STR">AWESOME</ProtocolType>
      <ProtocolVersion TYPE="INT">1</ProtocolVersion>
      <ProtocolDescription TYPE="STR">Super awesome protocol</ProtocolDescription>
    </RECORD>
  </_ProtocolInfo>
</MyAwesomeProtocol>
```
