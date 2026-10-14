# Graphical Client Arguments

`WizardGraphicalClient` will happily spit out a list of CLI arguments if passed
`-?` but this is outdated.

This page attempts to correctly document a list of available arguments.

| Argument               | Usage                         | Description                                |
| ---------------------- | ----------------------------- | ------------------------------------------ |
| `-?`                   | `-?`                          | Print outdated arguments list              |
| `-A`                   | `-A [STRING]`                 | Locale                                     |
| `-CS`                  | `-CS`                         | Dumps the Client Signature                 |
| `-C`                   | `-C [STRING]`                 | Character name/ID                          |
| `-D`                   | `-D [PATH]`                   | Path to data root dir                      |
| `-EF_OVERFLOW`         | `-EF_OVERFLOW`                | Enable overflow detection                  |
| `-EF_UNDERFLOW`        | `-EF_UNDERFLOW`               | Enable underflow detection                 |
| `-G`                   | `-G [PATH]`                   | Path to log file                           |
| `-HD`                  | `-HD`                         | Enable heap debugging                      |
| `-HS`                  | `-HS`                         | Enable heap server                         |
| `-IgnoreMissingParams` | `-IgnoreMissingParams`        | Ignore missing parameters                  |
| `-L`                   | `-L [HOST] [PORT]`            | Login server to use                        |
| `-O`                   | `-O`                          | Whether to log all resource requests       |
| `-PT`                  | `-PT [INT]`                   | "PatchClientPatchTime"                     |
| `-ST`                  | `-ST`                         | "-Steam Required"                          |
| `-T`                   | `-T [STRING]`                 | "Test Local Zone"                          |
| `-UN`                  | `-UN [0/1]`                   | Whether to force unique character names    |
| `-U`                   | `-U ..[UID] [CK2] [USERNAME]` | User credentials passed in by patch client |
| `-V`                   | `-V`                          | "DoServerSelection"                        |
| `-X`                   | `-X`                          | "Dump Classes to Filename"                 |
| `-c`                   | `-c`                          | Checks compression for VolumeWAD files     |
| `-u`                   | `-u [STRING],[STRING],...`    | Wildcards for uncompressed VolumeWAD files |
