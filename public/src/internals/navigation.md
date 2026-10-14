# Navigation Maps

The `zonemap.nav` file stores a navigation graph and zone names. Integers and
floats are little-endian. Let `N` be `node_count`, `M` be `link_count`, and `Z`
be `zone_count`.

Offsets start at the beginning of `zonemap.nav`:

| Offset                  | Type                | Field      | Description                   |
| ----------------------- | ------------------- | ---------- | ----------------------------- |
| `0x00`                  | `u16`               | last_id    | Last node ID                  |
| `0x02`                  | `u32`               | node_count | Number of navigation nodes    |
| `0x06`                  | `NavigationNode[N]` | nodes      | Navigation nodes              |
| `0x06 + 14 * N`         | `u32`               | link_count | Number of navigation links    |
| `0x0A + 14 * N`         | `NavigationLink[M]` | links      | Connections between nodes     |
| `0x0A + 14 * N + 4 * M` | `u32`               | zone_count | Number of zone names          |
| `0x0E + 14 * N + 4 * M` | `ZoneName[Z]`       | zone_names | Zone names indexed by node ID |

`NavigationNode` has this layout:

| Offset | Type  | Field | Description  |
| ------ | ----- | ----- | ------------ |
| `0x00` | `f32` | x     | X coordinate |
| `0x04` | `f32` | y     | Y coordinate |
| `0x08` | `f32` | z     | Z coordinate |
| `0x0C` | `u16` | id    | Node ID      |

`NavigationLink` has this layout:

| Offset | Type  | Field  | Description                     |
| ------ | ----- | ------ | ------------------------------- |
| `0x00` | `u16` | first  | ID of the first connected node  |
| `0x02` | `u16` | second | ID of the second connected node |

Each `ZoneName` has this layout. Let `L` be its byte length:

| Offset | Type    | Field  | Description           |
| ------ | ------- | ------ | --------------------- |
| `0x00` | `i32`   | length | Number of UTF-8 bytes |
| `0x04` | `u8[L]` | value  | Zone name             |

A link connects the nodes identified by its `first` and `second` IDs. The
zone-name list is indexed by node ID.

## Compass Markers

Compass marker objects contain a localization text key, a three-component
location, and an object ID. The text key identifies the marker's displayed
label; English compass labels are stored in `WizardCompassLocs.lang` under
`Locale/English` in `Root.wad`.
