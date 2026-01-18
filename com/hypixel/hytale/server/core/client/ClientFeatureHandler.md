---
description: Architectural reference for ClientFeatureHandler
---

# ClientFeatureHandler

**Package:** com.hypixel.hytale.server.core.client
**Type:** Utility

## Definition
```java
// Signature
public class ClientFeatureHandler {
```

## Architecture & Concepts
The ClientFeatureHandler is a stateless, static utility class that acts as a global dispatcher for client-side feature flags. Its primary role is to broadcast the state of a given ClientFeature to every active World instance managed by the central Universe singleton.

This class provides a crucial layer of abstraction, decoupling the system that originates a feature change (e.g., a network packet handler, a command processor) from the underlying world management system. Instead of requiring callers to fetch the list of all worlds and iterate over them, ClientFeatureHandler exposes a single, high-level entry point to perform a server-wide state change. It embodies the Facade pattern for the complex operation of "update all worlds with this feature".

## Lifecycle & Ownership
- **Creation:** As a static utility class, ClientFeatureHandler is never instantiated. Its methods are available immediately after the class is loaded by the Java Virtual Machine.
- **Scope:** The class and its methods are globally accessible throughout the server's runtime. It is entirely stateless and its lifetime is tied to the JVM process itself.
- **Destruction:** Not applicable. The class is unloaded when the server application shuts down.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no member fields, caches no data, and all its operations are pure functions of their inputs and the global Universe state.

- **Thread Safety:** **Not guaranteed.** While the class itself has no internal state to protect, its methods operate on the shared, mutable state of the Universe singleton.
    - Calls to `register` and `unregister` iterate over the collection returned by `Universe.get().getWorlds()`. If another thread modifies this collection (e.g., by adding or removing a World) during iteration, a `ConcurrentModificationException` will occur.
    - **WARNING:** The caller is responsible for ensuring that no concurrent modifications are made to the Universe's world list while these methods are executing. Synchronization, if required, must be handled externally, typically by locking on the Universe object.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| register(feature) | static void | O(N) | Enables the specified feature across all N active worlds. This is a global, blocking operation. |
| unregister(feature) | static void | O(N) | Disables the specified feature across all N active worlds. This is a global, blocking operation. |

## Integration Patterns

### Standard Usage
The class is intended to be used via direct static calls from any system that needs to toggle a feature flag globally. This typically occurs when a client's capabilities are first identified or when a server-wide setting is changed.

```java
// Enable a specific feature for all players and worlds
ClientFeatureHandler.register(ClientFeature.ADVANCED_RENDERING);

// Later, disable it
ClientFeatureHandler.unregister(ClientFeature.ADVANCED_RENDERING);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance with `new ClientFeatureHandler()`. It provides no value, as all methods are static.
- **Concurrent World Modification:** Do not call `register` or `unregister` from one thread while another thread might be adding or removing a World from the Universe. This will lead to runtime exceptions and unpredictable state.

```java
// ANTI-PATTERN: Potential race condition
// Thread A
new Thread(() -> ClientFeatureHandler.register(someFeature)).start();

// Thread B - This could cause a ConcurrentModificationException
new Thread(() -> Universe.get().unloadWorld("world_lobby")).start();
```

## Data Pipeline
ClientFeatureHandler acts as a command broadcaster rather than a step in a data processing pipeline. It takes a command (register/unregister a feature) and dispatches it to multiple subsystems (all World instances).

> Flow:
> External Trigger (e.g., Network Handler, Console Command) -> **ClientFeatureHandler.register()** -> Universe.get().getWorlds() -> World.registerFeature() [Invoked for each World]

