# NC (NPC Control) Protocol

## What is NC?

NC stands for "NPC Control". It's a separate connection from the main game server that handles all the script-related stuff: weapons, classes, and NPCs (Non-Player Characters).

Think of it like this: The main game server (RC) is like the front desk of a hotel - it handles players, chat, warps, and general game stuff. The NC server is like the maintenance room - it handles all the scripts that make weapons work, classes function, and NPCs behave.

## Why is NC Separate?

NC is separate from the main game server for a few reasons:

1. **Security:** Scripts are sensitive. You don't want just anyone to be able to upload scripts. By separating it, servers can control who gets access to script management.

2. **Performance:** Script operations can be heavy. By having a separate connection, the main game server doesn't get slowed down by script uploads and downloads.

3. **Organization:** It keeps things clean. Game server stuff stays on the game server, script stuff stays on the NC server.

## How to Connect to NC

You can't just connect directly to the NC server. You have to go through the game server first. Here's the process:

```
1. Connect to game server (RC)
2. Log in and get authenticated
3. Look for a special player called "(npcserver)"
4. Ask that player for the NC server address
5. Connect to the NC server using that address
6. Log in to NC with your account and password
```

## Finding the NPC Server Address

This is the tricky part that trips up a lot of people.

**The Problem:**
When you connect to a game server, you'll see players joining. One of these players will be named "(npcserver)" - this is a special system player, not a real person. You need to remember this player's ID number.

**How to find it:**
When the server sends you packet 55 (PLO_ADDPLAYER - which means "a player joined"), you'll see something like:
- Player ID: 3
- Player name: "(npcserver)"

