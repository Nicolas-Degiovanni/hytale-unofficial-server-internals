---
description: Architectural reference for RecursivePrefabLoader
---

# RecursivePrefabLoader

**Package:** com.hypixel.hytale.builtin.buildertools.utils
**Type:** Transient Stateful Service

## Definition
```java
// Signature
public abstract class RecursivePrefabLoader<T> implements BiFunction<String, Random, T> {
```

## Architecture & Concepts
The RecursivePrefabLoader is a stateful, single-use processor responsible for loading and composing hierarchical game structures, known as prefabs. It acts as the orchestrator for resolving a single logical prefab name into a complex, fully realized object graph, which may be composed of multiple nested prefabs.

This class embodies the **Template Method** design pattern. It defines the skeleton of the loading algorithm—including file resolution, weighted selection, recursion depth checking, and cycle detection—while deferring the concrete implementation of loading a single prefab file to its subclasses via the abstract *loadPrefab* method.

Its primary architectural responsibilities include:
*   **Path Resolution:** Translating a logical prefab name (e.g., "house") into one or more physical file paths (e.g., "house_variant1.prefab.json", "house_variant2.prefab.json").
*   **Variant Selection:** Applying weighted or random selection logic to choose a single file when multiple variants exist for a given name.
*   **Recursive Composition:** Scanning a loaded prefab for special placeholder blocks (containing a PrefabSpawnerState component) and recursively invoking the loading process for the specified child prefab.
*   **Safety and Termination:** Enforcing a maximum recursion depth (MAX_RECURSION_DEPTH) and detecting cyclic dependencies (e.g., Prefab A contains Prefab B, which contains Prefab A) to prevent infinite loops and stack overflows.

The generic type T represents the in-memory representation of the loaded prefab, such as a BlockSelection or another custom data structure.

### Lifecycle & Ownership
- **Creation:** An instance is created manually by a higher-level system, such as a world generator or a build tool command. It is configured at creation time with a root directory for prefabs and a loader function for processing individual files.
- **Scope:** The lifecycle of a RecursivePrefabLoader instance is intentionally brief and tied to a single top-level load operation. Its internal state, particularly the set of visited files for cycle detection, is only valid for the duration of one call to *load* or *apply*.
- **Destruction:** The object is intended to be discarded and made eligible for garbage collection immediately after the load operation completes.

**WARNING:** Reusing a single instance for multiple independent load operations is a critical anti-pattern. Doing so will cause state to leak between calls, leading to incorrect behavior and false-positive cycle detection errors.

## Internal State & Concurrency
- **State:** This class is highly **mutable**. It maintains internal state through the *visitedFiles* set and the *depthTracker* integer. This state is essential for the correctness and safety of a single, nested loading operation.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads or used for concurrent load operations. The mutable internal state is accessed and modified without any synchronization mechanisms. Concurrent calls to *load* on the same instance will result in a corrupted state, race conditions, and unpredictable behavior. Each thread performing a load operation must create and use its own dedicated instance.

## API Surface
The public API is intentionally minimal, exposing only the entry points for initiating a load operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(name, random) | T | Complex | Implements the BiFunction interface. Delegates to the load method. Throws PrefabLoadException on failure. |
| load(name, random) | T | Complex | Primary entry point. Initiates the recursive loading process for the named prefab. Throws PrefabLoadException on failure. |

## Integration Patterns

### Standard Usage
A developer must extend RecursivePrefabLoader and provide a concrete implementation for the *loadPrefab* method. The most common use case is to load prefabs into a BlockSelection for placement in the world.

```java
// How a developer should normally use this
// 1. Define a concrete loader implementation
public class MyPrefabLoader extends RecursivePrefabLoader<BlockSelection> {
    public MyPrefabLoader(Path root, Function<String, BlockSelection> baseLoader) {
        super(root, baseLoader);
    }

    @Override
    protected BlockSelection loadPrefab(int x, int y, int z, String file, PrefabRotation rot, Random rand) {
        // Implementation to load a single prefab file into a BlockSelection
        // and handle recursive loading of children...
    }
}

// 2. Instantiate and use for a single operation
Path prefabRoot = Path.of("server/prefabs");
Function<String, BlockSelection> singleFileLoader = ...; // Function to load one .prefab.json
Random random = new Random();

// Create a new loader for each top-level operation
RecursivePrefabLoader<BlockSelection> loader = new MyPrefabLoader(prefabRoot, singleFileLoader);
BlockSelection finalStructure = loader.load("dungeons/level_1_entrance", random);

// The 'loader' instance should now be discarded.
```

### Anti-Patterns (Do NOT do this)
- **Instance Reuse:** Do not create a single loader instance and reuse it for multiple calls to *load*. The internal *visitedFiles* state will persist, causing subsequent loads to fail with false cycle detection errors.
- **Concurrent Access:** Do not share an instance across multiple threads. This will lead to race conditions and a corrupted internal state. Each thread must have its own instance.
- **Incorrect State Management:** Modifying the internal state (*visitedFiles*, *depthTracker*) from a subclass is unsupported and will break the loader's core safety guarantees.

## Data Pipeline
The data flow for a single load operation is a recursive descent through the prefab hierarchy.

> Flow:
> Prefab Name & Random -> **RecursivePrefabLoader.load()** -> File System Resolution -> Weighted/Random Selection -> Subclass.loadPrefab() -> In-Memory Object (e.g. BlockSelection) -> Scan for PrefabSpawnerState -> (Recursive Call to **load()**) -> Merge Child Object -> Return Final Composed Object

