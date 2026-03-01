# k6 WebSocket API Reference

## Overview

k6 provides two WebSocket modules:
- **`k6/websockets`** (recommended) — Uses global event loop, supports multiple concurrent connections per VU
- **`k6/ws`** (legacy) — Uses local event loop, one connection per VU

This reference covers the recommended `k6/websockets` module.

## Quick Start

```javascript
import { WebSocket } from 'k6/websockets';
import { check } from 'k6';

export const options = {
  vus: 10,
  duration: '30s',
};

export default function () {
  const url = 'wss://echo.websocket.org';
  const ws = new WebSocket(url);

  ws.onopen = () => {
    console.log('Connected');
    ws.send('Hello from k6!');
  };

  ws.onmessage = (event) => {
    console.log('Received:', event.data);
    ws.close();
  };

  ws.onerror = (event) => {
    console.error('Error:', event.error());
  };

  ws.onclose = (event) => {
    console.log('Disconnected:', event.code);
  };
}
```

## Creating a WebSocket Connection

```javascript
import { WebSocket } from 'k6/websockets';

// Basic connection
const ws = new WebSocket('wss://example.com/ws');

// With subprotocols
const ws = new WebSocket('wss://example.com/ws', ['graphql-ws', 'json']);

// With connection parameters
const ws = new WebSocket('wss://example.com/ws', null, {
  headers: {
    'Authorization': 'Bearer token123',
    'X-Custom-Header': 'value',
  },
  tags: { name: 'MyWebSocket' },
  jar: http.cookieJar(),        // Custom cookie jar
  compression: 'deflate',       // Enable compression
});
```

## Connection Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `headers` | object | Custom HTTP headers for the initial handshake |
| `tags` | object | Custom metric tags |
| `jar` | CookieJar | Custom cookie jar for the handshake |
| `compression` | string | `'deflate'` for per-message compression |

## WebSocket Properties

| Property | Type | Description |
|----------|------|-------------|
| `readyState` | number | `0` CONNECTING, `1` OPEN, `2` CLOSING, `3` CLOSED |
| `url` | string | Resolved connection URL |
| `bufferedAmount` | number | Bytes queued but not yet sent |
| `binaryType` | string | `'blob'` (default) or `'arraybuffer'` |

## Event Handlers

### onopen — Connection Established

```javascript
ws.onopen = () => {
  console.log('Connection established');
  ws.send('Initial message');
};

// Alternative: addEventListener
ws.addEventListener('open', (event) => {
  console.log('Connected');
});
```

### onmessage — Message Received

```javascript
ws.onmessage = (event) => {
  const data = event.data;

  // Text message
  if (typeof data === 'string') {
    const json = JSON.parse(data);
    console.log('JSON message:', json);
  }
};

// Multiple handlers via addEventListener
ws.addEventListener('message', (event) => {
  console.log('Handler 1:', event.data);
});
ws.addEventListener('message', (event) => {
  console.log('Handler 2:', event.data);
});
```

### onclose — Connection Closed

```javascript
ws.onclose = (event) => {
  console.log('Closed with code:', event.code);
  // Common close codes:
  // 1000 - Normal closure
  // 1001 - Going away
  // 1006 - Abnormal closure (no close frame)
  // 1011 - Server error
};
```

### onerror — Error Occurred

```javascript
ws.onerror = (event) => {
  console.error('WebSocket error:', event.error());
};
```

### onping / onpong — Ping/Pong Frames

```javascript
ws.addEventListener('ping', () => {
  console.log('Received ping from server');
});

ws.addEventListener('pong', () => {
  console.log('Received pong from server');
});
```

## Sending Messages

```javascript
// Text message
ws.send('Hello, server!');

// JSON message
ws.send(JSON.stringify({ type: 'subscribe', channel: 'updates' }));

// Binary data
const buffer = new ArrayBuffer(4);
const view = new Uint8Array(buffer);
view[0] = 0x01;
ws.send(buffer);
```

## Closing Connection

```javascript
// Normal close
ws.close();

// Close with code
ws.close(1000); // Normal closure

// Common close codes:
// 1000 - Normal
// 1001 - Going away
// 1008 - Policy violation
// 1011 - Server error
```

## Ping

```javascript
ws.ping(); // Send ping to server (pong is automatic)
```

## Patterns

### Request-Response Pattern

