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

## Unary RPC Calls

```javascript
// Basic invoke
const response = client.invoke('package.Service/Method', {
  field1: 'value1',
  field2: 42,
});

// With metadata (headers)
const response = client.invoke('package.Service/Method', requestMessage, {
  metadata: {
    'authorization': 'Bearer token123',
    'x-request-id': 'abc-123',
    'bin-data-bin': encoding.b64encode('binary data'), // Binary metadata (key ends with -bin)
  },
});

// With custom tags and timeout
const response = client.invoke('package.Service/Method', requestMessage, {
  tags: { name: 'MyRPC' },
  timeout: '5s',
});

// Async invoke
const promise = client.asyncInvoke('package.Service/Method', requestMessage);
// ... do other work ...
const response = await promise;
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

### Client Streaming

```javascript
const stream = new grpc.Stream(client, 'package.Service/ClientStreamMethod');

stream.on('data', (response) => {
  check(response, {
    'stream response OK': (r) => r.status === grpc.StatusOK,
  });
});

stream.on('error', (err) => {
  console.error('Stream error:', JSON.stringify(err));
});

stream.on('end', () => {
  console.log('Stream ended');
});

// Send multiple messages
stream.write({ data: 'message 1' });
stream.write({ data: 'message 2' });
stream.write({ data: 'message 3' });
stream.end(); // Signal end of client stream
```

### Server Streaming

```javascript
const stream = new grpc.Stream(client, 'package.Service/ServerStreamMethod');

stream.on('data', (message) => {
  console.log('Received:', JSON.stringify(message));
});

stream.on('error', (err) => {
  console.error('Error:', JSON.stringify(err));
});

stream.on('end', () => {
  console.log('All messages received');
});

// Send single request to start server stream
stream.write({ query: 'search term' });
```

### Bidirectional Streaming

```javascript
const stream = new grpc.Stream(client, 'package.Service/BidiStreamMethod', {
  metadata: { 'authorization': 'Bearer token123' },
});

stream.on('data', (message) => {
  console.log('Server:', JSON.stringify(message));
  // Respond to server messages
  if (message.needsReply) {
    stream.write({ reply: 'acknowledged' });
  }
});

stream.on('error', (err) => {
  console.error('Error:', JSON.stringify(err));
});

stream.on('end', () => {
  console.log('Stream ended');
});

stream.write({ action: 'subscribe', topic: 'updates' });
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
