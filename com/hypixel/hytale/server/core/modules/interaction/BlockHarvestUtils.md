---
description: Architectural reference for BlockHarvestUtils
---

# BlockHarvestUtils

**Package:** com.hypixel.hytale.server.core.modules.interaction
**Type:** Utility

## Definition
```java
// Signature
public class BlockHarvestUtils {
```

## Architecture & Concepts
BlockHarvestUtils is a static utility class that serves as the central processing hub for all server-side logic related to damaging, breaking, and harvesting blocks. It is a stateless orchestrator that connects multiple core engine systems to execute the complex lifecycle of a block interaction.

This class does not manage its own state. Instead, it operates on world data provided through its method arguments, primarily `ComponentAccessor` and `Ref` objects which grant safe, temporary access to the underlying Entity-Component-System (ECS) data stores.

Its primary responsibilities include:
- Calculating the damage dealt to a block based on the tool used, the block's material properties, and global gameplay configuration.
- Applying durability loss to items used for harvesting.
- Managing the `BlockHealth` component, which tracks the damage state of individual blocks.
- Orchestrating the block-breaking sequence, including state transitions for blocks that change form when harvested (e.g., farmland).
- Determining and spawning item drops based on loot tables and block configuration.
- Triggering associated visual and audio effects, such as particles and sounds, through other utility classes like `ParticleUtil` and `SoundUtil`.
- Firing critical game events like `DamageBlockEvent` and `BreakBlockEvent`, allowing other systems to intercept and modify interaction outcomes.

It acts as a critical bridge between player input handlers, physics systems, and the world state representation within chunks.

### Lifecycle & Ownership
- **Creation:** As a static utility class, BlockHarvestUtils is never instantiated. Its methods are loaded by the JVM and become available for the entire application lifetime.
- **Scope:** Application-wide. Its static methods can be called from any part of the server codebase that has access to the class.
- **Destruction:** The class is unloaded only when the JVM shuts down. There is no concept of instance-based cleanup.

## Internal State & Concurrency
- **State:** BlockHarvestUtils is entirely stateless. It contains no member fields and all operations are performed on data passed in as method parameters. Its behavior is deterministic based on its inputs.
- **Thread Safety:** The class itself is inherently thread-safe due to its stateless nature. However, it is **not safe** to call its methods from arbitrary threads. The methods perform direct, unsynchronized mutations on world state components (e.g., `BlockChunk`, `BlockHealthChunk`).

**WARNING:** All methods within BlockHarvestUtils must be invoked exclusively from the main server thread, which processes the game tick. Calling these methods from a network thread, an asynchronous task, or any other context will lead to severe data corruption, race conditions, and server instability. The caller is solely responsible for ensuring thread context.

## API Surface
The public API is designed around two primary actions: damaging a block and breaking a block.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| performBlockDamage(...) | boolean | High | The primary entry point for block interaction. Calculates and applies damage, handles durability, and may trigger a block break. Returns true if the block was broken. |
| performBlockBreak(...) | void | High | Forces a block to break. Handles multi-block structures, fires events, and orchestrates item drops and world updates. |
| naturallyRemoveBlock(...) | void | High | A lower-level function for breaking a block and spawning its drops. Used by physics and other world systems. |
| getDrops(...) | List<ItemStack> | O(N) | Calculates the list of item drops for a given block type and quantity, processing loot tables via the ItemModule. N is the quantity. |

## Integration Patterns

### Standard Usage
The most common pattern involves an interaction system (e.g., handling player input) acquiring the necessary world context and invoking `performBlockDamage`.

```java
// In a system that processes player actions
// Assume 'playerRef', 'targetBlockPos', and 'world' are available

// Get accessors to the world state
ComponentAccessor<EntityStore> entityStore = world.getEntityStore().getStore();
ComponentAccessor<ChunkStore> chunkStore = world.getChunkStore().getStore();

// Find the chunk containing the block
long chunkIndex = ChunkUtil.indexChunkFromBlock(targetBlockPos.x, targetBlockPos.z);
Ref<ChunkStore> chunkRef = chunkStore.getExternalData().getChunkReference(chunkIndex);

if (chunkRef != null && chunkRef.isValid()) {
    ItemStack heldItem = player.getInventory().getHeldItemStack();
    
    // Execute the block damage logic
    BlockHarvestUtils.performBlockDamage(
        player,
        playerRef,
        targetBlockPos,
        heldItem,
        null, // tool
        null, // toolId
        false, // matchTool
        1.0f, // damageScale
        0, // setBlockSettings
        chunkRef,
        entityStore,
        chunkStore
    );
}
```

### Anti-Patterns (Do NOT do this)
- **Asynchronous Execution:** Never call any method from this class outside the main server tick. This will bypass all concurrency controls and corrupt chunk data.
- **Ignoring Return Values:** The `performBlockDamage` method returns a boolean indicating if the block was fully broken. Logic that depends on the block's final state must check this value.
- **Invalid References:** Passing a null or invalid `Ref<ChunkStore>` will result in `NullPointerException` or other runtime exceptions. Always validate chunk references before calling.
- **Incorrect `setBlockSettings`:** The `setBlockSettings` parameter is a powerful bitmask that controls side effects like sound, particles, and drops. Using incorrect flags can lead to silent failures or visually inconsistent behavior (e.g., a block breaking with no sound or drops).

## Data Pipeline
The data flow for a typical player-initiated block damage event is complex, involving multiple modules and data transformations.

> Flow:
> Player Input Packet -> Network Handler -> Player Interaction System -> **BlockHarvestUtils.performBlockDamage**
> 
> **Inside performBlockDamage:**
> 1.  Reads `BlockType` and `Item` asset data.
> 2.  Calculates damage and durability cost.
> 3.  Dispatches `DamageBlockEvent` to the Event Bus.
> 4.  Mutates `BlockHealthChunk` component state.
> 5.  **If block breaks:** Calls `performBlockBreak`.
> 6.  Dispatches `BreakBlockEvent`.
> 7.  Mutates `WorldChunk` and `BlockChunk` to remove the block.
> 8.  Calls `ItemModule` to resolve loot tables (`getDrops`).
> 9.  Calls `EntityModule` to spawn `Item` entities into the world.
> 10. Calls `SoundUtil` and `ParticleUtil` to generate client-side effects.
> 
> **Output:**
> Modified Chunk Components -> World State Change -> Network Packets (Block Update, Entity Spawn, Sound Event) -> Client Render

