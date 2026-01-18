---
description: Architectural reference for EnumSetHolder
---

# EnumSetHolder

**Package:** com.hypixel.hytale.server.npc.asset.builder.holder
**Type:** Transient Data Holder

## Definition
```java
// Signature
public class EnumSetHolder<E extends Enum<E>> extends ArrayHolder {
```

## Architecture & Concepts

The EnumSetHolder is a specialized component within the server's NPC Asset Builder framework. It serves as a strongly-typed container for configuration values that represent a collection of enums, such as a set of valid states or capabilities for an NPC.

Architecturally, it extends the more generic ArrayHolder, inheriting its ability to parse and manage dynamic string array expressions from JSON configuration files. The primary responsibility of EnumSetHolder is to perform the critical translation from this raw string array into a highly optimized `java.util.EnumSet`. This provides both type safety and significant performance benefits at runtime compared to using a standard List or Set of strings.

A key concept is its support for both static and dynamic resolution:
*   **Static Resolution:** If the underlying JSON value is a simple, constant array of strings, the EnumSet is computed once during the asset loading phase and cached internally.
*   **Dynamic Resolution:** If the value is an expression that depends on game state, the EnumSet is re-evaluated on each call to the `get` method, using the provided ExecutionContext to resolve the expression. This allows for dynamic NPC behaviors defined directly in configuration.

## Lifecycle & Ownership

-   **Creation:** An EnumSetHolder is instantiated exclusively by a `BuilderBase` subclass during the deserialization of an NPC asset file. It is an internal implementation detail of the asset pipeline and is never created directly by game logic developers.
-   **Scope:** The object's lifetime is strictly bound to the construction process of its parent asset. It is a short-lived, transient object that holds an intermediate representation of a configuration value.
-   **Destruction:** It becomes eligible for garbage collection as soon as the parent builder completes its work and the final, resolved EnumSet is transferred to the runtime NPC data model.

## Internal State & Concurrency

-   **State:** Mutable. The internal `value` field, which holds the final EnumSet, is populated lazily. For performance, it caches the enum's class and its constant values upon initialization. For static expressions, the resolved `value` is also cached after the first retrieval.
-   **Thread Safety:** **This class is not thread-safe.** It is designed for single-threaded access within the asset loading pipeline. The internal state is mutated without locks or other synchronization primitives. Accessing an instance from multiple threads will result in unpredictable behavior and race conditions, especially for dynamic expressions.

## API Surface

The public contract is focused on initialization and value retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| readJSON(...) | void | O(N) | Initializes the holder from a JSON element. Parses the underlying string array and caches enum metadata. This method MUST be called before any other. |
| get(executionContext) | EnumSet | O(1) / O(N) | Resolves and returns the final EnumSet. Complexity is O(1) for static, cached values and O(N) for dynamic expressions, where N is the number of strings to resolve. |
| validate(context) | void | O(1) / O(N) | A pre-flight check used by the asset system to ensure the underlying expression can be resolved without errors. |

## Integration Patterns

### Standard Usage

This class is not intended for direct use in game systems. The following example illustrates its intended use within a custom asset builder, where it reads a set of "patrol_flags" from a JSON object.

```java
// Within a custom BuilderBase subclass...

// Define the holder for the configuration property
EnumSetHolder<PatrolFlag> patrolFlags = new EnumSetHolder<>();

// During the build process, initialize it from JSON
// json is a JsonObject representing the NPC asset
patrolFlags.readJSON(
    json.get("patrolFlags"),
    EnumSet.of(PatrolFlag.DAYLIGHT_ONLY), // Default value
    PatrolFlag.class,
    "patrolFlags",
    builderParameters
);

// The resolved EnumSet is later retrieved and passed to the runtime object
NpcData runtimeData = new NpcData();
runtimeData.setPatrolFlags(patrolFlags.get(executionContext));
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new EnumSetHolder()` in general game logic. Its lifecycle is managed entirely by the asset building framework.
-   **State Mutation After Read:** Do not attempt to modify the holder's state after the initial `readJSON` call. It is designed to be write-once.
-   **Concurrent Access:** Never share an EnumSetHolder instance across multiple threads. The lack of synchronization will lead to data corruption.
-   **Premature Retrieval:** Calling `get` before `readJSON` has been successfully executed will result in a NullPointerException or other unhandled exceptions.

## Data Pipeline

The EnumSetHolder acts as a transformation stage in the NPC asset loading pipeline. It converts raw text data from a configuration file into a type-safe, high-performance runtime data structure.

> Flow:
> JSON File -> GSON Parser -> JsonElement -> **EnumSetHolder.readJSON** -> Internal String Expression -> **EnumSetHolder.get(context)** -> `java.util.EnumSet` -> Runtime NPC Component

