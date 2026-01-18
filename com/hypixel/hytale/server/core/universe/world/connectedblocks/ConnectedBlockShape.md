---
description: Architectural reference for ConnectedBlockShape
---

# ConnectedBlockShape

**Package:** com.hypixel.hytale.server.core.universe.world.connectedblocks
**Type:** Data Model / DTO

## Definition
```java
// Signature
public class ConnectedBlockShape {
```

## Architecture & Concepts
The ConnectedBlockShape class is a pure data model that defines a specific visual representation for a block based on its adjacent neighbors. It is a core component of the "connected blocks" system, which allows blocks like fences, walls, and glass panes to dynamically change their model to form continuous structures.

This class does not contain any logic. Instead, it serves as an in-memory representation of configuration data loaded from game asset files. Its primary role is to be deserialized from a data source (like JSON) via its static CODEC field. The Hytale engine uses this structured data to determine which block model to render in a given position.

A ConnectedBlockShape is defined by two key properties:
1.  **patternsToMatchAnyOf**: A collection of conditions describing the required state of neighboring blocks. If any of these patterns are met, this shape is considered a candidate for rendering.
2.  **faceTags**: Metadata applied to the block's faces when this shape is active. This typically influences texture mapping, lighting, and which faces are culled.

This class is a leaf node in the asset definition tree, typically owned by a higher-level BlockDefinition object.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the Hytale serialization framework using the public static `CODEC` field. The engine's asset loader reads a block definition file, finds the relevant data section, and invokes the codec to construct a ConnectedBlockShape object.
-   **Scope:** The object's lifetime is bound to the asset it defines. It is loaded during the engine's asset initialization phase and persists in a read-only state for the entire game session.
-   **Destruction:** The object is marked for garbage collection when its parent asset registry is unloaded, typically upon server shutdown or when a game world is closed.

## Internal State & Concurrency
-   **State:** The state of a ConnectedBlockShape is **effectively immutable** after its creation via the codec. The internal fields are populated once during deserialization and are not intended to be modified thereafter.
-   **Thread Safety:** This class is **thread-safe for reads**. Due to its immutable nature, multiple engine threads (e.g., world generation, block ticking, rendering) can safely access its data simultaneously without requiring locks or other synchronization primitives.

## API Surface
The public contract is minimal, providing read-only access to the deserialized configuration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPatternsToMatchAnyOf() | CustomTemplateConnectedBlockPattern[] | O(1) | Retrieves the array of neighbor patterns that can trigger this shape. |
| getFaceTags() | ConnectedBlockFaceTags | O(1) | Retrieves the metadata tags associated with the block's faces for this shape. |

## Integration Patterns

### Standard Usage
This class is not intended for direct instantiation or manipulation. Engine systems retrieve it from a parent BlockDefinition and use it as read-only data to evaluate block states.

```java
// Hypothetical usage within a block state calculation system
void updateBlockAppearance(World world, Position pos) {
    BlockDefinition definition = world.getBlock(pos).getDefinition();
    
    // Iterate through all possible shapes for this block type
    for (ConnectedBlockShape shape : definition.getConnectedBlockShapes()) {
        // Check if any of the shape's patterns match the current world state
        for (CustomTemplateConnectedBlockPattern pattern : shape.getPatternsToMatchAnyOf()) {
            if (pattern.matches(world, pos)) {
                // A match is found. Apply this shape's properties for rendering.
                RenderableBlock renderable = getRenderableForPosition(pos);
                renderable.setFaceTags(shape.getFaceTags());
                return; // Stop after finding the first matching shape
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new ConnectedBlockShape()`. The class is designed to be data-driven and instantiated only by the asset serialization system. Manual creation bypasses the asset pipeline and will lead to disconnected, non-functional state.
-   **State Mutation:** Do not attempt to modify the contents of the array returned by `getPatternsToMatchAnyOf`. The object is shared globally as a read-only asset. Modifying it at runtime will cause unpredictable behavior and visual glitches across the entire game.

## Data Pipeline
The ConnectedBlockShape is a critical link in the chain that transforms declarative asset files into rendered pixels in the game world.

> Flow:
> Block Definition Asset (JSON) -> Asset Loading Service -> **ConnectedBlockShape.CODEC** -> In-Memory **ConnectedBlockShape** -> Block State Calculation Engine -> Render System

