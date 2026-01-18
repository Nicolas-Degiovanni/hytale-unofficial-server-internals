---
description: Architectural reference for RemovalCondition
---

# RemovalCondition

**Package:** com.hypixel.hytale.builtin.instances.removal
**Type:** Strategy Interface

## Definition
```java
// Signature
public interface RemovalCondition {
```

## Architecture & Concepts
The RemovalCondition interface defines a contract for implementing world pruning and garbage collection logic. It embodies the Strategy Pattern, decoupling the core world management system from the specific rules that determine when a world instance is eligible for removal. This allows for a highly configurable and extensible system where server administrators can define complex world lifecycle rules through data, rather than hard-coded logic.

The static CODEC field is the central mechanism for this data-driven design. It enables the server to deserialize various concrete implementations of RemovalCondition from configuration files. The server's world persistence layer queries these conditions to decide whether to offload or permanently delete world data that is no longer active or relevant.

**WARNING:** Implementations of this interface are critical to server stability and data integrity. A faulty implementation can lead to premature world deletion and permanent data loss.

## Lifecycle & Ownership
As an interface, RemovalCondition itself has no lifecycle. The following pertains to its concrete implementations.

-   **Creation:** Instances are almost exclusively created by the server's configuration loading system during bootstrap or world initialization. The system uses the static CODEC field to look up the correct implementation based on a "Type" identifier in the configuration data and deserializes it.
-   **Scope:** The lifetime of a RemovalCondition instance is tied to the scope of the configuration in which it is defined. Typically, it persists as long as the server or a specific world is running.
-   **Destruction:** Instances are garbage collected when their owning configuration is unloaded, usually during a server shutdown or a dynamic world unload event.

## Internal State & Concurrency
-   **State:** The interface is inherently stateless. Implementations should be designed to be immutable or have their state configured entirely at creation time. Mutable state within a RemovalCondition is a significant anti-pattern and can lead to unpredictable behavior.
-   **Thread Safety:** The interface itself provides no thread safety guarantees. It is the absolute responsibility of the implementing class to be thread-safe. The world management system may invoke the shouldRemoveWorld method from a background maintenance thread. Therefore, all implementations **MUST** be thread-safe without external synchronization.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| shouldRemoveWorld(Store<ChunkStore> var1) | boolean | Varies | Evaluates the condition against a given world's chunk storage. Returns true if the world should be removed. |

## Integration Patterns

### Standard Usage
A server administrator defines a set of removal conditions in a world configuration file. The server deserializes these into a list of RemovalCondition objects and periodically evaluates them against active worlds.

```java
// Conceptual server-side usage
// This code does not exist in this file but illustrates the pattern.

List<RemovalCondition> conditions = worldConfig.getRemovalConditions();
World targetWorld = server.getWorld("example_world");

boolean shouldPrune = conditions.stream()
    .anyMatch(condition -> condition.shouldRemoveWorld(targetWorld.getChunkStore()));

if (shouldPrune) {
    server.scheduleForRemoval(targetWorld);
}
```

### Anti-Patterns (Do NOT do this)
-   **Blocking Operations:** Do not perform long-running or blocking operations (e.g., network calls, heavy disk I/O) within the shouldRemoveWorld method. This can stall the server's world maintenance thread, causing severe performance degradation.
-   **Stateful Implementations:** Avoid creating implementations that rely on mutable internal state that changes over time. This can lead to non-deterministic behavior and race conditions if the condition is evaluated concurrently for multiple worlds.
-   **Ignoring the CODEC:** Do not bypass the CODEC system for serialization. The engine relies on this mechanism for configuration hot-reloading and data-driven world management.

## Data Pipeline
The data flow for this component is primarily driven by server configuration and maintenance cycles.

> Flow:
> Server Configuration File (HOCON/JSON) -> Deserializer (using CODEC) -> **RemovalCondition Instance** -> World Maintenance System -> Evaluation of shouldRemoveWorld -> World Removal Decision

