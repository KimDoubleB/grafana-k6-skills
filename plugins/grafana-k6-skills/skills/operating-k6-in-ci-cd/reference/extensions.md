# k6 Extensions (xk6) Reference

## Overview

k6 extensions add capabilities beyond the core k6 feature set. Types:
- **JavaScript extensions** — New JS APIs and modules
- **Output extensions** — Send metrics to custom backends
- **Secret source extensions** — Provide secrets at runtime

## Automatic Resolution (k6 v1.2.0+)

Official extensions auto-load when imported:

```javascript
import faker from 'k6/x/faker';    // Auto-resolved at runtime
import sql from 'k6/x/sql';        // Auto-resolved at runtime
```

Community extensions require opt-in:

```bash
K6_ENABLE_COMMUNITY_EXTENSIONS=true k6 run script.js
```

## Building Custom k6 Binary (xk6)

### Using Go

```bash
# Install xk6
go install go.k6.io/xk6/cmd/xk6@latest

# Build with extensions
xk6 build \
  --with github.com/grafana/xk6-faker \
  --with github.com/grafana/xk6-sql

# Build specific k6 version
xk6 build v1.2.0 \
  --with github.com/grafana/xk6-faker
```

### Using Docker

```bash
# Build custom binary
docker run --rm -v $(pwd):/output grafana/xk6 build \
  --with github.com/grafana/xk6-faker \
  --output /output/k6

# Run with custom binary
./k6 run script.js
```

### Dockerfile for CI/CD

```dockerfile
FROM grafana/xk6 AS builder
RUN xk6 build \
  --with github.com/grafana/xk6-faker \
  --with github.com/grafana/xk6-sql \
  --output /k6

FROM alpine:3
COPY --from=builder /k6 /usr/local/bin/k6
ENTRYPOINT ["k6"]
```

## Popular Official Extensions

| Extension | Import | Description |
|-----------|--------|-------------|
| xk6-faker | `k6/x/faker` | Generate fake/random data |
| xk6-sql | `k6/x/sql` | Database testing (PostgreSQL, MySQL, SQLite) |
| xk6-disruptor | `k6/x/disruptor` | Fault injection for Kubernetes |

## Popular Community Extensions

| Extension | Description |
|-----------|-------------|
| xk6-output-influxdb | Send metrics to InfluxDB |
| xk6-kafka | Apache Kafka testing |
| xk6-amqp | AMQP/RabbitMQ testing |
| xk6-redis | Redis testing |
| xk6-mqtt | MQTT protocol testing |
| xk6-cable | ActionCable testing |
| xk6-exec | Execute external commands |

## Extension Usage Examples

### xk6-faker (Data Generation)

```javascript
import faker from 'k6/x/faker';

export default function () {
  const user = {
    name: faker.person.firstName() + ' ' + faker.person.lastName(),
    email: faker.internet.email(),
    phone: faker.phone.number(),
    address: faker.location.streetAddress(),
    company: faker.company.name(),
  };

  http.post(url, JSON.stringify(user), {
    headers: { 'Content-Type': 'application/json' },
  });
}
```

### xk6-sql (Database Testing)

```javascript
import sql from 'k6/x/sql';
import driver from 'k6/x/sql/driver/postgres';

const db = sql.open(driver, 'postgres://user:pass@localhost:5432/testdb');

export function setup() {
  db.exec(`CREATE TABLE IF NOT EXISTS test (id SERIAL PRIMARY KEY, name VARCHAR(50))`);
}

export default function () {
  db.exec(`INSERT INTO test (name) VALUES ('test_${Date.now()}')`);
  const results = sql.query(db, 'SELECT COUNT(*) as count FROM test');
}

export function teardown() {
  db.close();
}
```

## Checking Dependencies

```bash
# Check what extensions a script needs
k6 deps script.js
```

## Extension Discovery

Browse available extensions:
- Official: https://grafana.com/docs/k6/latest/extensions/
- Community: https://github.com/topics/xk6
