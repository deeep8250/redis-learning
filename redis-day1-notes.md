# Redis — Day 1 Learning Notes
> Hours 1, 2 & 3 | Fundamentals, CLI Hands-on, TTL & Expiry

---

## Hour 1 — Understanding Redis

### What is Redis?
A super-fast, **in-memory key-value store**. Data lives in RAM — not on disk. That's the entire reason it's fast.

### Why does it exist — what problem does PostgreSQL not solve?
PostgreSQL is brilliant but has one physical constraint: **every read/write touches disk.**

| Problem | PostgreSQL | Redis |
|---|---|---|
| Speed at high volume | Slows under load | Built for millions of ops |
| Expiring data | Painful | Native TTL built-in |
| Real-time counters | Race conditions | Atomic operations |
| Sessions & caching | Overkill/slow | Perfect fit |
| Complex queries & joins | Perfect | Not for this |
| Permanent storage | Perfect | Not primary use |

### What does "in-memory" actually mean?
Data lives in the **server's RAM** — not your local device.

```
Your Browser → Internet → Server (Redis lives here in RAM)
                                  ↓
                           PostgreSQL (on server disk)
```

**Speed comparison:**
- RAM: ~100 nanoseconds (grabbing from your desk)
- SSD: ~100 microseconds (walking to a shelf)
- Hard Disk: ~10 milliseconds (driving to a warehouse)

### When to use Redis vs PostgreSQL

> **Rule of thumb:** If losing this data for 10 seconds would cause a real problem → PostgreSQL. If it's fast-access, temporary, or repeated reads → Redis.

| Situation | Use |
|---|---|
| User signs up, stores profile | PostgreSQL |
| User logs in, store session | Redis |
| Fetch product details (shared) | Redis cache |
| Place an order | PostgreSQL |
| OTP verification | Redis |
| Past orders history | PostgreSQL |
| Rate limit API calls | Redis |
| Real-time leaderboard | Redis |
| Financial transactions | PostgreSQL only |
| Background job queue | Redis |

### 4 Core Real-World Use Cases

**1. Caching** — Store results of expensive DB queries. Serve repeated reads from Redis, skip the DB.

**2. Sessions** — Store login sessions per user. Checked on every request. Fast + supports expiry.

**3. Rate Limiting** — Atomic counters + TTL. Increment on each API call, block when limit hit, auto-reset.

**4. Queues** — Push slow tasks (emails, image processing) to Redis queue. Background worker picks them up. User experience stays instant.

---

## Hour 2 — CLI Hands-On

### Setup (Docker — proper way)

```bash
# Pull Redis image explicitly
docker pull redis

# Verify image downloaded
docker images

# Create and run container
docker run --name redis-local -p 6379:6379 redis

# Connect via redis-cli (new terminal)
docker exec -it redis-local redis-cli

# Verify connection
PING  →  PONG
```

### Core Commands

#### `SET` and `GET`
```
SET name "Kushmanda"    →  OK
GET name                →  "Kushmanda"
GET age                 →  (nil)         ← key doesn't exist
```
- `SET` on existing key **silently overwrites** — no warning
- `(nil)` = key not found, not an error

#### `DEL`
```
SET product "laptop"
DEL product             →  (integer) 1   ← found and deleted
DEL product             →  (integer) 0   ← nothing to delete
DEL a b c               →  (integer) 3   ← deleted 3 keys
```
Returns **count of keys actually deleted** — not true/false.

#### `EXISTS`
```
SET email "k@gmail.com"
EXISTS email            →  (integer) 1
EXISTS phone            →  (integer) 0
EXISTS email email email →  (integer) 3  ← counts every mention
```
Returns count — not boolean. Duplicates are counted separately.

#### `KEYS` — pattern matching
```
KEYS *              →  all keys
KEYS user:*         →  keys starting with "user:"
KEYS pro*           →  keys starting with "pro"
```
⚠️ Never use `KEYS *` in production — scans everything, blocks Redis.

#### `FLUSHALL`
```
FLUSHALL    →  OK    ← deletes EVERYTHING instantly, no undo
```

