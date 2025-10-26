# Docker HEALTHCHECK Support Implementation

## Changes Made

All changes implement exposing Docker HEALTHCHECK status through the Workers container API.

### 1. Docker API Schema (`src/workerd/server/docker-api.capnp`)
- Added `Health` struct to represent Docker HEALTHCHECK status
- Added `health` field to `ContainerState` struct
- Health status values: "none", "starting", "healthy", "unhealthy"

### 2. Container Client (`src/workerd/server/container-client.h` & `.c++`)
- Updated `InspectResponse` struct to include `healthStatus` field
- Modified `inspectContainer()` to parse health status from Docker API response
- Updated `status()` RPC implementation to return health status
- Health status is extracted from `State.Health.Status` in Docker inspect response

### 3. Container RPC Interface (`src/workerd/io/container.capnp`)
- Updated `status()` method signature to return both `running` and `health`
- Added documentation for health status values

### 4. Container API (`src/workerd/api/container.h` & `.c++`)
- Added `health` private member variable to `Container` class
- Implemented `getHealth()` public method
- Exposed `health` as read-only property in JavaScript
- Updated constructor to accept health parameter

### 5. Durable Object State (`src/workerd/api/actor-state.h` & `.c++`)
- Updated `DurableObjectState` constructor to accept `containerHealth` parameter
- Passed health status to `Container` constructor when creating container instances

### 6. Worker Actor (`src/workerd/io/worker.c++`)
- Modified `ensureConstructedImpl()` to fetch and store health status from RPC call
- Passed health status through to `DurableObjectState` constructor

## How It Works

1. When a Durable Object with a container starts, `ensureConstructedImpl()` calls `container.status()` RPC
2. The RPC calls Docker inspect API and parses the `State.Health.Status` field
3. Health status is returned alongside the `running` boolean
4. Health status is stored in the `Container` instance and exposed via `container.health` property
5. JavaScript code can now access `this.container.health` to check container health

## Usage Example

```javascript
export class MyContainer extends DurableObject {
  async constructor(ctx, env) {
    super(ctx, env);

    // Wait for container to be healthy before handling requests
    await this.waitUntilHealthy();
  }

  async waitUntilHealthy(timeout = 60000) {
    const start = Date.now();

    while (this.ctx.container.health !== 'healthy') {
      if (Date.now() - start > timeout) {
        throw new Error(`Container failed to become healthy: ${this.ctx.container.health}`);
      }

      console.log(`Container health: ${this.ctx.container.health}, waiting...`);
      await new Promise(resolve => setTimeout(resolve, 1000));
    }

    console.log('Container is healthy!');
  }

  async fetch(request) {
    // Container is guaranteed to be healthy here
    return new Response('OK');
  }
}
```

## Testing

To test these changes:

1. Build workerd with the changes
2. Create a Dockerfile with HEALTHCHECK:
   ```dockerfile
   HEALTHCHECK --interval=5s --timeout=3s --retries=3 \
     CMD curl -f http://localhost:5005/ || exit 1
   ```
3. Run a Durable Object with the container
4. Access `container.health` property
5. Verify it returns correct values: "starting" → "healthy"

## Backward Compatibility

- Containers without HEALTHCHECK will return empty string `""` for health
- Existing code that doesn't use `container.health` continues to work unchanged
- The `running` property behavior is unchanged

## Files Modified

- `src/workerd/server/docker-api.capnp`
- `src/workerd/server/container-client.h`
- `src/workerd/server/container-client.c++`
- `src/workerd/io/container.capnp`
- `src/workerd/api/container.h`
- `src/workerd/api/container.c++`
- `src/workerd/api/actor-state.h`
- `src/workerd/api/actor-state.c++`
- `src/workerd/io/worker.c++`

## Benefits

- **No more manual polling**: Can check health status directly instead of retry loops with timeouts
- **Standardized**: Uses Docker's built-in HEALTHCHECK mechanism
- **Efficient**: Health status is fetched once during DO initialization
- **Compatible**: Works with existing Docker health check definitions
