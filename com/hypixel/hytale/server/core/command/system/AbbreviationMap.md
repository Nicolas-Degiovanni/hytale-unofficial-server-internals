---
description: Architectural reference for AbbreviationMap
---

# AbbreviationMap

**Package:** com.hypixel.hytale.server.core.command.system
**Type:** Utility

## Definition
```java
// Signature
public class AbbreviationMap<Value> {
```

## Architecture & Concepts

The AbbreviationMap is a specialized, immutable, key-value data structure designed for high-performance lookups where keys can be specified by unambiguous prefixes. Its primary role within the server architecture is to facilitate user-friendly command parsing, allowing players and systems to use shortened versions of command names or arguments.

The core principle is ambiguity resolution. During its construction phase, the AbbreviationMap pre-calculates all possible prefixes for every key provided. If a prefix (e.g., "com") could resolve to multiple distinct full keys (e.g., "command" and "combat"), that prefix is deliberately invalidated by mapping it to a null value. This ensures that any successful lookup is guaranteed to be unambiguous.

This preemptive invalidation makes the `get` operation extremely fast, reducing runtime logic to a simple hash map lookup. The underlying implementation leverages the `fastutil` library's `Object2ObjectOpenHashMap` for memory efficiency and performance, which is critical in a high-throughput environment like a game server's command system.

### Lifecycle & Ownership

-   **Creation:** An AbbreviationMap is never instantiated directly. Instead, it is constructed using the fluent `AbbreviationMapBuilder`, which is obtained via the static `create()` factory method. The builder accumulates key-value pairs and computes the final, immutable map upon calling its `build()` method. This pattern is typically employed during a system's initialization phase, such as when a `CommandRegistry` is loading and registering all available commands.
-   **Scope:** The lifecycle of an AbbreviationMap instance is bound to its owner. In the context of a command system, it is expected to persist for the entire server session.
-   **Destruction:** The object is managed by the Java garbage collector. There are no native resources or explicit cleanup methods required. It is reclaimed once its owning object is garbage collected.

## Internal State & Concurrency

-   **State:** The AbbreviationMap class is **immutable**. Once the `build()` method on its builder is called, the internal map is wrapped in an unmodifiable view, and its state can no longer be altered. All lookup data is computed and finalized at creation time. The associated `AbbreviationMapBuilder` is, by contrast, a mutable, stateful object.

-   **Thread Safety:**
    -   **AbbreviationMap:** Instances are **thread-safe for all read operations**. Its immutability guarantees that concurrent calls to `get` will not interfere with each other.
    -   **AbbreviationMapBuilder:** The builder is **not thread-safe**. It is designed for single-threaded use during an initialization context. Accessing a builder instance from multiple threads without external synchronization will lead to a corrupt internal state and unpredictable behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| create() | AbbreviationMapBuilder | O(1) | Static factory. Returns a new builder instance. |
| get(abbreviation) | Value | O(1) | Retrieves the value for the given key or unambiguous prefix. Returns null if not found or if the prefix is ambiguous. |
| AbbreviationMapBuilder.put(key, value) | AbbreviationMapBuilder | O(1) | Registers a key-value pair. Throws `IllegalArgumentException` if the full key is a duplicate. |
| AbbreviationMapBuilder.build() | AbbreviationMap | O(N\*L) | Consumes the builder to produce an immutable AbbreviationMap. Complexity is proportional to the number of keys (N) and their average length (L). |

## Integration Patterns

### Standard Usage

The AbbreviationMap is intended to be built once during an initialization phase and then used for frequent read operations. The standard pattern involves using the builder to populate the map and then storing the resulting immutable instance for the lifetime of the service.

```java
// 1. Obtain a builder
AbbreviationMap.AbbreviationMapBuilder<Runnable> commandBuilder = AbbreviationMap.create();

// 2. Populate the builder during system initialization
commandBuilder.put("teleport", () -> System.out.println("Teleporting..."));
commandBuilder.put("tell", () -> System.out.println("Messaging..."));
commandBuilder.put("gamemode", () -> System.out.println("Changing gamemode..."));

// 3. Build the final, immutable map
AbbreviationMap<Runnable> commandMap = commandBuilder.build();

// 4. Use the map for lookups at runtime
// This will resolve to the "teleport" command
Runnable command = commandMap.get("tele");
if (command != null) {
    command.run();
}

// This will return null because "t" is ambiguous ("teleport", "tell")
Runnable ambiguousCommand = commandMap.get("t");
```

### Anti-Patterns (Do NOT do this)

-   **Builder Reuse:** Do not call `build()` on a builder and then attempt to continue using it. The builder's state is not reset, and its behavior is undefined for subsequent calls. Always create a new builder for a new map.
-   **Concurrent Builder Modification:** Never share an `AbbreviationMapBuilder` instance across multiple threads without explicit, external locking. The builder's internal state is not protected against race conditions.
-   **Relying on Ambiguous Prefixes:** Do not write logic that expects an ambiguous prefix to resolve to a value. The design explicitly nullifies these prefixes to enforce clarity and prevent bugs. Always check the result of `get` for null.

## Data Pipeline

The AbbreviationMap functions as a resolution step within a larger data flow, typically for processing user or system input.

> Flow:
> Raw Input String -> Command Parser -> **AbbreviationMap.get(token)** -> Resolved Command Object -> Command Executor

