---
description: Architectural reference for RelationalShader
---

# RelationalShader

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.shaders
**Type:** Transient

## Definition
```java
// Signature
public class RelationalShader<T> implements Shader<T> {
```

## Architecture & Concepts
The RelationalShader is a specialized implementation of the Shader interface that functions as a high-performance dispatcher or router. It embodies the Strategy pattern, allowing the world generation pipeline to dynamically select a specific shading behavior based on an input value.

Its primary role is to replace complex conditional logic (e.g., long if-else-if chains or switch statements) with a clean, data-driven map lookup. This is fundamental for creating context-sensitive generation rules, such as "if the current block is Stone, apply the OreVeinShader; if it is Dirt, apply the RootShader; otherwise, do nothing."

The component is built around two core concepts:
1.  **Relations Map:** A map where keys are potential input values and values are the specific Shader instances to execute for that input.
2.  **Fallback Shader:** A mandatory default Shader, `onMissingKey`, that is invoked for any input value not explicitly defined in the relations map. This ensures deterministic behavior and prevents generation failures from unhandled cases.

This design makes generator logic highly extensible, as new rules can be added by simply inserting entries into the map without modifying the core shading algorithm.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly via its constructor (`new RelationalShader(...)`) during the setup phase of a world generator or a specific generation stage. It is typically built and configured by a factory or builder responsible for assembling the overall generation logic.
-   **Scope:** The object's lifetime is ephemeral, scoped to the specific generation task for which it was configured. It does not persist between different generation sessions or stages.
-   **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the owning generator completes its work and all references to the instance are dropped. No manual cleanup is necessary.

## Internal State & Concurrency
-   **State:** The internal state is **mutable** during the initial configuration phase via the `addRelation` method. The state consists of the `relations` HashMap and the reference to the `onMissingKey` Shader. After configuration, it is typically treated as immutable during its operational use in the `shade` methods.

-   **Thread Safety:** This class is **not thread-safe**. The internal `relations` map is a standard, non-synchronized HashMap.

    **WARNING:** Modifying the shader by calling `addRelation` from one thread while another thread is executing a `shade` method will result in a `ConcurrentModificationException` or other undefined, non-deterministic behavior. The intended lifecycle is to configure the object completely on a single thread before it is used for any generation work.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addRelation(key, value) | RelationalShader<T> | O(1) avg | Configures a routing rule by mapping an input key to a delegate Shader. Returns the instance to support a fluent, builder-style pattern. |
| shade(current, seed...) | T | O(1) avg | Executes the shader logic. Performs a key lookup on the `current` value and delegates the call to the mapped Shader or the fallback `onMissingKey` Shader. |

## Integration Patterns

### Standard Usage
The intended usage follows a configure-then-use pattern. The instance is created with a default fallback, configured with specific rules using the fluent API, and then passed into the generation pipeline to be executed.

```java
// 1. Define a fallback shader (e.g., one that does nothing)
Shader<Block> noOpShader = new NoOpShader<>();

// 2. Create and configure the RelationalShader
RelationalShader<Block> blockProcessor = new RelationalShader<>(noOpShader)
    .addRelation(Blocks.STONE, new OreVeinShader())
    .addRelation(Blocks.DIRT, new RootPlacementShader());

// 3. Use the configured shader within a generator
Block currentBlock = Blocks.STONE;
Block newBlock = blockProcessor.shade(currentBlock, worldSeed, x, z);
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Never call `addRelation` after the shader has been passed to a multi-threaded generation engine. All configuration must be completed upfront.
-   **Stateful Delegates:** While RelationalShader itself is stateful during configuration, avoid using delegate Shaders that contain mutable state if the RelationalShader will be used across multiple chunks or regions concurrently. This can introduce subtle cross-contamination bugs in world generation.
-   **Null Fallback:** The constructor is annotated with Nonnull, but if a null is somehow provided, it will cause a `NullPointerException` for any unmapped input key. Always provide a safe, well-defined fallback like a `NoOpShader`.

## Data Pipeline
The RelationalShader acts as a conditional branching node within a larger data processing pipeline. It does not transform data itself but rather directs the data to the correct transformation component.

> Flow:
> Input Value (e.g., Block ID) -> **RelationalShader** (Key Lookup) -> (Delegation) -> Sub-Shader (e.g., OreVeinShader) -> Transformed Output Value

