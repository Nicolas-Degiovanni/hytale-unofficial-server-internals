---
description: Architectural reference for the Density abstract class, the core of the procedural world generation system.
---

# Density

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density
**Type:** Abstract Base Class / Strategy Pattern Component

## Definition
```java
// Signature
public abstract class Density {
   // ... inner class and static fields ...
}
```

## Architecture & Concepts

The **Density** class is the fundamental building block of Hytale's procedural world generation system. It represents a single, atomic operation within a larger computational graph that determines the shape and composition of the game world. At its core, an implementation of **Density** is a function that takes a 3D position and other contextual information, and returns a single scalar value—the "density". This value is typically used to decide whether a given point in space is solid ground or empty air.

This class and its subclasses form a directed acyclic graph (DAG). Simple **Density** nodes, such as noise generators, produce raw values. These are then fed into more complex composite nodes, such as adders, multipliers, or clamps, which combine and transform the inputs to produce the final, complex terrain features.

The design heavily utilizes the Strategy and Composite design patterns. Each concrete subclass of **Density** is a specific strategy for calculating a value, and these strategies can be composed into a tree-like structure to define the entire world generation algorithm. The system is data-driven; these graphs are typically defined in configuration files and instantiated at runtime, allowing designers to craft new worlds without modifying engine code.

The nested **Density.Context** class is a critical component of this architecture. It acts as a parameter object, bundling all necessary inputs for a density calculation. This prevents method signature bloat and provides a flexible mechanism for passing new types of data through the generation pipeline in the future.

## Lifecycle & Ownership

- **Creation:** Concrete instances of **Density** are not created directly via constructors. They are instantiated by a world generator configuration loader, which parses data files (e.g., JSON or HOCON) defining the generation graph. This process typically occurs once when a world is loaded or a server starts.
- **Scope:** A **Density** object graph is immutable and stateless after its initial configuration. It persists for the lifetime of the world generator instance, which is typically the entire server session or client play session.
- **Destruction:** The entire graph of **Density** objects is garbage collected when the world generator is unloaded, for example, when a server shuts down or the client returns to the main menu.

## Internal State & Concurrency

- **State:** The **Density** base class is stateless. Concrete implementations are **required** to be stateless and immutable after their initial configuration via `setInputs`. All variable data required for a calculation is passed via the **Density.Context** object. This design is paramount for deterministic and parallelized world generation.
- **Thread Safety:** Implementations must be unconditionally thread-safe. The world generation engine processes chunks in parallel across multiple worker threads. A single, shared instance of the **Density** graph is used by all workers simultaneously. The stateless design, where all transient data is scoped to the **Context** object on each thread's stack, is the primary mechanism for ensuring thread safety. Any deviation from this pattern in a subclass will lead to severe concurrency bugs and non-deterministic world generation.

## API Surface

The public contract is minimal, focusing exclusively on graph composition and evaluation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context) | double | Varies | **Core Method.** Evaluates the density at a given point. The complexity depends entirely on the implementation and the depth of its input graph. |
| setInputs(Density[]) | void | O(N) | Configures this node's inputs from other **Density** nodes. Called once during the initialization of the generator graph. |

## Integration Patterns

### Standard Usage

A developer or designer will almost never interact with a **Density** object directly in Java code. Instead, they define the graph structure in configuration files. A higher-level system, the **TerrainDensityProvider**, is responsible for invoking the root of the **Density** graph for each voxel within a chunk.

```java
// Conceptual example of how the engine uses a Density graph.
// This code does not appear in typical gameplay logic.

// 1. The graph is loaded from configuration at startup.
Density rootNode = worldGenerator.getDensityGraph();

// 2. For each point in a chunk, a context is created.
for (int x = 0; x < 16; x++) {
    for (int y = 0; y < 256; y++) {
        for (int z = 0; z < 16; z++) {
            Vector3d worldPos = chunk.toWorld(x, y, z);
            Density.Context context = new Density.Context(worldPos, workerId, ...);

            // 3. The graph is evaluated for that point.
            double densityValue = rootNode.process(context);

            // 4. The result determines the block type.
            if (densityValue > 0) {
                chunk.setBlock(x, y, z, Block.STONE);
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)

- **Stateful Implementations:** Never store mutable state as a field in a **Density** subclass. This will break thread safety and cause unpredictable, non-deterministic world generation. All calculations must depend only on the inputs from the **Context** object and the node's immutable configuration.
- **Direct Instantiation:** Do not use `new MyDensity()` in game logic. **Density** graphs are part of the world's static configuration and must be managed by the world generator's loading system.
- **Caching Results:** Avoid caching results within a **Density** object itself. The engine may have its own higher-level caching strategies. Implementing custom caching can interfere with this and violate the statelessness requirement.

## Data Pipeline

The **Density** class is a central processing stage in the procedural generation data pipeline. It transforms spatial coordinates into a scalar field that defines the fundamental shape of the world.

> Flow:
> Chunk Generation Request -> Voxel Coordinate Iteration -> **Density.Context** Creation -> **Density Graph Evaluation** -> Density Value -> Block Type Decision -> Chunk Data Population

