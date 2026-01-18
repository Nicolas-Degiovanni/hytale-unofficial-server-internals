---
description: Architectural reference for IRegistry
---

# IRegistry

**Package:** com.hypixel.hytale.server.core.plugin.registry
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface IRegistry {
```

## Architecture & Concepts
The IRegistry interface defines the fundamental contract for any component that manages a collection of objects within the Hytale server environment. It serves as a foundational abstraction for service locators, plugin managers, and other catalog-like systems.

By conforming to this interface, a class signals to the server's core lifecycle manager that it holds resources or maintains state which must be explicitly released during a controlled shutdown. This contract is critical for preventing resource leaks, ensuring data is saved correctly, and allowing for a graceful server termination. It is a core component of the server's Inversion of Control (IoC) container, establishing a standardized teardown protocol.

## Lifecycle & Ownership
As an interface, IRegistry does not have its own lifecycle. Instead, it *defines* a critical phase in the lifecycle of its implementing classes.

- **Creation:** Implementations are typically instantiated by the core server bootstrap process or a parent plugin manager. They are long-lived, foundational services.
- **Scope:** An IRegistry implementation is expected to persist for the entire duration of the server session. Its purpose is to manage objects that are globally or regionally accessible.
- **Destruction:** The server's primary shutdown sequence is responsible for invoking the `shutdown` method on all registered IRegistry instances. This is the sole supported mechanism for teardown. Manual invocation outside of this sequence can lead to an unstable server state.

## Internal State & Concurrency
The IRegistry interface is stateless.

- **State:** This contract makes no assumptions about the internal state of implementing classes. Implementations are typically stateful, often containing collections (Maps, Lists) of the objects they manage.
- **Thread Safety:** The interface itself provides no concurrency guarantees. **WARNING:** Implementations are solely responsible for ensuring their own thread safety. It is common for registries to be accessed from multiple game-loop or network threads, making synchronization (e.g., using ConcurrentHashMap or explicit locks) a critical implementation detail.

## API Surface
The public contract consists of a single method designed for lifecycle management.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| shutdown() | void | O(N) | Signals the registry to release all managed resources, unregister listeners, and perform cleanup. Complexity is typically linear to the number of items managed. |

## Integration Patterns

### Standard Usage
This interface is not meant to be used directly by typical game logic or plugin developers. It is implemented by core systems that need to participate in the server's managed lifecycle.

```java
// Example of a class implementing the contract
public class PluginRegistry implements IRegistry {
    private final Map<String, Plugin> loadedPlugins = new ConcurrentHashMap<>();

    // ... methods to register and retrieve plugins

    @Override
    public void shutdown() {
        // This is the critical implementation of the contract.
        // The server core will call this method during shutdown.
        for (Plugin plugin : loadedPlugins.values()) {
            plugin.disable();
        }
        loadedPlugins.clear();
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Invocation:** Do not call `shutdown` directly. The server's lifecycle manager is the exclusive owner of this process. Calling it prematurely will break dependencies and destabilize the server.
- **Incomplete Cleanup:** An implementation of `shutdown` that fails to release all resources (e.g., file handles, network connections, database pools) constitutes a resource leak and violates the contract.
- **Blocking Operations:** The `shutdown` method should execute in a timely manner. Performing long-running or indefinite blocking operations within this method can prevent the server from terminating correctly.

## Data Pipeline
This interface does not participate in a data processing pipeline. It is part of the server's control flow for lifecycle management.

> Flow:
> Server Shutdown Signal -> LifecycleManager -> **IRegistry.shutdown()** -> Resource Release

