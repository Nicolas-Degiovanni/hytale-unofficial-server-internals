---
description: Architectural reference for DestroyableBlockState
---

# DestroyableBlockState

**Package:** com.hypixel.hytale.server.core.universe.world.meta.state
**Type:** Interface

## Definition
```java
// Signature
@Deprecated
public interface DestroyableBlockState {
```

## Architecture & Concepts
The DestroyableBlockState interface defines a behavioral contract for block state objects that require custom logic to be executed upon their destruction. It serves as a hook into the server's world modification pipeline, allowing specific block types to perform cleanup, trigger events, or spawn entities when they are removed from the world.

This interface is a component of the world's block entity and metadata system. The world engine, when processing a block removal operation, performs a type check to see if the target block's state object implements DestroyableBlockState. If it does, the engine invokes the onDestroy method before proceeding with the standard block removal and chunk update logic.

**WARNING:** This interface is marked as **Deprecated**. New development should not implement this interface. It is maintained for backward compatibility with older world data or legacy block types. Future engine versions may remove support entirely. Consult the primary block management system for the modern equivalent, which likely uses an event-driven or component-based approach.

## Lifecycle & Ownership
As an interface, DestroyableBlockState has no lifecycle of its own. The lifecycle and ownership semantics apply to the concrete classes that implement it.

- **Creation:** An object implementing this interface is created by the world's BlockStateFactory or deserialized from chunk data when a chunk is loaded. Its creation is tied to the placement of a specific block type in the world.
- **Scope:** The object's lifetime is coupled directly to the block it represents. It persists as long as the block exists in the world, either in a loaded chunk in memory or serialized to disk.
- **Destruction:** The implementing object is marked for garbage collection when the block is destroyed and the onDestroy callback has completed. The world engine is the sole owner and is responsible for releasing all references.

## Internal State & Concurrency
This interface is stateless. It defines a behavior, not data.

- **State:** Implementors of this interface are expected to be stateful, containing data relevant to the block they represent. The onDestroy method will typically read this state to perform its logic.
- **Thread Safety:** Implementations are **not** required to be thread-safe. The world engine guarantees that all modifications to a specific block state, including the invocation of onDestroy, occur on the main server thread for that world or dimension. Accessing an implementing object from other threads will result in undefined behavior and world corruption.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onDestroy() | void | O(N) | Callback invoked by the world engine immediately before the block is removed. Complexity depends on implementation. |

## Integration Patterns

### Standard Usage
**WARNING:** The following pattern is deprecated and should not be used for new block types.

Historically, a developer would implement this interface on a custom BlockState class to handle cleanup logic.

```java
// This is a deprecated pattern. Do NOT replicate.
public class LegacyChestState extends BlockState implements DestroyableBlockState {

    private Inventory contents;

    @Override
    public void onDestroy() {
        // Logic to drop the chest's contents into the world
        World world = this.getWorld();
        Location loc = this.getLocation();
        world.dropItems(loc, contents.getItems());
        System.out.println("LegacyChestState destroyed and items dropped.");
    }
}
```

### Anti-Patterns (Do NOT do this)
- **New Implementations:** Do not create new classes that implement DestroyableBlockState. Use the modern, preferred event bus or component system for block behaviors.
- **Direct Invocation:** Never call the onDestroy method directly. It is designed to be a callback invoked exclusively by the server's world management engine during its block destruction sequence. Calling it manually will bypass engine safety checks and lead to state desynchronization.

## Data Pipeline
The data flow for this component is triggered by a world modification event, typically initiated by a player or system.

> Flow:
> Player Action (e.g., breaking a block) -> Server World Logic -> World::setBlock(pos, AIR) -> Engine retrieves old BlockState -> **Engine checks: `instanceof DestroyableBlockState`** -> `onDestroy()` callback invoked -> BlockState reference is released -> Chunk data is updated

