# Adding Docker HEALTHCHECK Support to Cloudflare Workers Containers

## Summary
Add support for exposing Docker HEALTHCHECK status through the Workers container API, allowing developers to properly wait for containers to be healthy before making requests.

## Problem
Currently, when a container starts, the port becomes connectable before the application inside is ready to accept requests. This causes `getTcpPort().fetch()` to hang until the application starts listening. Users must work around this by:
1. Using `AbortSignal.timeout()` and retry loops
2. Using `Promise.race()` with timeouts
3. Manual polling with error handling

Docker already has HEALTHCHECK support, but this isn't exposed through the Workers API.

## Proposed Solution

### 1. Update Docker API Schema (`src/workerd/server/docker-api.capnp`)

Add Health struct to ContainerState:

```capnp
struct ContainerState {
  status @0 :Text $Json.name("Status");
  running @1 :Bool $Json.name("Running");
  paused @2 :Bool $Json.name("Paused");
  restarting @3 :Bool $Json.name("Restarting");
  oomKilled @4 :Bool $Json.name("OOMKilled");
  dead @5 :Bool $Json.name("Dead");
  pid @6 :UInt32 $Json.name("Pid");
  exitCode @7 :Int32 $Json.name("ExitCode");
  error @8 :Text $Json.name("Error");
  startedAt @9 :Text $Json.name("StartedAt");
  finishedAt @10 :Text $Json.name("FinishedAt");
  health @11 :Health $Json.name("Health");  # NEW
}

struct Health {
  # Docker HEALTHCHECK status
  # See: https://docs.docker.com/engine/api/v1.43/#tag/Container/operation/ContainerInspect
  status @0 :Text $Json.name("Status");  # "none", "starting", "healthy", "unhealthy"
  failingStreak @1 :UInt32 $Json.name("FailingStreak");
  # Could also add log @2 :List(HealthLog) if needed
}
```

### 2. Update InspectResponse (`src/workerd/server/container-client.h`)

```c++
struct InspectResponse {
  bool isRunning;
  kj::HashMap<uint16_t, uint16_t> ports;
  kj::Maybe<kj::String> healthStatus;  // NEW: "none", "starting", "healthy", "unhealthy"
};
```

### 3. Update inspectContainer() (`src/workerd/server/container-client.c++`)

Parse health status from Docker API response:

```c++
kj::Promise<ContainerClient::InspectResponse> ContainerClient::inspectContainer() {
  // ... existing code ...

  // Parse health status if available
  kj::Maybe<kj::String> healthStatus = kj::none;
  if (state.hasHealth()) {
    auto health = state.getHealth();
    if (health.hasStatus()) {
      healthStatus = kj::str(health.getStatus());
    }
  }

  co_return InspectResponse{
    .isRunning = running,
    .ports = kj::mv(portMappings),
    .healthStatus = kj::mv(healthStatus)
  };
}
```

### 4. Expose through Container API (`src/workerd/api/container.h`)

Add health property to Container class:

```c++
class Container: public jsg::Object {
 public:
  // ... existing code ...

  jsg::Optional<kj::String> getHealth() {
    return health;
  }

  JSG_RESOURCE_TYPE(Container, CompatibilityFlags::Reader flags) {
    JSG_READONLY_PROTOTYPE_PROPERTY(running, getRunning);
    JSG_READONLY_PROTOTYPE_PROPERTY(health, getHealth);  // NEW
    // ... rest of methods ...
  }

 private:
  IoOwn<rpc::Container::Client> rpcClient;
  bool running;
  jsg::Optional<kj::String> health;  // NEW
  // ... rest of fields ...
};
```

### 5. Update Container RPC interface (`src/workerd/io/container.capnp`)

```capnp
interface Container @0x9aaceefc06523bca {
  status @0 () -> (running :Bool, health :Text);  # Add health to return
  # ... rest of interface ...
}
```

### 6. Update Container constructor to fetch health

In `src/workerd/api/container.c++`:

```c++
Container::Container(rpc::Container::Client rpcClient, bool running)
    : rpcClient(kj::mv(rpcClient)), running(running) {
  // Optionally fetch health status on construction
  // Or make it lazy-loaded via a getter
}
```

## Usage Example

After implementation, users could write:

```javascript
export class FavaContainer extends DurableObject {
  async waitForFlask() {
    while (this.container.health !== 'healthy') {
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }
}
```

Or even better with a helper:

```javascript
async waitUntilHealthy(timeout = 60000) {
  const start = Date.now();
  while (this.container.health !== 'healthy') {
    if (Date.now() - start > timeout) {
      throw new Error('Container failed to become healthy');
    }
    await new Promise(resolve => setTimeout(resolve, 1000));
  }
}
```

## Testing

1. Create a test Dockerfile with HEALTHCHECK
2. Verify `container.health` returns correct values
3. Test all states: "none", "starting", "healthy", "unhealthy"
4. Ensure backward compatibility (containers without HEALTHCHECK return "none")

## Feasibility

✅ **Highly Feasible**:
- Docker API already exposes `State.Health` in inspect response
- workerd already uses Docker API v1.50
- Similar to existing `running` property implementation
- No new Docker features needed
- Backward compatible (health status is optional)

## Related Files

- `src/workerd/api/container.h` - Container API
- `src/workerd/api/container.c++` - Container implementation
- `src/workerd/server/container-client.h` - Docker client interface
- `src/workerd/server/container-client.c++` - Docker client implementation
- `src/workerd/server/docker-api.capnp` - Docker API schema
- `src/workerd/io/container.capnp` - Container RPC interface

## References

- Docker Engine API: https://docs.docker.com/engine/api/v1.43/#tag/Container/operation/ContainerInspect
- Docker HEALTHCHECK: https://docs.docker.com/reference/dockerfile/#healthcheck
- Issue: https://github.com/cloudflare/workers-sdk/issues/10859
