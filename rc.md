# Remote Control Protocol

## What is RC?

RC stands for "Remote Control". It's the main connection to a game server that lets you control and manage the server. Think of it like a remote control for your TV - but instead of changing channels, you're managing players, files, accounts, and server settings.

RC is different from NC (NPC Control). RC handles:
- Player management (warping, kicking, viewing stats)
- File browser (uploading/downloading files)
- Account management (editing accounts, banning)
- Server settings (flags, options)
- Chat and messaging

NC handles:
- Scripts (weapons, classes, NPCs)
- NPC database operations

## The Connection Process

Here's how you connect to a game server:

```
1. Connect to the server's IP and port
2. Send a login packet with your account and password
3. Server checks if you're allowed to connect
4. Server sends back a "signature" packet (packet 25 - PLO_SIGNATURE) - this means you're authenticated
5. You're now connected and can send commands
```

**Important:** You get the server's IP and port from the listserver (see [Listserver Protocol](#/listserver)).

### Login Packet Format

The login packet is sent as packet type 6 (PLI_TOALL) before encryption is enabled:

```
"v" + version_string + account (get1PlusTextNetString) + password (get1PlusTextNetString) + os_info
```

- `version_string`: `"GSERV025"` (note the leading `"v"` before it makes the full header `"vGSERV025"`)
- Account and password use `get1PlusTextNetString` encoding (see Helper Functions)
- `os_info`: `"win,<pcid>,<pcid>,\"\""` or `"mac,<pcid>,<pcid>,\"\""`

**After sending the login packet, the encryption key is immediately set to `0x56`.** All subsequent packets are encrypted with this key. The server does not send the key in any packet — `0x56` is the fixed well-known RC client key.

**PLO_SIGNATURE (packet 25):** When received, this means authentication was successful. The server sends one byte of payload: the value `73` (encoded as GByte: `0x49`), which historically told the client that more than eight players may be connected. The presence of this packet is the authentication signal; the payload byte can be ignored by RC clients.

## Packet Encoding Basics

Before we talk about what packets do, we need to understand how data is encoded. Graal uses special encoding to make sure all data is readable text (no weird control characters).

### GByte

A GByte is a single number from 0 to 255, but it's stored as a readable character.

**How it works:**
- To encode: Add 32 to your number
- To decode: Subtract 32 from the byte you read

**Example:**
- The number 5 becomes the byte 37 (5 + 32)
- The byte 37 becomes the number 5 (37 - 32)

**Why?** This ensures the byte value is always a printable character (space and above in ASCII).

**Hex Example:**
```
Number: 5
Encoded: 0x25 (37 in decimal, 0x20 + 0x05)
ASCII: '%' (readable character)

Number: 100
Encoded: 0x84 (132 in decimal, 0x20 + 0x64)
ASCII: '\x84' (still printable)
```

### GShort

A GShort is a number from 0 to 65535, stored in 2 bytes.

**How it works:**
- Split your number into two 7-bit parts
- Encode each part as a GByte (add 32)
- First byte = high 7 bits, second byte = low 7 bits

**Example:**
- Number 1000
- High part: 1000 >> 7 = 7
- Low part: 1000 & 0x7F = 104
- Encoded: (7 + 32), (104 + 32) = 39, 136

**To decode:**
- High = (byte1 - 32) << 7
- Low = (byte2 - 32)
- Value = High + Low

**Hex Example:**
```
Number: 1000 (0x03E8)
High: 7 (0x07)
Low: 104 (0x68)
Encoded: 0x27 0x88 (39, 136)
         ^^   ^^
         7+32 104+32

Decode: (0x27 - 0x20) << 7 + (0x88 - 0x20)
      = 7 << 7 + 104
      = 896 + 104
      = 1000 ✓
```

### GInt

A GInt is a 4-byte number (up to about 4 billion). It's stored as 4 GBytes, each holding 7 bits.

**How it works:**
- Split into 4 parts of 7 bits each
- Encode each as a GByte

**To decode:**
- Read 4 bytes, subtract 32 from each
- Combine: (byte1-32) << 21 + (byte2-32) << 14 + (byte3-32) << 7 + (byte4-32)

### GString

A GString is text with a length prefix.

**Format:**
- First byte: Length of the string (as a GByte, so add 32)
- Rest: The actual string bytes

**Example:**
- String "hello" (length 5)
- Encoded: (5 + 32) = 37, then 'h', 'e', 'l', 'l', 'o'
- So: 37, 104, 101, 108, 108, 111

**Hex Example:**
```
String: "hello"
Length: 5
Encoded: 0x25 0x68 0x65 0x6C 0x6C 0x6F
         ^^   ^^   ^^   ^^   ^^   ^^
         5+32 'h'  'e'  'l'  'l'  'o'
         (37) (104)(101)(108)(108)(111)
```

### GInt5 (5-byte number)

Some things need really big numbers (like file sizes or timestamps). These use 5 bytes, each holding 7 bits.

**How it works:**
- Just like GInt, but with 5 bytes instead of 4
- Gives you 35 bits total (about 34 billion)

**To encode:**
```
b[0] = ((value >> 28) & 0x7F) + 32
b[1] = ((value >> 21) & 0x7F) + 32
b[2] = ((value >> 14) & 0x7F) + 32
b[3] = ((value >> 7) & 0x7F) + 32
b[4] = (value & 0x7F) + 32
```

**To decode:**
```
value = ((b[0]-32) << 28) | ((b[1]-32) << 21) | ((b[2]-32) << 14) | ((b[3]-32) << 7) | (b[4]-32)
```

**Common mistake:** People think this is 8 raw bytes. It's not! It's 5 GByte-encoded values.

## Packet Structure

Every packet has this structure:

```
[Packet ID: GByte] [Payload: variable] [End marker: 0x0A]
```

