# k6 Extensions (xk6) Reference

## Overview

k6 extensions add capabilities beyond the core k6 feature set. Types:
- **JavaScript extensions** — New JS APIs and modules (`import ... from 'k6/x/<name>'`)
- **Output extensions** — Send metrics to custom backends (`--out <name>`)
- **Secret source extensions** — Provide secrets at runtime (`--secret-source=<name>=...`)
- **Subcommand extensions** — Extra CLI commands under `k6 x <name>` (auto-resolved since k6 v1.7.0), for example `k6 x docs`, `k6 x explore`, `k6 x mcp`, `k6 x agent`

## Automatic Resolution

Any extension listed in the k6 extension registry auto-loads when imported. k6 detects the `k6/x/*` import, provisions a binary that includes the extension through the k6 build service, caches it, and re-runs the script transparently.

```javascript
import faker from 'k6/x/faker';    // Auto-resolved at runtime
import sql from 'k6/x/sql';        // Auto-resolved at runtime
import { Socket } from 'k6/x/tcp'; // Auto-resolved at runtime
```

| Behavior | Version |
|----------|---------|
| JavaScript extensions resolve on import | k6 v1.2.0+ |
| Community-tier extensions resolve without a flag | k6 v2.0.0+. `v1.x:` set `K6_ENABLE_COMMUNITY_EXTENSIONS=true` (variable removed in v2.0.0) |
| Subcommand extensions resolve on `k6 x <name>` | k6 v1.7.0+ |
| `k6 x` lists the subcommands available in the registry | k6 v2.1.0+ |
| Disable auto-resolution | `K6_AUTO_EXTENSION_RESOLUTION=false` (`v1.x:` `K6_BINARY_PROVISIONING`, removed in v2.0.0) |
| Provisioning logs (cache hits, downloads, retries) appear in k6's log output | k6 v1.8.0 / v2.0.0+ |

Auto-resolution needs network access to the build service. Air-gapped runners must use a pre-built binary (next section) or a custom image.

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
xk6 build v2.3.0 \
  --with github.com/grafana/xk6-faker
```

**Module path:** k6 v2 lives at `go.k6.io/k6/v2`. Extensions must import `go.k6.io/k6/v2/js/modules` (and friends) to build into a v2 binary; an extension that still imports `go.k6.io/k6` only builds with `xk6 build v1.8.2` (or `grafana/k6:v1`). Check an extension's `go.mod` before relying on it.

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

## Official Extensions (registry tier `official`)

| Extension | Import | Description |
|-----------|--------|-------------|
| xk6-faker | `k6/x/faker` | Generate fake/random data |
| xk6-sql | `k6/x/sql` + `k6/x/sql/driver/postgres`, `.../mysql` | Database testing; each driver is its own official extension |
| xk6-tcp | `k6/x/tcp` | Raw TCP sockets: `Socket` with `connect()`, `write()`, `on('data')`, TLS, `tcp_*` metrics |
| xk6-redis | `k6/x/redis` | Redis client. Replaces `k6/experimental/redis` (removed in k6 v2.0.0) |
| xk6-mqtt | `k6/x/mqtt` | MQTT protocol testing |
| xk6-dns | `k6/x/dns` | DNS lookups and resolution timing |
| xk6-icmp | `k6/x/icmp` | ICMP ping |

Tier and current versions: `https://registry.k6.io/` or `k6 x explore` (subcommand extension).

## Community Extensions (registry tier `community`)

Resolve automatically on k6 v2.x; `v1.x:` require `K6_ENABLE_COMMUNITY_EXTENSIONS=true`.

| Extension | Description |
|-----------|-------------|
| xk6-kafka | Apache Kafka testing |
| xk6-amqp | AMQP/RabbitMQ testing |
| xk6-tls | TLS handshake inspection |
| xk6-output-influxdb | Send metrics to InfluxDB v2 (the built-in `--out influxdb` output speaks the v1 protocol) |
| xk6-cable | ActionCable testing |
| xk6-exec | Execute external commands |
| xk6-sql drivers for Azure SQL, ClickHouse, SQL Server | `k6/x/sql/driver/<name>` |

## Archived Extensions

| Extension | Status | Guidance |
|-----------|--------|----------|
| xk6-disruptor (`k6/x/disruptor`) | Archived by Grafana Labs in February 2026; last release v0.3.17 (2025-10); still listed in the registry as community tier | Built against `go.k6.io/k6` v1, so it only works with k6 **v1.x** binaries (`grafana/k6:v1`, `xk6 build v1.8.2`). Expect no fixes. See `/k6:testing-resilience` for how the plugin handles it |

The web dashboard (former xk6-dashboard) is part of the k6 binary since v2.0.0 (`K6_WEB_DASHBOARD=true`); no extension is needed.

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
# List the extensions a script (or archive) needs, k6 v1.6.0+
k6 deps script.js
k6 deps --json script.js

# Archives created with k6 archive record the same information in metadata.json ("dependencies"), k6 v1.8.0 / v2.0.0+
```

## Extension Discovery

- `k6 x explore` — browse the registry from the terminal (subcommand extension, auto-resolved)
- Registry: https://registry.k6.io/
- Official docs: https://grafana.com/docs/k6/latest/extensions/
- Community: https://github.com/topics/xk6

## k6's AI Subcommands

| Command | Purpose |
|---------|---------|
| `k6 x docs` | k6 documentation in the terminal, written for agents and humans |
| `k6 x mcp` | k6 MCP server (`validate_script`, `run_script`, docs lookup); `--transport=http` for shared hosts |
| `k6 x agent init <claude-code\|cursor\|vscode-copilot\|codex-cli\|opencode\|cline>` | Installs Grafana's bundled k6 skills and registers the MCP server in the editor (k6 v2.0+) |
| `k6 x agent status` / `list` / `skills show <name>` | Inspect what was installed |
