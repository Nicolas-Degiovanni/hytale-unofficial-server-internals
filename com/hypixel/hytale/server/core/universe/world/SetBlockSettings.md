---
description: Architectural reference for SetBlockSettings
---

# SetBlockSettings

**Package:** com.hypixel.hytale.server.core.universe.world
**Type:** Utility

## Definition
```java
// Signature
public class SetBlockSettings {
```

## Architecture & Concepts
SetBlockSettings is a static utility class that defines a bitmask-based contract for all world modification operations, specifically the placement or alteration of blocks. It is not a system that performs work, but rather a schema that provides a rich, configurable set of options to the core world engine.

The primary architectural purpose of this class is to decouple the *intent* of a block modification from the *mechanics* of its execution. By combining flags, a calling system (such as player interaction logic, world generation, or a scripting API) can precisely control the side effects of a `setBlock` operation. This avoids the "boolean parameter explosion" anti-pattern, where a method signature would become unwieldy with numerous true/false arguments (e.g., `setBlock(..., boolean notify, boolean updatePhysics, boolean sendParticles)`).

Instead, it employs a bitwise flag pattern, which is highly performant and common in low-level engine development. Each constant represents a single, orthogonal instruction to the world system.

### Lifecycle & Ownership
- **Creation:** This class is never instantiated. As a container for `public static final` constants, its fields are initialized by the Java ClassLoader when the class is first referenced by the server runtime.
- **Scope:** Application-wide. The settings are globally available and immutable for the entire lifetime of the server process.
- **Destruction:** The class and its static data are unloaded by the JVM upon server shutdown.

## Internal State & Concurrency
- **State:** SetBlockSettings is stateless and deeply immutable. Its fields are compile-time constants and cannot be altered at runtime.
- **Thread Safety:** This class is inherently thread-safe. Its constants can be safely read from any thread without requiring synchronization primitives. The responsibility for thread-safe application of these settings lies with the world modification systems that consume them.

## API Surface
The public contract consists entirely of static integer flags.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| NONE | int (flag) | O(1) | The default behavior. All side effects (notifications, physics, etc.) are enabled. |
| NO_NOTIFY | int (flag) | O(1) | Prevents neighbor blocks from receiving a block update notification. **Warning:** Can cause visual or logical inconsistencies if used improperly. |
| NO_UPDATE_STATE | int (flag) | O(1) | Prevents the block's state from being re-evaluated after placement. Useful for bulk operations where state is pre-calculated. |
| NO_SEND_PARTICLES | int (flag) | O(1) | Suppresses the creation of block-related particles (e.g., breaking or placement effects). |
| NO_SET_FILLER | int (flag) | O(1) | Instructs the engine to not place a secondary "filler" block, such as dirt under grass. |
| NO_BREAK_FILLER | int (flag) | O(1) | Prevents the destruction of an associated filler block when the primary block is removed. |
| PHYSICS | int (flag) | O(1) | Forces a physics update on the block, causing it to fall or react to gravity if applicable. |
| FORCE_CHANGED | int (flag) | O(1) | Forces the world to treat the block as changed, even if the new block type is identical to the old one. |
| NO_UPDATE_NEIGHBOR_CONNECTIONS | int (flag) | O(1) | Skips the recalculation of connections for adjacent blocks (e.g., fences, walls). |
| PERFORM_BLOCK_UPDATE | int (flag) | O(1) | Explicitly triggers a block update notification on the target block itself after placement. |
| NO_UPDATE_HEIGHTMAP | int (flag) | O(1) | Skips updating the chunk's heightmap. **Critical Warning:** Only use for subterranean or temporary changes to avoid severe lighting or AI pathing bugs. |
| NO_SEND_AUDIO | int (flag) | O(1) | Suppresses the playback of sounds associated with the block modification. |
| NO_DROP_ITEMS | int (flag) | O(1) | Prevents the block from dropping its corresponding item when broken or replaced. |

## Integration Patterns

### Standard Usage
Flags are combined using the bitwise OR operator (`|`) to create a composite settings integer. This integer is then passed to a world modification function.

```java
// Example: Place a stone block silently without notifying neighbors or dropping items.
// This is a typical pattern for large-scale, programmatic world edits.

import com.hypixel.hytale.server.core.universe.world.SetBlockSettings;

World world = server.getUniverse().getWorld();
Position targetPosition = new Position(100, 64, 250);
BlockState stoneBlock = BlockState.STONE;

int settings = SetBlockSettings.NO_NOTIFY | SetBlockSettings.NO_SEND_AUDIO | SetBlockSettings.NO_DROP_ITEMS;

world.setBlock(targetPosition, stoneBlock, settings);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The class has no public constructor and provides no value when instantiated. `new SetBlockSettings()` will fail and serves no purpose.
- **Arithmetic Combination:** Do not use arithmetic addition (`+`) to combine flags. While it may work for some combinations, it is not semantically correct and will fail if flags are ever changed. Always use the bitwise OR operator (`|`).
- **Ignoring Defaults:** Passing a raw `0` is acceptable, but for clarity, `SetBlockSettings.NONE` should always be used to signify default behavior.

## Data Pipeline
SetBlockSettings does not process data itself. It provides configuration metadata that directs the flow of a subsequent world update operation.

> Flow:
> High-Level Game Logic -> Creates `settings` integer using **SetBlockSettings** flags -> `World.setBlock(pos, state, settings)` -> World Engine parses `settings` -> Conditional Execution of Sub-systems (Physics, Networking, Audio, etc.) -> Final Block State in Chunk Data