```javascript
export default function () {
  const ws = new WebSocket('wss://api.example.com/ws');
  let responseReceived = false;

  ws.onopen = () => {
    ws.send(JSON.stringify({ action: 'getData', id: 123 }));
  };

  ws.onmessage = (event) => {
    const data = JSON.parse(event.data);
    check(data, {
      'has result': (d) => d.result !== undefined,
      'status is success': (d) => d.status === 'success',
    });
    responseReceived = true;
    ws.close();
  };

  ws.onerror = (event) => {
    console.error('Error:', event.error());
  };
}
```

### Subscription Pattern (Pub/Sub)

```javascript
import { setTimeout } from 'k6/timers';

export default function () {
  const ws = new WebSocket('wss://api.example.com/ws');
  let messageCount = 0;

  ws.onopen = () => {
    // Subscribe to channels
    ws.send(JSON.stringify({ type: 'subscribe', channel: 'prices' }));
    ws.send(JSON.stringify({ type: 'subscribe', channel: 'trades' }));

    // Close after 10 seconds
    setTimeout(() => {
      ws.send(JSON.stringify({ type: 'unsubscribe', channel: 'prices' }));
      ws.send(JSON.stringify({ type: 'unsubscribe', channel: 'trades' }));
      ws.close();
    }, 10000);
  };

  ws.onmessage = (event) => {
    messageCount++;
    const data = JSON.parse(event.data);
    check(data, {
      'valid message': (d) => d.channel !== undefined && d.data !== undefined,
    });
  };
}
```

### Chat/Interactive Pattern

```javascript
import { setTimeout } from 'k6/timers';

export default function () {
  const ws = new WebSocket('wss://chat.example.com/ws');

  ws.onopen = () => {
    ws.send(JSON.stringify({ type: 'join', room: 'loadtest' }));

    // Simulate periodic messages
    let i = 0;
    const interval = setInterval(() => {
      ws.send(JSON.stringify({
        type: 'message',
        text: `Hello from VU ${__VU}, message ${i}`,
      }));
      i++;
      if (i >= 5) {
        clearInterval(interval);
        ws.send(JSON.stringify({ type: 'leave', room: 'loadtest' }));
        ws.close();
      }
    }, 1000);
  };

  ws.onmessage = (event) => {
    const msg = JSON.parse(event.data);
    if (msg.type === 'message') {
      console.log(`${msg.user}: ${msg.text}`);
    }
  };
}
```

### Multiple Concurrent Connections

The `k6/websockets` module supports multiple connections per VU:

```javascript
export default function () {
  const ws1 = new WebSocket('wss://api.example.com/stream1');
  const ws2 = new WebSocket('wss://api.example.com/stream2');

  ws1.onmessage = (event) => {
    console.log('Stream 1:', event.data);
  };

  ws2.onmessage = (event) => {
    console.log('Stream 2:', event.data);
  };

  ws1.onopen = () => ws1.send('subscribe');
  ws2.onopen = () => ws2.send('subscribe');

  setTimeout(() => {
    ws1.close();
    ws2.close();
  }, 10000);
}
```

## WebSocket Metrics

k6 automatically emits these metrics for WebSocket connections:

| Metric | Type | Description |
|--------|------|-------------|
| `ws_connecting` | Trend | Time to establish WebSocket connection |
| `ws_session_duration` | Trend | Total session duration |
| `ws_msgs_sent` | Counter | Number of messages sent |
| `ws_msgs_received` | Counter | Number of messages received |
| `ws_ping` | Trend | Ping round-trip time |
| `ws_sessions` | Counter | Number of sessions started |

## Legacy Module (k6/ws)

The `k6/ws` module uses a callback-based pattern with a local event loop:

```javascript
import ws from 'k6/ws';

export default function () {
  const res = ws.connect('wss://echo.websocket.org', {}, function (socket) {
    socket.on('open', () => {
      socket.send('Hello');
      socket.setInterval(() => {
        socket.send('Periodic message');
      }, 1000);
      socket.setTimeout(() => {
        socket.close();
      }, 5000);
    });

    socket.on('message', (data) => {
      console.log('Received:', data);
    });

    socket.on('close', () => {
      console.log('Disconnected');
    });
  });

  check(res, { 'status is 101': (r) => r && r.status === 101 });
}
```

Key differences from `k6/websockets`:
- Uses callback function wrapping all logic
- `socket.setInterval()` and `socket.setTimeout()` instead of `k6/timers`
- Only one connection per VU at a time
- Returns response object with status 101
