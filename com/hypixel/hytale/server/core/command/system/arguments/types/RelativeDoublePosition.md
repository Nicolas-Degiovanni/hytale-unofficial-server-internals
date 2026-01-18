---
description: Architectural reference for RelativeDoublePosition
---

# RelativeDoublePosition

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Transient

## Definition
```java
// Signature
public class RelativeDoublePosition {
```

## Architecture & Concepts
The RelativeDoublePosition class is a specialized value object that serves as an intermediate representation for 3D world coordinates within the server's command system. Its primary function is to encapsulate the complex logic of resolving potentially relative coordinates, commonly expressed with tilde (~) notation, into an absolute world position as a Vector3d.

This class acts as a bridge between the command argument parser and the game world state. It is not a simple data container; it holds the unresolved coordinate components and requires access to a *base position* (typically the command sender's location) and the *World* object to perform its final calculation. This design decouples the parsing of command syntax from the execution logic, allowing the system to handle coordinate resolution in a standardized, reusable manner.

The most notable feature is its context-aware resolution. The calculation for the Y-axis, `resolveYAtWorldCoords`, implies that it may perform world-aware calculations, such as finding the ground level at the target X/Z coordinates, rather than performing a simple mathematical addition.

## Lifecycle & Ownership
- **Creation:** An instance of RelativeDoublePosition is created by the command argument parsing framework when it successfully parses three consecutive coordinate arguments from a command string. The framework first creates individual Coord objects and then uses them to construct this class.
- **Scope:** The object's lifetime is exceptionally short. It exists only for the duration of a single command's execution. It is created, its `getRelativePosition` method is called to produce a final Vector3d, and it is then discarded.
- **Destruction:** The instance becomes eligible for garbage collection as soon as the command execution method that created it returns. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** This class is **effectively immutable**. Its internal fields, `x`, `y`, and `z`, are final references to Coord objects. Once constructed, its state cannot be modified. Its methods do not cause side effects on its own state.
- **Thread Safety:** The class itself is inherently thread-safe due to its immutability. However, its methods operate on shared, mutable state from the game engine, specifically the `World` and `EntityStore` objects.

   **WARNING:** Calling `getRelativePosition` from any thread other than the main server thread is extremely dangerous. The caller must ensure that all access to the `World` and `ComponentAccessor` parameters is properly synchronized to prevent race conditions and data corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getRelativePosition(base, world) | Vector3d | O(World Lookup) | Resolves the coordinates relative to a provided base Vector3d. Complexity is dominated by the Y-coordinate resolution which may query world data structures. |
| getRelativePosition(context, world, accessor) | Vector3d | O(World Lookup) | High-level resolver. Determines the base position from the CommandContext and resolves coordinates. Throws GeneralCommandException if a non-player uses relative coordinates. |
| isRelative() | boolean | O(1) | Returns true if any of the three coordinate components are relative. Used to trigger context-aware logic. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly by most developers. It is an internal component of the command system. A command definition would declare a position argument, and the framework handles the instantiation and resolution behind the scenes.

```java
// Hypothetical command execution logic
// The 'pos' object is provided by the command framework after parsing.
public void execute(CommandContext context, RelativeDoublePosition pos) {
    World world = context.getWorld();
    ComponentAccessor<EntityStore> accessor = ...;

    // Resolve the parsed argument into a final world position
    Vector3d targetLocation = pos.getRelativePosition(context, world, accessor);

    // Use the final position for game logic
    teleportEntity(context.getSender(), targetLocation);
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Caching:** Do not cache an instance of RelativeDoublePosition. Its purpose is to perform a one-time calculation based on the world state *at the moment of command execution*. Caching it would lead to using stale data.
- **Cross-Thread Resolution:** Do not pass this object to a worker thread to resolve its position later. The `World` and `EntityStore` objects it needs for resolution are not thread-safe and must be accessed from the main server tick.
- **Incorrect Base Position:** Manually calling `getRelativePosition(Vector3d.ZERO, world)` for a relative position will produce logically incorrect results, as it will be relative to the world origin instead of the intended entity.

## Data Pipeline
The flow of data from a raw command string to a usable world position follows a clear, multi-stage pipeline.

> Flow:
> Raw Command String (`/teleport ~10 5 ~-20`) -> Command Argument Parser -> 3x `Coord` objects -> **`RelativeDoublePosition`** -> `getRelativePosition()` call -> Final `Vector3d` -> Game Logic (e.g., Teleportation System)

