---
description: Architectural reference for IntCoord
---

# IntCoord

**Package:** com.hypixel.hytale.server.core.command.system.arguments.types
**Type:** Transient

## Definition
```java
// Signature
public class IntCoord {
```

## Architecture & Concepts
IntCoord is a specialized value object designed to parse and resolve a single integer coordinate from a command string. It is a fundamental building block of the server's command system, responsible for interpreting advanced coordinate notations such as relative positioning (`~`), chunk-based coordinates (`c`), and height-based coordinates (`_`).

This class encapsulates the complex logic of transforming a symbolic coordinate into a concrete world position. By isolating this responsibility, it decouples the high-level command argument parsers (e.g., for `Vec3`) from the low-level details of world state and coordinate arithmetic. Its primary function is to serve as an intermediate representation between a raw string token and a final integer value used by game logic.

## Lifecycle & Ownership
- **Creation:** Instances are exclusively created via the static factory method `IntCoord.parse(String)`. This occurs deep within the command processing pipeline when a command argument requires coordinate resolution.
- **Scope:** The lifecycle of an IntCoord instance is extremely brief. It is created on-demand during the execution of a single command and is typically discarded once the final coordinate has been resolved.
- **Destruction:** As a short-lived, unmanaged object, it is reclaimed by the Java garbage collector as soon as it falls out of the scope of the command handler that created it.

## Internal State & Concurrency
- **State:** The object is **Immutable**. Its internal state, including the `value` and the boolean flags (`height`, `relative`, `chunk`), is set once during construction and cannot be modified thereafter. This design guarantees predictable behavior.
- **Thread Safety:** IntCoord is inherently **thread-safe** due to its immutability. Instances can be safely shared and accessed across multiple threads without requiring any external synchronization or locks. However, in practice, its usage is confined to the single thread processing a given command.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| parse(String str) | static IntCoord | O(N) | **Primary Factory.** Parses a string token into an IntCoord instance. Throws NumberFormatException for invalid integer parts. |
| resolveXZ(int base) | int | O(1) | Resolves the coordinate for a horizontal axis (X or Z). The base value is used for relative calculations. |
| resolveYAtWorldCoords(int base, ChunkStore store, int x, int z) | int | O(log N) | Resolves the Y-axis coordinate. If the `height` flag is set, this method queries the ChunkStore for the highest block at the given X/Z position, which can be a costly operation. |
| isRelative() | boolean | O(1) | Returns true if the coordinate was prefixed with `~`. |
| isChunk() | boolean | O(1) | Returns true if the coordinate was prefixed with `c`. |
| isHeight() | boolean | O(1) | Returns true if the coordinate was prefixed with `_`. |

## Integration Patterns

### Standard Usage
The class is intended to be used by higher-level argument parsers to resolve vector components. The parser calls `parse` for each string component and then uses the resulting IntCoord objects to calculate a final world position relative to a command's execution context.

```java
// Example: Resolving a teleport destination
// Assume input strings "c1", "_", "~-5"
IntCoord xCoord = IntCoord.parse("c1");
IntCoord yCoord = IntCoord.parse("_");
IntCoord zCoord = IntCoord.parse("~-5");

// Command source provides the base coordinates
int baseX = player.getPosition().getX();
int baseY = player.getPosition().getY();
int baseZ = player.getPosition().getZ();

// ChunkStore is retrieved from the world context
ChunkStore chunkStore = world.getChunkStore();

int finalX = xCoord.resolveXZ(baseX); // Resolves to 1 * 32 = 32
int finalY = yCoord.resolveYAtWorldCoords(baseY, chunkStore, finalX, baseZ); // Resolves to height at (32, baseZ)
int finalZ = zCoord.resolveXZ(baseZ); // Resolves to baseZ - 5

// Use finalX, finalY, finalZ for game logic
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new IntCoord(...)`. The static `parse` method is the sole entry point and public contract for creating instances. Bypassing it will lead to behavior that is inconsistent with the server's command syntax.
- **State Caching:** Do not cache or reuse IntCoord instances. They are lightweight and represent a specific, contextual parsing result. Caching them is unnecessary and can lead to subtle bugs if the execution context (e.g., base coordinates) changes.
- **Excessive Height Lookups:** Avoid calling `resolveYAtWorldCoords` in performance-critical loops. The underlying query to the ChunkStore involves lookups that are significantly more expensive than simple arithmetic.

## Data Pipeline
The flow of data through this component is linear and part of the larger command processing pipeline.

> Flow:
> Raw Command String -> Command Argument Tokenizer -> **IntCoord.parse()** -> **IntCoord Instance** -> `resolve...()` Method -> Final Integer Coordinate -> Game System (e.g., Teleportation, Block Placement)

