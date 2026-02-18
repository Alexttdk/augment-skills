---
type: agent_requested
description: Apply when building real-time features with WebSockets — connection lifecycle, pub/sub, heartbeat/reconnection, horizontal scaling, or WebSocket security.
---

# WebSocket Engineer

## When to Use
- Building real-time communication (chat, live updates, notifications, presence)
- Deciding between WebSocket vs SSE vs polling
- Implementing reconnection with exponential backoff
- Scaling WebSocket servers horizontally
- Securing WebSocket connections

## WebSocket vs Alternatives

| Mechanism | Use When | Avoid When |
|-----------|----------|-----------|
| WebSocket | Bidirectional, low latency, persistent connection needed | Simple one-way updates |
| SSE (Server-Sent Events) | Server→client only, HTTP/2 compatible, simple setup | Client→server messaging needed |
| Long Polling | Legacy browser support required | Modern stacks (use SSE/WS) |
| HTTP Polling | Very infrequent updates (>30s) | Frequent real-time events |

**Rule of thumb:** If only the server pushes data → SSE. If bidirectional → WebSocket.

## Connection Lifecycle

```
Client                         Server
  │── HTTP Upgrade request ──► │  (Sec-WebSocket-Key, headers)
  │◄── 101 Switching Protocols ─│  (Sec-WebSocket-Accept)
  │                             │
  │◄──── ping frame (30s) ──────│  heartbeat
  │───── pong frame ───────────►│  liveness confirmed
  │                             │
  │◄──── message frames ────────│  bidirectional data
  │───── message frames ───────►│
  │                             │
  │── close(1000) ────────────►│  normal closure
  │◄─ close acknowledged ───────│
```

### Heartbeat / Dead Connection Detection
```javascript
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws) => {
  ws.isAlive = true;
  ws.on('pong', () => { ws.isAlive = true; });
});

// Ping all clients every 30s; terminate unresponsive ones
const interval = setInterval(() => {
  wss.clients.forEach((ws) => {
    if (!ws.isAlive) return ws.terminate();
    ws.isAlive = false;
    ws.ping();
  });
}, 30_000);

wss.on('close', () => clearInterval(interval));
```

### Client Reconnection with Exponential Backoff
```javascript
class ReconnectingSocket {
  constructor(url) {
    this.url = url;
    this.attempt = 0;
    this.maxDelay = 30_000;
    this.connect();
  }

  connect() {
    this.ws = new WebSocket(this.url);
    this.ws.onopen  = () => { this.attempt = 0; console.log('connected'); };
    this.ws.onclose = () => this.scheduleReconnect();
  }

  scheduleReconnect() {
    const delay = Math.min(1000 * 2 ** this.attempt++, this.maxDelay);
    const jitter = Math.random() * 1000; // Prevent thundering herd
    setTimeout(() => this.connect(), delay + jitter);
  }
}
```

## Message Patterns

### Rooms / Channel Grouping (Socket.IO)
```javascript
io.on('connection', (socket) => {
  socket.on('join-room', (roomId) => {
    socket.join(roomId);
    socket.to(roomId).emit('user-joined', { userId: socket.id });
  });

  socket.on('message', ({ roomId, text }) => {
    io.to(roomId).emit('message', { userId: socket.id, text, ts: Date.now() });
  });
});

// Broadcasting variants
io.emit('event', data);                  // Everyone
socket.broadcast.emit('event', data);    // Everyone except sender
io.to('room1').emit('event', data);      // Room only
socket.volatile.emit('update', data);    // Ok to drop if client not ready
```

### Acknowledgments (RPC-over-WebSocket)
```javascript
// Client calls server and awaits response
socket.timeout(5000).emit('save-data', payload, (err, response) => {
  if (err) console.error('Timeout or server error');
  else console.log('Saved:', response);
});

// Server handles and acknowledges
socket.on('save-data', async (data, callback) => {
  try {
    await db.save(data);
    callback({ success: true });
  } catch (e) {
    callback({ success: false, error: e.message });
  }
});
```

## Scaling WebSocket Horizontally

**Problem**: Clients on Server A can't receive messages emitted on Server B.

**Solution**: Redis pub/sub adapter + sticky sessions.

```javascript
// Socket.IO with Redis adapter
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('redis');

const pubClient = createClient({ url: 'redis://redis:6379' });
const subClient = pubClient.duplicate();
await Promise.all([pubClient.connect(), subClient.connect()]);
io.adapter(createAdapter(pubClient, subClient));

// Now: io.to('room').emit() broadcasts across ALL server instances
```

**Load balancer config:** Enable sticky sessions (IP hash or cookie-based) so a client always reconnects to the same node.

```nginx
upstream websocket_servers {
  ip_hash;   # Sticky sessions
  server ws1:3000;
  server ws2:3000;
}
```

**Serverless WebSocket (AWS API Gateway / Cloudflare Durable Objects):**
- Store connection IDs in DynamoDB/KV
- Use connection management API to push messages to offline nodes
- Good for low-frequency updates; not ideal for high-frequency streaming

## Security

```javascript
// 1. Authenticate on connect (JWT in handshake auth)
io.use(async (socket, next) => {
  try {
    const token = socket.handshake.auth.token;
    socket.user = await verifyJWT(token);
    next();
  } catch (err) {
    next(new Error('Authentication failed'));
  }
});

// 2. Validate origin (CORS equivalent for WS)
const io = new Server(server, {
  cors: { origin: ['https://app.example.com'], credentials: true }
});

// 3. Rate limit events per connection
const rateLimiter = new Map();
socket.use(([event], next) => {
  const key = `${socket.id}:${event}`;
  const count = (rateLimiter.get(key) || 0) + 1;
  rateLimiter.set(key, count);
  if (count > 100) return next(new Error('Rate limit exceeded'));
  next();
});
```

**Security checklist:**
- Always authenticate before allowing any events
- Validate message schema server-side (never trust client data)
- Use WSS (TLS) in production — never plain WS
- Never broadcast sensitive data to room without authorization check
- Set `maxPayload` to prevent memory exhaustion attacks

## Constraints

**MUST DO:** Authenticate connections · implement heartbeat/ping-pong · exponential backoff reconnection · sticky sessions for load balancing · Redis adapter for multi-node · queue messages during disconnection · log connection metrics

**MUST NOT:** Skip connection auth · store large state in memory without clustering strategy · broadcast sensitive data to all clients · ignore connection limits · forget cleanup on disconnect

## References
- Source: https://github.com/Jeffallan/claude-skills/tree/main/skills/websocket-engineer

