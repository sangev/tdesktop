# How MTProto Servers Work

**Author:** Research compiled for understanding Telegram-compatible server infrastructure
**Date:** 2025-11-06
**Related:** See RESEARCH_FINDINGS.md for infrastructure options

---

## Table of Contents
1. [Overview](#overview)
2. [Protocol Architecture](#protocol-architecture)
3. [Server Components](#server-components)
4. [Authentication Process](#authentication-process)
5. [Message Flow](#message-flow)
6. [Data Centers & Distribution](#data-centers--distribution)
7. [Implementation Example: Teamgram](#implementation-example-teamgram)
8. [Key Technical Details](#key-technical-details)

---

## Overview

**MTProto** (Mobile Transport Protocol) is Telegram's custom protocol for client-server communication. It's designed specifically for mobile applications, prioritizing:
- Low latency
- Battery efficiency
- Reliable message delivery
- Strong encryption
- Multi-data center support

An MTProto server is responsible for:
- Authenticating clients
- Routing messages between users
- Storing messages and media
- Managing user sessions
- Synchronizing data across devices
- Handling group chats, channels, and calls

---

## Protocol Architecture

MTProto consists of **three independent layers**:

### 1. High-Level Component (API Layer)
**What it does:** Defines the API methods and data structures

**Examples:**
- `messages.sendMessage` - Send a text message
- `users.getFullUser` - Get user information
- `auth.sendCode` - Send authentication code

**Format:** Uses TL (Type Language) schema to define:
```
messages.sendMessage#520c3870 flags:# no_webpage:flags.1?true
  silent:flags.5?true background:flags.6?true
  peer:InputPeer random_id:long message:string = Updates;
```

### 2. Cryptographic/Authorization Layer
**What it does:** Handles encryption and authentication

**Key features:**
- **AES-256-IGE** encryption for message bodies
- **SHA-256** hashing for message keys
- **2048-bit RSA** for initial key exchange
- **Diffie-Hellman** for generating shared secrets

**Message structure (encrypted):**
```
[64-bit auth_key_id][128-bit msg_key][encrypted_data]
```

### 3. Transport Layer
**What it does:** Delivers messages over network protocols

**Supported transports:**
- **TCP** (multiple modes: Full, Intermediate, Abridged, Padded Intermediate)
- **HTTP/HTTPS** (for restrictive networks)
- **WebSocket** (for web clients)
- **UDP** (for voice/video calls)

---

## Server Components

An MTProto server implementation typically consists of:

### 1. API Server (Gateway)
**Responsibilities:**
- Accept incoming client connections
- Parse MTProto messages
- Route API calls to appropriate services
- Return responses to clients
- Manage multiple data center connections

**Technical details:**
- Listens on ports (typically 443 for HTTPS, various for TCP)
- Handles thousands of concurrent connections
- Validates message format and timing
- Enforces rate limits

### 2. Authentication Service
**Responsibilities:**
- Generate and verify authorization keys
- Handle phone number verification
- Manage 2FA (two-factor authentication)
- Issue and validate session tokens
- Handle logout and session termination

**Security features:**
- RSA key signing during registration
- Diffie-Hellman key exchange validation
- Time synchronization checks (±300 seconds)
- Message ID validation (prevent replay attacks)

### 3. Message Router
**Responsibilities:**
- Route messages to recipients
- Handle offline message storage
- Manage message delivery confirmations
- Support multi-device synchronization
- Queue messages for offline users

**How it works:**
1. Client sends message with recipient ID
2. Router looks up recipient's active sessions
3. If online: immediate delivery to all sessions
4. If offline: store in database for later retrieval
5. Send delivery confirmation back to sender

### 4. Storage Service
**Responsibilities:**
- Store user profiles and contacts
- Store chat history
- Store media files (photos, videos, documents)
- Handle database replication
- Manage data retention policies

**Data types:**
- **User data:** Profiles, contacts, settings
- **Messages:** Text, media references, metadata
- **Files:** Photos, videos, documents, voice messages
- **State data:** Read/unread status, pinned chats

### 5. File Storage (Object Storage)
**Responsibilities:**
- Store large media files
- Provide CDN functionality
- Handle file uploads/downloads
- Manage file encryption
- Optimize bandwidth usage

**Typical implementation:**
- MinIO or similar S3-compatible storage
- Chunked file uploads/downloads
- Separate encryption keys per file
- CDN distribution for popular content

### 6. Session Manager
**Responsibilities:**
- Track active client sessions
- Handle reconnections
- Manage session timeouts
- Coordinate multi-device access
- Handle session conflicts

**Session lifecycle:**
1. Client authenticates → session created
2. Client sends/receives messages → session active
3. Connection drops → session kept alive (configurable timeout)
4. Reconnect → resume existing session
5. Logout or timeout → session destroyed

---

## Authentication Process

The authentication process establishes a **shared secret (auth_key)** between client and server using Diffie-Hellman key exchange.

### Phase 1: DH Exchange Initiation

**Client → Server:** `req_pq_multi`
```
Client generates: nonce (128-bit random)
Client sends: { nonce }
```

**Server → Client:** `resPQ`
```
Server generates: server_nonce (128-bit random)
Server generates: pq (product of two large primes)
Server sends: { nonce, server_nonce, pq, rsa_fingerprints[] }
```

**Purpose:**
- Establish unique session identifiers (nonces)
- Provide proof-of-work challenge (factorize pq)
- Server identifies itself via RSA fingerprints

### Phase 2: Proof of Work & Server Authentication

**Client processes:**
1. **Factorizes pq** into primes p and q (proof of work)
2. **Generates new_nonce** (256-bit random) - will become part of auth_key
3. **Creates encrypted payload:**
   ```
   inner_data = { pq, p, q, nonce, server_nonce, new_nonce }
   encrypted = RSA_OAEP(inner_data, server_public_key)
   ```

**Client → Server:** `req_DH_params`
```
{ nonce, server_nonce, p, q, encrypted_data }
```

**Server processes:**
1. **Decrypts** using private RSA key
2. **Validates** all nonces match
3. **Generates DH parameters:**
   - `dh_prime`: Safe 2048-bit prime
   - `g`: Generator (typically 3 or 7)
   - `a`: Random 2048-bit number (server's secret)
   - `g_a = g^a mod dh_prime` (server's public key)

**Server → Client:** `server_DH_params_ok`
```
{ nonce, server_nonce, encrypted_answer }

Where encrypted_answer contains:
{ nonce, server_nonce, g, dh_prime, g_a, server_time }
```

### Phase 3: DH Key Exchange Completion

**Client processes:**
1. **Validates dh_prime** is a safe prime (security check)
2. **Generates b** (random 2048-bit number - client's secret)
3. **Computes g_b = g^b mod dh_prime** (client's public key)
4. **Computes auth_key = g_a^b mod dh_prime** (shared secret!)

**Client → Server:** `set_client_DH_params`
```
{ nonce, server_nonce, encrypted_data }

Where encrypted_data contains:
{ nonce, server_nonce, retry_id, g_b }
```

**Server processes:**
1. **Computes auth_key = g_b^a mod dh_prime** (same shared secret!)
2. **Verifies** both sides computed same key using message hash

**Server → Client:** `dh_gen_ok`
```
{ nonce, server_nonce, new_nonce_hash }
```

**Result:** Both sides now have identical 2048-bit `auth_key`

### Deriving Encryption Keys

From the `auth_key`, both sides derive:

```
auth_key_id = SHA1(auth_key)[last 64 bits]  // Identifies the key
msg_key = SHA256(auth_key_part + message)[96:128]  // Per-message key

# Encryption keys derived from auth_key + msg_key:
aes_key = SHA256(msg_key + auth_key_part1) +
          SHA256(auth_key_part2 + msg_key)

aes_iv = SHA256(auth_key_part3 + msg_key) +
         SHA256(msg_key + auth_key_part4)
```

**All subsequent messages are encrypted with AES-256-IGE using these keys.**

---

## Message Flow

### Sending a Message

**1. Client Side (Telegram Desktop):**

```
User types "Hello" → send button clicked

↓ Application layer (UI)

Construct API call:
  messages.sendMessage(
    peer = @username,
    random_id = 12345678,
    message = "Hello"
  )

↓ Serialization layer

Convert to TL binary format:
  [constructor_id][flags][peer][random_id][message]

↓ Encryption layer (mtproto/session.cpp)

1. Generate msg_id (based on timestamp)
2. Add session_id, seq_no
3. Encrypt with AES-256-IGE
4. Prepend auth_key_id + msg_key

↓ Transport layer (mtproto/connection_tcp.cpp)

Wrap in TCP transport:
  - Abridged: [length][packet]
  - Intermediate: [length][packet]
  - Full: [length][seq_no][packet][crc32]

↓ Network

Send over TCP/HTTPS/WebSocket to server
```

**2. Server Side (MTProto Server):**

```
↓ Receive TCP packet

↓ Transport layer

Extract encrypted message from transport wrapper

↓ Authentication

Lookup auth_key using auth_key_id
If not found → reject with auth_error

↓ Decryption

1. Extract msg_key
2. Derive AES key/IV from auth_key + msg_key
3. Decrypt message body with AES-256-IGE
4. Validate msg_key hash matches decrypted data

↓ Validation

1. Check message timestamp (must be within ±300 seconds)
2. Check msg_id has correct parity (even = client)
3. Check msg_id is greater than previous (prevent replay)
4. Verify session_id is valid

↓ Deserialization

Parse TL binary format → API call object:
  messages.sendMessage {
    peer: User ID 12345
    message: "Hello"
  }

↓ Business Logic (Message Router)

1. Validate sender has permission to message recipient
2. Assign unique message_id
3. Save to database:
   - messages table: (id, from, to, text, date)
   - dialogs table: update last_message
4. Check if recipient is online

↓ Message Delivery

If recipient online:
  FOR EACH active session:
    - Encrypt message with session's auth_key
    - Send via push notification system
    - Send via active WebSocket/TCP connection

If recipient offline:
  - Store in pending_messages queue
  - Send push notification to device
  - Wait for user to come online

↓ Acknowledgment

Send back to sender:
  Updates {
    message_id: 999,
    pts: 123,  // state update sequence
    date: 1234567890
  }
```

**3. Recipient Side (Any Client):**

```
↓ Receive encrypted message from server

↓ Decrypt (same as step 2 above)

↓ Process Update

Update local database:
  - Add message to chat history
  - Update dialog list
  - Increment unread counter
  - Trigger notification

↓ UI Update

Display message in chat window
Show notification
Update badge count
```

### Message Routing Optimization

**Direct Routing (Same Data Center):**
```
Client A (DC1) → Server (DC1) → Client B (DC1)
Latency: ~50ms
```

**Cross-Data Center Routing:**
```
Client A (DC1) → Server (DC1) → Server (DC5) → Client B (DC5)
Latency: ~150ms (depends on geographic distance)
```

**Offline Message Storage:**
```
Client A → Server → Database (persistent storage)
                 → Redis (quick access cache)
                 → Push notification service

Client B comes online → Query Redis for pending messages
                      → Deliver all queued messages
                      → Mark as delivered
```

---

## Data Centers & Distribution

### Data Center Types

MTProto uses **four types of data centers**:

#### 1. Regular DCs
**Purpose:** Primary user data storage
**Contains:**
- User accounts and profiles
- Chat histories
- Contact lists
- User settings
- Secret chats (on user's device only)

**Example:** Telegram has DCs in:
- DC1: Miami, USA
- DC2: Amsterdam, Netherlands
- DC3: Miami, USA
- DC4: Amsterdam, Netherlands
- DC5: Singapore

#### 2. Media Cluster DCs
**Purpose:** Optimized for large files
**Contains:**
- Photos and images
- Videos
- Voice messages
- Documents and files
- Stickers and GIFs

**Features:**
- Optimized for high bandwidth
- Geographic distribution for fast access
- Automatic CDN-like distribution

#### 3. Temporary DCs
**Purpose:** Short-term encrypted connections
**Used for:**
- Secret chats (end-to-end encrypted)
- Temporary file transfers
- Short-lived authentication sessions

**Characteristics:**
- Keys are not persisted long-term
- Destroyed after use
- Perfect Forward Secrecy (PFS)

#### 4. CDN DCs
**Purpose:** Content delivery network
**Contains:**
- Cached popular media
- Public channel content
- Frequently accessed files

**Features:**
- Read-only access
- Special encryption (can be served by third parties)
- Reduced server load

### How Clients Choose Data Centers

**1. Initial Connection:**
```
Client starts → Connects to built-in DC address
             → Server returns config with all DC addresses
             → Client connects to geographically closest DC
```

**2. User Registration:**
```
New user → Registers on current DC
        → That becomes user's "home DC"
        → All user data stored in home DC
```

**3. Accessing Data:**
```
Client needs data → Check which DC stores it
                 → If same DC: direct request
                 → If different DC: redirect or proxy
```

**4. Multi-DC Operations:**

When users from different DCs interact:
```
User A (DC1) messages User B (DC5):

1. A's client → A's home DC (DC1)
2. DC1 saves message in A's outbox
3. DC1 replicates to DC5
4. DC5 delivers to B
5. B's client ← DC5

Both users see message, stored in their respective DCs
```

### Data Synchronization

**Problem:** User has multiple devices in different locations

**Solution:** Multi-DC synchronization

```
User opens Telegram on 3 devices:
- Phone (DC1 - closest to user)
- Laptop (DC2 - home internet)
- Work PC (DC4 - office)

Message arrives:
1. Stored in user's home DC (e.g., DC1)
2. Home DC pushes to all active sessions:
   - Direct push to DC1 session (phone)
   - Proxy to DC2 session (laptop)
   - Proxy to DC4 session (work PC)
3. All devices receive simultaneously
```

**State Synchronization:**

Telegram uses **pts** (persistent timestamp sequence) for sync:
```
Each update has:
- pts: sequence number
- pts_count: how many updates

Client syncs by:
1. Storing local pts value
2. Comparing with server pts
3. Requesting missing updates if gap exists
```

---

## Implementation Example: Teamgram

Teamgram is the most popular open-source MTProto server. Here's how it works:

### Architecture Components

```
┌─────────────────────────────────────────────────────────┐
│                    CLIENT LAYER                         │
│  (Telegram Desktop, Mobile Apps, Web Clients)          │
└────────────────┬────────────────────────────────────────┘
                 │ MTProto over TCP/HTTP/WebSocket
                 ↓
┌─────────────────────────────────────────────────────────┐
│                   TEAMGRAM SERVER                       │
│                                                          │
│  ┌─────────────────────────────────────────────────┐  │
│  │          API Gateway (Go Services)              │  │
│  │  - auth.service     (authentication)            │  │
│  │  - session.service  (session management)        │  │
│  │  - msg.service      (message routing)           │  │
│  │  - user.service     (user data)                 │  │
│  │  - chat.service     (group chats)               │  │
│  │  - media.service    (file handling)             │  │
│  └────────┬─────────────────────────────────────────┘  │
│           │                                             │
│           ↓                                             │
│  ┌─────────────────────────────────────────────────┐  │
│  │         Service Mesh (gRPC/etcd)                │  │
│  │  - Service discovery                            │  │
│  │  - Load balancing                               │  │
│  │  - Health checks                                │  │
│  └────────┬─────────────────────────────────────────┘  │
└───────────┼──────────────────────────────────────────────┘
            │
            ↓
┌─────────────────────────────────────────────────────────┐
│               INFRASTRUCTURE LAYER                       │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │  MySQL   │  │  Redis   │  │   etcd   │  │ Kafka  │ │
│  │          │  │          │  │          │  │        │ │
│  │ User     │  │ Session  │  │ Service  │  │ Event  │ │
│  │ Messages │  │ Cache    │  │ Config   │  │ Queue  │ │
│  │ Chats    │  │ Online   │  │ Discovery│  │ Logs   │ │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘ │
│                                                          │
│  ┌──────────────────────────────────────┐               │
│  │         MinIO (Object Storage)       │               │
│  │  - Photos, videos, documents         │               │
│  │  - Chunked storage                   │               │
│  │  - S3-compatible API                 │               │
│  └──────────────────────────────────────┘               │
│                                                          │
│  ┌──────────────────────────────────────┐               │
│  │           FFmpeg (Media)             │               │
│  │  - Video transcoding                 │               │
│  │  - Thumbnail generation              │               │
│  │  - Audio processing                  │               │
│  └──────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────┘
```

### Component Responsibilities

**1. MySQL (Persistent Storage)**
```sql
Tables:
- users (id, phone, first_name, last_name, username, auth_key_id)
- messages (id, from_id, to_id, text, date, media_id)
- chats (id, title, photo, date, members_count)
- chat_participants (chat_id, user_id, inviter_id, date)
- auth_keys (auth_key_id, auth_key, user_id)
- dialogs (user_id, peer_id, top_message, unread_count)
```

**2. Redis (Hot Data Cache)**
```
Keys:
- session:{user_id}:{auth_key_id} = {device_info, online_status}
- online:{user_id} = timestamp
- auth_key:{auth_key_id} = {key_data, user_id}
- pending_messages:{user_id} = [msg_id1, msg_id2, ...]
- user_state:{user_id} = {pts, qts, seq}
```

**3. etcd (Service Discovery)**
```
Keys:
- /services/auth/node1 = {ip:port, health}
- /services/msg/node1 = {ip:port, health}
- /services/user/node1 = {ip:port, health}
- /config/rate_limits = {values}
```

**4. Kafka (Event Streaming)**
```
Topics:
- messages.sent (new messages)
- messages.delivered (delivery confirmations)
- users.online (presence updates)
- updates.push (push notifications)
```

**5. MinIO (File Storage)**
```
Buckets:
- photos/ (images and avatars)
- videos/ (video files)
- documents/ (files and documents)
- voice/ (voice messages)
- thumbnails/ (generated previews)
```

### Message Flow in Teamgram

**Example: User A sends "Hello" to User B**

```
1. RECEIVE MESSAGE FROM CLIENT A
   ↓
   auth.service:
   - Validates auth_key_id
   - Retrieves user_id from Redis/MySQL

2. SESSION CHECK
   ↓
   session.service:
   - Verifies session is active
   - Updates last_activity timestamp

3. MESSAGE PROCESSING
   ↓
   msg.service:
   - Generates unique message_id
   - Validates recipient exists
   - Checks permissions (not blocked, etc.)

4. PERSISTENCE
   ↓
   MySQL:
   - INSERT INTO messages (from_id=A, to_id=B, text="Hello", date=now())
   - UPDATE dialogs SET top_message=msg_id, unread_count=+1

5. EVENT PUBLISHING
   ↓
   Kafka:
   - PUBLISH to "messages.sent" topic
   - msg.service and user.service subscribe to this

6. RECIPIENT CHECK
   ↓
   Redis:
   - CHECK online:{user_B}
   - GET session:{user_B}:* (all B's sessions)

7. DELIVERY (if B is online)
   ↓
   FOR EACH B's active session:
     - Encrypt message with session's auth_key
     - Send via TCP connection
     - Add to pending_acks

   IF B is offline:
     - Redis: LPUSH pending_messages:B msg_id
     - Trigger push notification

8. ACKNOWLEDGMENT TO SENDER A
   ↓
   - Encrypt Updates with A's auth_key
   - Send back:
     {
       message_id: 999,
       date: 1234567890,
       pts: 123
     }

9. DELIVERY CONFIRMATION (when B receives)
   ↓
   Kafka:
   - PUBLISH to "messages.delivered"
   - Update sender A: "✓✓ Delivered"
```

### How Teamgram Handles Scale

**Horizontal Scaling:**
```
Load Balancer (nginx/haproxy)
    ↓
┌─────────┬─────────┬─────────┐
│ Gateway │ Gateway │ Gateway │
│  Node 1 │  Node 2 │  Node 3 │
└────┬────┴────┬────┴────┬────┘
     │         │         │
     └────────┬┴─────────┘
              ↓
       etcd (discovers services)
              ↓
     ┌────────┴────────┐
     ↓                 ↓
┌─────────┐      ┌─────────┐
│ msg.svc │      │ msg.svc │
│ Node 1  │      │ Node 2  │
└─────────┘      └─────────┘
```

**Session Stickiness:**
- User connects → assigned to gateway node
- Session stored in Redis with gateway_id
- Reconnects go to same gateway (or any if failed)

**Database Sharding (planned for scale):**
```
Users 1-1M     → MySQL Shard 1
Users 1M-2M    → MySQL Shard 2
Users 2M-3M    → MySQL Shard 3
...
```

---

## Key Technical Details

### Message ID Generation

```
msg_id = (unix_time_seconds * 2^32) + sequence_number

Rules:
- Must be divisible by 4
- Client messages: even number
- Server messages: odd number
- Must be greater than previous msg_id
- Must be within ±300 seconds of server time
```

Example:
```
Time: 1699900000 seconds since epoch
Sequence: 42

msg_id = (1699900000 * 4294967296) + 42 * 4
       = 7301444669124288168
```

### Encryption Details

**Message structure before encryption:**
```
┌─────────────────────────────────────────────────────┐
│ salt (64-bit)                                       │
├─────────────────────────────────────────────────────┤
│ session_id (64-bit)                                 │
├─────────────────────────────────────────────────────┤
│ msg_id (64-bit)                                     │
├─────────────────────────────────────────────────────┤
│ seq_no (32-bit)                                     │
├─────────────────────────────────────────────────────┤
│ length (32-bit)                                     │
├─────────────────────────────────────────────────────┤
│ message body (TL serialized)                        │
└─────────────────────────────────────────────────────┘
```

**After encryption:**
```
┌─────────────────────────────────────────────────────┐
│ auth_key_id (64-bit)                                │
├─────────────────────────────────────────────────────┤
│ msg_key (128-bit)                                   │
├─────────────────────────────────────────────────────┤
│ encrypted_data (AES-256-IGE)                        │
└─────────────────────────────────────────────────────┘
```

### Transport Modes

**1. Abridged (most efficient):**
```
[1-byte: 0xef] (mode indicator)
[1-4 bytes: length] [packet data]
[1-4 bytes: length] [packet data]
...
```

**2. Intermediate:**
```
[4-byte: 0xeeeeeeee] (mode indicator)
[4 bytes: length] [packet data]
[4 bytes: length] [packet data]
...
```

**3. Padded Intermediate:**
```
[4-byte: 0xdddddddd] (mode indicator)
[4 bytes: length] [packet data] [random padding]
[4 bytes: length] [packet data] [random padding]
...
```

**4. Full (legacy, with CRC):**
```
[4 bytes: length] [4 bytes: seq_no] [packet data] [4 bytes: CRC32]
```

### Rate Limiting

MTProto servers implement rate limiting to prevent abuse:

```
Per user limits:
- Messages: 30/second, burst 100
- API calls: 60/second
- File uploads: 10/minute
- Group messages: 20/second

Per IP limits:
- Connections: 100/IP
- Auth attempts: 5/minute
- Registration: 1/hour
```

### Error Handling

Common MTProto error codes:

```
-404 NOT_FOUND           - Auth key not found
-420 FLOOD_WAIT_X        - Too many requests, wait X seconds
-500 INTERNAL            - Server error
303 SEE_OTHER_DC         - User data is in different DC
400 BAD_REQUEST          - Invalid request format
401 UNAUTHORIZED         - Invalid auth key
```

Client must handle:
- Temporary network failures → retry with exponential backoff
- DC migration → reconnect to different DC
- Session reset → re-authenticate
- Flood errors → respect wait time

---

## Summary

**How MTProto servers work in a nutshell:**

1. **Clients connect** using TCP/HTTP/WebSocket
2. **Authentication** via Diffie-Hellman establishes shared secret
3. **All messages encrypted** with AES-256 using derived keys
4. **Server validates** timing, nonces, and message structure
5. **API calls parsed** from TL binary format
6. **Business logic executes** (store message, route to recipient)
7. **Database stores** persistent data (MySQL)
8. **Cache layer** (Redis) for hot data and sessions
9. **Message queue** (Kafka) for async processing
10. **Files stored** in object storage (MinIO/S3)
11. **Multi-DC replication** keeps data synchronized
12. **Push notifications** sent for offline users
13. **Responses encrypted** and sent back to client

**Key architectural principles:**
- **Separation of concerns:** Each service handles specific functionality
- **Scalability:** Horizontal scaling via service replication
- **Reliability:** Multi-DC redundancy and message queuing
- **Security:** Strong encryption, validation, and rate limiting
- **Efficiency:** Caching, CDN, optimized protocols

---

## Further Reading

- **Official MTProto Docs:** https://core.telegram.org/mtproto
- **Teamgram GitHub:** https://github.com/teamgram/teamgram-server
- **Telegram Desktop Source:** This repository (client implementation)
- **TL Language Spec:** https://core.telegram.org/mtproto/TL

---

**Related Documents:**
- `RESEARCH_FINDINGS.md` - Infrastructure options comparison
- `LICENSE` - GPL v3 license for this codebase
- `/docs/api_credentials.md` - How to get Telegram API keys
