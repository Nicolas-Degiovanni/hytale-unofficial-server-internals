---
description: Architectural reference for FunctionManager
---

# FunctionManager

**Package:** com.hypixel.hytale.function
**Type:** Singleton

## Definition
```java
// Signature
public final class FunctionManager implements IService, IReloadableResource {
```

## Architecture & Concepts
The FunctionManager is the central authority for the loading, compilation, and execution of server-side game logic, known as Functions. It serves as the primary bridge between the core game engine events and user-defined scripts, enabling content creators to customize and extend gameplay behavior without modifying the engine's source code.

Architecturally, this system is positioned as a high-level service that consumes low-level game events. It subscribes to the server's main EventBus and listens for specific triggers, such as player actions, entity lifecycle events, or scheduled ticks. Upon receiving a relevant event, it identifies the appropriate Function to execute, constructs an ExecutionContext containing relevant world state and event data, and dispatches the call to its internal script execution engine.

The manager abstracts the underlying scripting language and runtime (e.g., a sandboxed JavaScript V8 instance or a proprietary bytecode interpreter). Its primary responsibility is not the execution itself, but the lifecycle management of scripts and the enforcement of performance and security constraints.

### Lifecycle & Ownership
- **Creation:** The FunctionManager is instantiated once per server session by the primary ServiceRegistry during the server's bootstrap sequence. Its initialization is a critical step in the startup process and must complete before the game world begins ticking.
- **Scope:** The instance is a session-scoped singleton, persisting for the entire lifetime of the running server. It is never re-created during a single session.
- **Destruction:** The instance is marked for garbage collection during the server shutdown sequence. The shutdown hook ensures that the underlying script execution engine is properly terminated and all file handles are released. Failure to shut down gracefully can lead to resource leaks.

## Internal State & Concurrency
- **State:** The FunctionManager maintains a highly mutable internal state. Its core component is a cache mapping a FunctionIdentifier to a pre-compiled, executable script object. This cache is populated during server startup and can be modified at runtime via hot-reloading commands.

- **Thread Safety:** This class is designed to be thread-safe, but with specific caveats. While the function cache itself uses concurrent collections to allow safe reads from multiple threads, all script *execution* is marshaled onto a dedicated, single-threaded executor. This design choice serializes all world-state mutations originating from scripts, preventing complex race conditions and ensuring deterministic outcomes.

**WARNING:** Directly accessing the internal cache from outside this class is strictly forbidden. All interactions must occur through the public API to guarantee thread safety.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| loadFunctions(source) | CompletableFuture<Void> | O(n) | Asynchronously discovers, loads, and compiles all functions from the given resource source. This is a high-latency operation. |
| invoke(id, context) | FunctionResult | O(1) | Executes a pre-compiled function identified by its ID with the provided context. Throws FunctionNotFoundException if the ID is not registered. |
| reload() | CompletableFuture<Void> | O(n) | Clears the entire function cache and re-triggers the load process. This is a disruptive operation intended for development environments only. |

## Integration Patterns

### Standard Usage
The FunctionManager should always be retrieved from the server's central service context. It is most commonly used within event handlers to trigger custom game logic.

```java
// Correctly invoking a function from an event listener
public void onPlayerBreaksBlock(PlayerBlockBreakEvent event) {
    FunctionManager funcManager = Server.get().getServiceRegistry().getService(FunctionManager.class);
    
    ExecutionContext context = new ExecutionContext.Builder()
        .withSource(event.getPlayer())
        .withTarget("block", event.getBlock())
        .build();
        
    FunctionIdentifier id = new FunctionIdentifier("my_mod", "on_block_break");
    funcManager.invoke(id, context);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call new FunctionManager(). The system relies on a single, managed instance. Direct instantiation will result in an uninitialized and non-functional object, leading to NullPointerExceptions.
- **Synchronous Loading:** Do not call loadFunctions().get() on the main server thread. This will block all server activity, including networking and world ticks, potentially causing a server crash.
- **Context Reuse:** The ExecutionContext object is transient and may be pooled. Do not store references to it outside the immediate scope of the call, as its internal state is not guaranteed to be valid after the invoke method returns.

## Data Pipeline
The FunctionManager acts as a critical processing stage in the server's event-to-logic pipeline. It transforms abstract game events into concrete, script-driven actions.

> Flow:
> Player Input -> Network Packet -> Server Game Loop -> **PlayerBlockBreakEvent** -> EventBus -> Custom EventHandler -> **FunctionManager.invoke()** -> Script Execution -> World State Mutation (e.g., drop item) -> Network Sync to Client

---

