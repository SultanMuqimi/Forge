# Forge — Real-Time & WebSockets Addon

**When to load this file:** The app needs live updates — chat, notifications, collaborative editing, live dashboards, presence indicators, multiplayer features, or any feature where users see each other's changes without refreshing.

This file adds real-time rules on top of the universal `CLAUDE.md` and the stack-specific rules. Do not repeat general rules here — apply both.

---

## CORE PRINCIPLES

1. **Pick the right transport for the job.** Not every "live" feature needs WebSockets.
2. **Server is the source of truth.** Clients never trust each other. All state flows through the server.
3. **Design for reconnection.** Networks drop. Mobile users switch towers. Laptops sleep. Assume disconnects are normal, not exceptional.
4. **Authorize every message.** A connected socket is not an authenticated one. Every event must be authorized against the user's identity and permissions.
5. **Scale with a broker.** One server handles thousands of connections. Multiple servers need Redis pub/sub (or similar) to coordinate — never try to share memory between instances.

---

## CHOOSING THE TRANSPORT

Pick the simplest mechanism that solves the user's problem.

| Feature | Use |
|---|---|
| Live notifications (one-way, server → client) | **SSE (Server-Sent Events)** |
| Streaming AI responses | **SSE** |
| Live dashboards, analytics | **SSE** or **polling** if update frequency is low |
| Chat, messaging | **WebSockets** |
| Collaborative editing (docs, whiteboards) | **WebSockets** |
| Multiplayer games | **WebSockets** (consider WebRTC for peer-to-peer) |
| Presence (who's online) | **WebSockets** |
| File upload progress | **HTTP + progress events** — not WebSockets |
| One-off confirmations | **HTTP request/response** — not real-time |

**Default:** If it's server-to-client only → SSE. If it's bidirectional → WebSockets.

---

## WEBSOCKETS

### Library choice per stack

- **Node.js (Fastify/Express):** Socket.IO for most apps, native `ws` for lightweight or protocol-specific needs
- **FastAPI (Python):** FastAPI's built-in `WebSocket` support, with Redis pub/sub for scaling
- **Next.js:** External WebSocket server — Next.js serverless does not support persistent connections. Use a separate Fastify + Socket.IO service, or a managed service (Pusher, Ably, Supabase Realtime)
- **ASP.NET Core:** SignalR — first-class, handles scaling with Azure SignalR Service or Redis backplane
- **Flutter / mobile:** `socket_io_client` or native `WebSocketChannel`

### Connection lifecycle (required implementation)

Every WebSocket implementation must handle:

1. **Connect** — authenticate before accepting the connection. Reject anonymous connections unless public by design.
2. **Authenticate** — verify JWT, session, or token in the connection handshake. Never trust the client's self-declared user ID.
3. **Join rooms/channels** — scope events to relevant users only (e.g., `room:${chatId}`). Never broadcast everything to everyone.
4. **Heartbeat** — ping/pong every 30 seconds. Disconnect if no pong within 60 seconds.
5. **Reconnect** — client must auto-reconnect with exponential backoff (1s, 2s, 4s, 8s, max 30s).
6. **Resume state** — on reconnect, client requests missed events since last known timestamp or event ID.
7. **Disconnect** — clean up rooms, emit presence-offline events, release resources.

### Event naming

- Use `resource:action` format: `message:new`, `message:updated`, `user:joined`, `order:shipped`
- Never use generic event names like `update` or `data` — always specify what changed
- Include the resource ID in the payload, never rely on positional args

### Payload shape

Every event payload must include:

```typescript
{
  id: string,           // unique event ID (UUID) for deduplication
  type: string,         // event type (e.g., "message:new")
  timestamp: number,    // server time in ms since epoch
  data: { ... },        // the actual payload
  version: number       // schema version for forward compatibility
}
```

---

## SERVER-SENT EVENTS (SSE)

SSE is simpler than WebSockets. Use it whenever the data flow is one-way (server → client).

### Server implementation

- Set headers: `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`
- Send periodic `: keepalive` comments every 15–30 seconds to prevent proxies from closing the connection
- Include event IDs so clients can resume with `Last-Event-ID` header after reconnect
- Handle client disconnects cleanly — release resources immediately

### Client implementation

- Use the native `EventSource` API in browsers, not a polyfill unless targeting legacy
- On error, let the browser reconnect automatically (that's what EventSource does by default)
- Handle each event type explicitly — never use a catch-all handler in production

### Common SSE use cases

- Streaming AI chat responses (token-by-token)
- Live notifications (new messages, alerts, status changes)
- Build progress, upload progress, job status
- Live read-only dashboards

---

## AUTHENTICATION & AUTHORIZATION

### Authentication on connection

- WebSocket handshake must include auth — either as a query param (`?token=...`), header, or first message
- Validate the token immediately — close the connection with a clear error code if invalid
- Never expose the token in URLs that get logged — prefer headers or an initial auth message

### Authorization per message

- Every incoming event must be authorized against the sender's identity
- Never trust client-sent user IDs, room IDs, or permissions — derive everything from the authenticated session
- A user can only join rooms they have permission to access
- A user can only send messages to rooms they're a member of
- Admin/moderator actions require re-verification, not just "they were admin when they connected"

### Session expiry

- If a user's session expires mid-connection, disconnect them immediately
- Do not let stale tokens persist on open sockets
- On reconnect, re-authenticate from scratch

---

## SCALING ACROSS MULTIPLE SERVERS

A single Node.js or FastAPI process handles 5,000–20,000 concurrent connections comfortably. Beyond that — or with multiple instances behind a load balancer — you need a broker.

### The problem

If User A connects to server 1 and User B connects to server 2, a message from A to B has no path unless the servers can talk to each other.

### The solution: Redis pub/sub (or equivalent)

- Every server subscribes to relevant Redis channels
- When server 1 receives a message for room X, it publishes to Redis channel `room:X`
- All servers subscribed to `room:X` receive the message and forward it to their local connected clients in that room
- Use Redis Cluster or a managed Redis (Upstash, Railway, AWS ElastiCache) in production

### Socket.IO adapter

- Socket.IO has an official Redis adapter — `@socket.io/redis-adapter`
- One line of code enables multi-server scaling
- Works with Redis Streams for message persistence if needed

### SignalR backplane

- ASP.NET Core SignalR uses Azure SignalR Service or Redis backplane for scaling
- Configure once in `Program.cs`, it handles the rest

### Managed alternatives

For non-developers or teams that don't want to run Redis:
- **Pusher** — simple, reliable, generous free tier
- **Ably** — feature-rich, strong guarantees
- **Supabase Realtime** — if already using Supabase, built-in with row-level security
- **Azure SignalR Service** — if on ASP.NET Core + Azure

---

## PRESENCE (WHO'S ONLINE)

Presence is deceptively complex. Users disconnect without notice, switch tabs, put phones to sleep.

- Track presence with a TTL in Redis: `SET user:${userId}:online true EX 60`
- Client sends a heartbeat every 30 seconds that refreshes the TTL
- On disconnect, explicitly delete the key
- "Online users in room" = intersection of room members and keys with TTL alive
- Never trust a single "connected" flag in memory — that breaks on server restarts

---

## RATE LIMITING & ABUSE

Real-time features are a DoS target. A single malicious client can flood the server.

- Rate limit events per socket — e.g., max 30 messages/second per connection
- Use a token bucket algorithm with Redis
- Close sockets that exceed rate limits after a warning
- Cap message payload size — reject anything over 64KB by default (raise only for specific use cases like file metadata)
- Reject malformed payloads immediately — do not let them reach business logic

---

## PERSISTENCE & MESSAGE DELIVERY

### For chat/messaging

- **Always persist messages before broadcasting.** The database is the source of truth, not the WebSocket event.
- Flow: client sends → server validates → server writes to DB → server broadcasts event → clients update
- If the DB write fails, do not broadcast. Return an error to the sender.

### For ephemeral events (typing indicators, cursor positions, presence)

- Do not persist — pure real-time, no DB write
- Fire and forget is acceptable

### Message ordering

- Always assign server timestamps — never trust client clocks
- For strict ordering, use an auto-incrementing sequence per room/channel
- Clients sort by sequence number, not arrival time

### Delivery guarantees

- WebSockets are **at-most-once** by default — if the client disconnects mid-send, the message is lost
- For **at-least-once**, require client ACKs and retry unacked messages
- For **exactly-once**, combine ACKs with server-side deduplication by message ID
- Most apps need at-most-once for ephemeral events, at-least-once for chat. Exactly-once is rarely worth the complexity.

---

## CLIENT-SIDE STATE MANAGEMENT

Real-time state changes constantly — manage it carefully.

- Use a dedicated store (Zustand, Redux, Riverpod) for real-time data — do not mix with local UI state
- Apply updates optimistically where safe (e.g., sending a chat message), reconcile with server response
- On reconnect, always re-fetch the current state from the server — never trust the local cache to be fresh
- Show connection status to users: online (hide or green dot), reconnecting (yellow), offline (red with retry button)

---

## SECURITY CHECKLIST

Before shipping any real-time feature:

- [ ] Authentication enforced on connection
- [ ] Authorization checked on every incoming event
- [ ] Room/channel membership validated on every send and receive
- [ ] Rate limiting per connection
- [ ] Payload size cap
- [ ] Input validation on every event payload (Zod, Pydantic, FluentValidation)
- [ ] Sensitive data never broadcast to rooms users don't belong to
- [ ] Disconnect on session expiry
- [ ] CSRF not needed for WebSockets (origin check instead — validate `Origin` header)
- [ ] WSS (TLS) in production, never WS
- [ ] No secrets or tokens echoed back in event payloads

---

## TESTING

- Unit test event handlers the same way you test HTTP route handlers — inject fake socket, assert behavior
- Integration test connection lifecycle: connect, auth, join room, send, receive, disconnect, reconnect
- Load test with realistic concurrent connection counts — tools like `artillery`, `k6` support WebSockets
- Test reconnection logic explicitly — force disconnects, verify state resumes correctly

---

## WHAT NOT TO DO

- Do not use WebSockets for one-off request/response — HTTP is simpler and cacheable
- Do not store session state in the WebSocket server's memory if running multiple instances
- Do not trust anything the client sends — user IDs, timestamps, room membership — verify server-side
- Do not broadcast user-specific data to shared channels
- Do not try to build your own real-time infrastructure if a managed service fits the budget — your time is better spent on the product
- Do not ignore backpressure — if a client can't keep up, buffer or drop events, don't block the server
