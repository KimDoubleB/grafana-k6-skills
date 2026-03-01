# k6 Results Visualization Reference

## Output Options

### Console Summary (Default)

k6 prints an end-of-test summary to stdout by default. No configuration needed.

### JSON Lines (Real-Time Streaming)

```bash
k6 run --out json=results.json script.js
```

Each line is a JSON object with metric data, timestamps, and tags. Useful for post-processing.

### CSV Output

```bash
k6 run --out csv=results.csv script.js
```

### Web Dashboard (Built-in)

```bash
K6_WEB_DASHBOARD=true k6 run script.js
# Access at http://localhost:5665
```

Real-time visualization in the browser. Shows charts for all metrics.

### Multiple Outputs

```bash
k6 run --out json=results.json --out csv=results.csv script.js
```

## handleSummary() Customization

Override the default end-of-test summary:

```javascript
import { textSummary } from 'https://jslib.k6.io/k6-summary/0.1.0/index.js';

export function handleSummary(data) {
  return {
    // Console output
    stdout: textSummary(data, { indent: ' ', enableColors: true }),
    // JSON file
    'summary.json': JSON.stringify(data, null, 2),
    // Custom HTML report
    'report.html': generateHtmlReport(data),
  };
}

function generateHtmlReport(data) {
  const metrics = data.metrics;
  return `
    <html>
    <body>
      <h1>k6 Test Report</h1>
      <h2>HTTP Request Duration</h2>
      <p>Average: ${metrics.http_req_duration.values.avg.toFixed(2)}ms</p>
      <p>P95: ${metrics.http_req_duration.values['p(95)'].toFixed(2)}ms</p>
      <p>P99: ${metrics.http_req_duration.values['p(99)'].toFixed(2)}ms</p>
      <h2>Error Rate</h2>
      <p>${(metrics.http_req_failed.values.rate * 100).toFixed(2)}%</p>
      <h2>Throughput</h2>
      <p>${metrics.http_reqs.values.rate.toFixed(2)} req/s</p>
    </body>
    </html>
  `;
}
```

### handleSummary Data Structure

```javascript
{
  root_group: {
    name: '',
    path: '',
    groups: [...],
    checks: [...],
  },
  metrics: {
    http_req_duration: {
      type: 'trend',
      contains: 'time',
      values: {
        avg: 131.01,
        min: 116.4,
        med: 127.6,
        max: 196.38,
        'p(90)': 146.21,
        'p(95)': 176.91,
      },
      thresholds: {
        'p(95)<500': { ok: true },
      },
    },
    // ... other metrics
  },
}
```

## Grafana Cloud k6

### Running with Cloud Results

```bash
# Set token
export K6_CLOUD_TOKEN=your-token-here

# Run locally, send results to cloud
k6 cloud run --local-execution script.js

# Run entirely in cloud
k6 cloud run script.js
```

### Cloud Dashboard Features

- Real-time metric visualization
- Performance trend comparison across runs
- Threshold pass/fail tracking
- Team sharing and collaboration
- Historical result storage

## Post-Processing JSON Results

### Using jq

```bash
# Extract all http_req_duration data points
cat results.json | jq 'select(.type=="Point" and .metric=="http_req_duration")'

# Get all failed requests
cat results.json | jq 'select(.type=="Point" and .metric=="http_req_failed" and .data.value==1)'
```

### Using Python

```python
import json

with open('results.json') as f:
    for line in f:
        data = json.loads(line)
        if data.get('metric') == 'http_req_duration':
            print(f"Duration: {data['data']['value']}ms")
```
