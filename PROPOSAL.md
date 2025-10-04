# Proposal: Expose Docker HEALTHCHECK Status in Workers Container API

## Summary

Add a `health` property to the Workers container API that exposes the Docker HEALTHCHECK status, allowing developers to programmatically wait for containers to be ready before processing requests.

## Motivation

### Current Problem

When a container-enabled Durable Object starts, the container port becomes accessible before the application inside is ready to accept requests. This causes `getTcpPort().fetch()` calls to hang indefinitely until the application starts listening.

**Current workarounds:**
- Using `AbortSignal.timeout()` with retry loops
- Manual polling with `Promise.race()` and timeouts
- Guessing appropriate sleep durations

**Example of current workaround:**
```javascript
async waitForFlask(maxAttempts = 60) {
  for (let i = 0; i < maxAttempts; i++) {
    try {
      const timeout = new Promise((_, reject) =>
        setTimeout(() => reject(new Error('timeout')), 3000)
      );

      const fetchPromise = this.container.getTcpPort(5005).fetch(
        'http://localhost:5005/',
        { method: 'HEAD' }
      );

      await Promise.race([fetchPromise, timeout]);
      return; // Success
    } catch (error) {
      // Retry...
    }
    await new Promise(resolve => setTimeout(resolve, 2000));
  }
}
```

This is error-prone and wastes time with arbitrary timeouts.

### Why This Matters

Docker already provides a standardized HEALTHCHECK mechanism that applications can define in their Dockerfile:

```dockerfile
HEALTHCHECK --interval=5s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:5005/ || exit 1
```

This health status is available via the Docker API but **not exposed** through the Workers container interface.

## Proposed Solution

Expose Docker's health status through a new `health` property on the `Container` class.

### API Design

```typescript
interface Container {
  readonly running: boolean;
  readonly health: string;  // NEW: "none" | "starting" | "healthy" | "unhealthy" | ""

  start(options?: ContainerStartupOptions): void;
  monitor(): Promise<void>;
  destroy(error?: any): Promise<void>;
  signal(signo: number): void;
  getTcpPort(port: number): Fetcher;
}
```

### Health Status Values

| Value | Meaning |
|-------|---------|
| `"none"` | No health check configured in container |
| `"starting"` | Container is starting, not yet healthy |
| `"healthy"` | Container passed health check and is ready |
| `"unhealthy"` | Container failed health check |
| `""` (empty) | Health status unavailable (backward compat) |

### Usage Example

**Before (with workaround):**
```javascript
export class FavaContainer extends DurableObject {
  constructor(ctx, env) {
    super(ctx, env);
    this.initPromise = this.waitForFlask(); // Complex retry logic
  }

  async waitForFlask(maxAttempts = 60) {
    // 30+ lines of retry/timeout code...
  }
}
```

**After (with health property):**
```javascript
export class FavaContainer extends DurableObject {
  constructor(ctx, env) {
    super(ctx, env);
    this.initPromise = this.waitUntilHealthy();
  }

  async waitUntilHealthy(timeout = 60000) {
    const start = Date.now();

    while (this.ctx.container.health !== 'healthy') {
      if (Date.now() - start > timeout) {
        throw new Error(`Container unhealthy: ${this.ctx.container.health}`);
      }
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
}
```

Much simpler, more reliable, and uses Docker's standard health checking.

## Implementation

I've implemented a working proof-of-concept in my fork: [`alycda/workerd#feature/docker-healthcheck-support`](https://github.com/alycda/workerd/tree/feature/docker-healthcheck-support)

### Changes Required

The implementation touches 9 files across the stack:

1. **Docker API Schema** (`docker-api.capnp`) - Add `Health` struct for parsing
2. **Container Client** (`container-client.h/c++`) - Parse health from Docker inspect
3. **RPC Interface** (`container.capnp`) - Include health in status response
4. **Container API** (`container.h/c++`) - Expose `health` property to JavaScript
5. **Worker/Actor** (`worker.c++`, `actor-state.h/c++`) - Pass health through initialization

### Technical Details

- Health status is fetched during Durable Object initialization via existing `container.status()` RPC
- Uses Docker Engine API's `/containers/{id}/json` endpoint (`State.Health.Status` field)
- Zero performance overhead - piggybacks on existing status check
- Fully backward compatible - containers without HEALTHCHECK return empty string

### Code Size

- **~50 lines of new code** (excluding comments/docs)
- **Minimal complexity** - mostly plumbing existing Docker API data

## Benefits

✅ **Standard**: Uses Docker's official HEALTHCHECK mechanism
✅ **Simple**: Eliminates complex retry logic
✅ **Reliable**: No more guessing timeout values
✅ **Efficient**: Single check during initialization
✅ **Compatible**: Works with any Docker health check definition
✅ **Backward Compatible**: Existing code continues to work unchanged

## Testing

The implementation can be tested with:

```dockerfile
FROM python:3.12-slim

# Install your app
COPY . /app
WORKDIR /app

# Define health check
HEALTHCHECK --interval=5s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:5005/ || exit 1

CMD ["python", "app.py"]
```

```javascript
export class MyContainer extends DurableObject {
  async fetch(request) {
    console.log(`Container health: ${this.ctx.container.health}`);

    // Wait for healthy status
    while (this.ctx.container.health !== 'healthy') {
      await new Promise(resolve => setTimeout(resolve, 1000));
    }

    // Now safe to forward requests
    return this.ctx.container.getTcpPort(5005).fetch(request);
  }
}
```

## Related Issues

- Original issue: https://github.com/cloudflare/workers-sdk/issues/10859
- Discussion with maintainer confirmed `AbortSignal` works but manual polling is required

## Alternative Approaches Considered

1. **Add `container.waitUntilHealthy()` method**
   - Pro: More ergonomic API
   - Con: Less flexible, hides underlying state

2. **Make `getTcpPort()` wait for healthy**
   - Pro: Automatic, no code changes
   - Con: Magic behavior, hard to debug, breaks for containers without health checks

3. **Status quo (use AbortSignal + manual polling)**
   - Pro: Already works
   - Con: Inefficient, complex, error-prone

**Exposing the `health` property is the best balance** - simple, explicit, flexible.

## Questions for Maintainers

1. Is this the right API design, or would you prefer a different approach?
2. Should health status be refreshed on subsequent accesses, or only at initialization?
3. Are there any edge cases or container runtimes this needs to handle?
4. Would you like me to add TypeScript types to `@cloudflare/workers-types`?

## Implementation Status

- ✅ Working implementation in fork
- ✅ Full stack integration (Docker API → JS API)
- ✅ Documentation and examples
- ⏳ Needs CI validation (will run on PR)
- ⏳ Needs integration tests
- ⏳ Needs TypeScript types update

I'm happy to refine the implementation based on feedback and open a PR when you're ready!

---

**Links:**
- Fork with implementation: https://github.com/alycda/workerd/tree/feature/docker-healthcheck-support
- Original bug report: https://github.com/cloudflare/workers-sdk/issues/10859
