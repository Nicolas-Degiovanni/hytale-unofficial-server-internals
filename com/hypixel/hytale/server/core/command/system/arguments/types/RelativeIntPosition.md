---
description: Architectural reference for RelativeIntPosition
---

# RelativeIntPosition

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Transient Value Object

## Definition
```java
// Signature
public class RelativeIntPosition {
```

## Architecture & Concepts
The RelativeIntPosition class is a specialized value object within the server's command system. Its primary responsibility is to represent and resolve a three-dimensional integer coordinate that may be either absolute or relative to an execution context. This class is fundamental for handling common command syntaxes like `/teleport ~5 100 ~-2`, where tilde notation signifies a position relative to the command's sender.

It acts as a translation layer between the abstract, string-based input from a user and a concrete, absolute **Vector3i** world position. To achieve this, it encapsulates context-aware resolution logic. The final calculated position depends on the source of the command; a command executed by a player will resolve relative coordinates against the player's current position, while a command from the server console will resolve them against the world origin or fail if relativity is required.

This component requires access to external game state, specifically an entity's **TransformComponent** for a positional base and the world's **ChunkStore** for height-aware Y-coordinate calculations.

## Lifecycle & Ownership
- **Creation:** An instance of RelativeIntPosition is created by the command argument parsing framework when it identifies a sequence of three coordinate arguments in a command string. It is not intended to be manually instantiated by command implementation logic.
- **Scope:** The object's lifetime is ephemeral, typically scoped to the execution of a single command. It is created, its resolution methods are called once, and it is then discarded.
- **Destruction:** As a short-lived object with no external resources to manage, it is reclaimed by the Java Garbage Collector once the command execution completes and the object is no longer referenced.

## Internal State & Concurrency
- **State:** The class is **immutable**. Its internal state consists of three final **IntCoord** fields representing the X, Y, and Z components. This state is set exclusively at construction time and cannot be modified thereafter. All methods are pure functions that compute new values based on this internal state and external data, without causing side effects.
- **Thread Safety:** RelativeIntPosition is inherently **thread-safe** due to its immutability. Instances can be safely shared and accessed across multiple threads. However, callers must ensure that the external state objects passed into its methods, such as **ComponentAccessor** and **ChunkStore**, are themselves safe for concurrent access. The methods of this class perform read-only operations on that external state.

## API Surface
The public API is designed for resolving coordinates from different levels of contextual information.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| RelativeIntPosition(x, y, z) | constructor | O(1) | Constructs a new position object from three IntCoord components. |
| getBlockPosition(context, accessor) | Vector3i | O(C) | Resolves the position using a CommandContext. Throws GeneralCommandException if relative coordinates are used without a valid player context. C represents chunk access time. |
| getBlockPosition(ref, accessor) | Vector3i | O(C) | Resolves the position relative to a specific entity reference. Assumes the reference is valid and has a TransformComponent. |
| getBlockPosition(base, chunkStore) | Vector3i | O(C) | The core resolution logic. Calculates the final block position from a base vector and a ChunkStore, used for height-aware Y-coordinate resolution. |
| isRelative() | boolean | O(1) | Returns true if any of the X, Y, or Z coordinates are relative, indicated by tilde notation in the original command. |

## Integration Patterns

### Standard Usage
This class is intended to be consumed as a pre-parsed argument within a command's execution logic. The command system's framework handles the parsing and creation.

```java
// In a command's execute method, after the framework has parsed arguments.
// Assume 'context' is the command's execution context.

// Retrieve the parsed argument by name and type.
RelativeIntPosition pos = context.getArgument("targetLocation", RelativeIntPosition.class);

// Retrieve the necessary world accessor from the context.
ComponentAccessor<EntityStore> accessor = context.getComponentAccessor();

// Resolve the potentially relative position into an absolute world coordinate.
Vector3i finalPosition = pos.getBlockPosition(context, accessor);

// Use the fully resolved, absolute position for game logic.
World world = accessor.getExternalData().getWorld();
world.setBlock(finalPosition, Block.STONE);
```

### Anti-Patterns (Do NOT do this)
- **Manual Resolution:** Do not extract the internal **IntCoord** objects and attempt to resolve the position manually. This bypasses the critical context-aware logic that correctly handles player-relative vs. origin-relative coordinates and world-height lookups for the Y-axis.
- **Ignoring Context:** Calling `getBlockPosition` with a hardcoded base position (e.g., Vector3d.ZERO) when a valid player context is available. This will incorrectly resolve tilde-notation coordinates from the world origin instead of the player's location. Always pass the full **CommandContext** when available.

## Data Pipeline
RelativeIntPosition serves as a transformation node in the command processing pipeline. It converts abstract, symbolic coordinate data into concrete, actionable world data.

> Flow:
> Raw Command String (`/setblock ~ 10 ~ stone`) -> Command Argument Parser -> **RelativeIntPosition** Instance -> `getBlockPosition(context)` -> Resolved **Vector3i** -> World Modification Logic

