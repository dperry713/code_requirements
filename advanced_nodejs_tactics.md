# Advanced Node.js Development Tactics
### Comprehensive Production-Ready Guide with Full Examples

**Target**: Node.js 20 / 22 LTS  
**Style**: Modern ESM, production-oriented

---

## Table of Contents

1. Modern Node.js Features
2. Asynchronous Patterns & Concurrency
3. Streams & Backpressure
4. Performance & Memory
5. Error Handling & Resilience
6. Observability
7. Configuration & Secrets
8. Validation
9. Caching
10. Testing
11. Security
12. Clustering & Scaling
13. Worker Threads
14. Native Addons / N-API
15. Architecture Patterns
16. Diagnostics Channel & AsyncLocalStorage Advanced
17. Custom Loaders / Module Hooks
18. Single Executable Applications (SEA)
19. Permission Model
20. QUIC / HTTP/3
21. Advanced Undici & Mocking
22. Graceful Shutdown & Resource Management

---

## 1–15. Core Topics

(ESM, AbortController, streams/pipeline, worker_threads, pino + OpenTelemetry, Zod, LRU + Redis, node:test, helmet & audits, cluster, N-API, hexagonal architecture — retained.)

---

## 16. Diagnostics Channel & AsyncLocalStorage Advanced

```js
import diagnostics_channel from 'node:diagnostics_channel';
import { AsyncLocalStorage } from 'node:async_hooks';

const channel = diagnostics_channel.channel('myapp:request');
const requestContext = new AsyncLocalStorage();

export function withRequest(ctx, fn) {
  return requestContext.run(ctx, () => {
    channel.publish({ type: 'start', ...ctx });
    return fn();
  });
}

channel.subscribe((message) => {
  // export to OpenTelemetry or custom metrics
});
```

---

## 17. Custom Loaders / Module Hooks

```js
export async function resolve(specifier, context, nextResolve) {
  if (specifier.endsWith('.yaml')) {
    return {
      shortCircuit: true,
      url: new URL(specifier, context.parentURL).href,
      format: 'json',
    };
  }
  return nextResolve(specifier, context);
}
```

---

## 18. Single Executable Applications (SEA)

```bash
node --experimental-sea-config sea-config.json
# Produces a portable executable containing your application
```

Ideal for CLI tools.

---

## 19. Permission Model

```bash
node --experimental-permission \
     --allow-fs-read=./data \
     --allow-fs-write=./logs \
     --allow-child-process=false \
     app.js
```

---

## 20. QUIC / HTTP/3

HTTP/3 support continues to mature via undici and native APIs. Track Node.js release notes.

---

## 21. Advanced Undici & Mocking

```js
import { MockAgent, setGlobalDispatcher } from 'undici';

const mockAgent = new MockAgent();
setGlobalDispatcher(mockAgent);

const pool = mockAgent.get('https://api.example.com');
pool.intercept({ path: '/users/1', method: 'GET' })
    .reply(200, { id: 1, name: 'Test' });
```

---

## 22. Graceful Shutdown & Resource Management

```js
const server = app.listen(3000);
const connections = new Set();

server.on('connection', (conn) => {
  connections.add(conn);
  conn.on('close', () => connections.delete(conn));
});

async function shutdown(signal) {
  console.log(`Received ${signal}, shutting down`);
  server.close();
  for (const conn of connections) conn.destroy();
  await closeDb();
  await redis.quit();
  process.exit(0);
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));
```

---

**Key Node.js Recommendations (Updated)**

- Diagnostics Channel + AsyncLocalStorage for context & instrumentation
- Undici MockAgent for HTTP tests
- Design for graceful shutdown from day one
- Explore SEA for distributable CLIs
- Track Permission Model and HTTP/3

*Document version: 2.0 — Expanded*
