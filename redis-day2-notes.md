# Redis — Day 1 Learning Notes
> Hours 1, 2, 3 & 4 | Fundamentals, CLI, TTL, All 5 Data Types

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
DEL product             →  (integer) 1   ← found and deleted
DEL product             →  (integer) 0   ← nothing to delete
DEL a b c               →  (integer) 3   ← deleted 3 keys
```
Returns **count of keys actually deleted** — not true/false.

#### `EXISTS`
```
EXISTS email            →  (integer) 1
EXISTS phone            →  (integer) 0
EXISTS email email email →  (integer) 3  ← counts every mention, duplicates included
```

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
FLUSHDB          →  delete all keys in current database only
FLUSHALL         →  delete all keys in ALL 16 databases
```

### Key Naming Convention
Redis has no folders or tables. Use `:` as separator by convention:
```
user:1042:profile
user:1042:session
session:abc123
product:456:details
```
Colon means nothing to Redis — just for human readability and enables pattern matching.

### Strings — the base data type
```
SET counter "0"
INCR counter        →  (integer) 1
INCRBY counter 10   →  (integer) 11
GET counter         →  "11"           ← still stored as string
```
```
SET counter "abc"
INCR counter    →  ERR value is not an integer
```
Quotes are optional unless value has spaces: `SET city "New York"`

---

## Hour 3 — TTL & Expiry

### Setting Expiry
```bash
SET otp "4821" EX 300       # preferred — set with expiry in seconds
SETEX otp 300 "4821"        # older syntax, same result
EXPIRE otp 300              # add expiry to existing key
SET otp "4821" PX 300000    # milliseconds instead of seconds
```
`EXPIRE` returns `1` if key found, `0` if key not found.

### TTL — Check Time Remaining
```
TTL key    →  positive number   key exists, X seconds remaining
TTL key    →  -1                key exists, NO expiry (permanent)
TTL key    →  -2                key does NOT exist (expired or never set)
```

### What Happens When a Key Expires
Redis **deletes the key entirely** — completely wiped from memory. After expiry:
```
GET otp       →  (nil)
EXISTS otp    →  0
TTL otp       →  -2
```

### PERSIST — Remove Expiry
```
SET session "active" EX 3600
PERSIST session      →  removes expiry timer
TTL session          →  -1    ← now permanent, still exists
```

### Real-World OTP System
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

## Hour 4 — Data Types

### Why data types exist
With plain Strings, updating one field of a stored object means fetch everything → modify in app → rewrite everything. Wasteful and risky. Each data type solves a specific problem Strings handle poorly.

---

### 1. Strings ✅ (covered in Hour 2)
Single values, counters, OTPs, cached JSON, session tokens.

---

### 2. Hashes — objects with multiple fields

```
HSET user:123 name "Deep"
HSET user:123 email "deep@gmail.com" city "Mumbai" plan "pro"
HGET user:123 name              →  "Deep"
HGETALL user:123                →  all fields and values alternating
HKEYS user:123                  →  all field names only
HVALS user:123                  →  all values only
HLEN user:123                   →  count of fields
HDEL user:123 plan              →  delete one field
```

`HSET` returns count of **new fields created** — returns 0 if just updating existing field.

⚠️ `HKEYS user:*` does NOT work — Hash commands need exact key name, no wildcards.

**Use when:** Storing objects with multiple attributes (user profile, product details).

---

### 3. Lists — ordered collections, queues

```
RPUSH email-queue "email to deep@gmail.com"    # add to right (back)
LPUSH email-queue "urgent task"                # add to left (front)
LRANGE email-queue 0 -1                        # get all items
LPOP email-queue                               # remove from left (front)
RPOP email-queue                               # remove from right (back)
```

**Real queue pattern (FIFO — first in, first out):**
```
RPUSH  →  add to back
LPOP   →  take from front
```

`LRANGE` index guide:
```
0 -1   →  entire list
0 0    →  first item only
0 -2   →  everything except last item
```

**Use when:** Background job queues, activity feeds, chat history.

---

### 4. Sets — unique values, no duplicates

```
SADD post:1:tags "redis" "backend" "database"
SADD post:1:tags "redis"       →  (integer) 0  ← duplicate silently ignored
SMEMBERS post:1:tags           →  all members
SCARD post:1:tags              →  count of members
SISMEMBER post:1:tags "redis"  →  1 (exists) or 0 (doesn't)
SREM post:1:tags "backend"     →  remove a member
```

**Use when:** Tags, unique visitors, users who liked a post, online users.

---

### 5. Sorted Sets — unique values with a score

```
ZADD leaderboard 920 "John"
ZADD leaderboard 850 "Deep"
ZADD leaderboard 760 "Sara"
ZRANGE leaderboard 0 -1          →  lowest to highest score
ZREVRANGE leaderboard 0 -1       →  highest to lowest score
ZSCORE leaderboard "John"        →  "920" (score returned as string)
ZRANK leaderboard "Deep"         →  position, 0 = lowest score
ZREVRANK leaderboard "Deep"      →  position, 0 = highest score
```

**Tiebreaker:** When two members have same score → sorted alphabetically by name.

**Use when:** Leaderboards, priority queues, rankings, trending content.

---

## All 5 Data Types — Summary

| Type | Best for | Key commands |
|---|---|---|
| String | Single values, counters, OTPs | SET, GET, INCR |
| Hash | Objects with multiple fields | HSET, HGET, HGETALL |
| List | Queues, ordered collections | RPUSH, LPOP, LRANGE |
| Set | Unique values, tags | SADD, SMEMBERS, SISMEMBER |
| Sorted Set | Leaderboards, rankings | ZADD, ZRANGE, ZREVRANK |

---

## Full Quick Reference Card

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
| HSET | `HSET key field value` | count new fields |
| HGET | `HGET key field` | value or (nil) |
| HGETALL | `HGETALL key` | all fields + values |
| HKEYS | `HKEYS key` | all field names |
| HVALS | `HVALS key` | all values |
| HLEN | `HLEN key` | field count |
| HDEL | `HDEL key field` | count deleted |
| RPUSH | `RPUSH key value` | list length |
| LPUSH | `LPUSH key value` | list length |
| LPOP | `LPOP key` | removed value |
| RPOP | `RPOP key` | removed value |
| LRANGE | `LRANGE key 0 -1` | list of values |
| SADD | `SADD key member...` | count added |
| SMEMBERS | `SMEMBERS key` | all members |
| SCARD | `SCARD key` | member count |
| SISMEMBER | `SISMEMBER key member` | 1 or 0 |
| SREM | `SREM key member` | count removed |
| ZADD | `ZADD key score member` | count added |
| ZRANGE | `ZRANGE key 0 -1` | members low→high |
| ZREVRANGE | `ZREVRANGE key 0 -1` | members high→low |
| ZSCORE | `ZSCORE key member` | score as string |
| ZRANK | `ZRANK key member` | position (0=lowest) |
| ZREVRANK | `ZREVRANK key member` | position (0=highest) |

---

## Day 2 Plan
- Hour 5 — Connect Redis to Code (Node.js or Python)
- Hour 6 — Build real features: caching layer, OTP system, session system, rate limiter
- Mini Challenge — Profile cache + rate limit simulation