Three levels of clearing:
```
DEL key1 key2   →  delete specific keys
FLUSHDB         →  delete all keys in current database
FLUSHALL        →  delete all keys in ALL databases
```

### Key Naming Convention
Redis has no folders or tables. Use `:` as separator by convention:
```
user:1042:profile
user:1042:session
session:abc123
product:456:details
```
Colon means nothing to Redis — it's just for human readability and enables pattern matching with `KEYS`.

### Strings — the base data type
Everything stored in Redis is a String internally:
```
SET counter "0"
INCR counter        →  (integer) 1    ← treats stored value as number
INCR counter        →  (integer) 2
INCRBY counter 10   →  (integer) 12   ← increment by custom amount
GET counter         →  "12"           ← still stored as string
```

```
SET counter "abc"
INCR counter    →  ERR value is not an integer   ← can't math on text
```

Quotes are optional unless value has spaces:
```
SET city Mumbai        →  OK
SET city "New York"    →  OK  ← quotes needed for spaces
```

### Multiple Databases
Redis ships with 16 databases (0–15). Default is database 0.
```
SELECT 1    →  switch to database 1 (isolated keyspace)
SELECT 0    →  back to default
```
In practice — almost nobody uses multiple databases. Use key naming conventions instead.

---

## Hour 3 — TTL & Expiry

### Setting Expiry

```bash
# Method 1 — set then expire
SET otp "4821"
EXPIRE otp 300          →  (integer) 1  ← found key, expiry set

# Method 2 — set with expiry in one command (preferred)
SET otp "4821" EX 300

# Method 3 — older syntax (you'll see in legacy code)
SETEX otp 300 "4821"

# Milliseconds instead of seconds
SET otp "4821" PX 300000
```

`EXPIRE` returns:
- `1` — key found, expiry set
- `0` — key not found, nothing happened

### TTL — Check Time Remaining

```
TTL otp    →  positive number   key exists, X seconds remaining
TTL otp    →  -1                key exists, NO expiry (permanent)
TTL otp    →  -2                key does NOT exist (expired or never set)
```

### What Happens When a Key Expires
Redis **deletes the key entirely** — not marked as stale, completely wiped from memory.

After expiry:
```
GET otp       →  (nil)
EXISTS otp    →  0
TTL otp       →  -2
```

### PERSIST — Remove Expiry
```
SET session "active" EX 3600
TTL session          →  3598
PERSIST session      →  removes expiry timer
TTL session          →  -1    ← now permanent
GET session          →  "active"  ← still exists
```

### Real-World Example — OTP System
```
1. User clicks "Send OTP"
2. Server generates "4821"
3. Server sends "4821" to user's phone via SMS
4. Server stores: SET otp:user:1042 "4821" EX 300
5. User enters "4821"
6. Server fetches: GET otp:user:1042  →  "4821"
7. Server compares internally: match ✅
8. Server deletes: DEL otp:user:1042
9. Login successful
```
Redis only stores and retrieves. Server does all thinking. OTP never sent back to client.

---

## Quick Reference Card

| Command | Syntax | Returns |
|---|---|---|
| SET | `SET key value` | OK |
| SET with expiry | `SET key value EX seconds` | OK |
| GET | `GET key` | value or (nil) |
| DEL | `DEL key1 key2...` | count deleted |
| EXISTS | `EXISTS key1 key2...` | count existing |
| KEYS | `KEYS pattern` | list of keys |
| FLUSHALL | `FLUSHALL` | OK |
| INCR | `INCR key` | new integer value |
| INCRBY | `INCRBY key amount` | new integer value |
| EXPIRE | `EXPIRE key seconds` | 1 or 0 |
| TTL | `TTL key` | seconds, -1, or -2 |
| PERSIST | `PERSIST key` | 1 or 0 |
| SELECT | `SELECT db` | OK |

---

## Next Up — Hour 4: Data Types
- Lists — ordered collections, queues
- Hashes — objects with multiple fields
- Sets — unique values
- Sorted Sets — leaderboards, priority queues
