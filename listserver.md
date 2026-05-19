# Listserver Protocol

## What is the Listserver?

The listserver is like a phone book for game servers. When you open your Remote Control client, it needs to know what servers are available to connect to. The listserver is a central computer that keeps track of all the game servers and sends you a list of them.

Think of it like this: You want to call a friend, but you don't know their phone number. You look it up in a phone book. The listserver is that phone book, but for game servers.

## How It Works

The listserver connection is very simple and short-lived. Here's what happens:

1. Your client connects to the listserver
2. Your client says "Hi, I'm a Remote Control client, here are my login details"
3. The listserver checks if you're allowed to connect
4. The listserver sends you a list of all available servers
5. The listserver says "Welcome!" and then immediately closes the connection

The whole thing takes about one second. This is normal - the listserver doesn't stay connected. It just gives you the list and then closes.

## Connection Details

**Where to connect:**
- Host: `listserver.graalonline.com`
- Port: `14922`

**Important:** The port is 14922, not 14900. Many people get this wrong.

## The Connection Process

Here's the step-by-step process:

```
Client                          Listserver
  |                                 |
  |--- Connect to port 14922 ------>|
  |                                 |
  |--- Send "Edition" packet ------>|
  |   (says who you are)            |
  |                                 |
  |--- Send "Login" packet -------->|
  |   (your account and password)   |
  |                                 |
  |<--- Server list packet ---------|
  |   (list of all servers)         |
  |                                 |
  |<--- Welcome message ------------|
  |                                 |
  |<--- Connection closed ----------|
  |   (this is normal!)             |
```

## The Edition Packet

Before you can log in, you need to tell the listserver what kind of client you are. This is called the "Edition" packet.

