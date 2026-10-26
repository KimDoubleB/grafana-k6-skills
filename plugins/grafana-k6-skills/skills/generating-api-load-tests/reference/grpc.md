# k6 gRPC API Reference

## Overview

k6 supports gRPC load testing via the `k6/net/grpc` module. Supports unary RPCs, client/server/bidirectional streaming, TLS, metadata, and server reflection.

## Quick Start

```javascript
import grpc from 'k6/net/grpc';
import { check, sleep } from 'k6';

const client = new grpc.Client();
client.load(['definitions'], 'hello.proto'); // Load in init context

export const options = {
  vus: 5,
  duration: '30s',
  thresholds: {
    grpc_req_duration: ['p(95)<500'],
  },
};

export default function () {
  client.connect('localhost:50051', { plaintext: true });

  const response = client.invoke('hello.HelloService/SayHello', {
    greeting: 'k6',
  });

  check(response, {
    'status is OK': (r) => r && r.status === grpc.StatusOK,
    'message contains greeting': (r) => r && r.message.reply.includes('k6'),
  });

  client.close();
  sleep(1);
}
```

## Client Setup

### Creating and Loading Protos

```javascript
const client = new grpc.Client();

// Load from proto files (must be in init context)
client.load(['path/to/import/dir'], 'service.proto');
client.load([], 'another.proto'); // No additional import paths

// Load from compiled protoset file
client.loadProtoset('path/to/service.protoset');
```

### Connecting

```javascript
// Plaintext (no TLS)
client.connect('localhost:50051', { plaintext: true });

// TLS with default settings
client.connect('api.example.com:443');

// TLS with custom certificates
client.connect('api.example.com:443', {
  tls: {
    cacerts: open('ca.pem'),       // CA certificate
    cert: open('client.pem'),       // Client certificate
    key: open('client-key.pem'),    // Client key
    password: 'keyfile-password',   // Key password (optional)
  },
});

// With server reflection (no proto files needed)
client.connect('localhost:50051', {
  plaintext: true,
  reflect: true,
});

// With timeout
client.connect('localhost:50051', {
  plaintext: true,
  timeout: '10s',
});

// Max message sizes
client.connect('localhost:50051', {
  plaintext: true,
  maxReceiveSize: 16 * 1024 * 1024, // 16MB (default 4MB)
  maxSendSize: 16 * 1024 * 1024,
});
```

## Connection Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `plaintext` | boolean | `false` | Use insecure connection (no TLS) |
| `reflect` | boolean | `false` | Use gRPC server reflection |
| `reflectMetadata` | object | `null` | Metadata for reflection request |
| `timeout` | string/number | `'60s'` | Connection timeout |
| `maxReceiveSize` | number | 4MB | Max message size to receive |
| `maxSendSize` | number | ~2GB | Max message size to send |
| `tls` | object | `null` | TLS settings (cert, key, password, cacerts) |

If the server omits an intermediate CA certificate, set the test-wide option `tlsAIAFetch: true` (k6 v2.3.0+) so k6 fetches it instead of failing the handshake. `v1.x:` provide `cacerts` with the full chain.

## Unary RPC Calls

```javascript
// Basic invoke
const response = client.invoke('package.Service/Method', {
  field1: 'value1',
  field2: 42,
});

// With metadata (headers)
const withMetadata = client.invoke('package.Service/Method', requestMessage, {
  metadata: {
    'authorization': 'Bearer token123',
    'x-request-id': 'abc-123',
    'bin-data-bin': encoding.b64encode('binary data'), // Binary metadata (key ends with -bin)
  },
});

// With custom tags and timeout
const withOptions = client.invoke('package.Service/Method', requestMessage, {
  tags: { name: 'MyRPC' },
  timeout: '5s',
});

// Async invoke (inside an async function)
const promise = client.asyncInvoke('package.Service/Method', requestMessage);
// ... do other work ...
const asyncResponse = await promise;
```

## Invoke Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `metadata` | object | Custom metadata headers |
| `tags` | object | Custom metric tags |
| `timeout` | string/number | Request timeout |
| `authority` | string | `:authority` pseudo-header override |
| `discardResponseMessage` | boolean | Discard response to reduce memory |

## Response Object

| Property | Type | Description |
|----------|------|-------------|
| `status` | number | gRPC status code |
| `message` | object | Response message as JSON (null if error) |
| `headers` | object | Response metadata headers |
| `trailers` | object | Response metadata trailers |
| `error` | object | Error details (if status != OK) |

## gRPC Status Codes

| Constant | Code | Description |
|----------|------|-------------|
| `grpc.StatusOK` | 0 | Success |
| `grpc.StatusCanceled` | 1 | Operation cancelled |
| `grpc.StatusUnknown` | 2 | Unknown error |
| `grpc.StatusInvalidArgument` | 3 | Invalid argument |
| `grpc.StatusDeadlineExceeded` | 4 | Deadline exceeded |
| `grpc.StatusNotFound` | 5 | Resource not found |
| `grpc.StatusAlreadyExists` | 6 | Resource already exists |
| `grpc.StatusPermissionDenied` | 7 | Permission denied |
| `grpc.StatusResourceExhausted` | 8 | Resource exhausted |
| `grpc.StatusFailedPrecondition` | 9 | Failed precondition |
| `grpc.StatusAborted` | 10 | Operation aborted |
| `grpc.StatusOutOfRange` | 11 | Out of range |
| `grpc.StatusUnimplemented` | 12 | Not implemented |
| `grpc.StatusInternal` | 13 | Internal error |
| `grpc.StatusUnavailable` | 14 | Service unavailable |
| `grpc.StatusDataLoss` | 15 | Data loss |
| `grpc.StatusUnauthenticated` | 16 | Unauthenticated |