You need to save that player ID (in this example, it's 3).

**Why this matters:**
When you ask for the NC server address, you must send the request TO the npcserver player. If you use the wrong player ID (like always using player ID 2), the server will ignore your request.

**The request:**
You send packet 94 (PLI_NPCSERVERQUERY) with:
- The npcserver player's ID (not a hardcoded number!)
- The word "location"

The server will then send you back packet 79 (PLO_NPCSERVERADDR) with the NC server address in the format "host,port" (like "198.27.86.142,14801").

**Special case:**
If the address is "127.0.0.1", that means the NC server is on the same computer as the game server. In this case, use the game server's IP address instead.

## Connecting to NC

Once you have the address, you connect to the NC server.

**The login process:**
1. Connect to the host and port you received
2. Send a login packet with:
   - The protocol version: "NCL21075"
   - Your account name (with length)
   - Your password (with length)
3. The server will authenticate you (sends packet 25 - PLO_SIGNATURE)
4. **The server automatically sends you:**
   - All database NPCs (packet 158 - PLO_NC_NPCADD for each NPC)
   - All classes (packet 163 - PLO_NC_CLASSADD for each class)
5. You're now connected to NC

**Important:** NC uses ZLIB compression with seed 0 (no encryption scrambling, just compression).

**Important:** Make sure to listen for and store all the class names from packet 163 when you first connect - this is how you get the list of available classes!

## Weapons

Weapons are scripts that players can use in the game. Think of them like tools - a sword, a bow, a magic staff, etc.

### Getting the Weapon List

To see what weapons exist on the server, you send packet 115 (PLI_NC_WEAPONLISTGET). This packet has no data - it's just the packet ID.

**Example:**
```
Packet 115 (PLI_NC_WEAPONLISTGET):
Byte 0: 0x93 (115 + 32 = 147)
Byte 1: 0x0A (end marker)
```

The server responds with packet 167 (PLO_NC_WEAPONLISTGET), which contains a list of all weapon names. Each weapon name is stored as:
- One byte for the name length (minus 32)
- The weapon name itself

You keep reading weapon names until you run out of data.

**Example Response:**
```
Weapons: "sword", "bow", "staff"

Packet 167 (PLO_NC_WEAPONLISTGET):
Byte 0: 0xE7 (167 + 32 = 199)
Byte 1: 0x25 (5 + 32 = 37, "sword" length)
Bytes 2-6: 0x73 0x77 0x6F 0x72 0x64 ("sword")
Byte 7: 0x23 (3 + 32 = 35, "bow" length)
Bytes 8-10: 0x62 0x6F 0x77 ("bow")
Byte 11: 0x27 (5 + 32 = 37, "staff" length)
Bytes 12-16: 0x73 0x74 0x61 0x66 0x66 ("staff")
Byte 17: 0x0A (end marker)
```

### Getting a Weapon Script

To download a weapon script, you send packet 116 (PLI_NC_WEAPONGET) with the weapon name as a raw string (no length prefix). The 0x0A at the end is the standard packet terminator, not a mid-packet separator.

**Example:**
```
Weapon: "sword"

Packet 116 (PLI_NC_WEAPONGET):
Byte 0: 0x94 (116 + 32 = 148)
Bytes 1-5: 0x73 0x77 0x6F 0x72 0x64 ("sword")
Byte 6: 0x0A (packet terminator)
```

The server responds with packet 192 (PLO_NC_WEAPONGET), which contains:
- The weapon name (length byte + name)
- The weapon image name (length byte + image name)
- The weapon script

**Note:** There is no unknown byte to skip at the start of this packet. The very first byte IS the weapon name length (a GByte-encoded Int8). Earlier documentation that said "skip one unknown byte" was incorrect. Read `name_length = readInt8()`, then `name = readString(name_length)`, then `image_length = readInt8()`, then `image = readString(image_length)`, then `script = readWString()`.

**Important:** Weapon scripts use a special encoding called "WString". In this format, newlines in the script are replaced with the byte 0xA7. When you read the script, you need to replace every 0xA7 byte with a newline (the Java reference client decodes 0xA7 as `\r\n`; for display purposes `\n` alone is also acceptable).

**Example:**
```
Original script:
function test() {
  player.chat = "Hello";
}

WString encoded (hex):
66 75 6E 63 74 69 6F 6E 20 74 65 73 74 28 29 20 7B A7 20 20 70 6C 61 79 65 72 2E 63 68 61 74 20 3D 20 22 48 65 6C 6C 6F 22 3B A7 7D

Decode by replacing 0xA7 with \n:
function test() {
  player.chat = "Hello";
}
```

**Hex breakdown:**
```
66 75 6E 63 74 69 6F 6E 20 74 65 73 74 28 29 20 7B
"function test() {"

A7
\n (newline)

20 20 70 6C 61 79 65 72 2E 63 68 61 74 20 3D 20 22 48 65 6C 6C 6F 22 3B
"  player.chat = \"Hello\";"

A7
\n (newline)

7D
"}"
```

### Uploading a Weapon

To upload a weapon script, you send packet 117 (PLI_NC_WEAPONADD) with:
- Weapon name length (GByte) + weapon name (truncated to 223 bytes if longer)
- Image name length (GByte) + image name (truncated to 223 bytes if longer)
- The script in WString format (newlines replaced by 0xA7, carriage returns stripped)

**Example weapon script:**

```gscript
//weapon script with a triggeraction
//Weapon name: Fire Ball
function onActionserverside() {
  if (params[0] == "mp") {
    player.mp -= 10;
  }
}

//#CLIENTSIDE
function onWeaponfired() {
  if (player.mp >= 10) {
    shootfireball(playerdir);
    triggeraction(0, 0, "serverside", "Fireball", "mp");
  } else {
    player.chat = "Not enough mp!";
  }
}
```

**Important:** When uploading, replace all newline characters (`\n`) with the byte `0xA7`. So the above script would become:
```
//weapon script with a triggeraction\xA7//Weapon name: Fire Ball\xA7function onActionserverside() {\xA7  if (params[0] == "mp") {\xA7    player.mp -= 10;\xA7  }\xA7}\xA7\xA7//#CLIENTSIDE\xA7function onWeaponfired() {\xA7  if (player.mp >= 10) {\xA7    shootfireball(playerdir);\xA7    triggeraction(0, 0, "serverside", "Fireball", "mp");\xA7  } else {\xA7    player.chat = "Not enough mp!";\xA7  }\xA7}
```

### Deleting a Weapon

To delete a weapon, send packet 118 (PLI_NC_WEAPONDELETE) with just the weapon name (raw string, no length prefix).

## Classes

Classes are like templates for NPCs. They define how an NPC behaves. Think of a class like a blueprint - you can create many NPCs from the same class.

### Finding Available Classes

When you connect to the NC server and authenticate, the server **automatically sends you all available classes** using packet 163 (PLO_NC_CLASSADD). You'll receive one packet 163 for each class on the server.

**Format of packet 163:**
- Packet ID: 163 (PLO_NC_CLASSADD)
- Class name (string, ends with newline)

**Important:** In GServer-v2, all PLO_NC_CLASSADD entries are sent as a **single bundled packet** rather than as separate packets. The server builds one `classPacket` CString and appends all class names to it as `>> (char)PLO_NC_CLASSADD << name << "\n"` in a single `sendPacket()` call. The result is multiple PLO_NC_CLASSADD records concatenated in one network packet. Your parser must handle this by reading PLO_NC_CLASSADD records from the packet payload until data is exhausted.

**Example:** If the server has classes "tree", "player", and "house_door", the server sends one packet containing:
- `[163]["tree\n"][163]["player\n"][163]["house_door\n"]` all in a single network payload.

This happens automatically right after NC authentication - you don't need to request it. Just listen for packet 163 and collect all the class names as they arrive.

### Getting a Class Script

To download a class script, you send packet 112 (PLI_NC_CLASSEDIT) with the class name as a raw string (no length prefix). The 0x0A that follows is the standard packet terminator.

The server responds with packet 162 (PLO_NC_CLASSGET), which contains:
- Class name length + class name
- The class script in "CommaText" format

**Important:** Class scripts use a completely different encoding than weapons. They use "CommaText" format.

### CommaText Format

CommaText is a way of storing multi-line text where each line is quoted and separated by commas. It looks like this:

```
"line1","line2","line with ""quotes""",""
```

**How to read it:**
1. Each line is wrapped in quotes
2. Lines are separated by commas
3. If a line contains a quote, it's doubled ("" means a single quote)
4. Empty lines are just "" (two quotes with nothing between them)

**Example:**
```
"function onCreated()","  this.chat = \"Hello\";",""
```

Decodes to:
```
function onCreated()
  this.chat = "Hello";

```

**The tricky part:**
- You need to track whether you're inside quotes or not
- When you see `""`, that's an escaped quote (not the end of the string)
- When you see `,` outside of quotes, that's the end of one line and start of another
- Strip the outer quotes from each line
- Replace `""` with `"` in the final text

### Uploading a Class

To upload a class, send packet 113 (PLI_NC_CLASSADD) with:
- Class name length (GByte) + class name (truncated to 223 bytes if longer)
- The script converted to CommaText format

**Example class script:**

```gscript
//#CLIENTSIDE
function onCreated() {
  setimg("tree.png");
  setshape(1, 32, 32);
}

function onPlayerTouchsMe() {
  player.chat = "You touched a tree!";
}
```

**Important:** Classes use CommaText format. Convert the script like this:
- Each line goes in quotes
- Lines are separated by commas
- Escaped quotes inside lines become `""`

So the above class would become:
```
"//#CLIENTSIDE","function onCreated() {","  setimg(\"tree.png\");","  setshape(1, 32, 32);","}","","function onPlayerTouchsMe() {","  player.chat = \"You touched a tree!\";","}"
```

### Deleting a Class

To delete a class, send packet 119 (PLI_NC_CLASSDELETE) with just the class name (raw string, no length prefix).

**Note:** There's no "list all classes" command. You have to know the class name, or browse the file system to find class files.

## NPCs

NPCs are characters in the game that aren't players. They can be shopkeepers, enemies, quest givers, etc.

### Two Types of NPCs

**Database NPCs:**
These are stored in a database on the server. They persist across server restarts. Each one has a unique ID number (usually starting from 1000).

**Level NPCs:**
These are NPCs that only exist on a specific level. They're not in the database - they're part of the level file itself.

### Getting the NPC List

**Important:** Database NPCs are automatically sent to you when you connect to the NC server. After authentication, the server sends packet 158 (PLO_NC_NPCADD) for each database NPC. You don't need to request them - just listen for these packets during connection.

Each packet 158 (PLO_NC_NPCADD) sent during login contains (from `sendLoginNC()` in TPlayerLogin.cpp):

```
[NPC ID: GInt (4 bytes, 7-bit per byte)]    -- written as >> (int)npc->getId()
[NPCPROP_NAME property: GByte ID + GString name]
[NPCPROP_TYPE property: GByte ID + GString type]
[NPCPROP_CURLEVEL property: GByte ID + GString level]
```

**Correction:** The NPC ID in PLO_NC_NPCADD is encoded as a **4-byte GInt** (using the standard GInt encoding: 4 × 7-bit bytes each +32), not as the Int24 (3-byte) encoding. The Int24 encoding applies to packets that take an NPC ID as a request argument (PLI_NC_NPCGET, etc.), but the server's `>> (int)npc->getId()` operator in the add packet uses GInt (4 bytes). Verify using the actual packet length in your client.

**NPC ID encoding:**
NPC IDs are stored as 3 bytes using a special encoding:
- First byte: (ID >> 14) + 32
- Second byte: ((ID >> 7) & 0x7F) + 32
- Third byte: (ID & 0x7F) + 32

To decode:
- ID = ((byte1 - 32) << 14) + ((byte2 - 32) << 7) + (byte3 - 32)

**Example:**
```
NPC ID: 1003

Encode:
  Byte 1: (1003 >> 14) + 32 = 0 + 32 = 32 (0x20)
  Byte 2: ((1003 >> 7) & 0x7F) + 32 = (7 & 0x7F) + 32 = 39 (0x27)
  Byte 3: (1003 & 0x7F) + 32 = 107 + 32 = 139 (0x8B)

Encoded: 0x20 0x27 0x8B

Decode:
  ID = ((0x20 - 0x20) << 14) + ((0x27 - 0x20) << 7) + (0x8B - 0x20)
     = (0 << 14) + (7 << 7) + 107
     = 0 + 896 + 107
     = 1003 ✓
```

### Getting an NPC Script

To download an NPC's script, send packet 106 (PLI_NC_NPCSCRIPTGET) with the NPC ID (3 bytes, encoded as above).

The server responds with packet 160 (PLO_NC_NPCSCRIPT), which contains:
- NPC ID (3 bytes)
- The script in CommaText format (same as classes)

### Getting NPC Attributes

NPC attributes are things like what level the NPC is on, its X and Y position, etc. These are NOT the script - they're just properties.

To get attributes, send packet 103 (PLI_NC_NPCGET) with the NPC ID.

The server responds with packet 157 (PLO_NC_NPCATTRIBUTES), which contains the attributes in CommaText format. Each line looks like:
```
.level:onlinestartlocal.nw
.xprecise:30.5
.yprecise:30
```

### Getting NPC Flags

NPC flags are special properties that control NPC behavior. To get them, send packet 108 (PLI_NC_NPCFLAGSGET) with the NPC ID.

The server responds with packet 161 (PLO_NC_NPCFLAGS), which contains the flags in CommaText format.

### Creating an NPC

To create a new database NPC, send packet 111 (PLI_NC_NPCADD) with a comma-separated string:
```
name,id,type,scripter,level,x,y
```

**Example:**
```
test,1003,OBJECT,Denveous,onlinestartlocal.nw,30.5,30
```

**Important:** NPC IDs usually start at 1000. Avoid using 10000, as that's reserved for a special Control-NPC.

**Important:** This packet must be sent over the NC connection, NOT the RC connection. If you send it over RC, the server will disconnect you with an "invalid data" error.

### Setting an NPC Script

To upload an NPC script, send packet 109 (PLI_NC_NPCSCRIPTSET) with:
- NPC ID (3 bytes)
- The script converted to CommaText format

**Example NPC script:**

```gscript
function onCreated() {
  // Initialize database NPC
  this.playerCount = 0;
}

function onPlayerLogin(pl) {
  // Handle player login event
  this.playerCount++;
  echo(pl.account @ " logged in. Total: " @ this.playerCount);
}
```

**Important:** NPC scripts use the same CommaText format as classes. Convert each line to quoted, comma-separated format:

```
"function onCreated() {","  // Initialize database NPC","  this.playerCount = 0;","}","","function onPlayerLogin(pl) {","  // Handle player login event","  this.playerCount++;","  echo(pl.account @ \" logged in. Total: \" @ this.playerCount);","}"
```

**Note:** Database NPCs are typically serverside-only and handle server events like `onPlayerLogin`, `onPlayerChat`, `onPlayerDisconnect`, etc. They don't need `//#CLIENTSIDE` because they're not visible in-game.

### Setting NPC Flags

To set NPC flags, send packet 110 (PLI_NC_NPCFLAGSSET) with:
- NPC ID (3 bytes)
- The flags in CommaText format

### Resetting an NPC

To reset an NPC back to its default state, send packet 105 (PLI_NC_NPCRESET) with just the NPC ID (3 bytes).

### Deleting an NPC

To delete an NPC from the database, send packet 104 (PLI_NC_NPCDELETE) with just the NPC ID (3 bytes).

### Warping an NPC

To move an NPC to a different location, send packet 107 (PLI_NC_NPCWARP) with:
- NPC ID (Int24, 3 bytes)
- X coordinate (GByte: integer tile coordinate × 2)
- Y coordinate (GByte: integer tile coordinate × 2)
- Level name (raw string, no length prefix)

**Coordinate encoding:**
The X and Y coordinates are in "tiles" (where 1 tile = 16 pixels). To encode them:
- Multiply the tile coordinate by 2
- Encode as a single byte (value + 32)

**Example:**
If you want to warp to X = 30.5 tiles:
- 30.5 × 2 = 61
- Encode 61 as a byte: 61 + 32 = 93 (0x5D)

The server will divide by 2.0 to get 30.5 tiles, then multiply by 16 to get 488 pixels.

**Why multiply by 2?**
This gives you half-tile precision. Without it, you could only warp to whole tiles (30, 31, 32, etc.). With it, you can warp to 30.5, 31.5, etc.

### Level NPCs

Level NPCs are different - they're part of the level file, not the database.

To get level NPCs, send packet 114 (PLI_NC_LOCALNPCSGET) with the level name.

The server responds with packet 164 (PLO_NC_LEVELDUMP), which contains a dump of all NPCs on that level.

### Level List Management

**Getting the level list:**
Send packet 150 (PLI_NC_LEVELLISTGET) with no data.

The server responds with packet 80 (PLO_NC_LEVELLIST) containing a list of all levels in CommaText format.

**Setting the level list:**
Send packet 151 (PLI_NC_LEVELLISTSET) with the level list in CommaText format.

This is used to update which levels are available on the server.

## Script Encoding Summary

This is important - each type of script uses a different encoding:

| Type | Packet | Encoding |
|------|--------|----------|
| Weapon | 192 | WString (0xA7 = newline) |
| Class | 159/162 | CommaText (quoted lines, comma-separated) |
| NPC | 160 | CommaText (same as classes) |

**Weapons:** Replace newlines with 0xA7 when uploading, replace 0xA7 with newlines when downloading.

**Classes and NPCs:** Use CommaText - convert lines to quoted, comma-separated format when uploading, parse quoted lines when downloading.

## Graal Sorting

When displaying weapons and classes, Graal uses a special sorting order:
- Symbols come before letters and numbers
- Within symbols, they're sorted normally
- Within letters/numbers, they're sorted normally

**Example order:**
```
!Setshapes
-Functions
Backpack
Sword
```

This is because symbols are treated as category 0, and alphanumeric as category 1. Category 0 always comes first.

## Common Problems

**Problem: Can't get NPC server address**
- **Solution:** Make sure you're using the npcserver player's ID, not a hardcoded number. Find the player with name "(npcserver)" and use their ID.

**Problem: Server disconnects when creating NPC**
- **Solution:** Make sure you're sending NPC creation packets over the NC connection, not the RC connection.

**Problem: Weapon script has weird characters instead of newlines**
- **Solution:** You forgot to replace 0xA7 bytes with newlines when reading the script.

**Problem: Class script is all on one line or has extra quotes**
- **Solution:** You're not parsing CommaText correctly. Make sure you're handling quoted strings and escaped quotes ("").

**Problem: Can't find a class**
- **Solution:** Classes are automatically sent to you when you connect to the NC server. After authentication, the server sends packet 163 (PLO_NC_CLASSADD) for each class. Make sure you're listening for these packets and storing the class names as they arrive. If you missed them during connection, you'll need to reconnect to get the class list again.

**Problem: NPC warp goes to wrong position**
- **Solution:** Make sure you're multiplying tile coordinates by 2 before encoding them as bytes.

## Complete NC Packet Reference

### Client to Server Packets (NC)

| ID | Name | Purpose | Format |
|----|------|---------|--------|
| 3 | PLI_NPCPROPS | Authenticate with NC server | "NCL21075" (raw, 8 bytes, no length prefix) + account length (GByte) + account + password length (GByte) + password |
| 103 | PLI_NC_NPCGET | Get NPC data | NPC ID (Int24) |
| 104 | PLI_NC_NPCDELETE | Delete NPC | NPC ID (Int24) |
| 105 | PLI_NC_NPCRESET | Reset NPC | NPC ID (Int24) |
| 106 | PLI_NC_NPCSCRIPTGET | Get NPC script | NPC ID (Int24) |
| 107 | PLI_NC_NPCWARP | Warp NPC | NPC ID (Int24) + X (GByte, tile×2) + Y (GByte, tile×2) + level name (raw string, no length prefix) |
| 108 | PLI_NC_NPCFLAGSGET | Get NPC flags | NPC ID (Int24) |
| 109 | PLI_NC_NPCSCRIPTSET | Set NPC script | NPC ID (Int24) + script (CommaText) |
| 110 | PLI_NC_NPCFLAGSSET | Set NPC flags | NPC ID (Int24) + flags (CommaText) |
| 111 | PLI_NC_NPCADD | Create NPC | "name,id,type,scripter,level,x,y" (CommaText) |
| 112 | PLI_NC_CLASSEDIT | Get class script | Class name (raw string, no length prefix; 0x0A is the packet terminator) |
| 113 | PLI_NC_CLASSADD | Upload class script | Class name length (GByte) + name + script (CommaText); name is truncated to 223 bytes if longer |
| 114 | PLI_NC_LOCALNPCSGET | Get level NPCs | Level name |
| 115 | PLI_NC_WEAPONLISTGET | Get weapon list | (empty) |
| 116 | PLI_NC_WEAPONGET | Get weapon script | Weapon name (raw string, no length prefix; 0x0A is the packet terminator) |
| 117 | PLI_NC_WEAPONADD | Upload weapon script | Name length (GByte) + name + image length (GByte) + image + script (WString); name and image each truncated to 223 bytes if longer |
| 118 | PLI_NC_WEAPONDELETE | Delete weapon | Weapon name (raw string) |
| 119 | PLI_NC_CLASSDELETE | Delete class | Class name (raw string) |
| 150 | PLI_NC_LEVELLISTGET | Get level list | (empty) |
| 151 | PLI_NC_LEVELLISTSET | Set level list | Levels (CommaText) |

### Server to Client Packets (NC)

| ID | Name | Purpose | Format |
|----|------|---------|--------|
| 16 | PLO_DISCMESSAGE | Disconnect message | Message |
| 25 | PLO_SIGNATURE | Authentication signature | Signature data |
| 47 | PLO_STAFFGUILDS | Staff guilds list | Guilds (CommaText) |
| 74 | PLO_RC_CHAT | NC chat message | Message |
| 80 | PLO_NC_LEVELLIST | Level list response | Levels (CommaText) |
| 157 | PLO_NC_NPCATTRIBUTES | NPC attributes response | Attributes (CommaText) |
| 158 | PLO_NC_NPCADD | NPC added to list | NPC ID (GInt, 4 bytes) + NPCPROP_NAME (GByte ID + GString) + NPCPROP_TYPE (GByte ID + GString) + NPCPROP_CURLEVEL (GByte ID + GString) |
| 159 | PLO_NC_NPCDELETE | NPC deleted from list | NPC ID (Int24) |
| 160 | PLO_NC_NPCSCRIPT | NPC script response | NPC ID (Int24) + script (CommaText) |
| 161 | PLO_NC_NPCFLAGS | NPC flags response | NPC ID (Int24) + flags (CommaText) |
| 162 | PLO_NC_CLASSGET | Class script response | Class name length (GByte) + name + script (CommaText) |
| 163 | PLO_NC_CLASSADD | Class added to list (sent automatically on NC connect) | Class name followed by `\n` (newline). Multiple PLO_NC_CLASSADD records are bundled into a single network packet during login — parse all records from the packet payload. |
| 164 | PLO_NC_LEVELDUMP | Level NPC dump | Level NPCs (CommaText) |
| 167 | PLO_NC_WEAPONLISTGET | Weapon list response | Repeated: name length (GByte) + name; read until end of packet |
| 188 | PLO_NC_CLASSDELETE | Class deleted from list | Class name (raw string, all remaining bytes in packet payload) |
| 192 | PLO_NC_WEAPONGET | Weapon script response | name length (GByte) + name + image length (GByte) + image + script (WString) |

### Int24 Encoding (NPC IDs)

NPC IDs use a special 3-byte encoding:
- First byte: (ID >> 14) + 32
- Second byte: ((ID >> 7) & 0x7F) + 32
- Third byte: (ID & 0x7F) + 32

**To decode:**
- ID = ((byte1 - 32) << 14) + ((byte2 - 32) << 7) + (byte3 - 32)

**To encode:**
- byte1 = ((ID >> 14) & 0xFF) + 32
- byte2 = ((ID >> 7) & 0x7F) + 32
- byte3 = (ID & 0x7F) + 32

## Summary

NC is the script management system:
- Separate from the main game server
- Handles weapons, classes, and NPCs
- Uses different encoding for each script type
- Requires going through the game server to get the NC server address

The key things to remember:
1. Find the npcserver player ID when connecting to the game server
2. Use that ID to request the NC server address
3. Weapons use WString (0xA7 for newlines)
4. Classes and NPCs use CommaText (quoted, comma-separated)
5. NPC operations must go over NC, not RC
6. NPC IDs use Int24 encoding (3 bytes)

