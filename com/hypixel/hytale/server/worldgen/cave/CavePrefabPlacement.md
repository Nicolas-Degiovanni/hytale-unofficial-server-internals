---
description: Architectural reference for CavePrefabPlacement
---

# CavePrefabPlacement

**Package:** com.hypixel.hytale.server.worldgen.cave
**Type:** Strategy Enumeration

## Definition
```java
// Signature
public enum CavePrefabPlacement {
```

## Architecture & Concepts
The CavePrefabPlacement enum defines a set of fixed, type-safe strategies for determining the vertical (Y-axis) placement of generated structures, known as prefabs, within a cave system. It is a core component of the procedural world generation pipeline, specifically for the cave decoration and population stages.

This class is a classic implementation of the **Strategy Pattern**. Each enum constant—CEILING, FLOOR, and DEFAULT—encapsulates a distinct height calculation algorithm. By passing a CavePrefabPlacement constant, the world generator can delegate the responsibility of position calculation without being coupled to the specific implementation. This design allows for clean, extensible, and readable generator code, avoiding the use of magic numbers or complex conditional logic for placement decisions.

The core logic is contained within the `PrefabPlacementFunction` functional interface, which is implemented by a lambda expression for each enum constant.

### Lifecycle & Ownership
- **Creation:** Enum constants are instantiated by the Java Virtual Machine during class loading. They are effectively compile-time constants that exist before any game-specific code is executed.
- **Scope:** Application-wide. These instances are static, globally accessible, and persist for the entire lifetime of the server process.
- **Destruction:** The enum instances are only garbage collected when the JVM shuts down. There is no manual lifecycle management.

## Internal State & Concurrency
- **State:** **Immutable**. Each enum constant holds a single, final reference to its corresponding PrefabPlacementFunction. Its internal state is defined at compile time and cannot be modified at runtime. The static constant NO_HEIGHT is also immutable.
- **Thread Safety:** **Inherently thread-safe**. As an immutable object with no mutable state, CavePrefabPlacement can be safely accessed and used by any number of threads simultaneously without requiring locks or other synchronization mechanisms. This is critical for performance in a multithreaded world generation environment where multiple chunks may be processed in parallel.

## API Surface
The public contract is minimal, focusing on retrieving the encapsulated placement strategy.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFunction() | PrefabPlacementFunction | O(1) | Returns the functional strategy for height calculation. |
| NO_HEIGHT | static final int | N/A | A sentinel value indicating that no valid height could be determined. |
| PrefabPlacementFunction.generate(...) | int | O(1) | Executes the placement logic. Complexity depends on the underlying CaveNode methods, but is typically constant time. |

## Integration Patterns

### Standard Usage
The intended use is for a world generation service to select a placement strategy and invoke its function to determine a Y-coordinate. The generator provides the necessary context, such as world coordinates and the local CaveNode data structure.

```java
// A world generator obtains the relevant CaveNode for a given column
CaveNode currentNode = caveSystem.getNodeAt(x, z);
CavePrefabPlacement placementStrategy = CavePrefabPlacement.FLOOR;

// Retrieve the function and generate the height
int y = placementStrategy.getFunction().generate(worldSeed, x, z, currentNode);

if (y != CavePrefabPlacement.NO_HEIGHT) {
    // Place the prefab at (x, y, z)
    world.placePrefab(prefab, x, y, z);
}
```

### Anti-Patterns (Do NOT do this)
- **External `switch` statements:** Avoid using a switch statement on the enum value to replicate the placement logic. This defeats the purpose of the Strategy Pattern and creates brittle code that must be updated if a new enum constant is added.

    ```java
    // DO NOT DO THIS
    int y;
    switch (placementStrategy) {
        case CEILING:
            y = currentNode.getCeilingPosition(seed, x, z); // Redundant and brittle
            break;
        case FLOOR:
            y = currentNode.getFloorPosition(seed, x, z);
            break;
        // ...
    }
    ```

- **Direct Instantiation:** Enums cannot be instantiated using the `new` keyword. This is enforced by the Java compiler.

## Data Pipeline
CavePrefabPlacement acts as a pure function node within the larger world generation data pipeline. It accepts generator state as input and produces a single integer coordinate as output, with no side effects.

> Flow:
> World Generator -> Provides (Seed, X, Z, CaveNode) -> **CavePrefabPlacement.PrefabPlacementFunction** -> Returns Y-Coordinate -> Prefab Placer Service

