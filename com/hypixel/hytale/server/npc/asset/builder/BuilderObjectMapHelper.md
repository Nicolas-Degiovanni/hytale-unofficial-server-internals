---
description: Architectural reference for BuilderObjectMapHelper
---

# BuilderObjectMapHelper

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient Helper

## Definition
```java
// Signature
public class BuilderObjectMapHelper<K, V> extends BuilderObjectArrayHelper<Map<K, V>, V> {
```

## Architecture & Concepts
The BuilderObjectMapHelper is a specialized component within the server-side asset deserialization pipeline. Its primary responsibility is to transform a JSON array of objects into a Java `Map<K, V>`. This class acts as a bridge between the list-oriented structure of JSON arrays and the key-value lookup requirements of core game systems.

Architecturally, it inherits from BuilderObjectArrayHelper, reusing the underlying mechanism for parsing a JSON array into a collection of individual object builders. The key innovation of this class is the addition of a key-extraction function, provided during its construction. This function is applied to each deserialized object to produce a unique key, which is then used to populate the final map.

This pattern is critical for defining complex asset structures, such as a collection of NPC animations or abilities, where each element must be addressable by a unique identifier (e.g., an animation name). The class enforces data integrity by throwing an exception if a duplicate key is detected, preventing ambiguous or corrupt asset state.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the internal `BuilderManager` during the asset loading process. Its creation is triggered when the `BuilderManager` reflects over an asset definition and encounters a field of type `Map`. The context of the parent asset being built is passed via the `BuilderContext` constructor argument.
- **Scope:** The lifecycle of a BuilderObjectMapHelper instance is extremely short. It is scoped to the construction of a single `Map` field within a single parent asset.
- **Destruction:** The object is immediately eligible for garbage collection after the parent builder has called the `build` method and the resulting map has been assigned to the asset field. It holds no persistent references and is not registered in any global context.

## Internal State & Concurrency
- **State:** The object is stateful but its state transitions through a well-defined, two-phase lifecycle.
    1.  **Configuration Phase:** During the `readConfig` call, its internal state (specifically the `builders` list inherited from its parent) is actively mutated as the JSON is parsed.
    2.  **Build Phase:** During the `build` call, the internal state is treated as read-only to produce the final `Map`. The `build` method itself is idempotent but produces a new `Map` instance on each invocation.
- **Thread Safety:** **This class is not thread-safe.** It is designed to operate within a single-threaded asset loading context. Concurrent calls to `readConfig` or `build` will result in unpredictable behavior, race conditions, and likely data corruption. All asset pipeline operations are expected to be synchronous.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(builderSupport) | Map<K, V> | O(N) | Constructs the final map from the configured builders. N is the number of elements. Throws `IllegalArgumentException` if a duplicate key is generated. |
| readConfig(...) | void | O(N) | Delegates to the parent class to parse a JSON array and populate the internal list of builders. |
| testEach(...) | T | O(M) | Iterates through each prospective element builder, applying a test function. M is the number of elements tested before the function returns a non-success result. Used for validation. |

## Integration Patterns

### Standard Usage
This class is part of the internal framework and is not intended for direct use by developers creating game content or systems. Its functionality is invoked implicitly by the asset loader based on the structure of an asset definition class.

A developer triggers its use by defining a `Map` in their asset class.

```java
// Conceptual: In an asset definition file (e.g., NpcBehavior.java)
// The presence of this Map field instructs the BuilderManager to use
// a BuilderObjectMapHelper during deserialization.

public class NpcBehavior {
    // The asset loader will require that the 'Animation' class has a field
    // that can be used as a String key (e.g., a 'name' field).
    public Map<String, Animation> animations;
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new BuilderObjectMapHelper()`. The asset `BuilderManager` is solely responsible for its lifecycle. Manual creation will result in a disconnected object that cannot participate in the asset build process.
- **Assuming Thread Safety:** Do not share instances of this helper or the parent builders across threads. The asset loading pipeline must be a single-threaded, synchronous operation.
- **Bypassing Key Uniqueness:** The duplicate key validation is a critical data integrity feature. Do not attempt to catch and ignore the `IllegalArgumentException`, as this indicates a fundamental flaw in the source asset data.

## Data Pipeline
The flow of data through this component is linear and unidirectional, transforming structured text into a hydrated in-memory object.

> Flow:
> JSON File -> Gson Parser -> `JsonElement` (Array) -> `BuilderObjectArrayHelper::readConfig` -> `List<BuilderObjectReferenceHelper>` -> **`BuilderObjectMapHelper::build`** -> `Map<K, V>` -> Asset Field Assignment