**Packet ID:** What kind of packet this is (the number + 32, so it's a readable character)

**Payload:** The actual data (depends on the packet type)

**End marker:** Always 0x0A (newline character) - this tells the server "packet is done"

**Example:**
- Packet 25 (PLO_SIGNATURE - signature/authentication)
- Encoded: (25 + 32) = 57, then payload, then 0x0A

**Complete Packet Example:**
```
Packet 25 (PLO_SIGNATURE) with payload "abc123":

Byte 0: 0x39 (25 + 32 = 57)
Bytes 1-6: 0x61 0x62 0x63 0x31 0x32 0x33 ("abc123")
Byte 7: 0x0A (end marker)

Complete: 39 61 62 63 31 32 33 0A
          ^^ ^^^^^^^^^^^^^^^^^^^^ ^^
          ID payload              end
```

## Compression and Encryption

Packets can be compressed and encrypted before being sent over the network.

**Compression:**
- If the packet is small (less than 40 bytes), it's sent as-is
- If it's larger, it's compressed with ZLIB

**Encryption:**
- Uses the same scrambler algorithm as the listserver
- The encryption key is `0x56` (fixed, set immediately after sending the login packet)
- Small packets (format byte 0x02): encrypt the first 48 bytes (12 groups of 4)
- Large packets (format byte 0x04): encrypt the first 16 bytes (4 groups of 4)

**Format byte:**
- Small packet: 0x02 (means "plain text, maybe encrypted")
- Large packet: 0x04 (means "ZLIB compressed, maybe encrypted")

**Wire format:**
```
[2-byte big-endian length] [format byte] [encrypted/compressed data]
```

The length field counts the format byte plus the data. The format byte is only present when encryption is active; without encryption the 2-byte length is followed directly by the compressed-or-plain payload.

## Player Properties

Players have lots of properties - their name, position, health, items, etc. These are sent in packet 8 (PLO_OTHERPLPROPS) or packet 72 (PLO_RC_PLAYERPROPSGET).

### Property System

Each property has:
- An ID number (0-86+)
- A data type (byte, short, int, string, special)
- The actual value

**How to read properties:**
1. Read the property ID
2. Look up what type that property is
3. Read that many bytes/data in that format
4. Repeat until you run out of data

**Important:** Different properties have different types! You can't just read everything as strings.

**Example: Reading Player Properties**

Given packet data:
```
Offset: 0x00 0x20 0x2B 0x54 0x65 0x73 0x74 0x50 0x6C 0x61 0x79 0x65 0x72 0x01 0x28 0x02 0x28
         ^^   ^^   ^^   ^^   ^^   ^^   ^^   ^^   ^^   ^^   ^^   ^^   ^^   ^^   ^^   ^^   ^^
         ID=0 len=11 "TestPlayer"     ID=1 val=20 ID=2 val=20
```

Reading process:
```
1. Read property ID: 0x20 - 0x20 = 0 (NICKNAME, GString)
   - Read length: 0x2B - 0x20 = 11
   - Read string: "TestPlayer" (11 bytes)
   
2. Read property ID: 0x01 (MAXPOWER, GByte)
   - Read value: 0x28 - 0x20 = 8 (max hearts = 8)
   
3. Read property ID: 0x02 (CURPOWER, GByte)
   - Read value: 0x28 - 0x20 = 8 (current hearts = 8/2 = 4)
```

Result: Player named "TestPlayer" with 4/8 hearts.

### Complete Player Property Table

The property IDs below are taken directly from the server's `PLPROP_*` enum in `TAccount.h` and the encoding logic in `TPlayerProps.cpp`. Earlier documentation that listed different IDs for X, Y, BOMBSCOUNT, GLOVEPOWER, etc. was incorrect.

| ID | Enum Name | Type | Notes |
|----|-----------|------|-------|
| 0 | PLPROP_NICKNAME | GString | Player's display name |
| 1 | PLPROP_MAXPOWER | GByte | Maximum hearts (raw value = hearts × 1) |
| 2 | PLPROP_CURPOWER | GByte | Current power; divide by 2.0 for actual hearts |
| 3 | PLPROP_RUPEESCOUNT | GInt (4 bytes, 7-bit each) | Rupees/currency; server encodes as `>> (int)gralatc` |
| 4 | PLPROP_ARROWSCOUNT | GByte | Arrow count |
| 5 | PLPROP_BOMBSCOUNT | GByte | Bomb count |
| 6 | PLPROP_GLOVEPOWER | GByte | Glove power level (0–3) |
| 7 | PLPROP_BOMBPOWER | GByte | Bomb power level (0–3) |
| 8 | PLPROP_SWORDPOWER | special | See sword encoding below |
| 9 | PLPROP_SHIELDPOWER | special | See shield encoding below |
| 10 | PLPROP_GANI | GString | Current GANI animation name (or bow image in pre-2.x clients) |
| 11 | PLPROP_HEADGIF | special | See head encoding below |
| 12 | PLPROP_CURCHAT | GString | Current chat message |
| 13 | PLPROP_COLORS | 5 × GByte | 5 raw GByte values: skin, coat, sleeves, shoes, belt |
| 14 | PLPROP_ID | GShort | Player's numeric ID (2 GBytes) |
| 15 | PLPROP_X | GByte | X tile position × 2 |
| 16 | PLPROP_Y | GByte | Y tile position × 2 |
| 17 | PLPROP_SPRITE | GByte | Direction sprite index |
| 18 | PLPROP_STATUS | GByte | Status flags bitmask (see below) |
| 19 | PLPROP_CARRYSPRITE | GByte | Carry sprite index |
| 20 | PLPROP_CURLEVEL | GString | Current level name. For RC clients the server always sends `" "` (a single space). |
| 21 | PLPROP_HORSEGIF | GString | Horse image name |
| 22 | PLPROP_HORSEBUSHES | GByte | Horse bushes count |
| 23 | PLPROP_EFFECTCOLORS | GByte | Effect colors (server always encodes 0) |
| 24 | PLPROP_CARRYNPC | GInt (4 bytes) | NPC ID being carried |
| 25 | PLPROP_APCOUNTER | GShort | AP counter + 1 |
| 26 | PLPROP_MAGICPOINTS | GByte | Magic/MP points |
| 27 | PLPROP_KILLSCOUNT | GInt (4 bytes) | Kill count |
| 28 | PLPROP_DEATHSCOUNT | GInt (4 bytes) | Death count |
| 29 | PLPROP_ONLINESECS | GInt (4 bytes) | Total online seconds |
| 30 | PLPROP_IPADDR | GInt5 (5 bytes) | IP address as 32-bit value encoded in 5 × 7-bit bytes |
| 31 | PLPROP_UDPPORT | GInt (4 bytes) | UDP port |
| 32 | PLPROP_ALIGNMENT | GByte | Alignment / AP value |
| 33 | PLPROP_ADDITFLAGS | GByte | Additional flags bitmask |
| 34 | PLPROP_ACCOUNTNAME | GString | Player's account name |
| 35 | PLPROP_BODYIMG | GString | Body image name |
| 36 | PLPROP_RATING | GInt (4 bytes) | Spar rating packed as `((rating & 0xFFF) << 9) \| (deviation & 0x1FF)` |
| 37 | PLPROP_GATTRIB1 | GString | Generic attribute 1 |
| 38 | PLPROP_GATTRIB2 | GString | Generic attribute 2 |
| 39 | PLPROP_GATTRIB3 | GString | Generic attribute 3 |
| 40 | PLPROP_GATTRIB4 | GString | Generic attribute 4 |
| 41 | PLPROP_GATTRIB5 | GString | Generic attribute 5 |
| 42 | PLPROP_ATTACHNPC | special | GByte type (always 0) + GInt NPC ID |
| 43 | PLPROP_GMAPLEVELX | GByte | GMAP level X position |
| 44 | PLPROP_GMAPLEVELY | GByte | GMAP level Y position |
| 45 | PLPROP_Z | GByte | Z position encoded as `(z + 0.5) + 50` |
| 46 | PLPROP_GATTRIB6 | GString | Generic attribute 6 |
| 47 | PLPROP_GATTRIB7 | GString | Generic attribute 7 |
| 48 | PLPROP_GATTRIB8 | GString | Generic attribute 8 |
| 49 | PLPROP_GATTRIB9 | GString | Generic attribute 9 |
| 50 | PLPROP_JOINLEAVELVL | GByte | Level join/leave flag (1 = joining, 0 = leaving) |
| 51 | PLPROP_PCONNECTED | (empty) | Connected flag — server writes nothing; skipped |
| 52 | PLPROP_PLANGUAGE | GString | Client language string (e.g. "English") |
| 53 | PLPROP_PSTATUSMSG | GByte | Status message index |
| 54 | PLPROP_GATTRIB10 | GString | Generic attribute 10 |
| 55 | PLPROP_GATTRIB11 | GString | Generic attribute 11 |
| 56 | PLPROP_GATTRIB12 | GString | Generic attribute 12 |
| 57 | PLPROP_GATTRIB13 | GString | Generic attribute 13 |
| 58 | PLPROP_GATTRIB14 | GString | Generic attribute 14 |
| 59 | PLPROP_GATTRIB15 | GString | Generic attribute 15 |
| 60 | PLPROP_GATTRIB16 | GString | Generic attribute 16 |
| 61 | PLPROP_GATTRIB17 | GString | Generic attribute 17 |
| 62 | PLPROP_GATTRIB18 | GString | Generic attribute 18 |
| 63 | PLPROP_GATTRIB19 | GString | Generic attribute 19 |
| 64 | PLPROP_GATTRIB20 | GString | Generic attribute 20 |
| 65 | PLPROP_GATTRIB21 | GString | Generic attribute 21 |
| 66 | PLPROP_GATTRIB22 | GString | Generic attribute 22 |
| 67 | PLPROP_GATTRIB23 | GString | Generic attribute 23 |
| 68 | PLPROP_GATTRIB24 | GString | Generic attribute 24 |
| 69 | PLPROP_GATTRIB25 | GString | Generic attribute 25 |
| 70 | PLPROP_GATTRIB26 | GString | Generic attribute 26 |
| 71 | PLPROP_GATTRIB27 | GString | Generic attribute 27 |
| 72 | PLPROP_GATTRIB28 | GString | Generic attribute 28 |
| 73 | PLPROP_GATTRIB29 | GString | Generic attribute 29 |
| 74 | PLPROP_GATTRIB30 | GString | Generic attribute 30 |
| 75 | PLPROP_OSTYPE | GString | OS type string (e.g. "wind") — requires client version 2.19+ |
| 76 | PLPROP_TEXTCODEPAGE | GInt (4 bytes) | Text codepage (e.g. 1252) — requires client version 2.19+ |
| 77 | PLPROP_UNKNOWN77 | (unknown) | (format unknown — debug required) |
| 78 | PLPROP_X2 | GShort | Pixel-precise X: `abs(x * 16) << 1`, bit 0 = sign (1 = negative) |
| 79 | PLPROP_Y2 | GShort | Pixel-precise Y: `abs(y * 16) << 1`, bit 0 = sign (1 = negative) |
| 80 | PLPROP_Z2 | GShort | Pixel-precise Z: `abs(z * 16) << 1`, bit 0 = sign (1 = negative) |
| 81 | PLPROP_UNKNOWN81 | GByte | Player list placement flag. Bit 0: show in players list; bit 1: show in servers tab; bit 2: show in channels tab. External/IRC players use this. |
| 82 | PLPROP_COMMUNITYNAME | GString | Community/guild display name |

**Properties 37–41 and 46–49 and 54–74** are the 30 generic player attributes (GATTRIB1 through GATTRIB30). All are GString-encoded. The server maps them using an `__attrPackets` lookup array rather than contiguous IDs, so the exact GATTRIB-to-property-ID mapping is fixed as shown above.

**Sword encoding (property 8):**

The server always sends two fields: a GByte power byte followed by a GByte-prefixed image string. The power byte is `swordPower + 30`. When reading:
- Decode first byte: `power_byte = readGByte()`. The raw sword power value is `power_byte - 30`.
- If `power_byte <= 4` (i.e., raw power 0–4 after the +30 offset was not applied because standard swords set the byte directly to 0–4 on the client side): standard sword — image name is implicit (`sword<power>.png`). The image length byte that follows is still present and will be the image name the server stored.
- The server writes: `>> (char)(swordPower + 30) >> (char)swordImg.length() << swordImg` — always two fields. For standard swords swordImg is `"sword<n>.png"` or `"sword<n>.gif"`; for custom swords it is the uploaded image name.

**Shield encoding (property 9):**

Same structure: `>> (char)(shieldPower + 10) >> (char)shieldImg.length() << shieldImg`.
- First byte is `shieldPower + 10`. Actual shield power = `byte_value - 10`.
- Second field is a GByte-prefixed image string (always present).
- For standard shields (power 0–3), shieldImg is `"shield<n>.png"` or `"shield<n>.gif"`.

**Head encoding (property 11):**

The server encodes: `>> (char)(headImg.length() + 100) << headImg`.
- Read first byte; subtract 100 to get the image name length.
- Read that many bytes for the image name.
- If `length + 100` overflows a GByte (headImg longer than 155 bytes), behavior is undefined — names are truncated in practice.

**Colors encoding (property 13):** 5 raw GByte values in order: skin, coat, sleeves, shoes, belt.

**Status flags (property 18):**
- Bit 0 (1): Paused
- Bit 1 (2): Hidden
- Bit 2 (4): Male
- Bit 3 (8): Dead
- Bit 4 (16): Weapons allowed
- Bit 5 (32): Hide sword
- Bit 6 (64): Has spin attack

**Property 20 - PLPROP_CURLEVEL:** Current level name (GString). For RC clients the server always encodes `" "` (a single space).

**Property 34 - PLPROP_ACCOUNTNAME:** Player's account name (GString).

**Property 30 - PLPROP_IPADDR:** IP address encoded with `writeGInt5(accountIp)` — a 5-byte GInt5. The 32-bit IP value is stored as a single integer.

**Property 36 - PLPROP_RATING:** Encoded as a 4-byte GInt (not 3 bytes). The packed value is `((rating & 0xFFF) << 9) | (deviation & 0x1FF)`. The earlier 3-byte description was incorrect.

**Properties 78-80 - PLPROP_X2, PLPROP_Y2, PLPROP_Z2:** Pixel-precise coordinates encoded as GShort.
- Encoding: `val = abs(coord * 16) << 1; if (coord < 0) val |= 0x0001`
- Bit 0 = sign (0 = positive, 1 = negative); bits 15-1 = absolute pixel value
- To decode tile position: `sign = val & 1; pixels = val >> 1; tile = pixels / 16.0 * (sign ? -1 : 1)`

### Packet 72 - Complete Player Data (PLO_RC_PLAYERPROPSGET)

Packet 72 (PLO_RC_PLAYERPROPSGET) sends ALL player properties at once. It's used when you request a player's full profile.

**Structure** (from `TPlayer::getPropsRC()` in TPlayerRC.cpp):

```
[Player ID: GShort]
[Account name length: GByte] [Account name: bytes]
[World name length: GByte] [World name: bytes]   -- hardcoded as "main" (4 bytes)
[Properties block length: GByte]
  [For each property in the RC set: Property ID (GByte) + property data]
[Flag count: GShort]
  [For each flag: Flag length (GByte) + flag bytes (format "name=value" or just "name")]
[Chest count: GShort]
  [For each chest: Chest data length (GByte) + [X GByte] [Y GByte] [chest identifier bytes]]
[Weapon count: GByte]
  [For each weapon: Weapon name length (GByte) + weapon name bytes]
```

**Notes:**
- The world name field is always `"main"` (GByte 4, then `"main"`) in GServer-v2.
- The properties block length is a single GByte — this limits the total encoded property data to 223 bytes maximum (GByte encoding caps at 223 for get1PlusTextNetString). If a player has many properties, only properties that fit are included.
- Flag names are encoded as raw `"name=value"` strings (or just `"name"` for flags with no value). The length byte is a raw GByte (not GString-style; maximum 223 bytes per flag).
- Chest data: each entry is `[total_length GByte][X GByte][Y GByte][identifier_string bytes]`.
- Weapon names: each entry is `[name_length GByte][name bytes]`.

### Setting Player Properties — PLI_RC_PLAYERPROPSSET2 (76)

Packet 76 (PLI_RC_PLAYERPROPSSET2) sets player properties identified by account name. The server reads it in `TPlayer::setPropsRC()`. The format mirrors what `getPropsRC()` produces, but note some differences:

```
[Account name: GString — read with readChars(readGUChar())]  <-- from PLI_RC_PLAYERPROPSSET2 handler
[World name: GString — readChars(readGUChar()), skipped/ignored by server]
[Properties block length: GByte — readChars(readGUChar()) reads this many prop bytes]
  [For each property: Property ID (raw GByte, NOT +32 encoded) + property data]
[Flag count: GShort]
  [For each flag: Flag length (GByte) + flag string ("name=value" or "name")]
[Chest count: GShort]
  [For each chest: Chest data length (GByte) + [X GByte][Y GByte][identifier bytes]]
[Weapon count: GByte]
  [For each weapon: Weapon name length (GByte) + weapon name bytes]
```

**Important:** The server reads the world name field with `readChars(readGUChar())` and discards it. Send `"\x24main"` (GByte 4, then "main") to match server output.

**Important:** Property IDs in the props block are raw byte values (0–82), NOT GByte-encoded (+32). The server reads them with `readGUChar()` which subtracts 32. So when writing, add 32 to each property ID byte as normal GByte encoding.

### Setting Nickname — PLI_PLAYERPROPS (packet 2)

To set the RC client's own nickname (display name), send packet 2 (PLI_PLAYERPROPS) with:
- A single space byte `0x20` (level name placeholder)
- Followed by the nickname as a `get1PlusTextNetString`

**Note on PLI_RC_PLAYERPROPSSET (packet 60):** Packet 60 is **empty/dead** in the server implementation (Java enum position 60 is `EMPTY60`). Do not use packet 60 to set player properties. The correct packet for setting properties by account name is packet 76. Setting your own RC client nickname uses packet 2 (PLI_PLAYERPROPS).

## File Browser

The file browser lets you upload, download, and manage files on the server.

### Opening the File Browser

Send packet 89 (PLI_RC_FILEBROWSER_START) with no data.

The server responds with:
- Packet 65 (PLO_RC_FILEBROWSER_DIRLIST): List of folders you can access
- Packet 66 (PLO_RC_FILEBROWSER_DIR): Contents of the default folder
- Packet 67 (PLO_RC_FILEBROWSER_MESSAGE): Welcome message

**CRITICAL: Buffer-Level Compression**

The server may send these packets bundled together and compressed with BZIP2. You MUST check for this before parsing packets!

**How to detect:**
- After receiving data and decrypting (if encrypted), check if buffer starts with 'BZ'
- If yes, decompress the ENTIRE buffer with bz2.decompress()
- Only then parse individual packets from the decompressed data

**Why this matters:**
- Without decompression, you'll see garbage packet IDs
- The entire response bundle is compressed, not individual packets
- This check must happen BEFORE any packet parsing
- This check must happen AFTER decryption (if encryption is enabled)

**Processing order:**
```
1. Receive network buffer
2. Decrypt buffer (if encryption enabled)
3. CHECK: Does buffer start with 'BZ'?
   YES → Decompress entire buffer with bz2.decompress()
   NO  → Continue to packet parsing
4. Parse individual packets from decompressed buffer
```

**Common mistake:** Trying to parse packets from compressed data directly. This will fail because the packet IDs are scrambled by compression.

### Folder List Format

Packet 65 contains a list of folders in this format:
```
r levels/*.nw
rw npcs/*.txt
r images/*
-r images/*.gif
```

**Format:** `[rights] [folder]/[pattern]`

**Rights:**
- `r` = allow read
- `w` = allow write
- `rw` = allow read and write (both permissions)
- `-r` = deny read
- `-w` = deny write

**How it works:**
- You can grant permissions: `r images/*` allows reading all files in images/
- You can grant both: `rw npcs/*.txt` allows reading and writing .txt files in npcs/
- You can deny specific patterns: `-r images/*.gif` denies reading .gif files (overrides the general rule)
- Multiple rules can apply - more specific patterns override general ones
- If a file doesn't match any rule, it has no permissions

**Important:** If you only grant `w` (write) without `r` (read), the user can write files but cannot see them in the folder listing - the folder will appear empty or may not show up at all.

**Packet 65 payload format:** The server sends the raw folder list run through `gtokenize()` (CommaText encoding). Each decoded line is one folder rule in the format `rights pattern` (e.g. `"rw levels/*"`). Use `gtokenizeReverse` / `guntokenize` to decode. There is no length prefix — the gtokenized bytes fill the entire packet payload after the packet ID byte.

### Folder Contents

Packet 66 (PLO_RC_FILEBROWSER_DIR) has TWO completely different purposes depending on context. You must detect which one it is.

**Purpose 1: Folder File Listing (Normal Use)**

After sending packet 90 (PLI_RC_FILEBROWSER_CD) with a folder path, the server responds with packet 66 (PLO_RC_FILEBROWSER_DIR) containing the file list.

**Format:**
- Some servers send plain Graal-encoded data (no Format.DYNAMIC)
- Some servers send Format.DYNAMIC encoded data
- You must detect which format is being used

**Detection:**
- Check first byte: if it's 0-5, it might be Format.DYNAMIC type OR a folder path length
- If bytes after first byte look like a valid folder path (printable ASCII with '/' or alphanumeric), it's plain Graal encoding
- Otherwise, treat first byte as Format.DYNAMIC type

**Format.DYNAMIC types:**
- Type 1: Encrypted (scrambled with current encryption key, 12 rotations)
- Type 2: ZLIB compressed
- Type 3: Encrypted + ZLIB
- Type 4: BZIP2 compressed
- Type 5: Encrypted + BZIP2

**Actual GServer-v2 Format** (from `msgPLI_RC_FILEBROWSER_START` and `msgPLI_RC_FILEBROWSER_CD` in TPlayerRC.cpp):

```
[Folder path: GString — GByte length + path bytes]
[For each file entry, prefixed by a literal space byte ' ':
  [File packet length: GByte]
  [File packet bytes:
    [Filename length: GByte] [Filename bytes]
    [Rights length: GByte] [Rights bytes]    -- e.g. "rw" or "r"
    [File size: GInt5 (5 bytes)]
    [Modification time: GInt5 (5 bytes)]
  ]
]
```

The server comment in the source reads: `// file packet: {CHAR name_length}{STRING name}{CHAR rights_length}{STRING rights}{INT5 file_size}{INT5 file_mod_time}` and `// files: {CHAR file_packet_length}{file_packet}[space]{CHAR file_packet_length}{file_packet}[space]`.

There is **no GShort message size field** before each file entry. The "message size GShort" mentioned in earlier documentation does not exist in this implementation. Each file entry is wrapped in `[GByte total_length][content]` with a literal space `0x20` separating consecutive entries.

**Purpose 2: Embedded Large File Download (Special Case)**

When downloading large files (especially level files), packet 66 can contain the first chunk of file data embedded directly in it. This is the "embedded protocol" mentioned in the large file section.

**How to detect:**
- After BZIP2 decompression, check if first byte = 100 (packet 68)
- If yes, and remaining payload > 1000 bytes, it's embedded file content
- Extract filename and first chunk as described in the large file section

**Important:** You need to track whether you're expecting a folder listing or a file download to know which format to use.

### Downloading Files

File downloads use different protocols depending on file size and type. This is one of the most complex parts of the RC protocol.

#### Small Files (Simple Protocol)

For small files (typically under 32KB):
1. Send packet 92 (PLI_RC_FILEBROWSER_DOWN) with the full file path
2. Server sends packet 102 (PLO_FILE) with complete file data
3. Done!

**Packet 102 format:**
- Timestamp (GInt5 - 5 bytes!)
- Filename length (GByte) + filename
- File content

#### Large Files - Two Different Protocols

There are TWO completely different protocols for large files. You must detect which one is being used.

**Protocol 1: Standalone Packets (Images, Audio, Small Large Files)**

This is the "normal" large file protocol:

```
1. Client → packet 92 (PLI_RC_FILEBROWSER_DOWN) with full path
2. Server → packet 68 (PLO_LARGEFILESTART) with basename only
3. Server → packet 84 (PLO_LARGEFILESIZE) with file size (GInt5)
4. Server → multiple packet 100 (PLO_RAWDATA) wrappers, each containing:
   - Packet 102 (PLO_FILE) chunk
5. Server → packet 69 (PLO_LARGEFILEEND) with basename only
```

**Critical details:**
- Packet 68 and 69 use the **basename only** (e.g., "bg.png")
- Packet 102 chunks use the **full path** (e.g., "levels/graphics/bg.png")
- You MUST use the full path from packet 92 as the key for tracking transfers
- If you use basename as key, chunks won't match and you'll get corrupted files

**Packet 100 (PLO_RAWDATA) wrapper:**
This wraps binary packets that might contain 0x0A bytes (which would be mistaken for packet delimiters).

**Format:**
- Packet ID: 100 + 32 = 132 (0x84)
- Length: GInt24 (3 bytes: ((length >> 14) & 0x7F) + 32, ((length >> 7) & 0x7F) + 32, (length & 0x7F) + 32)
- End byte: 0x0A
- Inner packet: Complete packet with ID + payload + trailing 0x0A

**CRITICAL BUG TO AVOID:** The inner packet has a trailing 0x0A that you MUST strip! If you don't, each chunk will be 1 byte too large, causing file corruption.

**How to unwrap packet 100:**
1. Read packet ID (should be 132/0x84)
2. Read 3 bytes for GInt24 length
3. Read and skip the 0x0A end byte
4. Read the inner packet (length bytes)
5. **STRIP the trailing 0x0A from the inner packet**
6. Process the inner packet normally

**Protocol 2: Embedded in Packet 66 (Levels, Very Large Files)**

Some servers (especially for level files) embed the first chunk directly in packet 66. This is more complex:

```
1. Client → packet 92 (PLI_RC_FILEBROWSER_DOWN) with full path
2. Server → packet 66 (PLO_RC_FILEBROWSER_DIR) BZIP2-compressed containing:
   - Packet 68 (PLO_LARGEFILESTART) header (byte 100 = 0x64)
   - Filename (null/newline/carriage return terminated)
   - Embedded first chunk (~32KB) with structure:
     [5-byte timestamp (GInt5)]
     [1-byte filename length + 32]
     [filename bytes]
     [first chunk of file content]
3. Server → packet 66 (PLO_RC_FILEBROWSER_DIR) BZIP2-compressed containing:
   - Packet 100 (PLO_RAWDATA) wrapper
   - Packet 102 (PLO_FILE) with second chunk
4. Server → more packet 66 with packet 100/102 chunks as needed
5. Server → packet 69 (PLO_LARGEFILEEND) with basename
```

**How to detect and handle embedded protocol:**

1. **Check for BZIP2 compression first:**
   - If buffer starts with 'BZ', decompress entire buffer first
   - This is buffer-level compression, not packet-level!

2. **Detect packet 68 (PLO_LARGEFILESTART) in packet 66:**
   - After BZIP2 decompression, check if first byte = 100 (packet 68 - PLO_LARGEFILESTART)
   - If yes, this is the embedded protocol

3. **Extract filename:**
   - Read bytes after packet 68 ID
   - Find first null byte, newline, or carriage return
   - Filename is everything before that terminator

4. **Check for embedded content:**
   - If remaining payload > 1000 bytes, embedded content follows
   - Search for filename string in remaining payload
   - File content starts IMMEDIATELY after filename match
   - Store this as the first chunk in your buffer

5. **Handle subsequent chunks:**
   - Next packet 66 will have packet 100 (PLO_RAWDATA) wrapper with packet 102 (PLO_FILE)
   - Unwrap packet 100, strip trailing 0x0A
   - Extract content from packet 102 and append to buffer

6. **Finalize:**
   - When packet 69 (PLO_LARGEFILEEND) arrives, combine all chunks and save file

**Why this matters:**
- Level files (.nw) use embedded protocol
- Missing the first chunk loses ~32KB of data
- Results in corrupted levels missing "GLEVNW01" header
- Images/audio usually use standalone protocol

**Implementation example:**
```python
file_transfers = {}
pending_file_download = None

def request_file(filepath):
    global pending_file_download
    pending_file_download = filepath  # Store full path!
    send_packet(92, filepath.encode('latin-1'))

def handle_packet_66(payload):
    # Check for BZIP2 compression
    if payload.startswith(b'BZ'):
        payload = bz2.decompress(payload)
    
    # Check if this is embedded bigfile protocol
    if len(payload) > 0 and payload[0] == 100:  # Packet 68 (PLO_LARGEFILESTART)
        # Extract filename
        filename_bytes = payload[1:]
        null_pos = min([p for p in [
            filename_bytes.find(b'\x00'),
            filename_bytes.find(b'\n'),
            filename_bytes.find(b'\r')
        ] if p != -1] + [len(filename_bytes)])
        
        filename = filename_bytes[:null_pos].decode('latin-1', errors='ignore').strip()
        content_offset = 1 + null_pos + 1
        remaining = payload[content_offset:] if content_offset < len(payload) else b''
        
        # Check for embedded content
        if len(remaining) > 1000:
            # Find filename in remaining payload
            fname_start = remaining.find(filename.encode('latin-1'))
            if fname_start != -1:
                # Skip timestamp (5 bytes) + filename length + filename
                offset = fname_start
                offset += 5  # Skip timestamp
                fname_len = remaining[offset] - 32
                offset += 1 + fname_len  # Skip filename
                actual_content = remaining[offset:]
                
                # Store first chunk
                full_path = pending_file_download if pending_file_download else filename
                file_transfers[full_path] = {
                    'buffer': bytearray(actual_content),
                    'size': 0,
                    'received': len(actual_content)
                }
        return
    
    # Normal folder listing or packet 100 wrapper
    # ... handle normally

def handle_packet_100(payload):
    # Read GInt24 length
    b0 = (payload[1] - 32) & 0x7F
    b1 = (payload[2] - 32) & 0x7F
    b2 = (payload[3] - 32) & 0x7F
    length = (b0 << 14) + (b1 << 7) + b2
    
    # Skip: packet_id(1) + length(3) + end_byte(1) = 5 bytes
    inner_packet = payload[5:5 + length]
    
    # CRITICAL: Strip trailing 0x0A
    if len(inner_packet) > 1 and inner_packet[-1] == 0x0A:
        inner_packet = inner_packet[:-1]
    
    # Process inner packet (should be packet 102 - PLO_FILE)
    inner_id = inner_packet[0] - 32
    inner_payload = inner_packet[1:]
    
    if inner_id == 102:  # PLO_FILE
        handle_packet_102(inner_payload)

def handle_packet_102(payload):
    # Read timestamp (GInt5)
    timestamp_bytes = [(payload[i] - 32) & 0x7F for i in range(5)]
    timestamp = (timestamp_bytes[0] << 28) | (timestamp_bytes[1] << 21) | \
                (timestamp_bytes[2] << 14) | (timestamp_bytes[3] << 7) | timestamp_bytes[4]
    
    # Read filename
    offset = 5
    filename_len = payload[offset] - 32
    offset += 1
    filename = payload[offset:offset + filename_len].decode('latin-1')
    offset += filename_len
    
    # Extract content
    content = payload[offset:]
    
    # Check if this is part of a large file transfer
    if filename in file_transfers:
        # Append to buffer
        file_transfers[filename]['buffer'].extend(content)
        file_transfers[filename]['received'] += len(content)
    else:
        # Standalone small file
        save_file(filename, content)

def handle_packet_68(payload):
    basename = payload.decode('latin-1')
    # Use full path from pending_file_download, not basename!
    full_path = pending_file_download if pending_file_download else basename
    file_transfers[full_path] = {'buffer': bytearray(), 'size': 0, 'received': 0}

def handle_packet_84(payload):
    # Read file size (GInt5)
    size_bytes = [(payload[i] - 32) & 0x7F for i in range(5)]
    file_size = (size_bytes[0] << 28) | (size_bytes[1] << 21) | \
                (size_bytes[2] << 14) | (size_bytes[3] << 7) | size_bytes[4]
    
    # Find transfer with size = 0 and set it
    for path in file_transfers:
        if file_transfers[path]['size'] == 0:
            file_transfers[path]['size'] = file_size
            break

def handle_packet_69(payload):
    basename = payload.decode('latin-1')
    # Use full path, not basename!
    full_path = pending_file_download if pending_file_download else basename
    
    if full_path in file_transfers:
        complete = bytes(file_transfers[full_path]['buffer'])
        save_file(full_path, complete)
        del file_transfers[full_path]
        pending_file_download = None
```

**Common bugs:**
1. Using basename instead of full path as key → chunks don't match → corruption
2. Not stripping trailing 0x0A from packet 100 → each chunk 1 byte too large → corruption
3. Not checking for BZIP2 compression → can't parse packets
4. Not detecting embedded protocol → lose first 32KB of level files
5. Reading GInt5 as 8 raw bytes → wrong file sizes/timestamps

### Uploading Files

Send packet 93 (PLI_RC_FILEBROWSER_UP) with the file data.

**Important:** Large files must be wrapped in packet 100 (PLO_RAWDATA), just like downloads.

**Format:**
- Inner packet: Packet 93 + filename (GString) + file content + 0x0A
- Wrap in packet 100: Packet 100 + length (GInt24) + 0x0A + inner packet

**Example: Uploading a simple text file**

For a small file like `test.txt` with content `Hello World`:

1. Create inner packet 93:
   - Packet ID: 93
   - Filename: "test.txt" (GString encoded)
   - Content: "Hello World"
   - End marker: 0x0A

2. For small files, you can send packet 93 directly without wrapping in packet 100.

3. For larger files, wrap the inner packet 93 in packet 100:
   - Packet ID: 100
   - Length: size of inner packet (GInt24 encoded)
   - End marker: 0x0A
   - Then the inner packet 93

**Hex Example:**
```
File: "test.txt"
Content: "Hello World"

Packet 93 (PLI_RC_FILEBROWSER_UPLOAD):
Byte 0: 0x7D (93 + 32 = 125)
Byte 1: 0x28 (8 + 32 = 40, filename length)
Bytes 2-9: 0x74 0x65 0x73 0x74 0x2E 0x74 0x78 0x74 ("test.txt")
Bytes 10-20: 0x48 0x65 0x6C 0x6C 0x6F 0x20 0x57 0x6F 0x72 0x6C 0x64 ("Hello World")
Byte 21: 0x0A (end marker)

Complete: 7D 28 74 65 73 74 2E 74 78 74 48 65 6C 6C 6F 20 57 6F 72 6C 64 0A
          ^^ ^^ ^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ ^^
          ID len "test.txt"          "Hello World"                end
```

### Other File Operations

**Delete file:** Send packet 97 (PLI_RC_FILEBROWSER_DELETE) with the file path (raw bytes, no length prefix)

**Example:**
```
File: "levels/test.nw"

Packet 97 (PLI_RC_FILEBROWSER_DELETE):
Bytes: 0x6C 0x65 0x76 0x65 0x6C 0x73 0x2F 0x74 0x65 0x73 0x74 0x2E 0x6E 0x77 0x0A
       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ ^^
       "levels/test.nw"                                                         end
```

**Rename file:** Send packet 98 (PLI_RC_FILEBROWSER_RENAME) with:
- Old name length (GByte) + old name
- New name length (GByte) + new name

**Example:**
```
Old: "test.txt" (8 bytes)
New: "renamed.txt" (11 bytes)

Packet 98 (PLI_RC_FILEBROWSER_RENAME):
Byte 0: 0x7E (98 + 32 = 130)
Byte 1: 0x28 (8 + 32 = 40, old name length)
Bytes 2-9: 0x74 0x65 0x73 0x74 0x2E 0x74 0x78 0x74 ("test.txt")
Byte 10: 0x2B (11 + 32 = 43, new name length)
Bytes 11-21: 0x72 0x65 0x6E 0x61 0x6D 0x65 0x64 0x2E 0x74 0x78 0x74 ("renamed.txt")
Byte 22: 0x0A (end marker)
```

**Move file:** Send packet 96 (PLI_RC_FILEBROWSER_MOVE) with (from `msgPLI_RC_FILEBROWSER_MOVE` in TPlayerRC.cpp):
- Destination directory (GString — GByte length + path bytes)
- File name (raw bytes, remainder of packet — the file to move from the current directory)

The server reads: `dir = readChars(readGUChar())` then `file = readString("")`. The source is `currentFolder + file`; the destination is `dir + file`.

**Correction:** The earlier documentation showing two raw byte paths was wrong. The destination is a GString-prefixed directory path, and the filename is the raw remainder. The file name is taken from the current directory (`lastFolder`), not from the packet.

**Delete folder:** Send packet 160 (PLI_RC_FOLDERDELETE) with the folder path (raw bytes, no length prefix)

**Close file browser:** Send packet 91 (PLI_RC_FILEBROWSER_END) with no data

**Get max upload size:** After opening file browser, server may send packet 103 (PLO_RC_MAXUPLOADFILESIZE) with the maximum file size allowed for uploads (GInt5).

## Account Management

### Getting Account Data

Send packet 77 (PLI_RC_ACCOUNTGET) with just the account name (no length prefix, just the raw bytes).

**Example:**
```
Account: "testaccount"

Packet 77 (PLI_RC_ACCOUNTGET):
Byte 0: 0x6D (77 + 32 = 109)
Bytes 1-11: 0x74 0x65 0x73 0x74 0x61 0x63 0x63 0x6F 0x75 0x6E 0x74 ("testaccount")
Byte 12: 0x0A (end marker)
```

Server responds with packet 73 (PLO_RC_ACCOUNTGET) containing (from `msgPLI_RC_ACCOUNTGET` in TPlayerRC.cpp):

```
[Account name: GString]
[Password: GByte 0 only]          -- always empty; GServer-v2 never sends the password
[Email: GString]
[Banned: GByte]                   -- 1 = banned, 0 = not banned
[Guest (loadOnly): GByte]         -- 1 = load-only/guest, 0 = normal
[Admin level: GByte 0]            -- always 0; field is deprecated in GServer-v2
[Admin worlds: GString]           -- hardcoded as "main" (GByte 4, then "main")
[Ban length: GString]             -- descriptive ban time string (from getBanLength())
[Ban reason: GString]             -- raw ban reason string (from getBanReason())
```

**Corrections to earlier documentation:**
- The password field is **always empty** — GServer-v2 sends a single `0x20` byte (GByte 0, meaning zero-length GString). No password is ever transmitted.
- The admin level field is **always 0** — it is deprecated. The server writes `>> (char)0`.
- The "admin worlds" field is **hardcoded** to the string `"main"` — it does not reflect the actual admin worlds configuration.
- Ban time is `getBanLength()` (a descriptive string), not a Unix timestamp or formatted date. Ban reason is `getBanReason()` (raw string, not CommaText).

### Setting Account Data

Send packet 78 (PLI_RC_ACCOUNTSET) with individual GString-prefixed fields (from `msgPLI_RC_ACCOUNTSET` in TPlayerRC.cpp). The format is **not** CommaText — each field is a separate GString:

```
[Account name: GString]       -- readChars(readGUChar())
[Password: GString]           -- readChars(readGUChar()); stored but may be ignored
[Email: GString]              -- readChars(readGUChar())
[Banned: GByte]               -- 0 = not banned, non-zero = banned
[Load-only/guest: GByte]      -- 0 = normal, non-zero = load-only
[Admin level: GByte]          -- readGUChar(), value is read and discarded (deprecated)
[World: GString]              -- readChars(readGUChar()), read and discarded
[Ban reason: GString]         -- readChars(readGUChar()); stored as ban reason
```

**Notes:**
- There is no separate ban-time field in this packet. Ban time is managed separately.
- The server only applies the ban/ban-reason fields if the sending RC has the `PLPERM_BAN` right.
- The password field is read but the GServer-v2 implementation does not save it to disk (account files are not password-managed by the game server in this build).
- To match the format that the server sends in packet 73, send an empty password GString (`\x20` = GByte 0).

## Player Management

### Getting Player Rights

Send packet 83 (PLI_RC_PLAYERRIGHTSGET) with the account name (raw bytes, no length prefix).

Server responds with packet 62 (PLO_RC_PLAYERRIGHTSGET) containing (from `msgPLI_RC_PLAYERRIGHTSGET` in TPlayerRC.cpp):

```
[Account name: GString]
[Rights value: GInt5 (5 bytes)]     -- written as >> (long long)rights, which uses writeGInt5
[IP range: GString]
[Folder access length: GShort]      -- always present, even if 0 (not optional)
[Folder access: bytes]              -- gtokenized folder list (length = field above)
```

The folder access section is **always present** — the `>> (short)folders.length() << folders` is unconditional. If the player has no folder access configured, the GShort will be 0 with no following bytes.

**Rights are stored as bits:**
- Bit 0: Warp to XY
- Bit 1: Warp to Player
- Bit 2: Warp player
- Bit 3: Update Level
- Bit 4: Disconnect
- Bit 5: View Attributes
- Bit 6: Set Attributes
- Bit 7: Set Own Attributes
- Bit 8: Reset Attributes
- Bit 9: Admin Message
- Bit 10: Change Rights
- Bit 11: Ban Players
- Bit 12: Comments
- (Bit 13: unused)
- Bit 14: Staff Accounts
- Bit 15: Server Flags
- Bit 16: Server Options
- Bit 17: Folder Configuration
- Bit 18: Folder Rights
- Bit 19: NPC Control

### Setting Player Rights

Send packet 84 (PLI_RC_PLAYERRIGHTSSET) with (from `msgPLI_RC_PLAYERRIGHTSSET` in TPlayerRC.cpp):

```
[Account name: GString]                   -- readChars(readGUChar())
[Rights value: GInt5 (5 bytes)]           -- readGUInt5()
[IP range: GString]                       -- readChars(readGUChar())
[Folder access length: GShort]            -- readGUShort()
[Folder access: bytes]                    -- readChars(length), then guntokenizeI()
```

The folder access bytes are a gtokenized newline-delimited list of folder rules. After reading, the server calls `guntokenizeI()` and then splits on `"\n"` to get individual folder entries.

### PLO_RC_PLAYERPROPS (71) — Deprecated

Packet 71 (PLO_RC_PLAYERPROPS) is listed in IEnums.h as deprecated: "Deprecated. Codr note: Unhandled by 6.037." GServer-v2 does not send this packet. Do not use it. Packet 72 (PLO_RC_PLAYERPROPSGET) is the active replacement.

### Getting Player Properties

There are THREE different packets for getting player properties. Use the one that matches how you identify the player:

**1. Packet 59 (PLI_RC_PLAYERPROPSGET) - DEPRECATED**
- This packet does nothing. Don't use it.

**2. Packet 73 (PLI_RC_PLAYERPROPSGET2) - By Player ID**
- Send: Player ID (GShort)
- Use this when you know the player's ID (from the player list)
- Server responds with packet 72 (PLO_RC_PLAYERPROPSGET) with all player properties

**3. Packet 74 (PLI_RC_PLAYERPROPSGET3) - By Account Name**
- Send: Account name (GString) + empty byte (32)
- Use this when you only know the account name
- Works for both online and offline players (loads from account file if offline)
- Server responds with packet 72 (PLO_RC_PLAYERPROPSGET) with all player properties

**Important:** Packet 74 can work even if the player is offline - it will load their account file and return the properties. Packet 73 only works for online players.

### Setting Player Properties

There are TWO active packets for setting player properties:

**1. Packet 2 (PLI_PLAYERPROPS) - Set own RC client nickname**

Used to set the RC client's own display name (nickname). Send:
- A single space byte `0x20` (level name placeholder)
- Nickname as `get1PlusTextNetString`

This is the only property modification packet 2 supports for RC clients.

**2. Packet 76 (PLI_RC_PLAYERPROPSSET2) - By Account Name**
- Send: Account name (GString) + complete player data (see Packet 72 format)
- Use this when you only know the account name
- Works for both online and offline players (loads from account file if offline)
- This is the recommended packet to use for setting any player property

**Note on packet 60:** Packet 60 (PLI_RC_PLAYERPROPSSET) is **not dead in GServer-v2** — the server has a functional handler `msgPLI_RC_PLAYERPROPSSET` that reads a player ID (GShort) and calls `setPropsRC()`. The source marks it with a comment "Deprecated?" but it does execute. The preferred packet is 76 (PLI_RC_PLAYERPROPSSET2) because it uses account name lookup (works for offline players), while packet 60 requires an active player ID.

### Getting Player Comments

Send packet 85 (PLI_RC_PLAYERCOMMENTSGET) with the account name (raw bytes, no length prefix).

Server responds with packet 63 (PLO_RC_PLAYERCOMMENTSGET) containing (from `msgPLI_RC_PLAYERCOMMENTSGET` in TPlayerRC.cpp):

```
[Account name: GString]
[Comments: raw bytes]     -- p->getComments() appended directly, no length prefix, no CommaText encoding
```

The comments field is the **raw stored comments string** with no wrapping or encoding applied. The server writes `>> (char)acc.length() << acc << p->getComments()` — the comments start immediately after the account GString. The comments may internally use CommaText if they were stored that way, but no encoding is added by this packet handler.

### Disconnecting a Player

Send packet 61 (PLI_RC_DISCONNECTPLAYER) with:
- Player ID (GShort)
- Reason (raw string bytes)

**Example:**
```
Player ID: 5
Reason: "You have been disconnected"

Packet 61 (PLI_RC_DISCONNECTPLAYER):
Byte 0: 0x5D (61 + 32 = 93)
Bytes 1-2: 0x25 0x25 (GShort: 5 encoded as 0x25 0x25)
Bytes 3-26: 0x59 0x6F 0x75 0x20 0x68 0x61 0x76 0x65 0x20 0x62 0x65 0x65 0x6E 0x20 0x64 0x69 0x73 0x63 0x6F 0x6E 0x6E 0x65 0x63 0x74 0x65 0x64 ("You have been disconnected")
Byte 27: 0x0A (end marker)
```

## Chat and Messaging

### Private Messages

Send packet 28 (PLI_PRIVATEMESSAGE) with:
- Number of players (GShort)
- For each player: Player ID (GShort)
- Message (CommaText format)

**Important:** The message uses CommaText encoding, not raw text.

**Example:**
```
Send to 2 players (IDs 5 and 7)
Message: "Hello there"

Packet 28 (PLI_PRIVATEMESSAGE):
Byte 0: 0x34 (28 + 32 = 60)
Bytes 1-2: 0x25 0x25 (GShort: 2 players)
Bytes 3-4: 0x25 0x25 (GShort: player ID 5)
Bytes 5-6: 0x25 0x27 (GShort: player ID 7)
Bytes 7-18: 0x22 0x48 0x65 0x6C 0x6C 0x6F 0x20 0x74 0x68 0x65 0x72 0x65 0x22 ("Hello there" in CommaText)
Byte 19: 0x0A (end marker)
```

### Admin Messages

**To all players:**
Send packet 63 (PLI_RC_ADMINMESSAGE) with just the message (raw bytes, no length prefix, no extra data).

**Common mistake:** Some implementations add extra data like Int16(1) or "test" string. Don't do this! Just send the raw message bytes.

**To specific player:**
Send packet 64 (PLI_RC_PRIVADMINMESSAGE) with:
- Player ID (GShort)
- Message (raw bytes, no length prefix)

**Important:** Packet 64 only supports ONE player. If you want to send to multiple players, send multiple separate packets.

**Example:**
```
Player ID: 5
Message: "Admin message here"

Packet 64 (PLI_RC_PRIVADMINMESSAGE):
Byte 0: 0x60 (64 + 32 = 96)
Bytes 1-2: 0x25 0x25 (GShort: player ID 5)
Bytes 3-22: 0x41 0x64 0x6D 0x69 0x6E 0x20 0x6D 0x65 0x73 0x73 0x61 0x67 0x65 0x20 0x68 0x65 0x72 0x65 ("Admin message here")
Byte 23: 0x0A (end marker)
```

### Regular "To All" Messages

Send packet 6 (PLI_TOALL) with:
- Message length (GByte)
- Message (raw bytes)

This is for regular players to send messages visible to everyone (not admin messages).

## Server Settings

### Getting Server Flags

Send packet 68 (PLI_RC_SERVERFLAGSGET) with no data.

Server responds with packet 61 (PLO_RC_SERVERFLAGSGET) containing (from `msgPLI_RC_SERVERFLAGSGET` in TPlayerRC.cpp):
- Flag count (GShort)
- For each flag: Flag string length (GByte) + flag string formatted as `"name=value"`

Each entry is a GString containing the flag in `"name=value"` format. Parse by splitting on `=`.

**Setting Server Flags:**
Send packet 69 (PLI_RC_SERVERFLAGSSET) with (from `msgPLI_RC_SERVERFLAGSSET` in TPlayerRC.cpp):
- Flag count (GShort)
- For each flag: Flag string length (GByte) + flag string (in `"name=value"` format, or just `"name"` for flags without a value)

The server reads each entry with `readChars(readGUChar())` and calls `server->setFlag()` on it, parsing `=` to split name from value.

### Getting Server Options

Send packet 51 (PLI_RC_SERVEROPTIONSGET) with no data.

Server responds with packet 76 (PLO_RC_SERVEROPTIONSGET) containing the server options in CommaText format.

**Setting Server Options:**
Send packet 52 (PLI_RC_SERVEROPTIONSSET) with options in CommaText format.

### Getting Folder Configuration

Send packet 53 (PLI_RC_FOLDERCONFIGGET) with no data.

Server responds with packet 77 (PLO_RC_FOLDERCONFIGGET) containing the folder configuration in CommaText format.

**Setting Folder Configuration:**
Send packet 54 (PLI_RC_FOLDERCONFIGSET) with configuration in CommaText format.

## Player Presence Updates

### PLO_ADDPLAYER (55) — Player Joined

The server sends packet 55 to RC clients when a player joins (or during the login player-list exchange). The format for RC clients (from `sendLogin()` in TPlayerLogin.cpp) is:

```
[Player ID: GShort]
[Account name length: GByte] [Account name: bytes]
[PLPROP_CURLEVEL GByte]  [level name GString]   -- always " " (space) for RC clients
[PLPROP_PSTATUSMSG GByte][status GByte]
[PLPROP_NICKNAME GByte]  [nickname GString]
[PLPROP_COMMUNITYNAME GByte][community name GString]
```

The packet is a stream of (property-ID byte + property-data) pairs immediately after the account name. The first field is a raw account name prefixed by its length byte (not PLPROP-style — there is no property ID before the account name). The properties that follow do use their PLPROP ID byte as a prefix.

Note: IEnums.h marks this packet as "Unhandled by 5.07+". For RC connections the server sends it regardless.

### PLO_DELPLAYER (56) — Player Left

```
[Player ID: GShort]
```

IEnums.h marks this as "Unhandled by 5.07+". The server sends it when a player disconnects.

## Player Warping

### Warp Player

Send packet 82 (PLI_RC_WARPPLAYER) with (from `msgPLI_RC_WARPPLAYER` in TPlayerRC.cpp):

```
[Player ID: GShort]
[X coordinate: GByte]    -- server reads with readGChar() then divides by 2.0
[Y coordinate: GByte]    -- server reads with readGChar() then divides by 2.0
[Level name: raw string] -- readString(""), remainder of packet, no length prefix
```

**Coordinate encoding:**
Multiply tile coordinate by 2, then encode as GByte (value + 32). The server decodes as `(float)(readGChar()) / 2.0f`.

**Verified:** Field order in the source is `readGUShort()` for player ID, then `readGChar()/2.0f` twice for X and Y, then `readString("")` for level name. The order matches what the docs state.

**Example:**
- Warp to X = 30.5 tiles, Y = 20 tiles
- X: 30.5 × 2 = 61, encode as 61 + 32 = 93
- Y: 20 × 2 = 40, encode as 40 + 32 = 72

**Hex Example:**
```
Player ID: 5
X: 30.5 tiles (61 encoded)
Y: 20 tiles (40 encoded)
Level: "onlinestartlocal.nw"

Packet 82 (PLI_RC_WARPPLAYER):
Byte 0: 0x72 (82 + 32 = 114)
Bytes 1-2: 0x25 0x25 (GShort: player ID 5)
Byte 3: 0x5D (61 + 32 = 93, X coordinate)
Byte 4: 0x48 (40 + 32 = 72, Y coordinate)
Bytes 5-24: 0x6F 0x6E 0x6C 0x69 0x6E 0x65 0x73 0x74 0x61 0x72 0x74 0x6C 0x6F 0x63 0x61 0x6C 0x2E 0x6E 0x77 ("onlinestartlocal.nw")
Byte 25: 0x0A (end marker)
```

## Player Profile Management

### Getting Player Profile

Send packet 80 (PLI_PROFILEGET) with the account name (raw bytes, no length prefix).

Server responds with packet 75 (PLO_PROFILE) containing profile data as a sequence of length-prefixed strings (GString format):
- Index 0: Account name
- Index 1: Real name
- Index 2: Age
- Index 3: Sex/gender
- Index 4: Country
- Index 5: Messenger
- Index 6: E-Mail
- Index 7: Homepage
- Index 8: Favorite hangout
- Index 9: Favorite quote
- Index 10: Online time
- Index 11+: Server extras (variable, server-defined)

### Setting Player Profile

Send packet 81 (PLI_PROFILESET) with:
- Account name (GString)
- For each profile field (indices 1-10): Field length (GByte) + field value

**Profile fields (in order, indices 1-10):**
1. Real name
2. Age
3. Sex/gender
4. Country
5. Messenger
6. E-Mail
7. Homepage
8. Favorite hangout
9. Favorite quote
10. Online time

## Account Operations

### Creating an Account

Send packet 70 (PLI_RC_ACCOUNTADD) with account creation data. The server reads individual GString fields (from `msgPLI_RC_ACCOUNTADD` in TPlayerRC.cpp):

```
[Account name: GString]       -- readChars(readGUChar())
[Password: GString]           -- readChars(readGUChar())
[Email: GString]              -- readChars(readGUChar())
[Banned: GByte]               -- 0 = not banned, non-zero = banned
[Load-only/guest: GByte]      -- 0 = normal, non-zero = load-only
[Admin level: GByte]          -- readGUChar(), deprecated, value discarded
```

This is **not** CommaText — it is a flat sequence of GString-prefixed fields followed by two GBytes.

### Deleting an Account

Send packet 71 (PLI_RC_ACCOUNTDEL) with the account name.

Server responds with packet 53 (PLO_RC_ACCOUNTDEL) confirming deletion.

### Getting Account List

Send packet 72 (PLI_RC_ACCOUNTLISTGET) with:
- Name pattern length (GByte) + name pattern (use "*" for all, "%" is converted to "*")
- Conditions length (GByte) + conditions (optional filter, can be empty)

**Example:**
- Get all accounts: Send `(char)1 "*" (char)0 ""`
- Get accounts matching "test*": Send `(char)5 "test*" (char)0 ""`

Server responds with packet 70 (PLO_RC_ACCOUNTLISTGET) containing (from `msgPLI_RC_ACCOUNTLISTGET` in TPlayerRC.cpp):
- For each matching account: Account name length (GByte) + account name bytes

The accounts are appended consecutively with no count prefix and no separator — read GByte-prefixed strings until the end of the packet.

## Player Ban Management

### Getting Player Ban Info

Send packet 87 (PLI_RC_PLAYERBANGET) with the account name (raw bytes, no length prefix).

Server responds with packet 64 (PLO_RC_PLAYERBANGET) containing (from `msgPLI_RC_PLAYERBANGET` in TPlayerRC.cpp):

```
[Account name: GString]
[Banned flag: GByte]        -- 1 = banned, 0 = not banned
[Ban reason: raw bytes]     -- p->getBanReason() appended directly, no length prefix
```

The server writes: `>> (char)acc.length() << acc >> (char)(p->getBanned() ? 1 : 0) << p->getBanReason()`

**The multi-section format (4 ban sections with computer ID, time remaining, reset timer, etc.) documented previously is incorrect for GServer-v2.** GServer-v2 uses a simple banned flag plus a reason string. The complex multi-section format may apply to the original Graal Online listserver ban system, which is separate from the game server ban system.

### Setting Player Ban

Send packet 88 (PLI_RC_PLAYERBANSET) with (from `msgPLI_RC_PLAYERBANSET` in TPlayerRC.cpp):

```
[Account name: GString]            -- readChars(readGUChar())
[Banned flag: GByte]               -- readGUChar(); 0 = unban, non-zero = ban
[Ban reason: raw bytes]            -- readString(""); remainder of packet
```

The server reads: account as GString, then one GByte for the ban flag, then the remainder of the packet as the ban reason string.

**The multi-section format documented previously is incorrect for GServer-v2.** The simple format above is what GServer-v2 actually implements. If the player is currently online and is being banned, the server disconnects them immediately.

## Resetting Player Properties

Send packet 75 (PLI_RC_PLAYERPROPSRESET) with the account name (raw bytes, no length prefix).

This resets the player's properties to default values. The server will respond with updated player properties.

## Level Operations

### Updating Levels

Send packet 62 (PLI_RC_UPDATELEVELS) with level update data.

This forces the server to reload level data from disk.

## NC Server Address Query

Send packet 94 (PLI_NPCSERVERQUERY) with:
- NPC server player ID (GShort) — the ID of the player named "(npcserver)"
- The string `"location"` (raw bytes)

Server responds with packet 79 (PLO_NPCSERVERADDR) containing:
- Server identifier (GShort — the raw value minus 0x1020 gives the internal server ID)
- NC server address as string: `"host,port"` (e.g., `"198.27.86.142,14801"`)

If the address is `"127.0.0.1"`, use the game server's IP address instead.

**Packet 78 (PLO_NC_CONTROL):** When the server echoes back an NC server query, you may receive packet 78. This packet contains the raw echo of your request. No action needed — just log it.

## Request Text / Send Text

These are general-purpose packets for requesting or sending arbitrary data to the server. They use a special command format with "GraalEngine" as the prefix.

### Request Text (Packet 152 - PLI_REQUESTTEXT)

This packet requests information from the server. The format is always CommaText with newline-separated fields.

**Format:**
```
"GraalEngine\n<type>\n<option>\n<additional_params>\n"
```

The server responds with packet 82 (PLO_SERVERTEXT) containing the requested data, also in CommaText format.

**Common Request Commands:**

**1. Cross-Server Player Lists:**
```
"GraalEngine\npmserverplayers\n<servername>\n"
```
- Requests list of players on another server
- Server forwards request to listserver
- Listserver asks target server for player list
- Response comes back as packet 82, then external players are added to your player list

**2. Remove Server from PM List:**
```
"GraalEngine\npmunmapserver\n<servername>\n"
```
- Removes all external players from the specified server
- Cleans up the player list

**3. Get Guild List:**
```
"GraalEngine\npmguilds\n"
```
- Requests list of guilds
- Server forwards to listserver
- Response via packet 82

**4. Get Server List:**
```
"GraalEngine\npmservers\nall\n"
```
- Requests list of all servers
- Used for cross-server messaging setup

**5. Listserver Commands:**

**Simple Server List:**
```
"GraalEngine\nlister\nsimplelist\nall\n"
```
- Requests simple server list from listserver
- Response via packet 82 with server list data

**Subscriptions:**
```
"GraalEngine\nlister\nsubscriptions\n"
```
- Requests subscription information
- Server responds with subscription status

**Ban Types:**
```
"GraalEngine\nlister\nbantypes\n"
```
- Requests list of available ban types and durations
- Response includes ban type names and time durations in seconds

**Global Items:**
```
"GraalEngine\nlister\ngetglobalitems\n"
```
- Requests global item information
- Response includes item details (autobill, bundle, creation time, etc.)

**Server Info:**
```
"GraalEngine\nlister\nserverinfo\n"
```
- Requests server information
- Server forwards to listserver

**6. Package Info:**
```
"GraalEngine\npackageinfo\n<package_name>\n"
```
- Requests information about an update package
- Response includes file count and total size

**7. IRC Commands:**
```
"GraalEngine\nirc\n<command>\n"
```
- IRC-related requests (see IRC section below)

### Send Text (Packet 154 - PLI_SENDTEXT)

This packet sends commands or data to the server. Same format as REQUESTTEXT but used for actions rather than queries.

**Format:**
```
"GraalEngine\n<type>\n<option>\n<params>\n"
```

**Common Send Commands:**

**1. Listserver Options:**
```
"GraalEngine\nlister\noptions\nglobalpms=true,buddytracking=true,showbuddies=true\n"
```
- Sets listserver options
- Options are comma-separated key=value pairs

**2. Buddy Management:**
```
"GraalEngine\nlister\nverifybuddies\n1,<pcid>\n"
```
- Verifies buddy tracking
- First param is usually 1, second is PC-ID

```
"GraalEngine\nlister\naddbuddy\n<account>\n"
```
- Adds a buddy to your buddy list
- Account name is gtokenized

```
"GraalEngine\nlister\ndeletebuddy\n<account>\n"
```
- Removes a buddy from your buddy list

**3. IRC Commands:**

**Login to IRC:**
```
"GraalEngine\nirc\nlogin\n-\n"
```
- Logs into IRC system
- Server creates IRC channel players (starting at ID 16000)
- Returns existing IRC channels as players

**Join IRC Channel:**
```
"GraalEngine\nirc\njoin\n#channel\n"
```
- Joins an IRC channel
- Server forwards to listserver
- Channel appears in player list

**Leave IRC Channel:**
```
"GraalEngine\nirc\npart\n#channel\n"
```
- Leaves an IRC channel
- Server forwards to listserver

**Send IRC Private Message:**
```
"GraalEngine\nirc\nprivmsg\n<target>\n<message>\n"
```
- Sends IRC private message
- If target is "IRCBot", handles special bot commands
- Otherwise forwards to listserver

**4. Server Info Request:**
```
"GraalEngine\nlister\nserverinfo\n"
```
- Requests server information
- Server forwards to listserver

**5. Get Ban Info (RC only):**
```
"GraalEngine\nlister\ngetban\n<pc_id>\n"
```
- Gets ban information for a PC-ID
- RC clients only
- Response via packet 82

### Server Text Response (Packet 82 - PLO_SERVERTEXT)

The server responds to REQUESTTEXT and some SENDTEXT commands with packet 82.

**Format:**
- Response data in CommaText format
- Structure matches the request format
- First field is usually the "weapon" (GraalEngine)
- Second field is the "type" (lister, irc, etc.)
- Third field is the "option" (command name)
- Additional fields contain the response data

**Example Responses:**

**Subscription Response:**
```
"GraalEngine\nlister\nsubscriptions\n<subscription_data>\n"
```

**Server List Response:**
```
"GraalEngine\nlister\nsimpleserverlist\n<server1_data>\n<server2_data>\n..."
```

**Package Info Response:**
```
"GraalEngine\npackageinfo\n<package_name>\n<file_count>\n<total_size>\n"
```

**Ban Types Response:**
```
"GraalEngine\nlister\nbantypes\n<ban_type_data>\n"
```
- Contains comma-separated ban types with durations
- Format: "Event Interruption,259200" (name, seconds)

### How REQUESTTEXT/SENDTEXT Works

**The Flow:**

1. **Client sends REQUESTTEXT (152):**
   - Format: "GraalEngine\n<type>\n<option>\n<params>\n" (gtokenized)
   - Server receives and parses the command

2. **Server processes the command:**
   - Checks the "type" field (lister, irc, pmserverplayers, etc.)
   - Checks the "option" field (specific command)
   - Executes the appropriate handler

3. **Server may forward to listserver:**
   - Some commands (pmserverplayers, pmservers, irc) are forwarded
   - Server sends SVO_REQUESTLIST or similar to listserver
   - Listserver processes and responds

4. **Server responds with SERVERTEXT (82):**
   - Response in same CommaText format
   - Client parses and handles the response

**Important Details:**

- **All data is gtokenized:** The entire command string is converted to CommaText format before sending
- **Newline-separated fields:** Within the gtokenized string, fields are separated by \n
- **Case-sensitive:** Commands are case-sensitive (e.g., "GraalEngine" not "graalengine")
- **IRC player IDs:** IRC channels get player IDs starting at 16000
- **External player IDs:** Cross-server players also get IDs starting at 16000
- **Property 81:** External players (IRC, cross-server) have property 81 set to indicate they're special

### Common Patterns

**Pattern 1: Query Information**
```
Client → REQUESTTEXT: "GraalEngine\nlister\nsubscriptions\n"
Server → SERVERTEXT: "GraalEngine\nlister\nsubscriptions\n<data>\n"
```

**Pattern 2: Forward to Listserver**
```
Client → REQUESTTEXT: "GraalEngine\npmserverplayers\nServerName\n"
Server → Listserver: SVO_REQUESTLIST
Listserver → Target Server: SVI_REQUESTTEXT
Target Server → Listserver: SVI_REQUESTTEXT (with player list)
Listserver → Server: SVI_REQUESTTEXT (with player list)
Server → Client: SERVERTEXT + PLO_ADDPLAYER packets for each external player
```

**Pattern 3: Action Command**
```
Client → SENDTEXT: "GraalEngine\nlister\naddbuddy\nAccountName\n"
Server → Listserver: Forwards command
Server → Client: SERVERTEXT (confirmation)
```

### Error Handling

If a REQUESTTEXT command fails or is invalid:
- Server may send SERVERTEXT with error information
- Server may send nothing (silent failure)
- Server logs the request for debugging

### Implementation Notes

**When sending REQUESTTEXT:**
1. Build the command string with newlines
2. Convert to CommaText using gtokenize()
3. Send as packet 152

**When receiving SERVERTEXT:**
1. Receive packet 82
2. Convert from CommaText using gtokenizeReverse()
3. Parse newline-separated fields
4. Handle based on type and option fields

**Example implementation:**
```python
# Request player list from another server
command = "GraalEngine\npmserverplayers\nServerName\n"
tokenized = gtokenize(command)
send_packet(152, tokenized.encode('latin-1'))

# Handle response
def handle_servertxt(payload):
    data = payload.decode('latin-1', errors='ignore')
    untokenized = gtokenizeReverse(data)
    parts = untokenized.split('\n')
    
    if len(parts) >= 3:
        if parts[0] == "GraalEngine":
            if parts[1] == "lister" and parts[2] == "simpleserverlist":
                # Parse server list
                servers = parts[3:]  # Remaining parts are server entries
                for server_data in servers:
                    # Parse each server entry
                    pass
            elif parts[1] == "pmserverplayers":
                # External players will be added via PLO_ADDPLAYER packets
                pass
```

## Helper Functions and Encoding Utilities

### get1PlusTextNetString

This function creates a length-prefixed string where the length byte can exceed 255 by using a special encoding.

**Format:**
- If string length <= 223: Normal GString (length + 32, then string)
- If string length > 223: Use 255 (0xFF) as length byte, then send remaining string (length - 223)

**Purpose:** Allows strings longer than 255 characters by using 255 as a special marker.

### gtokenize (CommaText Encoding)

Converts multi-line text to CommaText format (quoted, comma-separated lines).

**Rules:**
- Empty lines become `""`
- Lines with spaces, commas, or special characters get quoted
- Quotes inside quoted strings are doubled (`"` becomes `""`)
- Lines without special characters don't need quotes

**Example:**
```
function test()
  this.chat = "Hello";
```

Becomes:
```
"function test()","  this.chat = ""Hello"";",""
```

### gtokenizeReverse (CommaText Decoding)

Converts CommaText format back to multi-line text.

**Rules:**
- Track whether you're inside quotes
- `""` inside quotes = escaped quote (single `"`)
- Commas outside quotes = line separator
- Strip outer quotes from each line

### GInt3 / Int24 Encoding

Used for NPC IDs and some other 24-bit values.

**Format:** 3 bytes, each holding 7 bits (21 bits total, but commonly called Int24)

**To encode:**
- byte1 = ((value >> 14) & 0xFF) + 32
- byte2 = ((value >> 7) & 0x7F) + 32
- byte3 = (value & 0x7F) + 32

**To decode:**
- value = ((byte1 - 32) << 14) + ((byte2 - 32) << 7) + (byte3 - 32)

## Common Problems

**Problem: Can't read player properties correctly**
- **Solution:** Make sure you're reading each property according to its type. Property 4 (ARROWSCOUNT) is a GByte, not a GString. Property 13 (COLORS) is 5 bytes, not 2.

**Problem: File download is corrupted**
- **Solution:** Make sure you're stripping the trailing 0x0A from packet 100's inner packet. Also check that you're using the full file path (from packet 92) as the key, not just the basename from packet 68.

**Problem: Admin message has garbage at the start**
- **Solution:** Don't add extra data to packet 63. Just send the raw message bytes.

**Problem: Can't send admin message to multiple players**
- **Solution:** Packet 64 only supports one player. Send separate packets for each player.

**Problem: Rights value is wrong**
- **Solution:** Make sure you're reading GInt5 (5 bytes), not 8 raw bytes. The readLong() function in some implementations reads 5 GByte-encoded values.

**Problem: File size or timestamp is way too big or negative**
- **Solution:** You're probably reading it as 8 raw bytes instead of 5 GByte-encoded values. Use GInt5 encoding.

**Problem: Packet 66 folder listing doesn't work**
- **Solution:** Check if the data is BZIP2 compressed (starts with 'BZ'). Also, some servers send plain Graal encoding (not Format.DYNAMIC), so check the first byte - if it looks like a valid folder path length, it's probably plain encoding.

**Problem: Sending rights or comments returns no response**
- **Solution:** Use packet 84 (PLI_RC_PLAYERRIGHTSSET) for setting player rights and packet 86 (PLI_RC_PLAYERCOMMENTSSET) for setting comments. These are the canonical packet numbers confirmed in both IEnums.h and TPlayerRC.cpp. Do not route through packet 155 (PLI_RC_LARGEFILESTART) — that packet is for large file upload handling, not rights/comments.

## Complete Packet Reference

### Client to Server Packets (PLI - Player Input)

| ID | Name | Direction | Purpose | Payload format |
|----|------|-----------|---------|----------------|
| 0 | PLI_LEVELWARP | C→S | Warp to level | Level name (raw) |
| 1 | PLI_BOARDMODIFY | C→S | Modify board | Board data |
| 2 | PLI_PLAYERPROPS | C→S | Set own player properties / set RC nickname | Space byte + nickname (get1PlusTextNetString) |
| 6 | PLI_TOALL | C→S | Send message to all players | Message length (GByte) + message |
| 28 | PLI_PRIVATEMESSAGE | C→S | Send private message | Player count (GShort) + player IDs (GShort each) + message (CommaText) |
| 51 | PLI_RC_SERVEROPTIONSGET | C→S | Get server options | (empty) |
| 52 | PLI_RC_SERVEROPTIONSSET | C→S | Set server options | Options (CommaText) |
| 53 | PLI_RC_FOLDERCONFIGGET | C→S | Get folder configuration | (empty) |
| 54 | PLI_RC_FOLDERCONFIGSET | C→S | Set folder configuration | Config (CommaText) |
| 55 | PLI_RC_RESPAWNSET | C→S | Set respawn settings | Respawn data |
| 56 | PLI_RC_HORSELIFESET | C→S | Set horse life settings | Horse life data |
| 57 | PLI_RC_APINCREMENTSET | C→S | Set AP increment settings | AP increment data |
| 58 | PLI_RC_BADDYRESPAWNSET | C→S | Set baddy respawn settings | Baddy respawn data |
| 59 | PLI_RC_PLAYERPROPSGET | C→S | Get player properties (DEPRECATED — does nothing) | (empty) |
| 60 | PLI_RC_PLAYERPROPSSET | C→S | Set player properties by player ID (partially deprecated) | Player ID (GShort) + world name (GString, ignored) + props block (GByte length + props bytes) + flags (GShort count + GString each) + chests (GShort count + chest data each) + weapons (GByte count + GString each). The server reads this as `msgPLI_RC_PLAYERPROPSSET` which is functional but labeled "deprecated?" in the source. |
| 61 | PLI_RC_DISCONNECTPLAYER | C→S | Disconnect a player | Player ID (GShort) + reason (raw bytes) |
| 62 | PLI_RC_UPDATELEVELS | C→S | Force server to reload levels | Level count (GShort) + level names (GString each) |
| 63 | PLI_RC_ADMINMESSAGE | C→S | Admin message to all | Raw message bytes only |
| 64 | PLI_RC_PRIVADMINMESSAGE | C→S | Admin message to one player | Player ID (GShort) + raw message bytes |
| 65 | PLI_RC_LISTRCS | C→S | List RC clients (DEPRECATED — does nothing) | (empty) |
| 66 | PLI_RC_DISCONNECTRC | C→S | Disconnect RC client (DEPRECATED — does nothing) | RC client ID |
| 67 | PLI_RC_APPLYREASON | C→S | Apply reason/note (DEPRECATED — does nothing) | Reason data |
| 68 | PLI_RC_SERVERFLAGSGET | C→S | Get server flags | (empty) |
| 69 | PLI_RC_SERVERFLAGSSET | C→S | Set server flags | Flag count (GShort) + flags (GString each) |
| 70 | PLI_RC_ACCOUNTADD | C→S | Create new account | Account name (GString) + password (GString) + email (GString) + banned (GByte) + load-only/guest (GByte) + admin level (GByte, deprecated/ignored) |
| 71 | PLI_RC_ACCOUNTDEL | C→S | Delete account | Account name (raw bytes) |
| 72 | PLI_RC_ACCOUNTLISTGET | C→S | Get account list | Name pattern (GString) + conditions (GString) |
| 73 | PLI_RC_PLAYERPROPSGET2 | C→S | Get player properties by ID | Player ID (GShort) |
| 74 | PLI_RC_PLAYERPROPSGET3 | C→S | Get player properties by account | Account name (GString) + empty byte (0x20) |
| 75 | PLI_RC_PLAYERPROPSRESET | C→S | Reset player properties | Account name (raw bytes) |
| 76 | PLI_RC_PLAYERPROPSSET2 | C→S | Set player properties by account | Account name (GString) + world (GString) + props length (GByte) + props + flags (GShort + GString each) + chests (GShort + GString each) + weapons (GByte + GString each) |
| 77 | PLI_RC_ACCOUNTGET | C→S | Get account data | Account name (raw bytes) |
| 78 | PLI_RC_ACCOUNTSET | C→S | Set account data | Account name (GString) + password (GString) + email (GString) + banned (GByte) + load-only/guest (GByte) + admin_level (GByte, deprecated/ignored) + world (GString, ignored) + ban_reason (GString) |
| 79 | PLI_RC_CHAT | C→S | Send RC chat message | Message (raw bytes) |
| 80 | PLI_PROFILEGET | C→S | Get player profile | Account name (raw bytes) |
| 81 | PLI_PROFILESET | C→S | Set player profile | Account name (GString) + fields 1-10 (GString each) |
| 82 | PLI_RC_WARPPLAYER | C→S | Warp player | Player ID (GShort) + X (GByte ×2) + Y (GByte ×2) + level name (raw bytes) |
| 83 | PLI_RC_PLAYERRIGHTSGET | C→S | Get player rights | Account name (raw bytes) |
| 84 | PLI_RC_PLAYERRIGHTSSET | C→S | Set player rights | Account name (GString) + rights (GInt5) + IP range (GString) + folder access length (GShort) + folder access (gtokenized newline-separated folder rules) |
| 85 | PLI_RC_PLAYERCOMMENTSGET | C→S | Get player comments | Account name (raw bytes) |
| 86 | PLI_RC_PLAYERCOMMENTSSET | C→S | Set player comments | Account name (GString) + comments (CommaText) |
| 87 | PLI_RC_PLAYERBANGET | C→S | Get player ban info | Account name (raw bytes) |
| 88 | PLI_RC_PLAYERBANSET | C→S | Set player ban | Account name (GString) + banned flag (GByte: 0=unban, non-zero=ban) + ban reason (raw bytes, remainder of packet) |
| 89 | PLI_RC_FILEBROWSER_START | C→S | Open file browser | (empty) |
| 90 | PLI_RC_FILEBROWSER_CD | C→S | Change folder | Folder path (raw bytes) |
| 91 | PLI_RC_FILEBROWSER_END | C→S | Close file browser | (empty) |
| 92 | PLI_RC_FILEBROWSER_DOWN | C→S | Download file | File path (raw bytes) |
| 93 | PLI_RC_FILEBROWSER_UP | C→S | Upload file | Filename (GString) + content — wrapped in packet 100 for large files |
| 94 | PLI_NPCSERVERQUERY | C→S | Request NC server address | NPC server player ID (GShort) + "location" (raw bytes) |
| 96 | PLI_RC_FILEBROWSER_MOVE | C→S | Move file | Destination directory (GString) + file name (raw bytes, remainder of packet); source is current browser directory + filename |
| 97 | PLI_RC_FILEBROWSER_DELETE | C→S | Delete file | File path (raw bytes) |
| 98 | PLI_RC_FILEBROWSER_RENAME | C→S | Rename file | Old name (GString) + new name (GString) |
| 100 | PLI_RAWDATA | C→S | Raw data wrapper | Length (GInt24) + 0x0A + inner packet |
| 130 | PLI_REQUESTUPDATEBOARD | C→S | Request board update | Level name + modtime (GInt5) + x/y/width/height (GShort each) |
| 152 | PLI_REQUESTTEXT | C→S | Request data from server | Request string (CommaText) |
| 154 | PLI_SENDTEXT | C→S | Send data to server | Data string (CommaText) |
| 155 | PLI_RC_LARGEFILESTART | C→S | Large file upload start | (used internally) |
| 156 | PLI_RC_LARGEFILEEND | C→S | Large file upload end | (used internally) |
| 157 | PLI_UPDATEGANI | C→S | Request GANI update | GANI data |
| 158 | PLI_UPDATESCRIPT | C→S | Request script from server | Script name |
| 159 | PLI_UPDATEPACKAGEREQUESTFILE | C→S | Request package file | Order (GByte) + filename + modtime (GInt5) |
| 160 | PLI_RC_FOLDERDELETE | C→S | Delete folder | Folder path (raw bytes) |
| 161 | PLI_UPDATECLASS | C→S | Request class update | Modtime (GInt5) + class name |

### Server to Client Packets (PLO - Player Output)

| ID | Name | Direction | Purpose | Payload format |
|----|------|-----------|---------|----------------|
| 0 | PLO_LEVELBOARD | S→C | Level board data | Board data |
| 1 | PLO_LEVELLINK | S→C | Level link data | Link data |
| 5 | PLO_LEVELSIGN | S->C | Level sign data | Sign data |
| 7 | PLO_BOARDMODIFY | S->C | Board modification | Board modification data |
| 6 | PLO_LEVELNAME | S→C | Current level name | Level name (raw) |
| 8 | PLO_OTHERPLPROPS | S→C | Other player properties | Player ID (GShort) + properties |
| 9 | PLO_PLAYERPROPS | S→C | Own player properties | Properties |
| 13 | PLO_TOALL | S→C | "To all" message | Player ID (GShort) + message length (GByte) + message |
| 14 | PLO_PLAYERWARP | S→C | Player warp response | Warp data |
| 15 | PLO_WARPFAILED | S→C | Warp failed | Failure reason |
| 16 | PLO_DISCMESSAGE | S→C | Disconnect message | Message (raw) |
| 25 | PLO_SIGNATURE | S→C | Authentication success | GByte value `73` (0x49) — historically indicates "more than 8 players possible". Presence of packet = auth success; payload byte can be ignored. |
| 30 | PLO_FILESENDFAILED | S→C | File send failed | Error message |
| 35 | PLO_RC_ADMINMESSAGE | S→C | Admin message display | Admin message data |
| 37 | PLO_PRIVATEMESSAGE | S→C | Private message received | Player ID (GShort) + CommaText sender + message |
| 39 | PLO_LEVELMODTIME | S→C | Level modification time | Level + modtime |
| 42 | PLO_NEWWORLDTIME | S→C | World time update | Time data (GInt4) |
| 45 | PLO_FILEUPTODATE | S→C | File is up to date | File info |
| 47 | PLO_STAFFGUILDS | S→C | Staff guilds list | Guilds (CommaText) |
| 48 | PLO_TRIGGERACTION | S→C | Trigger action response | Action data |
| 49 | PLO_PLAYERWARP2 | S→C | Player warp response (v2) | X/Y/Z + gmap level X/Y |
| 51 | PLO_RC_ACCOUNTSTATUS | S→C | Account status | Status data |
| 52 | PLO_RC_ACCOUNTNAME | S→C | Account name response | Account name |
| 53 | PLO_RC_ACCOUNTDEL | S→C | Account deleted confirmation | Account name |
| 55 | PLO_ADDPLAYER | S→C | Player joined | Player ID (GShort) + account name length (GByte) + account name + selected player properties (PLPROP_CURLEVEL, PLPROP_PSTATUSMSG, PLPROP_NICKNAME, PLPROP_COMMUNITYNAME each preceded by their PLPROP ID byte) |
| 56 | PLO_DELPLAYER | S→C | Player left | Player ID (GShort) |
| 61 | PLO_RC_SERVERFLAGSGET | S→C | Server flags response | Flag count (GShort) + flag strings (GString each, format "name=value") |
| 62 | PLO_RC_PLAYERRIGHTSGET | S→C | Player rights response | Account (GString) + rights (GInt5) + IP range (GString) + folder access length (GShort, always present) + folder access (gtokenized folder rules) |
| 63 | PLO_RC_PLAYERCOMMENTSGET | S→C | Player comments response | Account (GString) + comments (raw bytes, no length prefix, no CommaText encoding) |
| 64 | PLO_RC_PLAYERBANGET | S→C | Player ban info response | Account (GString) + banned flag (GByte: 1=banned) + ban reason (raw bytes, no length prefix) |
| 65 | PLO_RC_FILEBROWSER_DIRLIST | S→C | Folder list | Folder rules (gtokenized/CommaText — each decoded line: "rights pattern"; no length prefix, fills remainder of packet) |
| 66 | PLO_RC_FILEBROWSER_DIR | S→C | Folder contents or embedded file chunk | Folder path (GString) + for each file: space byte + [GByte entry_length + [filename GString + rights GString + size GInt5 + modtime GInt5]] — or embedded bigfile protocol |
| 67 | PLO_RC_FILEBROWSER_MESSAGE | S→C | File browser log/message | Message (CommaText) |
| 68 | PLO_LARGEFILESTART | S→C | Large file transfer start | Filename (raw bytes, basename only) |
| 69 | PLO_LARGEFILEEND | S→C | Large file transfer end | Filename (raw bytes, basename only) |
| 70 | PLO_RC_ACCOUNTLISTGET | S→C | Account list response | Account names (GString each, repeated) |
| 71 | PLO_RC_PLAYERPROPS | S→C | Deprecated — server no longer sends this | Per IEnums.h: "Deprecated. Codr note: Unhandled by 6.037." No handler exists in current RC clients. |
| 72 | PLO_RC_PLAYERPROPSGET | S→C | Player properties response | Player ID (GShort) + account (GString) + world (GString, hardcoded "main") + props block length (GByte) + props + flag count (GShort) + flags (GByte len + "name=value" bytes each) + chest count (GShort) + chests (GByte len + [X GByte][Y GByte][id bytes] each) + weapon count (GByte) + weapons (GByte len + name bytes each) |
| 73 | PLO_RC_ACCOUNTGET | S→C | Account data response | Account (GString) + password (GByte 0 — always empty) + email (GString) + banned (GByte) + guest/loadOnly (GByte) + admin_level (GByte 0 — always zero) + admin_worlds (GString, hardcoded "main") + ban_length (GString) + ban_reason (GString) |
| 74 | PLO_RC_CHAT | S→C | RC chat message | Message (raw) |
| 75 | PLO_PROFILE | S→C | Player profile response | Profile fields (GString each): account, real name, age, sex, country, messenger, email, homepage, hangout, quote, online time, [server extras...] |
| 76 | PLO_RC_SERVEROPTIONSGET | S→C | Server options response | Options (CommaText) |
| 77 | PLO_RC_FOLDERCONFIGGET | S→C | Folder config response | Config (CommaText) |
| 78 | PLO_NC_CONTROL | S→C | NC server query echo | Raw echo of request (format unknown — debug required) |
| 79 | PLO_NPCSERVERADDR | S→C | NC server address | Server ID (GShort) + "host,port" (raw bytes) |
| 82 | PLO_SERVERTEXT | S→C | Server text message | Text data (CommaText) |
| 84 | PLO_LARGEFILESIZE | S→C | Large file size | File size (GInt5) |
| 100 | PLO_RAWDATA | S→C | Raw data wrapper | Length (GInt24) + 0x0A + inner packet |
| 101 | PLO_BOARDPACKET | S->C | Board packet | Board packet data |
| 102 | PLO_FILE | S→C | File data chunk | Timestamp (GInt5) + filename (GString) + content |
| 103 | PLO_RC_MAXUPLOADFILESIZE | S→C | Max upload size | Max size (GInt5) — server sends `>> (long long)(1048576 * 20)` = 20 MiB default |
| 180 | PLO_STATUSLIST | S→C | Status list | Status data |
| 190 | PLO_UNKNOWN190 | S→C | Triggers client IRC/listserver initialization | No payload. Per IEnums.h: "Sending this packet makes the client login to IRC, request bantypes, pmguilds, pmservers, globalpms, buddytracking and stuff... Mainly tied to setTex[t]." Sent during RC and client login after the weapon list. |
| 194 | PLO_CLEARWEAPONS | S→C | Clears client weapon list | No payload. The server sends this to RC clients during login (before sending the player list) and to regular clients. IEnums.h confirms: `PLO_CLEARWEAPONS = 194`. |

## Summary

RC is the main control interface:
- Handles players, files, accounts, and server settings
- Uses special encoding (GByte, GShort, GInt, GString, GInt5)
- Packets can be compressed and encrypted
- Different properties have different data types
- File operations need special handling for large files
- Account and player data use CommaText for multi-line fields

The key things to remember:
1. Always check property types before reading
2. Large files use packet 100 wrapper - strip the trailing 0x0A!
3. GInt5 is 5 bytes, not 8
4. Admin messages are just raw bytes, no extra data
5. Some data uses CommaText encoding (multi-line text)
6. Packet 60 (PLI_RC_PLAYERPROPSSET) works by player ID; packet 76 (PLI_RC_PLAYERPROPSSET2) works by account name and supports offline players — prefer packet 76. Use packet 2 for setting the RC client's own nickname.
7. The encryption key is 0x56, set after sending the login packet
