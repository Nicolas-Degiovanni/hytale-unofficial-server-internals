---
description: Architectural reference for PlaceBlockSettings
---

# PlaceBlockSettings

**Package:** com.hypixel.hytale.server.core.universe.world
**Type:** Utility / Constants Class

## Definition
```java
// Signature
public class PlaceBlockSettings {
```

## Architecture & Concepts

PlaceBlockSettings is a stateless utility class that defines a set of bitmask flags used to control the behavior of block placement operations within the server world. It is not a system in itself, but rather a configuration contract that decouples the act of changing a block from the side effects of that change.

This class embodies the **Flags Enum** pattern, where multiple boolean options are encoded into a single integer. This allows methods like `World.setBlock` to accept a flexible and efficient set of instructions for how to handle the consequences of a block update, such as triggering physics, updating neighboring blocks, or recalculating visual connections.

By externalizing these settings, the engine allows different systems (e.g., world generation, player actions, administrative commands) to perform block modifications with varying levels of performance impact and gameplay consequence. For example, bulk world generation can use the `NONE` flag for maximum performance, while a player placing a block requires flags that trigger lighting and redstone updates.

## Lifecycle & Ownership

-   **Creation:** As a class containing only static final members, PlaceBlockSettings is never instantiated. Its constants are initialized by the JVM ClassLoader when the class is first referenced.
-   **Scope:** The class and its constants are available for the entire lifetime of the server application.
-   **Destruction:** The class is unloaded when the server shuts down and its ClassLoader is garbage collected.

## Internal State & Concurrency

-   **State:** This class is stateless and its constants are immutable.
-   **Thread Safety:** PlaceBlockSettings is inherently thread-safe. Its `public static final` fields are constants that can be safely read from any thread without synchronization. The responsibility for thread-safe application of these settings lies with the calling system (e.g., the World object).

## API Surface

The public contract consists entirely of static integer constants intended for use as bitwise flags.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NONE | int | O(1) | A setting of 0. Instructs the world to perform a "silent" or "raw" block placement with no subsequent side effects. |
| PERFORM_BLOCK_UPDATE | int | O(1) | Flag (2). When set, the engine will notify the placed block and its immediate neighbors of the change, allowing them to react. |
| UPDATE_CONNECTIONS | int | O(1) | Flag (8). When set, instructs the engine to re-evaluate and update the models of blocks that connect to their neighbors, such as fences, walls, and glass panes. |

## Integration Patterns

### Standard Usage

Flags should be combined using the bitwise OR operator and passed to world modification methods. This is the primary and intended use case.

```java
// Example: A standard block placement by a player
World world = server.getUniverse().getWorld();
Position targetPos = ...;
BlockState newBlock = ...;

int placementFlags = PlaceBlockSettings.PERFORM_BLOCK_UPDATE | PlaceBlockSettings.UPDATE_CONNECTIONS;

world.setBlock(targetPos, newBlock, placementFlags);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never create an instance with `new PlaceBlockSettings()`. It provides no value, wastes memory, and violates the design intent of a static utility class.
-   **Using Magic Numbers:** Avoid using the literal integer values (e.g., `world.setBlock(pos, block, 2)`). Always use the named constants for code clarity, maintainability, and to prevent errors if the flag values ever change.
-   **Arithmetic Addition:** Do not combine flags with the `+` operator. While it may work for the current values, the correct and idiomatic operator for combining bitmasks is the bitwise OR `|`.

## Data Pipeline

PlaceBlockSettings does not process data itself; it provides configuration that directs a data pipeline. The flags act as control signals within the world modification logic.

> Control Flow:
> Caller (Player Action, World Gen) -> Constructs `int` flags using **PlaceBlockSettings** -> `World.setBlock(pos, block, flags)` -> World Core checks flags -> Conditional Execution (Neighbor Updates, Connection Logic)

