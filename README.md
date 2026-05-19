# RC Protocol Documentation

Welcome to the RC Protocol documentation. This documentation explains how the Remote Control (RC) protocol works for Graal Online servers.

## Documentation Files

### [Listserver Protocol](#/listserver)
Explains how to connect to the listserver to get a list of available game servers. This is the first step - you need to know what servers exist before you can connect to them.

**Key topics:**
- Server discovery
- Authentication
- The scrambler encryption algorithm
- Cross-server messaging

### [NC Protocol](#/nc)
Explains the NPC Control (NC) protocol. This is a separate connection that handles all script-related operations: weapons, classes, and NPCs.

**Key topics:**
- Connecting to the NC server
- Weapon and class script management
- NPC database operations
- Script encoding formats (WString vs CommaText)

### [RC Protocol](#/rc)
Explains the main Remote Control (RC) protocol. This is the primary connection to a game server for managing players, files, accounts, and server settings.

**Key topics:**
- Packet encoding (GByte, GShort, GInt, GString)
- Player properties system
- File browser operations
- Account and player management
- Chat and messaging

## How to Use This Documentation

Start with **Listserver Protocol** to understand how to discover servers. Then read **RC Protocol** to understand the main protocol. Finally, read **NC Protocol** if you need to work with scripts.

Each document is written to be understandable even if you're new to the protocol. They focus on explaining *how* things work logically, not on showing code implementations.

## Common Concepts

All three protocols share some common concepts:

- **GByte encoding:** Numbers are stored as readable characters (add 32 to encode, subtract 32 to decode)
- **Packet structure:** Every packet has an ID, payload, and end marker (0x0A)
- **Compression:** Large packets are compressed with ZLIB
- **Encryption:** Some packets use a scrambler algorithm for encryption

See **RC Protocol** for detailed explanations of these encoding schemes.

## Troubleshooting

Each document has a "Common Problems" section at the end. If you're having issues, check there first. The most common problems are:

- Using wrong port numbers
- Getting encoding wrong (especially GInt5 vs 8 raw bytes)
- Forgetting to strip trailing bytes from wrapped packets
- Using wrong player IDs for NC server requests