**What it contains:**
- Client type: The number 7 (identifies this as a Remote Control client)
- Seed: The number 4 (used for encryption)
- Version string: "G3D30123" (this tells the server what version of Graal you're using)
- Client suffix: "rc2" (identifies the Remote Control client variant)

**Important details:**
- The strings are NOT separated by null bytes (zero bytes)
- Everything is compressed with ZLIB before sending
- The version string must be "G3D30123" - other versions like "GSERV025" will be rejected

**Example Edition Packet:**

Raw payload (before compression):
```
Byte 0: 0x27 (7 + 32, GByte encoded)
Byte 1: 0x24 (4 + 32, GByte encoded)
Bytes 2-9: "G3D30123" (8 bytes, ASCII)
Bytes 10-12: "rc2" (3 bytes, ASCII)
Byte 13: 0x0A (packet terminator)
```

Complete packet structure:
```
1. Build the 14-byte payload as shown above
2. Compress the payload with ZLIB
3. Send: [2-byte big-endian length] + [compressed data]
   (No format byte is prepended — unlike the login packet)
```

**Example (hex representation of raw payload before compression):**
```
Raw: 27 24 47 33 44 33 30 31 32 33 72 63 32 0A
     ^^ ^^ ^^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^ ^^
     7  4  "G3D30123"              "rc2"     \n
```

**Common mistakes:**
- Using the wrong version string (like "GSERV025" instead of "G3D30123")
- Adding null bytes between strings (don't do this!)
- Using the wrong port number

## The Login Packet

After sending the edition packet, you send your login information.

**What it contains:**
- Packet type: 1 (this means "I want the server list")
- Your account name (with its length)
- Your password (with its length)

**Note on packet type:** The login packet type is **1**, not 0. The value 0 in the packet reference table refers to the server's response packet (SERVERLIST), not the client login type. This is a common source of confusion.

**Format:**
The login packet uses a special format called "DYNAMIC" format. This means:
- If the packet is small (less than 40 bytes), it's sent as plain text (format type 0x02)
- If the packet is larger (40 bytes or more), it's compressed with ZLIB (format type 0x04)
- Then it's encrypted using the scrambler algorithm
- The 2-byte length field includes the format byte itself

**Example Login Packet:**

Raw payload (before compression/encryption):
```
Byte 0: 0x21 (1 + 32, GByte encoded - packet type 1)
Byte 1: 0x2B (11 + 32, GByte encoded - account length)
Bytes 2-12: "testaccount" (11 bytes, ASCII)
Byte 13: 0x2A (10 + 32, GByte encoded - password length)
Bytes 14-23: "mypassword" (10 bytes, ASCII)
Byte 24: 0x0A (newline terminator)
```

**Example with account "testaccount" and password "mypassword":**
```
Raw payload: 21 2B 74 65 73 74 61 63 63 6F 75 6E 74 2A 6D 79 70 61 73 73 77 6F 72 64 0A
             ^^ ^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ ^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ ^^
             1  11 "testaccount"                10 "mypassword"                  \n
```

**Complete packet structure:**
```
1. If payload >= 40 bytes: Compress with ZLIB (format type 0x04)
   If payload < 40 bytes: Use as-is (format type 0x02)
2. Encrypt using scrambler algorithm with seed 4
3. Send: [2-byte length (includes format byte)] + [format type byte] + [encrypted/compressed data]
```

**Example (small packet, no compression):**
```
Length: 00 19 (25 bytes)
Format: 02 (no compression)
Data: [25 bytes of scrambled data]
```

**Example (large packet, with compression):**
```
Length: 00 2F (47 bytes)
Format: 04 (ZLIB compressed)
Data: [47 bytes of scrambled, compressed data]
```

## The Scrambler Algorithm

The listserver uses encryption to protect your login information. This is called the "scrambler" algorithm.

**How it works:**
Think of it like this: You have a secret code wheel. You start with a specific number (the "mask"). For every 4 bytes of data, you rotate the code wheel to a new position. Then you use that position to scramble the data.

**The process:**
1. Start with a starting number (the mask): `0x4A80B38`
2. The number of rotation groups depends on the format type:
   - Format 0x02 (no ZLIB): 12 rotation groups (encrypts up to the first 48 bytes)
   - Format 0x04 (ZLIB compressed): 4 rotation groups (encrypts up to the first 16 bytes)
3. For each group of 4 bytes:
   - Rotate the mask: `mask = (mask * 0x8088405 + seed) & 0xFFFFFFFF`
   - XOR each byte in the 4-byte group: `result[i] ^= (mask >> (8 * (i % 4))) & 0xFF`
4. Bytes beyond the last rotation group are passed through unchanged.

The same function is used for both encryption (sending) and decryption (receiving). XOR is its own inverse, so you apply it the same way in both directions.

**Constants summary:**
- Initial mask: `0x4A80B38`
- Rotation multiplier: `0x8088405`
- Seed (listserver): `4`
- Rotation groups for format 0x02: `12`
- Rotation groups for format 0x04: `4`

**Important:** The mask rotates every 4 bytes, NOT every single byte. This is a common mistake - people try to rotate it for every byte, which breaks everything.

**Why it matters:**
If you get the scrambler wrong, the server can't read your login packet. It will either reject you or send back garbage data that you can't understand.

## Receiving the Server List

After you send your login, the server sends back a response.

**The response format:**
1. 2-byte big-endian length (number of bytes that follow)
2. Format byte (first byte of those N bytes): usually 0x04 (ZLIB compressed and scrambled)
3. Remaining bytes: the encrypted and compressed server list payload

**To read it:**
1. Read 2 bytes as a big-endian unsigned integer to get the packet length
2. Read that many bytes into a buffer
3. The first byte of the buffer is the format type (0x04)
4. Pass the remaining bytes (buffer[1:]) through the scrambler with seed 4 and format 0x04 (4 rotation groups) to decrypt
5. ZLIB decompress the decrypted result
6. Parse the server list from the decompressed data

**The server list structure:**
- Packet type: 0 (means "here's the server list")
- Number of servers: How many servers are in the list
- For each server:
  - Number of attributes
  - Server name
  - Language
  - Description
  - URL
  - Version
  - Player count
  - IP address
  - Port number

**Example Server List Response:**

After unscrambling and decompressing:
```
Byte 0: 0x20 (0 + 32, GByte - packet type 0)
Byte 1: 0x21 (1 + 32, GByte - 1 server in list)
Byte 2: 0x28 (8 + 32, GByte - 8 attributes for this server)
Byte 3: 0x0C (12 + 32, GByte - server name length)
Bytes 4-15: "Test Server" (12 bytes)
Byte 16: 0x02 (2 + 32, GByte - language length)
Bytes 17-18: "en" (2 bytes)
Byte 19: 0x0D (13 + 32, GByte - description length)
Bytes 20-32: "A test server" (13 bytes)
... (continues with URL, version, player count, IP, port)
```

**Example (simplified, showing structure):**
```
20 21 28 0C 54 65 73 74 20 53 65 72 76 65 72 02 65 6E ...
^^ ^^ ^^ ^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ ^^ ^^
0  1  8  12 "Test Server"                     2  "en"
```

## Server List Example

After parsing, you might get something like:

```
Server: U Dev WanderingWitch
Type: Hidden
IP: 198.27.86.142
Port: 14713
Players: 9
```

The server type can be:
- P = Premium (Gold)
- H = Hosted
- 3 = 3D server
- U = Hidden/Unlisted

If there is no type in the name it's considered a "Classic" server, which are the main servers showcased. 

## The Welcome Message

After the server list, you'll receive a packet type 2, which is just a welcome message like "Welcome to Graal!". This is just a friendly message - you can ignore it.

## Connection Closed is Normal

After sending the server list and welcome message, the listserver closes the connection. This is expected behavior! Don't treat it as an error.

The listserver is designed to be a quick "give and go" service:
- You ask for the server list
- It gives you the list
- It closes the connection

You're supposed to save the server list in your client and use it to connect to individual game servers.

## Cross-Server Messaging

The listserver also helps with messaging between players on different servers. This is how you can send private messages to someone who's on a different server than you.

**How it works:**

1. **Requesting a player list from another server:**
   - You tell your game server: "Show me players on ServerB"
   - Your game server asks the listserver: "What players are on ServerB?"
   - The listserver forwards this to ServerB
   - ServerB sends back a list of its players
   - The listserver forwards this to your game server
   - Your game server shows you the players as "Nickname (on ServerB)"

2. **Sending a PM to someone on another server:**
   - You send a PM to "Nickname (on ServerB)"
   - Your game server sees this is an external player
   - Your game server sends the message to the listserver
   - The listserver forwards it to ServerB
   - ServerB delivers it to the player

**External player IDs:**
- Players from other servers get special IDs starting at 16000
- This keeps them separate from regular players on your server
- They show up in your player list with "(on ServerName)" after their name

## Common Problems and Solutions

**Problem: Server disconnects immediately with an error**
- **Solution:** Check your version string. It must be "G3D30123", not "GSERV025" or any other version.

**Problem: Can't decrypt the response - getting garbage data**
- **Solution:** Make sure your scrambler rotates the mask every 4 bytes, not every byte. Also check that you're using seed 4.

**Problem: Server list is empty after connecting**
- **Solution:** This might be normal if there are no servers, but more likely the connection closed before you could read the list. Make sure you're reading the response before the connection closes. Also, on retail servers, the list only shows servers you have RC rights on - if you have rights to no RC servers, the list will be empty. It could also indicate a listserver outage.

**Problem: Using port 14900 instead of 14922**
- **Solution:** Always use port 14922. Port 14900 is old and doesn't work anymore.

**Problem: Adding null bytes between strings in the edition packet**
- **Solution:** Don't add null bytes. The strings should be right next to each other with no separators.

## Listserver Packet Reference

### Client to Listserver

**Edition Packet:**
- Not a numbered packet — sent as raw data before any login
- Payload format: Client type (GByte 7 = 0x27) + seed (GByte 4 = 0x24) + "G3D30123" (8 bytes, raw ASCII) + "rc2" (3 bytes, raw ASCII) + 0x0A
- The strings are concatenated with no separators between them
- Wire format: ZLIB compress the entire payload, then send [2-byte big-endian length] + [compressed data]
- No format byte is sent with the edition packet (unlike the login packet which includes a format byte)
- Example payload before compression: 0x27 0x24 "G3D30123" "rc2" 0x0A

**Login Packet:**
- Packet type: 1 (PLI_SERVERLIST)
- Format: Packet type (GByte) + account length (GByte) + account + password length (GByte) + password + 0x0A
- Wire: [2-byte big-endian length (includes format byte)] + [format byte 0x02 or 0x04] + [scrambled data]
- Uses DYNAMIC format (compressed with ZLIB if payload >= 40 bytes, then scrambled with seed 4)

### Listserver to Client

| Packet Type | Name | Purpose | Format |
|-------------|------|---------|--------|
| 0 | SERVERLIST | Server list response | Server count (GByte) + server entries |
| 2 | WELCOME | Welcome message | Message text (raw string) |
| 4 | DISCONNECT | Disconnect message | Message text (raw string); treat as a rejection — abort connection |
| 16 | ERROR | Error message | Error text (raw string, read until first newline); treat as a rejection — abort connection |

**Handling packets 4 and 16:** If you receive packet type 4 (DISCONNECT) or 16 (ERROR) instead of packet type 0 (SERVERLIST), the listserver has rejected your login. Print the message for diagnostic purposes and stop — no server list will follow. Common reasons: wrong version string in the edition packet, or credentials not recognized.

**Packet 0 (SERVERLIST) Structure:**
- Packet type: 0 (GByte)
- Server count: Number of servers (GByte)
- For each server:
  - Attribute count (GByte)
  - For each attribute:
    - Attribute length (GByte)
    - Attribute value (string)
  - Attributes typically include: name, language, description, URL, version, player count, IP, port

**Server Attributes (in order):**
1. Server name (with type prefix: P=Premium, H=Hosted, 3=3D, U=Hidden)
2. Language
3. Description
4. URL
5. Version
6. Player count
7. IP address
8. Port number

### Cross-Server Messaging Packets

These packets are sent through the game server, which forwards them to the listserver:

**SVO_REQUESTLIST (26):**
- Game server → Listserver
- Format: Packet type + player ID (GShort) + request data (CommaText)
- Used to request player lists from other servers

**SVI_REQUESTTEXT (19):**
- Listserver ↔ Game servers
- Format: Packet type + player ID (GShort) + data (CommaText)
- Forwards requests and responses between servers

**SVO_PMPLAYER (29):**
- Game server → Listserver
- Format: Packet type + player ID (GShort) + PM data (CommaText)
- Data: "servername\nsender_account\nsender_nick\nGraalEngine\npmplayer\ntarget_account\nmessage"

**SVI_PMPLAYER (29):**
- Listserver → Game server
- Format: Packet type + PM data (CommaText)
- Forwards PMs between servers

## Summary

The listserver is simple:
1. Connect
2. Say who you are (edition packet)
3. Log in (login packet)
4. Get the server list
5. Connection closes (this is normal!)

The tricky parts are:
- Getting the version string right ("G3D30123")
- Using the correct port (14922)
- Using login packet type 1, not 0
- Implementing the scrambler algorithm correctly (rotate every 4 bytes; 12 groups for 0x02, 4 groups for 0x04)
- Not adding extra null bytes

Once you get the server list, you can connect to individual game servers using the IP addresses and port numbers from the list.

The listserver also handles cross-server messaging by forwarding packets between game servers, allowing players on different servers to see each other and send private messages.

