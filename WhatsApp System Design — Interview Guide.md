# WhatsApp System Design — Interview Guide

Nov 15, 2026 · @Meet

WhatsApp is a relay, not an archive: stateful WebSocket gateways deliver messages in real time, a Cassandra inbox holds them only until they are delivered, Kafka fans them out, and Redis tracks who is connected where. This guide is the full 45–60 minute interview answer, in the order you present it, with the follow-ups panels dig into.

## 1. Interview plan and clarifying questions

Never jump to boxes and arrows. Spend the first five minutes agreeing scope; interviewers mark you down for skipping it.

| Phase | Time | What you produce |
| --- | --- | --- |
| Clarify + requirements | 5 min | Functional, non-functional, out of scope |
| Capacity estimation | 3–5 min | QPS, connections, storage |
| API design | 5 min | WebSocket events + REST endpoints |
| Data model | 5 min | Tables, keys, which store and why |
| High-level design + flows | 10–15 min | Architecture diagram, send/receive flows |
| Deep dives + trade-offs | 15 min | Ordering, failures, scaling, Q&A |

**Clarifying questions to ask (and the assumption you state):**

1. 1:1 only, or groups? How large? → *Groups up to 1,000 members.*
2. Keep history on the server forever, or until delivered? → *Until delivered, like WhatsApp.*
3. Multiple devices per user? → *Phone is primary; multi-device as an extension.*
4. End-to-end encryption required? → *Yes.*
5. Media? Voice and video calls? → *Media yes; calls out of scope.*
6. What scale? → *2B users, 500M connected at the same time.*

## 2. Requirements

Seven functional features, and non-functional targets led by under-200 ms delivery and zero message loss.

### Functional

1. Send and receive 1:1 messages in real time
2. Group messaging, up to 1,000 members
3. Message status: sent ✓, delivered ✓✓, read (blue ✓✓)
4. Offline users receive their messages when they reconnect
5. Online status and last seen
6. Media: images, video, documents
7. Push notifications when the app is not active

**Out of scope:** voice and video calls, stories, payments, message search.

### Non-functional

| Requirement | Target and reason |
| --- | --- |
| Low latency | Under 200 ms end to end when both users are online |
| High availability | 99.99%; an outage of a chat app is instantly visible |
| Durability | A message the server has acked to the sender is never lost |
| Ordering | Messages in one chat appear in the order they were sent |
| Delivery guarantee | At-least-once delivery + dedup, so each message shows exactly once |
| Scalability | Scale out to hundreds of millions of open connections |
| Security | End-to-end encrypted; the server sees only ciphertext |
| Consistency | Eventual for presence and receipts; strict order within a chat |

**Say this:** *"On CAP, I pick availability and partition tolerance. Presence can be a few seconds stale, but sending a message must never fail."*

## 3. Capacity estimation

The bottleneck is open connections and fan-out, not storage: about 3M messages per second at peak across roughly 5,000 gateway servers.

| Quantity | Working | Result |
| --- | --- | --- |
| Users | Given | 2B total, 500M online at once |
| Messages per second | 100B per day ÷ 86,400 s | \~1.2M avg, \~3M peak (2–3×) |
| Gateway servers | 500M connections ÷ 100K per server | \~5,000 (+ headroom → 6–7K) |
| Text storage | 100B × 100 bytes | \~10 TB per day written |
| Text kept | Deleted after delivery | Tens of TB (undelivered + short buffer) |
| Media | 5% of messages × 200 KB | \~1 PB per day to blob storage + CDN |
| Text bandwidth | 3M per s × 100 bytes | \~300 MB/s, small |

Media never flows through chat servers; clients upload to and download from blob storage and the CDN directly.

**Say this:** *"Storage is small because we delete on delivery. The hard parts are holding 500M sockets and fanning out 3M messages a second."*

## 4. API design

Real-time chat runs over one WebSocket per device; everything that is not real-time is plain REST.

### WebSocket: client → server

```
send_message  { client_msg_id, chat_id, type: text|media, ciphertext, media_ref? }
ack           { msg_id }                    // received → "delivered"
read          { chat_id, up_to_msg_id }     // read receipt
typing        { chat_id }
heartbeat     { }
```

### WebSocket: server → client

```
message_ack   { client_msg_id, msg_id, server_ts }   // ✓ sent
new_message   { msg_id, chat_id, sender_id, ciphertext, ts }
receipt       { msg_id, user_id, status: delivered|read }
presence      { user_id, status, last_seen }
```

### REST

```
POST /v1/media/upload-url          → presigned S3 URL
GET  /v1/sync?after_msg_id=X       → undelivered messages after reconnect
POST /v1/groups                    → create a group
PUT  /v1/groups/{id}/members       → add or remove members
GET  /v1/users/{id}/keys           → E2E public key bundle
```