## Streaming

### Streaming Requires an Async Default Function

Stream events (`data`, `error`, `end`) are delivered through handlers that only run when the JavaScript call stack is empty. In a synchronous `default` function the handlers would fire only after the function returns, so wrap the stream in a `Promise` that resolves on `end` and `await` it:

```javascript
export default async function () {
  client.connect('localhost:50051', { plaintext: true });
  const stream = new grpc.Stream(client, 'package.Service/Method');

  await new Promise((resolve, reject) => {
    stream.on('data', (msg) => { /* handle message */ });
    stream.on('error', (err) => reject(err));
    stream.on('end', () => { client.close(); resolve(); });

    stream.write({ /* first message */ });
    stream.end();   // client/bidi streams: signal no more client messages
  });
}
```

Do not use `sleep()` to "wait" for stream events; it blocks the event loop and the handlers never run.

### Client Streaming

```javascript
export default async function () {
  client.connect('localhost:50051', { plaintext: true });
  const stream = new grpc.Stream(client, 'package.Service/ClientStreamMethod');

  await new Promise((resolve, reject) => {
    stream.on('data', (response) => {
      check(response, { 'stream response OK': (r) => r.status === grpc.StatusOK });
    });
    stream.on('error', (err) => {
      console.error('Stream error:', JSON.stringify(err));
      reject(err);
    });
    stream.on('end', () => {
      console.log('Stream ended');
      client.close();
      resolve();
    });

    stream.write({ data: 'message 1' });
    stream.write({ data: 'message 2' });
    stream.write({ data: 'message 3' });
    stream.end(); // Signal end of client stream
  });
}
```

### Server Streaming

```javascript
export default async function () {
  client.connect('localhost:50051', { plaintext: true });
  const stream = new grpc.Stream(client, 'package.Service/ServerStreamMethod');

  await new Promise((resolve, reject) => {
    stream.on('data', (message) => {
      console.log('Received:', JSON.stringify(message));
    });
    stream.on('error', (err) => reject(err));
    stream.on('end', () => {
      console.log('All messages received');
      client.close();
      resolve();
    });

    stream.write({ query: 'search term' }); // Single request starts the server stream
  });
}
```

### Bidirectional Streaming

```javascript
export default async function () {
  client.connect('localhost:50051', { plaintext: true });
  const stream = new grpc.Stream(client, 'package.Service/BidiStreamMethod', {
    metadata: { authorization: 'Bearer token123' },
  });

  await new Promise((resolve, reject) => {
    let received = 0;
    stream.on('data', (message) => {
      console.log('Server:', JSON.stringify(message));
      if (message.needsReply) stream.write({ reply: 'acknowledged' });
      if (++received >= 10) stream.end(); // Decide when the client side is done
    });
    stream.on('error', (err) => reject(err));
    stream.on('end', () => {
      client.close();
      resolve();
    });

    stream.write({ action: 'subscribe', topic: 'updates' });
  });
}
```

## Stream API

| Method/Event | Description |
|-------------|-------------|
| `new grpc.Stream(client, url, [params])` | Create stream |
| `stream.write(message)` | Send message to server |
| `stream.end()` | Signal end of client messages |
| `stream.on('data', handler)` | Handle incoming messages |
| `stream.on('error', handler)` | Handle errors |
| `stream.on('end', handler)` | Handle stream completion |

The `async-metric-context` feature flag (k6 v2.3.0+, `--features async-metric-context`) makes stream handlers keep the tags and `group` active when the stream was created; without it, metrics emitted inside handlers may lose the group tag.

## Complete Example: Authenticated gRPC Test

```javascript
import grpc from 'k6/net/grpc';
import { check, sleep } from 'k6';

const client = new grpc.Client();
client.load(['proto'], 'user_service.proto');

export const options = {
  scenarios: {
    grpc_test: {
      executor: 'constant-arrival-rate',
      rate: 50,
      timeUnit: '1s',
      duration: '1m',
      preAllocatedVUs: 20,
    },
  },
  thresholds: {
    grpc_req_duration: ['p(95)<300', 'p(99)<500'],
  },
};

export function setup() {
  client.connect('auth.example.com:443');
  const authRes = client.invoke('auth.AuthService/Login', {
    username: 'loadtest',
    password: 'secret',
  });
  client.close();
  return { token: authRes.message.token };
}

export default function (data) {
  client.connect('api.example.com:443');

  const res = client.invoke('user.UserService/GetUser', { id: 1 }, {
    metadata: { authorization: `Bearer ${data.token}` },
    tags: { name: 'GetUser' },
  });

  check(res, {
    'status OK': (r) => r && r.status === grpc.StatusOK,
    'user found': (r) => r && r.message !== null,
  });

  client.close();
  sleep(0.5);
}
```
