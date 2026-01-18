---
description: Architectural reference for MaterialSet
---

# MaterialSet

**Package:** com.hypixel.hytale.builtin.hytalegenerator
**Type:** Utility

## Definition
```java
// Signature
public class MaterialSet implements Predicate<Material> {
```

## Architecture & Concepts
The **MaterialSet** is a high-performance, immutable data structure designed for efficient material membership testing within the world generation pipeline. It functions as a specialized predicate, optimized for the frequent checks required by algorithms like biome placement, ore distribution, and terrain carving.

Its core architectural purpose is to provide a memory-efficient and CPU-performant way to answer the question: "Does this given material belong to a predefined group?". To achieve this, it avoids storing direct references to **Material** objects. Instead, it pre-computes and stores their integer hash codes (**hashMaterialIds**) in a specialized primitive collection from the *fastutil* library. This design significantly reduces memory overhead and garbage collector pressure.

The class supports two fundamental modes of operation, controlled by the **isInclusive** flag:
1.  **Inclusion Mode (Whitelist):** The set defines a list of materials that are allowed. The `test` method returns true only for materials within this set.
2.  **Exclusion Mode (Blacklist):** The set defines a list of materials that are *not* allowed. The `test` method returns true for any material *not* in this set.

This dual-mode capability makes **MaterialSet** a highly versatile component for defining complex world generation rules.

## Lifecycle & Ownership
-   **Creation:** A **MaterialSet** is a transient object. It is instantiated on-demand by higher-level world generation components, such as a biome definition or a feature placement rule, which require a specific set of materials to operate on.
-   **Scope:** Its lifetime is typically bound to its owning component. For example, a **MaterialSet** defining which stone types an ore can spawn in will exist as long as the corresponding ore generation rule is loaded and active. It is not a global or session-scoped object.
-   **Destruction:** The object is managed by the Java garbage collector and is eligible for cleanup once it is no longer referenced. It holds no native resources and requires no explicit destruction.

## Internal State & Concurrency
-   **State:** **Immutable**. Once a **MaterialSet** is constructed, its internal state cannot be changed. The material hash mask is stored in an unmodifiable `IntSet`, and the inclusivity flag is final. This is a critical design guarantee.

-   **Thread Safety:** **Fully thread-safe**. Due to its immutable nature, a **MaterialSet** instance can be safely shared and accessed by multiple threads concurrently without any external synchronization or locks. This is essential for parallelized world generation chunks, where multiple threads may need to evaluate generation rules simultaneously.

## API Surface
The public contract is minimal, focusing on construction and predicate testing.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(Material value) | boolean | O(1) avg. | Checks if the given **Material** satisfies the set's conditions (inclusion or exclusion). |
| test(int hashMaterialIds) | boolean | O(1) avg. | Overloaded, high-performance check using a pre-computed material hash. Avoids **Material** object access. |

## Integration Patterns

### Standard Usage
A **MaterialSet** should be created once as part of a generator's configuration and then reused. It is commonly used with Java Streams or passed directly to algorithms that accept a **Predicate**.

```java
// Example: Defining a set of materials that a specific feature can replace.
List<Material> replaceableMaterials = List.of(Material.STONE, Material.DIRT, Material.GRAVEL);

// Create an inclusive set (a whitelist).
Predicate<Material> canBeReplaced = new MaterialSet(true, replaceableMaterials);

// Use the predicate to filter or test.
if (canBeReplaced.test(world.getMaterialAt(pos))) {
    // ... logic to place feature
}
```

### Anti-Patterns (Do NOT do this)
-   **Redundant Instantiation:** Avoid creating a new **MaterialSet** inside a tight loop (e.g., for every block in a chunk). This negates all performance benefits and creates excessive garbage. Instantiate it once and cache it.

    ```java
    // BAD: New set created for every single block
    for (int x = 0; x < 16; x++) {
        for (int z = 0; z < 16; z++) {
            // This is extremely inefficient.
            MaterialSet stoneSet = new MaterialSet(true, List.of(Material.STONE));
            if (stoneSet.test(world.getMaterialAt(x, y, z))) {
                // ...
            }
        }
    }
    ```

## Data Pipeline
The **MaterialSet** acts as a filter or gate within a larger data flow. It does not transform data but rather selects it.

> Flow:
> World Generator → Proposes a **Material** for a location → **MaterialSet.test(Material)** → `boolean` result → Generator commits or rejects the material based on the result.