**Why `client_msg_id`?** The client generates it as a UUID. When a flaky network makes the client retry, the server dedupes on `(sender_id, client_msg_id)`, so the recipient never sees a duplicate.

## 5. Data model

Messages live in Cassandra only until delivered, partitioned by recipient; sessions, presence and dedup are short-lived Redis keys; users, devices and groups sit in a sharded metadata store.

&#91;embedded content: data model · 3 stores, 7 tables\]

### Core table

```
user_inbox
  PRIMARY KEY ((user_id), msg_id)   -- partition by recipient, sort by msg_id
  msg_id      bigint      -- Snowflake, sortable by time
  chat_id     text
  sender_id   bigint
  type        text
  ciphertext  blob
  media_ref   text
  created_at  timestamp
```

- **Reconnect query:** `SELECT * FROM user_inbox WHERE user_id = ? AND msg_id > last_acked` — one partition, one range scan.
- **On ack:** delete the row. Undelivered rows carry a \~30-day TTL.
- **Groups:** `group_messages` stores the content once; `day` in the partition key stops a busy group from growing one giant partition.
- **Media** lives in S3 with a TTL; the message carries only `media_ref` and the decryption key.

**Say this:** *"Message IDs are 64-bit Snowflake IDs, sortable by time, so ordering never depends on device clocks."*

## 6. High-level architecture

Gateways only move bytes between sockets and the backend; the message service owns IDs, dedup, storage and routing, so each layer scales on its own.

&#91;embedded content: WhatsApp architecture · 10 components\]

The gateway arrow is two-way: the message service also pushes messages back out to the recipient's gateway. Kafka feeds group fan-out workers and the push service asynchronously.

| Component | Job | Why this choice |
| --- | --- | --- |
| Load balancer (L4) | Spread new connections across gateways | WebSockets are long-lived TCP; no need to parse HTTP |
| Chat gateway | Hold WebSockets, relay frames | Stateless in logic, so it deploys and scales independently |
| Session store (Redis) | Map user + device → gateway | Sub-ms lookups, TTL refreshed by heartbeat |
| Message service | Assign `msg_id`, dedup, persist, route | The single place that decides order and durability |
| Group service | Member lists for fan-out | Cached; groups change rarely |
| Presence service | Heartbeats, online, last seen | Redis TTL keys, pushed only to open chats |
| Message store (Cassandra) | Inbox of undelivered messages | Huge write load, partitioned by user |
| Kafka | Fan-out, receipts, push events | Decouples slow consumers from the send path |
| Push service | APNs / FCM for offline users | Wake-up hint only; the inbox is the truth |
| Blob store + CDN | Media files | Keeps large files off the chat path |

**Say this:** *"Gateways hold no business logic, so I can scale sockets and message logic independently."*

## 7. Core flows

Every flow follows one rule: save first, then ack. Five flows cover the requirements; draw the first two on the board.

### Flow 1: 1:1 message, B online

&#91;embedded content: 1:1 send, B online · 13 steps\]

1. A sends `send_message` with a `client_msg_id` to its gateway.
2. The gateway forwards to the message service, which dedupes, assigns `msg_id` and writes to B's inbox.
3. Only after the write, A gets `message_ack` → one tick.
4. Redis says B is on gateway B; the service delivers and gateway B pushes `new_message`.
5. B acks → the inbox row is deleted → A gets two ticks.
6. B opens the chat → `read` → A gets blue ticks.

### Flow 2: B offline

1. Steps 1–3 are identical; the message is safe in B's inbox.
2. Redis has no session for B → an event goes to Kafka → the push service calls APNs or FCM.
3. B opens the app, co-nnects the WebSocket, calls `GET /sync?after_msg_id=<last acked>`.
4. B acks everything received → A's messages turn to two ticks.

The push is only a wake-up hint and can be dropped; the inbox is the source of truth.

### Flow 3: Group message

&#91;embedded content: group fan-out · online and offline members\]

1. The message service saves one copy in `group_messages`, not 1,000.
2. Kafka carries one eve-nt per message; fan-out workers read the cached member list.
3. Online members get it through their gateway; offline members get an inbox pointer and a push.
4. Read receipts are tracked per member and shown to the sender as a count ("read by 12").

### Flow 4: Media

1. The client encrypts the file and asks for a presigned URL.
2. It uploads straight to S3, then sends a normal message carrying `media_ref` and the decryption key inside the E2E payload.
3. Recipients download from the CDN and decrypt on the device.

### Flow 5: Presence

1. Each client sends a heartbeat every 30 s: `SET presence:{user} online EX 60`.
2. No heartbeat for 60 s → the key expires → offline, and `last_seen` is stored.
3. Status changes go only to users who have that chat open, never to every contact.

