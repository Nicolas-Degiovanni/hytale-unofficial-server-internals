---
description: Architectural reference for TimeModule
---

# TimeModule

**Package:** com.hypixel.hytale.server.core.modules.time
**Type:** Singleton

## Definition
```java
// Signature
public class TimeModule extends JavaPlugin {
```

## Architecture & Concepts
The TimeModule is a core server plugin responsible for bootstrapping the server's time-keeping infrastructure. It does not manage time itself but instead acts as a registrar and initializer within the server's Entity Component System (ECS) framework.

Its primary function is to register the necessary ECS constructs for time management with the central `EntityStoreRegistry`. This includes:
*   **Resources:** Singleton data components that hold the global time state, such as `WorldTimeResource` and `TimeResource`.
*   **Systems:** The logic that operates on these resources each tick, such as `WorldTimeSystems.Ticking` which advances time, and `TimePacketSystem` which synchronizes it with clients.
*   **Commands:** Administrative commands like `TimeCommand` that allow operators to manipulate time.

By centralizing the registration logic here, the TimeModule ensures that all time-related systems are initialized correctly and in the proper order during server startup. This design decouples the time-keeping logic (the Systems) from the server's module loading and lifecycle management, adhering to a clean separation of concerns.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the server's `PluginLoader` during the initial server bootstrap sequence. The constructor requires a `JavaPluginInit` context, which is supplied by the loader. A static reference is stored upon construction, enforcing the Singleton pattern.
- **Scope:** Session-scoped. The TimeModule instance persists for the entire lifetime of the server process. Its lifecycle is directly tied to the server's main run loop.
- **Destruction:** The instance is decommissioned when the server initiates its shutdown sequence and unloads all plugins. The Java garbage collector will reclaim the memory once the plugin's ClassLoader is unloaded.

## Internal State & Concurrency
- **State:** The internal state of the TimeModule is minimal and becomes effectively immutable after the `setup` method completes. It holds two fields, `worldTimeResourceType` and `timeResourceType`, which are handles assigned once during initialization. The actual mutable time data is stored within the `WorldTimeResource` and `TimeResource` instances, which are owned and managed by the `EntityStore`.

- **Thread Safety:** This class is not thread-safe during initialization. The `setup` method, which modifies its internal state, is designed to be called only once by the main server thread during startup. After this phase, the object is safe for concurrent access, as its public methods only return the pre-computed, immutable `ResourceType` handles. All subsequent time-related state changes are managed by the ECS Systems, which are responsible for their own concurrency and thread-safety guarantees.

## API Surface
The public API is minimal, primarily exposing handles to the resources it registers. Direct interaction with this module is uncommon; most game logic interacts with the resources via the `EntityStore`.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | static TimeModule | O(1) | Retrieves the global singleton instance of the TimeModule. |
| getWorldTimeResourceType() | ResourceType | O(1) | Returns the registered type handle for the WorldTimeResource. |
| getTimeResourceType() | ResourceType | O(1) | Returns the registered type handle for the global TimeResource. |

## Integration Patterns

### Standard Usage
Direct interaction with the TimeModule is rare. It is primarily used by the server's core to bootstrap the time systems. Other plugins that need to interoperate at a low level with the time resources might retrieve the type handles, but most will interact with the resources directly through the `EntityStore`.

```java
// Example of a system retrieving a time resource registered by this module.
// Note: Direct interaction with TimeModule is not shown as it's not a standard pattern.

// Within another ECS System:
EntityStore store = ...;
WorldTimeResource time = store.getResource(timeModule.getWorldTimeResourceType());
long currentTime = time.getTimeOfDay();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new TimeModule()`. The class requires a specific `JavaPluginInit` context provided by the plugin loader. Manual instantiation will bypass the server's lifecycle management, overwrite the static singleton instance, and result in a system-wide inconsistent state or a runtime crash.
- **Manual Setup:** Do not invoke the `setup` method manually. This method is part of the plugin lifecycle and is guaranteed to be called exactly once by the plugin manager. Calling it again will cause duplicate registration of systems and resources, leading to unpredictable behavior.
- **Premature Access:** Do not attempt to call `get()` to retrieve the instance before the server's plugin loading phase is complete. This will return a null reference.

## Data Pipeline
The TimeModule does not process data directly. Instead, it *establishes* the data processing pipeline for time by registering the relevant systems. The logical flow of time data after this module has been initialized is as follows:

> Flow:
> Game Loop Tick -> **WorldTimeSystems.Ticking** (System) -> Mutates **WorldTimeResource** (Resource) -> **TimePacketSystem** (System) reads resource -> Creates Network Packet -> Sent to Client

