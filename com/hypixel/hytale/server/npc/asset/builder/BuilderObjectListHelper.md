---
description: Architectural reference for BuilderObjectListHelper
---

# BuilderObjectListHelper

**Package:** com.hypixel.hytale.server.npc.asset.builder
**Type:** Transient

## Definition
```java
// Signature
public class BuilderObjectListHelper<T> extends BuilderObjectArrayHelper<List<T>, T> {
```

## Architecture & Concepts
The BuilderObjectListHelper is a specialized component within the server-side NPC asset construction framework. It functions as an aggregator and composer, responsible for orchestrating the creation of a `java.util.List` from a collection of subordinate builders.

Architecturally, this class embodies the Composite pattern within a larger Builder pattern. It does not create the final `T` objects itself; instead, it delegates this responsibility to a series of `BuilderObjectReferenceHelper` instances that it contains. Its primary role is to iterate over these child builders, invoke their respective build processes, and collect the results into a single list.

This helper extends the more generic `BuilderObjectArrayHelper`, refining its behavior to produce a concrete `List` implementation. The choice of `it.unimi.dsi.fastutil.objects.ObjectArrayList` for the list implementation is a deliberate performance optimization, favoring reduced memory overhead and faster operations over the standard Java Collections Framework for the specific use case of asset loading.

## Lifecycle & Ownership
- **Creation:** An instance of BuilderObjectListHelper is created by the parent asset building system, typically a parser or a higher-level builder, when it encounters a data structure that represents a list of object references (e.g., a JSON array of objects). The `BuilderContext` supplied during construction links it to the overarching build operation.
- **Scope:** The object's lifetime is strictly bound to the scope of the parent asset build process. It is a short-lived, transient object that exists only to fulfill a single list-building request.
- **Destruction:** It holds no persistent state and manages no external resources. The object is eligible for garbage collection immediately after the `build` method returns and its result is consumed by the caller.

## Internal State & Concurrency
- **State:** The core state of this class is the `builders` collection, which is inherited from its parent. This collection is mutable during the initial configuration phase when child builders are added. During the execution of the `build` method, this state is treated as immutable. The class does not cache the result of a build; each call to `build` generates a new list.
- **Thread Safety:** This class is **not thread-safe**. It is designed for synchronous, single-threaded execution within the asset loading pipeline. Concurrent access to an instance, especially modifying its internal builder list while a `build` operation is in progress, will result in undefined behavior and likely a `ConcurrentModificationException`.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| build(BuilderSupport) | List<T> | O(N) | Orchestrates the construction of a list. Iterates through N internal builders, invoking `build` on each. Returns a new `ObjectArrayList` or `null` if no builders are present. |

## Integration Patterns

### Standard Usage
This class is an internal framework component and is not intended for direct instantiation by game logic developers. It is used by the asset deserialization system to hydrate object graphs from configuration files.

```java
// Conceptual example of framework usage
// This code would exist within a higher-level asset parser.

// When the parser encounters a list definition:
BuilderObjectListHelper<ArmorPiece> listBuilder = new BuilderObjectListHelper<>(ArmorPiece.class, context);

// The parser populates the listBuilder with builders for each item in the list
for (JsonElement item : jsonArray) {
    BuilderObjectReferenceHelper<ArmorPiece> itemRef = createReferenceBuilder(item, context);
    listBuilder.add(itemRef); // Method inherited from parent
}

// The parser later invokes the build process
List<ArmorPiece> finalArmorSet = listBuilder.build(builderSupport);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not manually construct this class with `new BuilderObjectListHelper()`. It must be created and managed by the asset building framework which provides the correct `BuilderContext`.
- **State Re-use:** Do not retain an instance of this helper after calling `build`. It is designed as a single-use object. Modifying its internal state after a build may have unintended consequences on subsequent builds.
- **Concurrent Execution:** Never share an instance of this helper across multiple threads. The asset building process is strictly single-threaded.

## Data Pipeline
The BuilderObjectListHelper acts as a processing stage in the transformation of raw configuration data into a fully realized in-memory game object.

> Flow:
> Raw Asset File (e.g., HOCON) -> Configuration Parser -> **BuilderObjectListHelper** -> Hydrated `List<T>` -> NPC Asset Component