## 8. Deep dives

Five topics panels push on; each has a one-line answer first, then the detail.

### Connection management

- **Why WebSocket:** chat needs two-way, low-latency traffic. Long polling wastes requests; SSE is server-to-client only.
- **Heartbeats** every \~30 s detect dead sockets, because phones lose signal without closing the connection cleanly.
- **Deploying gateways:** drain each server. Stop new connections and tell clients to reconnect in batches, so you avoid a thundering herd.

### Message ordering

- Device clocks drift, so the server assigns `msg_id` (64-bit Snowflake: timestamp + machine + sequence, sortable by time).
- Route every message of one chat to the same Kafka partition (key = `chat_id`), so it is processed in order.
- In groups, the server's order is the truth and every client renders that same order.

### Exactly-once for the user

- Networks cause retries, and retries cause duplicates.
- The server dedupes on `client_msg_id` using a short-lived dedup cache in Redis.
- The client dedupes on `msg_id` before rendering.
- At-least-once delivery + idempotency = exactly once as the user sees it.

### Scaling the gateways

- Gateways scale out horizontally; the session store is the routing table.
- Optional: consistent hashing of user → gateway reduces Redis lookups.
- Keep business logic out of gateways so they deploy and scale independently.

### Failure handling

| Failure | What happens |
| --- | --- |
| Gateway crashes | Clients reconnect to another gateway, update their session, resync from the last ack |
| Message service dies before saving | No ack reaches the sender, it retries, dedup absorbs the retry |
| Stale session (Redis says G7, G7 is gone) | Send to G7 fails, treat B as offline, push; the inbox keeps the message |
| Kafka consumer lag | Delivery slows, nothing is lost; alert on lag |
| Cassandra node down | Replication factor 3 + QUORUM writes keep serving |

**Rule to repeat:** save first, then ack. An ack before the write is a message that can vanish in a crash.

## 9. Trade-offs and bottlenecks

Raise these yourself before the interviewer does; naming the cost of each choice is what separates SDE II answers.

| Decision | We gain | We give up |
| --- | --- | --- |
| Fan-out on write (groups ≤ 1K) | Fast reads, simple clients | Write cost grows with group size; huge channels need fan-out on read |
| Delete after delivery | Small storage, better privacy | No server-side history or search |
| End-to-end encryption | Server never sees content | No server-side search, spam scanning or moderation of content |
| Push events via Kafka | Decoupled, retryable, rate-limited | Slightly higher notification latency |
| Presence with TTL | Cheap at 500M users | Status can be 30–60 s stale |
| AP over CP | Sends never fail | Receipts and presence are eventually consistent |

## 10. Q&A bank

Twelve follow-ups that come up most, each with a short answer you can expand.

**Why Cassandra over MySQL?** Around 3M writes per second, a simple access pattern (partition by user, range scan by `msg_id`), linear scale-out and no joins. MySQL would need heavy manual sharding.

**How does the server route a message that is E2E encrypted?** Only the content is encrypted. Routing metadata (sender, recipient, `chat_id`) stays readable.

**How does E2E work in groups?** Sender keys: each member shares a sender key with the group once, then encrypts each message once instead of N times.

**A user has three devices. What changes?** Each device has its own session and keys. The sender encrypts per device, and delivery is tracked per device. "Delivered" means at least one device acked.

**How do you keep last seen for 2B users?** Write it only on heartbeat expiry or disconnect, into Redis, and persist it asynchronously. Never poll.

**A 1,000-member group sends 10 messages a second. Fan-out load?** 10K deliveries per second for that group. Kafka workers handle it; batch by gateway so each gateway gets one batch for all its members.

**A user has been offline for 30 days?** Undelivered messages carry a TTL of about 30 days; after that they are dropped.

**Hot partitions?** Busy groups bucket by `(group_id, day)`. A user receiving huge volumes gets their inbox spread across partitions.

**Multi-region?** Users connect to the nearest region; their inbox lives in their home region; cross-region messages travel over a replicated queue, accepting extra latency.

**Spam and rate limiting?** A token bucket per user at the gateway, caps on new chats per hour, and reputation scoring on metadata, since content is encrypted.

**How do you prove no messages are lost?** Synthetic canary users sending tracked messages, alerts on sent-versus-delivered mismatches, and chaos tests that kill gateways.

**Why not HTTP polling?** 500M users polling every few seconds is hundreds of millions of mostly empty requests per second: expensive and slower.

## Closing summary

*"Stateful WebSocket gateways handle real-time delivery, a write-heavy Cassandra inbox gives durability, Kafka decouples fan-out, and Redis holds sessions and presence. Server-assigned Snowflake IDs give ordering, client message IDs prevent duplicates. The main trade-offs are the fan-out strategy and E2E encryption limiting server-side features."*
