# Testing Side Effects

Patterns for making backbone tests deterministic against external services, filesystems, time, and processes. All fakes below must carry a **fidelity contract** — see "Fidelity Contract" in the backbone plan interview (SKILL.md §2d). A fake without one is a liability, not a test double.

## Fakes

In-process server speaking the real wire protocol. Gold standard: deterministic, fast, no network.

### WebSocket fake (streaming tail)

Real streaming services parse the query at handshake before subscribing the client. A fake that pushes messages without reading the query (or the client's first frame) hides handshake bugs — the most common fidelity hole. Assert the handshake *before* sending, and record every client frame so the test can assert on it:

```typescript
import { WebSocketServer } from 'ws';

async function fakeStreamTail(port: number, messages: object[] = [], expectedQuery?: string) {
  const wss = new WebSocketServer({ port });
  await new Promise<void>(resolve => wss.on('listening', resolve));
  const assignedPort = (wss.address() as any).port;
  const received: string[] = [];
  wss.on('connection', (ws, req) => {
    const url = new URL(req.url!, 'http://localhost');
    if (expectedQuery !== undefined) {
      const q = url.searchParams.get('query');
      if (q !== expectedQuery) {
        ws.close(1008, `expected query '${expectedQuery}', got '${q}'`);
        return;
      }
    }
    ws.on('message', (data) => received.push(data.toString()));
    for (const msg of messages) ws.send(JSON.stringify(msg));
  });
  return { close: () => wss.close(), received, port: assignedPort };
}
```

### HTTP fake (REST APIs)

```typescript
import http from 'http';

async function fakeHTTP(
  port: number,
  handler: (req: http.IncomingMessage, body: Buffer) => { status: number; headers?: Record<string, string>; body: object }
) {
  const server = http.createServer((req, res) => {
    const chunks: Buffer[] = [];
    req.on('data', (c) => chunks.push(c));
    req.on('end', () => {
      const { status, headers, body } = handler(req, Buffer.concat(chunks));
      res.writeHead(status, { 'Content-Type': 'application/json', ...headers });
      res.end(JSON.stringify(body));
    });
  });
  await new Promise<void>(resolve => server.listen(port, resolve));
  const assignedPort = (server.address() as any).port;
  return { close: () => server.close(), port: assignedPort };
}
```

Model error responses, not just the happy path. A fake that only ever returns 200 hides retry/backoff bugs. Cover the shapes the real client must handle: `429 Retry-After`, `503`, `5xx-with-backoff`, malformed bodies, slow/dropped connections.

### Asserting requests

A fake must verify the client sent the right request. HTTP example:

```typescript
const seen: string[] = [];
const fake = await fakeHTTP(0, (req) => {
  seen.push(req.url!);
  expect(req.headers.authorization).toBe('Bearer test-token');
  return { status: 200, body: { data: 'ok' } };
});
// ... run system ...
expect(seen).toContain('/api/v0/projects/myproject/events/');
```

The WebSocket example above exposes `received` for the parallel assertion — assert on the client's first frame, the query it sent, and the order of frames it emits. WebSocket frames are easier to drop silently than HTTP bodies; the assertion is more important, not less.

### Port allocation

Hardcoded ports collide when tests parallelize. Pass `port: 0` and read the assigned port from `server.address().port`, then thread it into the system under test via env/config. This keeps the determinism requirement intact under parallel runners.

## Record/Replay

Capture real responses once, check fixtures into version control, replay in tests. A fixture is faithful **at capture time only** — it drifts with the service. Every fixture file must open with a header recording the source URL, service version, and capture date; the plan flags a re-record when the cited version is bumped.

```typescript
// Record (run once manually against the real service)
import fs from 'fs';
const res = await fetch('https://api.example.com/v1/stream?...');
const captured = {
  // Fixture header — required before any payload.
  capturedAt: '2025-06-28T14:03:22Z',
  source: 'https://api.example.com/v1/stream',
  serviceVersion: 'v2.7.0',
  payload: await res.text(),
};
fs.writeFileSync('fixtures/stream-response.json', JSON.stringify(captured, null, 2));

// Replay (test)
import nock from 'nock';
const fixture = JSON.parse(fs.readFileSync('fixtures/stream-response.json', 'utf-8'));
nock('https://api.example.com').get('/v1/stream').reply(200, fixture.payload);
```

Redact secrets before committing fixtures.

## Contract Stubs

Hardcoded minimal response. Cheapest to build, fragile when the service changes. A stub is still a simulation of a real system — it needs a **citation** (the spec clause, doc page, or recorded exchange it was hardcoded from) and a **drift trigger** (re-derive when the cited source version changes). Without those, "stub" is paperwork, not a contract.

```typescript
// Citation: openapi.yml §4.2 @ abc123 — service v3.2.1
const stubStreamMessage = (values: [string, string][] = []) => ({
  streams: [{ stream: { service: 'test' }, values }],
  dropped_entries: [],
});

// Citation: events-api.yml §Events §2 @ abc123 — service v2.7.0
const stubApiEvent = (id: string, title: string) => ({
  id, title, 'event.type': 'error',
  dateCreated: new Date().toISOString(),
});
```

## Live Sandbox

Real service, test credentials. For smoke tests only — flag as optional in the plan. Sandboxes are nondeterministic (eventual consistency, shared state, rate limits). Require **idempotent setup** (the test can be run repeatedly without polluting state) and a **reproducibility flag** so a passing run is interpretable.

```markdown
### Task N: Smoke — Live upstream connection

**Step N — Smoke test:**
Run `MY_TOOL=1 npx tsx cli.ts --timeout 30`
Expected: connects, receives >=1 record within 30s, exits 0.
Setup: reset sandbox via `scripts/seed-sandbox.sh` (idempotent).

Skip if `UPSTREAM_TOKEN` is unset or offline.
```

## File System

```typescript
import { mkdtemp, rm } from 'fs/promises';
import { join } from 'path';
import { tmpdir } from 'os';

let dir: string;
beforeEach(async () => { dir = await mkdtemp(join(tmpdir(), 'test-')); });
afterEach(async () => { await rm(dir, { recursive: true }); });
```

If the CLI writes config files, point it at the temp dir via env vars.

## Time

```typescript
beforeEach(() => {
  jest.useFakeTimers();
  jest.setSystemTime(new Date('2025-06-28T14:03:22.123Z'));
});
afterEach(() => { jest.useRealTimers(); });
```

For CLI tests, pin the `TZ` env var: `TZ=UTC` produces predictable timestamp output.

## CLI Process Spawning

For tools tested at the CLI level:

```typescript
import { spawn } from 'child_process';

function runCli(
  args: string[],
  env: Record<string, string> = {},
  timeoutMs = 5000
): Promise<{ stdout: string; stderr: string; code: number }> {
  return new Promise((resolve, reject) => {
    const proc = spawn('node', ['dist/cli.js', ...args], {
      env: { ...process.env, ...env },
      stdio: 'pipe',
    });
    let stdout = '', stderr = '';
    proc.stdout.on('data', (d: Buffer) => stdout += d.toString());
    proc.stderr.on('data', (d: Buffer) => stderr += d.toString());
    const timer = setTimeout(() => { proc.kill(); reject(new Error('timeout')); }, timeoutMs);
    proc.on('close', (code) => { clearTimeout(timer); resolve({ stdout, stderr, code: code ?? 0 }); });
  });
}

// Usage in a backbone test — note the OS-assigned port threaded in via env.
let fake: Awaited<ReturnType<typeof fakeStreamTail>>;
let port: number;
beforeAll(async () => {
  fake = await fakeStreamTail(0, [stubStreamMessage([['1719600000000000000', 'hello']])], '{job="test"}');
  port = fake.port;
  process.env.UPSTREAM_URL = `localhost:${port}`;
  process.env.UPSTREAM_QUERY = '{job="test"}';
  process.env.UPSTREAM_TOKEN = 't';
  process.env.TZ = 'UTC';
});
afterAll(() => fake.close());

test('prints startup summary with one source', async () => {
  const { stdout, code } = await runCli([]);
  expect(code).toBe(0);
  expect(stdout).toContain('Tailing 1 source');
  expect(fake.received).toEqual([]);
});
```

## Summary

| Technique | Deterministic | Speed | Setup | Fidelity contract | When |
|-----------|:---:|---:|:---:|------|------|
| Fake | Yes | Fast | Medium | **Shape** (spec + service version) **and runtime** (dual exec OR replay with normalization) | Primary choice |
| Record/Replay | Yes | Fast | Low | Fixture header (source, version, capture date); re-record on version bump | Complex wire formats |
| Contract Stub | Yes | Fast | Lowest | Shape via **citation** + drift trigger; note residual risk | Early dev, simple APIs |
| Live Sandbox | No | Slow | Low | Idempotent setup + reproducibility flag | Smoke only, optional |
| Temp dir | Yes | Fast | Low | n/a (not a service double) | File output |
| Fake timers | Yes | Fast | Low | n/a | Timestamp output |
| Process spawn | Yes | Med | Low | Subprocess inherits the above | CLI tools |